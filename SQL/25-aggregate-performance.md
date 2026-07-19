# Topic 25 — Aggregate Performance
### SQL Mastery Curriculum — Phase 4: Aggregation and Grouping

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a warehouse and someone dumps 10 million receipts on the floor and asks: "Tell me the total sales for every product."

You have two fundamentally different ways to do this.

**Strategy A — the pigeonhole wall (HashAggregate).** You put up a wall of labelled boxes, one per product. You walk through the receipts one by one. For each receipt you glance at the product, find its box on the wall, and drop the running total in. You never sort anything. You just need enough wall space to have one box per distinct product open at once. When the last receipt is processed, every box already holds its final total. Fast — one pass — but only works if the wall of boxes fits in the room. If there are 50 million distinct products, you run out of wall space.

**Strategy B — sort then sweep (GroupAggregate).** You first sort all 10 million receipts into one giant pile ordered by product. Now all the "Widget" receipts are physically adjacent, then all the "Gadget" receipts, and so on. You sweep through the sorted pile: as long as the product stays "Widget" you keep adding; the moment it changes to "Gadget" you write out the Widget total and start fresh. You only ever hold **one** running total at a time — memory is tiny. But sorting 10 million receipts first is expensive.

That is the entire drama of aggregate performance. HashAggregate is a single pass with a memory cost proportional to the number of **distinct groups**. GroupAggregate needs sorted input but uses almost no memory. The planner picks between them based on how many groups it expects and whether the data is already sorted (say, arriving straight off an index).

And the fun twist: what happens when the pigeonhole wall doesn't fit in the room? Before PostgreSQL 13, the planner had to guess in advance and would refuse to use HashAggregate if it thought it might overflow. From PG13 onward, HashAggregate can **spill the overflowing boxes to disk** and finish the job anyway — this is "hashagg spill," and it changed how you reason about `work_mem` for aggregation forever.

---

## 2. Connection to SQL Internals

Aggregation is where the executor stops being a pipe that streams rows and becomes a machine that **materialises state**. There are exactly two physical strategies for `GROUP BY` in PostgreSQL, and both are executor nodes with distinct internal data structures.

**HashAggregate** builds an in-memory **hash table** keyed by the grouping columns. Each hash table entry holds the group key plus the *transition state* of every aggregate (the running `sum`, the `count`, the accumulator for `avg`, etc.). This is the same hash table machinery used by Hash Join (Topic 17), but here it accumulates aggregate state rather than storing rows to probe. The table lives in `work_mem`. When it overflows in PG13+, overflowing groups' input rows are written to temporary spill files on disk and processed in later passes.

**GroupAggregate** (shown as `GroupAggregate` in plans, sometimes preceded by a `Sort` node) consumes input that is **already sorted on the grouping keys**. It keeps only the current group's transition state in memory — O(1) memory regardless of group count. The sort that feeds it either comes from an explicit `Sort` node (spending CPU and possibly disk via external merge sort) or, ideally, from an **Index Scan** whose B-tree order already matches the `GROUP BY` columns, giving aggregation for free.

Underneath both sit the **aggregate transition functions**. `SUM(x)` is not magic — it is a state value plus a transition function `sfunc` that folds each input row into the state, and a final function `ffunc` that produces the output. `avg` carries a two-element state `(sum, count)`. `count(*)` carries an int8 counter. Understanding this is why `SUM` over a billion rows uses constant memory per group but `array_agg` or `string_agg` can blow up — their state grows with the number of rows in the group.

Other internals in play: the **buffer pool** (aggregation reads heap or index pages through shared buffers), **parallel query** (multiple workers each build partial aggregates, then a `Finalize Aggregate` node combines them — this requires *combine functions* for each aggregate), **MVCC visibility** (aggregation still respects snapshot visibility; it never counts dead tuples unless you use approximations), and the **statistics** in `pg_statistic` — specifically `n_distinct`, which the planner uses to estimate group count and therefore to choose Hash vs Group in the first place.

---

## 3. Logical Execution Order Context

```
FROM / JOIN          ← rows are produced and joined first
WHERE                ← row-level filter, BEFORE grouping (cannot see aggregates)
GROUP BY             ← THIS TOPIC — rows collapse into groups here
HAVING               ← group-level filter, AFTER aggregation (can see aggregates)
SELECT               ← aggregate expressions computed / projected
DISTINCT
window functions     ← run after GROUP BY, over the grouped rows
ORDER BY
LIMIT
```

Aggregation sits at the pivot point of the whole pipeline. Everything before it operates on individual rows; everything after it operates on already-collapsed groups. Three consequences dominate performance:

1. **`WHERE` runs before `GROUP BY`.** Every predicate you can express as a row filter shrinks the input to aggregation. Pushing selectivity into `WHERE` (not `HAVING`) is the single biggest aggregation optimisation, because it reduces the number of rows the hash table or sort must process. `HAVING total > 100` cannot be evaluated until after grouping; `WHERE status = 'completed'` can.

2. **`GROUP BY` determines whether an ORDER BY later is free.** If the executor chose GroupAggregate, the output already arrives sorted on the grouping keys — a subsequent `ORDER BY` on those same keys needs no extra sort. If it chose HashAggregate, output order is undefined and an `ORDER BY` forces a separate sort. This is why the *same* query can be faster with a plan that looks slower in isolation.

3. **`LIMIT` usually does NOT save aggregation work.** A `LIMIT 10` on a `GROUP BY` query still has to compute *every* group before it knows which 10 to return (unless an ordered index lets it stop early). Aggregation is a blocking operator: HashAggregate emits nothing until the last input row is consumed. This surprises people expecting `LIMIT` to be cheap.

Remember from Topic 20 (GROUP BY Fundamentals) that `GROUP BY` collapses the fan-out produced by joins in Topic 11. This topic is about *how fast* that collapse happens.

---

## 4. What Is Aggregate Performance?

Aggregate performance is the study of how PostgreSQL physically executes `GROUP BY` and aggregate functions (`SUM`, `COUNT`, `AVG`, `MIN`, `MAX`, `array_agg`, …), how it chooses between the HashAggregate and GroupAggregate strategies, how memory (`work_mem`) governs whether that execution stays in RAM or spills to disk, and how indexes, parallelism, and query restructuring change the cost.

```sql
SELECT
    customer_id,                          -- │ grouping key: defines the buckets
    COUNT(*)            AS order_count,   -- │ aggregate: rows per group
    SUM(total_amount)   AS revenue,       -- │ aggregate: constant-state accumulator
    AVG(total_amount)   AS avg_order,     -- │ aggregate: carries (sum,count) state
    MAX(created_at)     AS last_order     -- │ aggregate: constant-state, index-friendly
FROM orders                               -- │ scanned source (Seq Scan or Index Scan)
WHERE status = 'completed'                -- │ pre-aggregation row filter (runs FIRST)
GROUP BY customer_id                      -- │ HashAggregate OR Sort+GroupAggregate
HAVING COUNT(*) > 5                       -- │ post-aggregation group filter
ORDER BY revenue DESC                     -- │ sort of the grouped output (may need Sort)
LIMIT 100;                                -- │ applied last; does NOT reduce grouping work
```

The keywords whose physical cost you must reason about:

```
GROUP BY <cols>
   │
   ├── HashAggregate     → build hash table keyed by <cols>; O(distinct groups) memory
   │                       one pass, no sort; spills to disk in PG13+ if > work_mem
   │
   └── GroupAggregate    → requires input sorted on <cols>
        │                  O(1) memory (one group's state at a time)
        ├── fed by Sort   → pays N log N sort cost (may spill: "external merge Disk")
        └── fed by Index  → sort is free; B-tree already ordered on <cols>
```

The planner's decision hinges on the estimated number of distinct groups (`n_distinct` from `pg_statistic`), the cost of sorting, whether a usefully-ordered index exists, and the size of `work_mem`.

---

## 5. Why Aggregate Performance Mastery Matters in Production

1. **Dashboards and reports are aggregation-bound.** Almost every analytics query, KPI tile, and admin report is a `GROUP BY`. When the "revenue by day" widget takes 40 seconds, the bottleneck is nearly always the aggregation strategy — a HashAggregate spilling to disk, or a needless sort of 50M rows. Knowing which one lets you fix it in minutes instead of guessing.

2. **`work_mem` misconfiguration silently destroys throughput.** If `work_mem` is too small, HashAggregate spills to disk (temp files, extra I/O, 5–10× slower) or the planner falls back to a sort that also spills. If it is too large and set globally, 200 concurrent connections each running a hash aggregate can request 200 × `work_mem` and drive the box into swap or the OOM killer. Getting this right is a production survival skill, not a micro-optimisation.

3. **The planner picks the wrong strategy when statistics are stale.** If `n_distinct` is wildly off (common on high-cardinality columns after a bulk load without `ANALYZE`), the planner may choose HashAggregate expecting 1,000 groups when there are 10 million, then spill catastrophically. Recognising this from `EXPLAIN ANALYZE` (estimated vs actual groups) is what separates a five-minute fix from a day of thrashing.

4. **Pre-aggregation is the difference between real-time and batch.** A query that aggregates 500M raw event rows on every dashboard load cannot be made fast by tuning alone. Knowing when to introduce a rollup table, a materialised view, or an incremental summary is the architectural decision that keeps a product responsive as data grows.

5. **Parallel aggregation is free performance you must not accidentally disable.** A single non-parallel-safe function, a low `max_parallel_workers_per_gather`, or a small table below the parallelism threshold can silently force single-threaded aggregation on a 100M-row scan. Understanding the `Partial Aggregate` / `Finalize Aggregate` plan shape lets you confirm you are actually using all your cores.

---

## 6. Deep Technical Content

### 6.1 HashAggregate — The One-Pass Strategy

HashAggregate is the default choice for `GROUP BY` on unsorted input with a manageable number of distinct groups. Mechanics:

1. Allocate a hash table in `work_mem`.
2. For each input row: compute the hash of the grouping key, locate (or create) the group's entry, and fold the row into each aggregate's transition state via its `sfunc`.
3. After the last input row, walk the hash table and emit one output row per group, calling each aggregate's final function `ffunc`.

Key properties:
- **Single pass** over the input. No sort required.
- **Memory ∝ number of distinct groups**, not number of input rows. 1 billion rows collapsing to 50 groups uses a tiny hash table. 1 billion rows collapsing to 500 million groups uses an enormous one.
- **Output order is undefined.** Do not rely on it; add `ORDER BY` if you need order.
- **Blocking operator.** Emits nothing until all input is consumed.

```sql
-- Classic HashAggregate candidate: few groups, many rows, no useful sort order
SELECT status, COUNT(*), SUM(total_amount)
FROM orders
GROUP BY status;
-- ~5 distinct statuses → tiny hash table → HashAggregate, one Seq Scan pass
```

### 6.2 GroupAggregate — The Sorted Strategy

GroupAggregate requires its input to be sorted on the grouping columns. It keeps only the current group's state, emitting a group the instant the key changes.

```sql
SELECT customer_id, COUNT(*), SUM(total_amount)
FROM orders
GROUP BY customer_id
ORDER BY customer_id;   -- if fed by an index on customer_id, sort is free
```

Two ways GroupAggregate gets its sorted input:

- **Explicit Sort node.** The planner inserts a `Sort` before `GroupAggregate`. This costs O(N log N) CPU and, if the data exceeds `work_mem`, spills to disk as an *external merge sort* (visible as `Sort Method: external merge Disk: NNNNkB`).
- **Ordered Index Scan.** If a B-tree index exists on the grouping columns, the executor reads rows in key order and feeds them straight to GroupAggregate — no sort node at all. This is the ideal: aggregation for the cost of an index scan.

Why the planner would ever prefer GroupAggregate over the one-pass HashAggregate:
- **Huge number of distinct groups** that would not fit in `work_mem` as a hash table (pre-PG13 this was mandatory; post-PG13 the planner still weighs spill cost).
- **The input is already sorted** (index available, or a prior operator produced sorted output) — then the sort is free and GroupAggregate wins outright.
- **A subsequent `ORDER BY` on the grouping keys** can reuse GroupAggregate's ordered output, avoiding a second sort.

### 6.3 How the Planner Chooses

The decision is cost-based and driven primarily by the estimated number of groups:

- The planner estimates distinct groups from `n_distinct` in `pg_statistic` (per column) or extended statistics (`CREATE STATISTICS`) for multi-column groupings.
- It costs a HashAggregate as roughly: input scan + hash table build, feasible only if `estimated_groups × avg_entry_size` fits in `work_mem` (or, in PG13+, plus the cost of spilling the excess).
- It costs a GroupAggregate as: input scan + sort (unless a sorted path exists) + near-zero aggregation memory.
- Whichever total cost is lower wins.

You can nudge or diagnose this:
```sql
SET enable_hashagg = off;   -- force GroupAggregate to compare plans (diagnostic only)
SET enable_sort = off;      -- discourage sort-based paths
-- Always reset afterwards; never leave these off in production.
```

Multi-column `GROUP BY` is where estimates go wrong most often, because per-column `n_distinct` values are multiplied assuming independence. Correlated columns (e.g. `city, country`) violate that badly. Fix with extended statistics:
```sql
CREATE STATISTICS orders_cust_status (ndistinct)
  ON customer_id, status FROM orders;
ANALYZE orders;
```

### 6.4 work_mem and Spilling to Disk

`work_mem` is the per-operation, per-node memory budget. A single query can use several multiples of it (one per sort, hash, or hash-aggregate node, and multiplied again by parallel workers). It is NOT a global cap.

**Sort spill (all versions):** When a `Sort` feeding GroupAggregate exceeds `work_mem`, PostgreSQL switches from an in-memory quicksort to an on-disk external merge sort. `EXPLAIN ANALYZE` shows:
```
Sort Method: external merge  Disk: 68912kB
```
versus the in-memory case:
```
Sort Method: quicksort  Memory: 24576kB
```

**HashAggregate spill (PG13+):** Before PostgreSQL 13, HashAggregate could **not** spill. If the planner mis-estimated and the hash table exceeded `work_mem` at runtime, it would keep allocating past `work_mem` and could exhaust memory — the planner therefore avoided HashAggregate whenever it feared overflow, often choosing a slower sort-based plan defensively. PostgreSQL 13 introduced **hash aggregation spill to disk**: when the hash table exceeds `work_mem`, overflowing groups' input is written to temp files and processed in additional passes. `EXPLAIN ANALYZE` shows:
```
HashAggregate ...
  Peak Memory Usage: 205000kB
  Disk Usage: 131072kB
  Batches: 5
```
`Batches > 1` and non-zero `Disk Usage` mean the hash aggregate spilled — the equivalent of an external merge sort for hashing. It works, but it is much slower than staying in RAM.

**The PG13 behavioural change to know for interviews:** because HashAggregate can now spill, the planner is far more willing to choose it even when it might exceed `work_mem`, since overflow is no longer catastrophic. A side effect is that some queries that used to (accidentally) use more than `work_mem` of RAM now spill to disk and run *slower but safer*. If you upgraded to PG13+ and a report got slower, a spilling HashAggregate is a prime suspect — the fix is raising `work_mem` for that workload. You can also set `hash_mem_multiplier` (PG13+) to give hash-based nodes (hash join and hash aggregate) more memory than plain sorts:
```sql
SET hash_mem_multiplier = 2.0;   -- hash nodes may use 2 × work_mem before spilling
```

**Setting work_mem safely:**
```sql
-- Do NOT crank the global default; it multiplies across connections and nodes.
-- Instead, raise it per-session or per-transaction for the heavy report:
SET LOCAL work_mem = '256MB';   -- only for this transaction
SELECT ... big GROUP BY ...;
```

### 6.5 Parallel Aggregation

For large scans, PostgreSQL can aggregate in parallel. The plan splits into two stages:

- **Partial Aggregate** — each parallel worker aggregates its slice of the table into a *partial* result (partial sums, partial counts).
- **Finalize Aggregate** — a single process combines the workers' partial results into the final answer using each aggregate's *combine function* (`combinefunc`).

```
Finalize GroupAggregate
  Group Key: customer_id
  ->  Gather Merge
        Workers Planned: 4
        ->  Partial GroupAggregate
              Group Key: customer_id
              ->  Sort
                    ->  Parallel Seq Scan on orders
```
or, for hashed:
```
Finalize HashAggregate
  ->  Gather
        Workers Planned: 4
        ->  Partial HashAggregate
              ->  Parallel Seq Scan on orders
```

Requirements and controls:
- The table (or its scan cost) must exceed thresholds: `min_parallel_table_scan_size` (default 8MB) and the planner must estimate a net benefit.
- `max_parallel_workers_per_gather` (default 2) caps workers per Gather; raise it for big analytics: `SET max_parallel_workers_per_gather = 4;`
- `max_parallel_workers` and `max_worker_processes` cap the cluster-wide pool.
- The aggregate must be **parallel-safe** and have a combine function. Standard aggregates (`sum`, `count`, `avg`, `min`, `max`, `bool_and/or`, `bit_and/or`) qualify. A custom aggregate without a `combinefunc`, or any function marked `PARALLEL UNSAFE` in the query, disables parallelism for the whole plan.
- `array_agg`/`string_agg` with an `ORDER BY` inside them, and `DISTINCT` inside an aggregate, restrict parallelism.

Each parallel worker gets its own `work_mem`, so a 4-worker parallel hash aggregate can use up to ~4 × `work_mem` (plus the leader). Budget accordingly.

### 6.6 Indexes That Help GROUP BY

An index helps aggregation in three distinct ways:

**(a) Providing sorted input for GroupAggregate.** A B-tree on the grouping columns lets the executor skip the sort:
```sql
CREATE INDEX ON orders (customer_id);
-- SELECT customer_id, SUM(total_amount) FROM orders GROUP BY customer_id;
-- can now be Index Scan → GroupAggregate, no Sort node
```
The column order in the index must be a prefix of (or match) the `GROUP BY` columns.

**(b) Index-Only Scans (covering indexes).** If the index contains every column the query touches, PostgreSQL reads only the index, never the heap:
```sql
CREATE INDEX ON orders (customer_id, total_amount);
-- SELECT customer_id, SUM(total_amount) FROM orders GROUP BY customer_id;
-- → Index Only Scan (no heap fetches, given the visibility map is current)
```
Keep the table well-vacuumed so the visibility map lets index-only scans avoid heap visits.

**(c) The MIN/MAX special case.** `MIN(col)` and `MAX(col)` on an indexed column do not scan at all — they become an index endpoint lookup:
```sql
CREATE INDEX ON orders (created_at);
SELECT MAX(created_at) FROM orders;
-- Plan: "Result" with an "InitPlan" doing a
-- Limit → Index Scan Backward → reads exactly ONE index entry. O(log N).
```
This only works for a *single* `MIN`/`MAX` per index and without a `GROUP BY` (or with a matching `GROUP BY` prefix in some cases). `SELECT MIN(x), MAX(x)` uses two such lookups.

**(d) Partial and expression indexes for filtered aggregation.** If you always aggregate `WHERE status = 'completed'`, a partial index shrinks the scan:
```sql
CREATE INDEX ON orders (customer_id, total_amount)
  WHERE status = 'completed';
```

### 6.7 Pre-Aggregation Strategies

When raw-row aggregation is too slow no matter the tuning, aggregate ahead of time.

**Materialised views** — precompute and store the result; refresh on a schedule:
```sql
CREATE MATERIALIZED VIEW daily_revenue AS
SELECT DATE_TRUNC('day', created_at) AS day,
       COUNT(*) AS orders, SUM(total_amount) AS revenue
FROM orders WHERE status = 'completed'
GROUP BY 1;

CREATE UNIQUE INDEX ON daily_revenue (day);   -- enables CONCURRENTLY refresh
REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue;
```
`CONCURRENTLY` avoids locking readers but requires a unique index and does more work.

**Rollup/summary tables with incremental maintenance** — maintain a running summary via triggers or a scheduled `INSERT ... ON CONFLICT DO UPDATE`:
```sql
INSERT INTO daily_revenue (day, orders, revenue)
SELECT DATE_TRUNC('day', created_at), COUNT(*), SUM(total_amount)
FROM orders
WHERE created_at >= CURRENT_DATE            -- only today's new rows
  AND status = 'completed'
GROUP BY 1
ON CONFLICT (day) DO UPDATE
SET orders  = EXCLUDED.orders,
    revenue = EXCLUDED.revenue;
```

**Two-phase aggregation (pre-aggregate before the join)** — collapse the fan-out early so the expensive join operates on far fewer rows. Recall from Topic 11 that joining to a one-to-many table multiplies rows; aggregating the many-side first avoids inflating the join:
```sql
-- Instead of joining then aggregating 20M order_items,
-- aggregate order_items per order first (fewer rows), then join.
WITH item_rollup AS (
  SELECT order_id, SUM(quantity * unit_price) AS order_total
  FROM order_items
  GROUP BY order_id
)
SELECT o.customer_id, SUM(ir.order_total) AS revenue
FROM item_rollup ir
JOIN orders o ON o.id = ir.order_id
GROUP BY o.customer_id;
```

### 6.8 FILTER, DISTINCT, and Aggregate State Cost

**`FILTER` clause** — conditional aggregation without extra passes; far cheaper than multiple correlated subqueries and cleaner than `SUM(CASE WHEN ...)`:
```sql
SELECT
  COUNT(*)                                         AS all_orders,
  COUNT(*) FILTER (WHERE status = 'completed')     AS completed,
  SUM(total_amount) FILTER (WHERE status = 'refunded') AS refunded_value
FROM orders;
-- One pass; each FILTER just gates whether the row folds into that aggregate's state.
```

**`COUNT(DISTINCT x)` is expensive.** DISTINCT inside an aggregate forces PostgreSQL to deduplicate, typically by sorting or hashing *per group*, and it **disables parallel aggregation** for that aggregate. On big data this is often the single slowest part of a report.
```sql
-- Slow at scale:
SELECT customer_id, COUNT(DISTINCT product_id) FROM order_items GROUP BY customer_id;

-- Faster pattern: pre-distinct, then count
SELECT customer_id, COUNT(*) FROM (
  SELECT DISTINCT customer_id, product_id FROM order_items
) d GROUP BY customer_id;

-- Or approximate with HLL / count-distinct extensions when exactness is optional.
```

**Growing-state aggregates.** `SUM`/`COUNT`/`AVG` carry constant-size state. `array_agg`, `string_agg`, `jsonb_agg` carry state that grows with the number of rows per group — a single huge group can exhaust `work_mem` on the aggregate state alone. Watch these on skewed data.

### 6.9 GROUP BY Skew and Cardinality

Aggregate cost is dominated by the number of distinct groups and their distribution:
- **Few groups, many rows** (e.g. `GROUP BY status`) → HashAggregate, tiny memory, very fast. Parallelism helps most here.
- **Many groups, roughly one row each** (e.g. `GROUP BY primary_key`) → the aggregation does almost no folding; cost is essentially the scan. A hash table nearly as large as the input — spill risk.
- **Skewed groups** (one group holds 90% of rows) → parallel workers imbalance; growing-state aggregates on the hot group can spill. Consider salting or separate handling of the hot key.

### 6.10 GROUPING SETS Interaction (bridge from Topic 24)

Recall from Topic 24 that `GROUPING SETS`/`ROLLUP`/`CUBE` compute multiple grouping levels in one pass. Physically, PostgreSQL implements them with a `MixedAggregate` (or GroupAggregate with multiple sort groups), and each grouping set has its own memory footprint. The same HashAggregate spill rules apply — a `CUBE` over high-cardinality columns can produce an enormous number of groups. `work_mem` and spilling matter *more* for grouping sets, not less, because you are materialising several groupings at once.

---

## 7. EXPLAIN — Aggregation in the Plan

### HashAggregate (few groups, in memory)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT status, COUNT(*), SUM(total_amount)
FROM orders
GROUP BY status;
```

```
HashAggregate  (cost=20834.00..20834.05 rows=5 width=48)
               (actual time=310.2..310.2 rows=5 loops=1)
  Group Key: status
  Batches: 1  Memory Usage: 24kB
  ->  Seq Scan on orders  (cost=0.00..15834.00 rows=1000000 width=16)
                          (actual time=0.01..120.4 rows=1000000 loops=1)
  Buffers: shared hit=5834
Planning Time: 0.2 ms
Execution Time: 310.4 ms
```

**Reading it:**
- `HashAggregate` with `Group Key: status` — hash table keyed by status.
- `Batches: 1` — the hash table fit entirely in `work_mem`; **no disk spill**. This is the healthy signal.
- `Memory Usage: 24kB` — only 5 groups, trivial memory.
- Cost is dominated by the `Seq Scan` reading all 1M rows; aggregation itself is cheap.

### HashAggregate spilling to disk (PG13+)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, SUM(total_amount)
FROM orders
GROUP BY customer_id;   -- customer_id has ~2M distinct values, work_mem = 4MB
```

```
HashAggregate  (cost=32218.00..44718.00 rows=2000000 width=40)
               (actual time=980.5..2140.7 rows=2000000 loops=1)
  Group Key: customer_id
  Batches: 9  Memory Usage: 4096kB  Disk Usage: 98304kB
  ->  Seq Scan on orders  (cost=0.00..15834.00 rows=1000000 width=16)
                          (actual time=0.02..150.1 rows=1000000 loops=1)
  Buffers: shared hit=5834 temp read=12000 written=12000
Planning Time: 0.3 ms
Execution Time: 2210.3 ms
```

**Reading it:**
- `Batches: 9` and `Disk Usage: 98304kB` — the hash aggregate **spilled to disk** across 9 passes. This is the PG13+ spill behaviour.
- `Memory Usage: 4096kB` equals `work_mem` (4MB) — it filled the budget then overflowed.
- `temp read/written=12000` in Buffers confirms temp-file I/O.
- **Fix:** raise `work_mem` for this query (`SET LOCAL work_mem = '256MB'`) so `Batches` drops to 1, or pre-aggregate.

### GroupAggregate fed by an Index Scan (sort is free)

```sql
EXPLAIN (ANALYZE)
SELECT customer_id, SUM(total_amount)
FROM orders
GROUP BY customer_id;   -- with an index on (customer_id, total_amount)
```

```
GroupAggregate  (cost=0.43..61000.00 rows=2000000 width=40)
                (actual time=0.05..1450.2 rows=2000000 loops=1)
  Group Key: customer_id
  ->  Index Only Scan using orders_cust_amt_idx on orders
        (cost=0.43..41000.00 rows=1000000 width=16)
        (actual time=0.03..600.1 rows=1000000 loops=1)
        Heap Fetches: 0
Planning Time: 0.2 ms
Execution Time: 1520.6 ms
```

**Reading it:**
- `GroupAggregate` with **no Sort node** — the `Index Only Scan` already delivers rows in `customer_id` order.
- `Heap Fetches: 0` — a true index-only scan; the heap was never touched (visibility map current).
- Constant aggregation memory (no Memory Usage / Batches line) — GroupAggregate holds one group at a time.
- This beats the spilling HashAggregate above for high-cardinality grouping: no temp files.

### GroupAggregate fed by an explicit external-merge Sort

```sql
EXPLAIN (ANALYZE)
SELECT customer_id, COUNT(*)
FROM orders
GROUP BY customer_id;   -- no useful index, work_mem = 4MB, 2M groups
```

```
GroupAggregate  (cost=140000.00..160000.00 rows=2000000 width=16)
                (actual time=2100.0..3300.5 rows=2000000 loops=1)
  Group Key: customer_id
  ->  Sort  (cost=140000.00..142500.00 rows=1000000 width=8)
            (actual time=2100.0..2600.3 rows=1000000 loops=1)
        Sort Key: customer_id
        Sort Method: external merge  Disk: 17624kB
        ->  Seq Scan on orders (actual rows=1000000 loops=1)
```

**Reading it:**
- The `Sort` feeding GroupAggregate spilled: `Sort Method: external merge  Disk: 17624kB`.
- Both strategies (this and the spilling HashAggregate) touch disk here; the planner picks whichever it costs lower. Raising `work_mem` fixes either.

### Parallel Aggregation

```sql
EXPLAIN (ANALYZE)
SELECT customer_id, SUM(total_amount)
FROM orders
GROUP BY customer_id;   -- 50M rows, max_parallel_workers_per_gather = 4
```

```
Finalize HashAggregate  (cost=... rows=2000000 width=40)
                        (actual time=4200.1..5100.6 rows=2000000 loops=1)
  Group Key: customer_id
  Batches: 1  Memory Usage: ...
  ->  Gather  (cost=... rows=8000000 width=40)
              (actual time=1200.2..3800.4 rows=8000000 loops=1)
        Workers Planned: 4  Workers Launched: 4
        ->  Partial HashAggregate  (actual time=1100.0..1400.0 rows=1600000 loops=5)
              Group Key: customer_id
              ->  Parallel Seq Scan on orders (actual rows=10000000 loops=5)
```

**Reading it:**
- `Partial HashAggregate` runs in each of 4 workers + leader (`loops=5`), each over its slice (`Parallel Seq Scan`).
- `Gather` collects the partials (note the partials total 8M rows — each worker emits its own copy of each group).
- `Finalize HashAggregate` combines them into the final 2M groups using `sum`'s combine function.
- If you see a plain `HashAggregate` with no Partial/Finalize on a huge table, parallelism was disabled — check `max_parallel_workers_per_gather` and for parallel-unsafe functions.

### What to Look For in EXPLAIN

| Symptom | Likely Problem | Fix |
|---|---|---|
| `HashAggregate ... Batches: N (N>1)  Disk Usage: …` | Hash table spilled to disk (PG13+) | Raise `work_mem` / `hash_mem_multiplier`; pre-aggregate |
| `Sort Method: external merge  Disk: …` before GroupAggregate | Sort spilled to disk | Raise `work_mem`; add index for sorted input |
| Estimated `rows` on the aggregate ≫ or ≪ actual | Bad `n_distinct` estimate | `ANALYZE`; add `CREATE STATISTICS` for multi-col |
| Plain (non-partial) aggregate on a huge table | Parallelism disabled | Raise `max_parallel_workers_per_gather`; remove parallel-unsafe funcs |
| `Sort` node before GroupAggregate that could be avoided | No index in `GROUP BY` order | Add B-tree on grouping columns |
| `Seq Scan` reading whole table for a filtered aggregate | No partial/covering index | Partial or covering index on filter + group cols |

---

## 8. Query Examples

### Example 1 — Basic: Low-Cardinality HashAggregate

```sql
-- Order counts and revenue by status.
-- ~5 distinct statuses → HashAggregate with a tiny in-memory hash table.
SELECT
  status,
  COUNT(*)          AS order_count,
  SUM(total_amount) AS revenue
FROM orders
GROUP BY status
ORDER BY revenue DESC;
```

### Example 2 — Intermediate: Filtered, Covering-Index-Friendly Aggregation

```sql
-- Monthly revenue for completed orders, last 12 months.
-- WHERE runs before GROUP BY, shrinking the input to aggregation.
-- A partial covering index makes this an Index Only Scan.
--
--   CREATE INDEX orders_completed_month_idx
--     ON orders (created_at, total_amount)
--     WHERE status = 'completed';
--
SELECT
  DATE_TRUNC('month', created_at)              AS month,
  COUNT(*)                                     AS orders,
  SUM(total_amount)                            AS revenue,
  AVG(total_amount)                            AS avg_order_value,
  COUNT(*) FILTER (WHERE total_amount > 500)   AS large_orders
FROM orders
WHERE status = 'completed'
  AND created_at >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '12 months'
GROUP BY 1
ORDER BY 1;
```

### Example 3 — Production Grade: Two-Phase Pre-Aggregation on Large Tables

```sql
-- Scenario:
--   order_items:  120M rows (many per order)
--   orders:        18M rows
--   Goal: revenue and average order value per customer, completed orders only.
-- Naive approach (join 120M items to orders, then GROUP BY customer) inflates the
--   join and forces a huge HashAggregate that spills.
-- Strategy: aggregate order_items to one row per order FIRST (collapses 120M → 18M),
--   then join the far smaller result and aggregate per customer.
-- Indexes assumed:
--   order_items(order_id) INCLUDE (quantity, unit_price)  -- covering for phase 1
--   orders(id) PRIMARY KEY, orders(status)
-- Perf expectation: phase 1 is a parallel index/seq scan + partial hash aggregate;
--   phase 2 joins 18M pre-aggregated rows. Set work_mem high for the session.

SET LOCAL work_mem = '256MB';
SET LOCAL max_parallel_workers_per_gather = 4;

WITH order_totals AS (
    SELECT
        oi.order_id,
        SUM(oi.quantity * oi.unit_price) AS order_total,
        SUM(oi.quantity)                 AS units
    FROM order_items oi
    GROUP BY oi.order_id                                   -- 120M → 18M rows
)
SELECT
    o.customer_id,
    COUNT(*)                              AS orders,        -- distinct orders (1 row each here)
    SUM(ot.order_total)                   AS revenue,
    AVG(ot.order_total)                   AS avg_order_value,
    SUM(ot.units)                         AS total_units
FROM order_totals ot
JOIN orders o ON o.id = ot.order_id
WHERE o.status = 'completed'
GROUP BY o.customer_id
HAVING SUM(ot.order_total) > 1000
ORDER BY revenue DESC
LIMIT 500;
```

```
EXPLAIN (ANALYZE, BUFFERS) [abridged]:

Limit  (actual time=8400.2..8400.5 rows=500 loops=1)
  ->  Sort  (actual rows=500 loops=1)
        Sort Key: (sum(ot.order_total)) DESC
        Sort Method: top-N heapsort  Memory: 210kB
        ->  Finalize GroupAggregate  (actual rows=1200000 loops=1)
              Group Key: o.customer_id
              Filter: (sum(ot.order_total) > 1000)
              ->  Gather Merge  Workers Planned: 4  Workers Launched: 4
                    ->  Partial GroupAggregate (actual rows=... loops=5)
                          Group Key: o.customer_id
                          ->  Sort  Sort Key: o.customer_id
                                Sort Method: external merge  Disk: 41000kB
                                ->  Parallel Hash Join
                                      Hash Cond: (ot.order_id = o.id)
                                      ->  Subquery Scan on ot
                                            ->  Finalize HashAggregate
                                                  Group Key: oi.order_id
                                                  Batches: 1  Memory Usage: 220000kB
                                                  ->  Gather ... Partial HashAggregate
                                                        ->  Parallel Seq Scan on order_items
                                      ->  Parallel Hash
                                            ->  Parallel Seq Scan on orders
                                                  Filter: (status = 'completed')
Execution Time: 8600.9 ms
```

Reading: phase 1 (`Finalize HashAggregate` over `order_items`) stays in memory (`Batches: 1`) because we raised `work_mem`. Phase 2 joins the collapsed rows and aggregates per customer in parallel. The `top-N heapsort` for the `LIMIT 500` avoids sorting all 1.2M groups fully. Without the pre-aggregation CTE, the join would materialise ~120M rows before grouping and the hash aggregate would spill heavily.

---

## 9. Wrong → Right Patterns

### Wrong 1: Filtering in HAVING instead of WHERE

```sql
-- WRONG: HAVING filters AFTER aggregation, so the aggregate still processes ALL rows
SELECT customer_id, SUM(total_amount) AS revenue
FROM orders
GROUP BY customer_id
HAVING customer_id IN (SELECT id FROM vip_customers);   -- row-level condition!
-- The engine aggregates EVERY customer, then discards non-VIPs. Wasted work on
-- millions of rows that could have been filtered before grouping.
```

```sql
-- RIGHT: push the row-level condition into WHERE (runs before GROUP BY)
SELECT o.customer_id, SUM(o.total_amount) AS revenue
FROM orders o
WHERE o.customer_id IN (SELECT id FROM vip_customers)   -- shrinks input to aggregation
GROUP BY o.customer_id;
-- HAVING is only for conditions on the aggregates themselves, e.g. HAVING SUM(...) > 1000
```

**Why at the execution level:** `WHERE` reduces the number of rows entering the HashAggregate/Sort. `HAVING` cannot — it is evaluated on already-computed groups. Any predicate that does not reference an aggregate belongs in `WHERE`.

### Wrong 2: Assuming work_mem is a global cap and cranking it globally

```sql
-- WRONG: set globally to "fix" spilling
ALTER SYSTEM SET work_mem = '512MB';   -- applied to EVERY operation, EVERY connection
-- With 200 connections each running a query with 2 sort/hash nodes and 4 workers,
-- worst case ≈ 200 × 2 × 4 × 512MB. The box OOMs.
```

```sql
-- RIGHT: raise it locally only for the heavy analytics transaction
BEGIN;
SET LOCAL work_mem = '512MB';        -- scoped to this transaction only
SELECT customer_id, SUM(total_amount) FROM orders GROUP BY customer_id;
COMMIT;
-- Or per-role for a reporting user: ALTER ROLE reporting SET work_mem = '512MB';
```

**Why:** `work_mem` is per-node, per-worker, per-connection. Global increases multiply uncontrollably. Scope the memory to the queries that actually need it.

### Wrong 3: COUNT(DISTINCT) killing parallelism and spilling

```sql
-- WRONG at scale: DISTINCT inside the aggregate disables parallel aggregation and
-- forces per-group deduplication (sort/hash) on 120M rows.
SELECT customer_id, COUNT(DISTINCT product_id) AS distinct_products
FROM order_items
GROUP BY customer_id;
-- Plan: single-threaded, big sort/hash for the DISTINCT. Slow.
```

```sql
-- RIGHT: deduplicate first (parallel-friendly), then aggregate the distinct pairs
SELECT customer_id, COUNT(*) AS distinct_products
FROM (
    SELECT DISTINCT customer_id, product_id
    FROM order_items
) d
GROUP BY customer_id;
-- The inner DISTINCT can parallelize; the outer COUNT(*) is a cheap HashAggregate.
```

**Why:** `COUNT(DISTINCT x)` carries a growing dedup state per group and is not parallel-safe as written. Materialising the distinct pairs first lets both stages parallelize and use plain aggregates.

### Wrong 4: Relying on GROUP BY output order

```sql
-- WRONG: assuming HashAggregate returns groups sorted
SELECT status, COUNT(*)
FROM orders
GROUP BY status;                        -- no ORDER BY
-- HashAggregate output order is UNDEFINED. It may look sorted today and change
-- after a stats update flips the plan to HashAggregate from GroupAggregate.
```

```sql
-- RIGHT: state the order you need explicitly
SELECT status, COUNT(*)
FROM orders
GROUP BY status
ORDER BY status;                        -- deterministic; free if plan is GroupAggregate
```

**Why:** only GroupAggregate produces ordered output, and the planner may switch strategies as data/stats change. Never depend on an implicit order.

### Wrong 5: Aggregating after the join instead of before it

```sql
-- WRONG: join 120M order_items to orders, THEN group by customer.
-- The join materializes 120M rows; the HashAggregate over them spills.
SELECT o.customer_id, SUM(oi.quantity * oi.unit_price) AS revenue
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
WHERE o.status = 'completed'
GROUP BY o.customer_id;
```

```sql
-- RIGHT: collapse order_items to one row per order first, then join and group.
WITH order_totals AS (
    SELECT order_id, SUM(quantity * unit_price) AS order_total
    FROM order_items
    GROUP BY order_id                       -- 120M → 18M rows
)
SELECT o.customer_id, SUM(ot.order_total) AS revenue
FROM order_totals ot
JOIN orders o ON o.id = ot.order_id
WHERE o.status = 'completed'
GROUP BY o.customer_id;
```

**Why:** pre-aggregation reduces the row count entering the join and the final aggregate, shrinking both memory pressure and spill risk. (This is the fan-out reasoning from Topic 11 applied to performance.)

---

## 10. Performance Profile

### Strategy comparison

| Strategy | When Chosen | Memory | Input Requirement | Spills? |
|---|---|---|---|---|
| HashAggregate | Unsorted input, groups fit `work_mem` | O(distinct groups) | none | Yes (PG13+), `Batches>1` |
| GroupAggregate (index-fed) | Sorted index on group cols exists | O(1) | pre-sorted input | No (index) |
| GroupAggregate (sort-fed) | Huge cardinality, no index | O(1) agg + O(N) sort | sort node | Yes (external merge) |
| Parallel (Partial+Finalize) | Large scan, parallel-safe aggs | ~workers × `work_mem` | none | Per-worker |

### Scaling at 1M / 10M / 100M rows

| Rows | Groups | Likely plan | Typical time (tuned) | Failure mode if untuned |
|---|---|---|---|---|
| 1M | ~5 | HashAggregate, in-mem | 100–350 ms | none — trivial |
| 1M | ~2M | HashAggregate spill or index GroupAgg | 1–2.5 s | spill if `work_mem` tiny |
| 10M | ~10 | Parallel HashAggregate | 300–900 ms | single-thread if no parallelism |
| 10M | ~5M | index-fed GroupAggregate | 1.5–4 s | disk spill on sort/hash |
| 100M | ~50 | Parallel Partial/Finalize HashAgg | 2–6 s | non-parallel = 20–40 s |
| 100M | ~50M | Pre-aggregation / rollup table | pre-aggregate offline | any live plan spills massively |

### Optimization checklist specific to aggregation

1. **Push predicates into `WHERE`** so fewer rows reach the aggregate.
2. **Match an index to the `GROUP BY` order** to get GroupAggregate with no sort (and Index Only Scan if covering).
3. **Raise `work_mem` per-session** for heavy reports so `Batches: 1` / in-memory sort.
4. **Use `hash_mem_multiplier`** (PG13+) to favour hash nodes without inflating sort memory.
5. **Enable parallelism**: `max_parallel_workers_per_gather` ≥ 4 for big scans; keep functions parallel-safe.
6. **Avoid `COUNT(DISTINCT)`** on hot paths; pre-distinct or approximate.
7. **Pre-aggregate before joins** to prevent fan-out inflation.
8. **Materialise** (matview/rollup) when the same aggregation runs repeatedly on slow-changing data.
9. **Keep statistics fresh** (`ANALYZE`, extended stats for correlated group columns) so the planner sizes the hash table right.
10. **Vacuum** so Index Only Scans skip the heap (`Heap Fetches: 0`).

---

## 11. Node.js Integration

### 11.1 Basic aggregation with pg

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function revenueByStatus() {
  const { rows } = await pool.query(
    `SELECT status,
            COUNT(*)          AS order_count,
            SUM(total_amount) AS revenue
     FROM orders
     GROUP BY status
     ORDER BY revenue DESC`
  );
  // NOTE: COUNT and SUM come back as STRINGS in node-postgres (bigint/numeric),
  // to avoid precision loss. Parse explicitly.
  return rows.map(r => ({
    status: r.status,
    orderCount: Number(r.order_count),
    revenue: Number(r.revenue),
  }));
}
```

### 11.2 Parameterised, filtered aggregation with a per-transaction work_mem

```javascript
async function monthlyRevenue(months = 12) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    // Scope the memory bump to THIS transaction only — never set it globally.
    await client.query(`SET LOCAL work_mem = '256MB'`);
    const { rows } = await client.query(
      `SELECT DATE_TRUNC('month', created_at) AS month,
              COUNT(*)                        AS orders,
              SUM(total_amount)               AS revenue,
              COUNT(*) FILTER (WHERE total_amount > 500) AS large_orders
       FROM orders
       WHERE status = 'completed'
         AND created_at >= DATE_TRUNC('month', CURRENT_DATE)
                           - ($1 || ' months')::INTERVAL
       GROUP BY 1
       ORDER BY 1`,
      [months]
    );
    await client.query('COMMIT');
    return rows;
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();
  }
}
```

### 11.3 Refreshing a rollup / materialised view from a worker

```javascript
// Run on a schedule (cron / pg_cron / a job queue), not on the request path.
async function refreshDailyRevenue() {
  // CONCURRENTLY avoids blocking dashboard reads; requires a unique index on the matview.
  await pool.query('REFRESH MATERIALIZED VIEW CONCURRENTLY daily_revenue');
}

// Dashboards then read the precomputed rollup — millisecond responses.
async function getDailyRevenue(days = 30) {
  const { rows } = await pool.query(
    `SELECT day, orders, revenue
     FROM daily_revenue
     WHERE day >= CURRENT_DATE - ($1 || ' days')::INTERVAL
     ORDER BY day`,
    [days]
  );
  return rows;
}
```

### 11.4 Streaming a large aggregation result

```javascript
// For very large GROUP BY outputs, stream rows instead of buffering all in memory.
import QueryStream from 'pg-query-stream';

async function streamCustomerRevenue(onRow) {
  const client = await pool.connect();
  try {
    const query = new QueryStream(
      `SELECT customer_id, SUM(total_amount) AS revenue
       FROM orders
       GROUP BY customer_id
       ORDER BY customer_id`   // ordered → GroupAggregate, streams incrementally
    );
    const stream = client.query(query);
    for await (const row of stream) onRow(row);
  } finally {
    client.release();
  }
}
```

A note on ORMs: every mainstream ORM can express `GROUP BY` with `SUM`/`COUNT`/`AVG`, but few give you control over `work_mem`, parallelism, `FILTER`, `COUNT(DISTINCT)` rewrites, or pre-aggregation CTEs. For performance-critical aggregation you almost always drop to raw SQL. See Section 12.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do aggregation?** — Yes, via `aggregate` and `groupBy`, but with a limited surface.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

const byStatus = await prisma.order.groupBy({
  by: ['status'],
  _count: { _all: true },
  _sum: { totalAmount: true },
  _avg: { totalAmount: true },
  having: { totalAmount: { _sum: { gt: 1000 } } },
  orderBy: { _sum: { totalAmount: 'desc' } },
});

// FILTER, COUNT(DISTINCT), DATE_TRUNC grouping, pre-aggregation CTEs → raw SQL:
const monthly = await prisma.$queryRaw`
  SELECT DATE_TRUNC('month', created_at) AS month,
         COUNT(*) AS orders, SUM(total_amount) AS revenue
  FROM orders WHERE status = 'completed'
  GROUP BY 1 ORDER BY 1`;
```

**Where it breaks:** no `DATE_TRUNC`/expression grouping, no `FILTER`, no `COUNT(DISTINCT)`, no control over `work_mem`/parallelism, no pre-aggregation CTE. **Verdict:** fine for simple single-table rollups; use `$queryRaw` for anything real.

### Drizzle ORM

**Can Drizzle do aggregation?** — Yes, close to SQL with typed helpers.

```typescript
import { db } from './db';
import { orders } from './schema';
import { sql, count, sum, avg, eq, gt } from 'drizzle-orm';

const byStatus = await db
  .select({
    status: orders.status,
    orderCount: count(),
    revenue: sum(orders.totalAmount),
    avgOrder: avg(orders.totalAmount),
    large: sql<number>`COUNT(*) FILTER (WHERE ${orders.totalAmount} > 500)`,
  })
  .from(orders)
  .where(eq(orders.status, 'completed'))
  .groupBy(orders.status)
  .having(gt(sum(orders.totalAmount), sql`1000`));
```

**Where it breaks:** `FILTER`, `DATE_TRUNC`, and pre-aggregation CTEs need the `sql` template, but everything stays composable. **Verdict:** best-in-class for expressive aggregation without leaving the ORM.

### Sequelize

**Can Sequelize do aggregation?** — Yes, via `attributes` + `fn`/`col` + `group`.

```javascript
const { fn, col, literal } = require('sequelize');
const rows = await Order.findAll({
  attributes: [
    'status',
    [fn('COUNT', col('id')), 'orderCount'],
    [fn('SUM', col('total_amount')), 'revenue'],
    [literal(`COUNT(*) FILTER (WHERE total_amount > 500)`), 'large'],
  ],
  where: { status: 'completed' },
  group: ['status'],
  order: [[literal('revenue'), 'DESC']],
  raw: true,
});
```

**Where it breaks:** verbose; `FILTER`, `DATE_TRUNC`, `HAVING` on aggregates, and CTEs need `literal()`, which loses type safety. **Verdict:** workable for basic group-bys; escape to `sequelize.query()` for complex reports.

### TypeORM

**Can TypeORM do aggregation?** — Yes, via QueryBuilder with `.getRawMany()`.

```typescript
const rows = await dataSource
  .createQueryBuilder()
  .select('o.status', 'status')
  .addSelect('COUNT(*)', 'orderCount')
  .addSelect('SUM(o.total_amount)', 'revenue')
  .addSelect(`COUNT(*) FILTER (WHERE o.total_amount > 500)`, 'large')
  .from('orders', 'o')
  .where('o.status = :s', { s: 'completed' })
  .groupBy('o.status')
  .having('SUM(o.total_amount) > :m', { m: 1000 })
  .orderBy('revenue', 'DESC')
  .getRawMany();
```

**Where it breaks:** aggregate expressions are raw strings (no type safety); pre-aggregation CTEs need `.addCommonTableExpression()` or raw SQL. **Verdict:** capable via QueryBuilder + `getRawMany`; use `dataSource.query()` for heavy analytics.

### Knex.js

**Can Knex do aggregation?** — Yes, the most SQL-transparent.

```javascript
const rows = await knex('orders')
  .select('status')
  .count('* as order_count')
  .sum('total_amount as revenue')
  .avg('total_amount as avg_order')
  .select(knex.raw(`COUNT(*) FILTER (WHERE total_amount > 500) AS large`))
  .where('status', 'completed')
  .groupBy('status')
  .havingRaw('SUM(total_amount) > ?', [1000])
  .orderBy('revenue', 'desc');

// Pre-aggregation CTE:
const report = await knex
  .with('order_totals', knex('order_items')
    .select('order_id')
    .sum(knex.raw('quantity * unit_price as order_total'))
    .groupBy('order_id'))
  .select('o.customer_id')
  .sum('ot.order_total as revenue')
  .from('order_totals as ot')
  .join('orders as o', 'o.id', 'ot.order_id')
  .where('o.status', 'completed')
  .groupBy('o.customer_id');
```

**Where it breaks:** `FILTER` still needs `knex.raw`, but CTEs and having are first-class. **Verdict:** most transparent; comfortably expresses production aggregation.

### ORM Summary Table

| ORM | GROUP BY | FILTER | COUNT(DISTINCT) | Pre-agg CTE | work_mem control | Verdict |
|---|---|---|---|---|---|---|
| Prisma | `groupBy` | raw | raw | raw | no | Simple rollups only |
| Drizzle | `.groupBy()` | `sql` | `sql` | `sql`/`with` | no (raw `SET`) | Best typed support |
| Sequelize | `group` | `literal` | `literal` | raw | no | Basic only; raw for reports |
| TypeORM | `.groupBy()` | raw string | raw | CTE/raw | no | Capable via QueryBuilder |
| Knex | `.groupBy()` | `raw` | `raw` | `.with()` | no (raw `SET`) | Most transparent |

None expose `work_mem`, `hash_mem_multiplier`, or parallelism settings — issue a `SET LOCAL` on the same connection/transaction before the query when you need them.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given `orders(id, customer_id, total_amount, status, created_at)`:

Write a query returning, per `status`: the number of orders, total revenue, and average order value. Order by revenue descending. Then explain: which physical strategy (HashAggregate or GroupAggregate) will the planner most likely choose, and why?

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines Topic 11 joins + this topic)

Given `orders(id, customer_id, status, created_at)` and `order_items(id, order_id, product_id, quantity, unit_price)`:

Write a query returning, per customer, for completed orders: the number of distinct orders, total revenue (`SUM(quantity * unit_price)`), and total units. Avoid the fan-out counting bug from Topic 11, and structure it so the aggregation over `order_items` happens *before* the join to keep the join small. Explain why your structure reduces spill risk.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; naive answer is wrong)

`events(id, user_id, event_type, occurred_at)` has 400M rows. You need daily active users (distinct `user_id`) per day for the last 90 days. The naive query:

```sql
SELECT DATE_TRUNC('day', occurred_at) AS day, COUNT(DISTINCT user_id) AS dau
FROM events
WHERE occurred_at >= CURRENT_DATE - INTERVAL '90 days'
GROUP BY 1;
```

runs for 4 minutes and spills heavily. Explain every reason it is slow (at least three), then write an approach that makes it fast for a dashboard that reloads often. Consider: `COUNT(DISTINCT)` cost, parallelism, indexing on `occurred_at`, and pre-aggregation/rollup. What would the rollup table and its incremental refresh look like?

```sql
-- Write your query / rollup design here
```

---

### Exercise 4 — Interview Simulation

You are handed this EXPLAIN fragment for a report that "suddenly got slow after we upgraded to PostgreSQL 14":

```
HashAggregate  (rows=3000000)
  Group Key: customer_id
  Batches: 17  Memory Usage: 4096kB  Disk Usage: 512000kB
  ->  Seq Scan on orders (rows=40000000)
```

1. Explain exactly what `Batches: 17` and `Disk Usage: 512000kB` mean.
2. Why might this have appeared only after the PG13/14 upgrade even though the query is unchanged?
3. Give three distinct fixes, ordered by how you would try them, with the reasoning for each.

```sql
-- Write your answer here
```

---

## 14. Interview Questions

### Junior Level

**Q: What is the difference between HashAggregate and GroupAggregate?**

Junior answer: "HashAggregate uses a hash table and GroupAggregate uses sorting."

Principal answer: HashAggregate builds an in-memory hash table keyed by the grouping columns and folds each input row into that group's aggregate state in a single pass — no sorting, memory proportional to the number of distinct groups. GroupAggregate requires input already sorted on the grouping keys (from an index or an explicit Sort) and keeps only the current group's state, so it uses O(1) memory but pays for the sort. The planner picks HashAggregate when groups fit in `work_mem` and there is no free sorted order; it picks GroupAggregate when the input is already sorted (index), when a subsequent ORDER BY can reuse the order, or when the group count is so large that a hash table would spill heavily.

Interviewer follow-up: "When would GroupAggregate be strictly better even though it needs a sort?" — When an index already provides the sorted order for free (no Sort node), turning aggregation into a single ordered index scan with constant memory and no spill risk; or when the query's ORDER BY on the grouping keys would otherwise require a separate sort anyway.

---

**Q: Where should you filter rows in a GROUP BY query — WHERE or HAVING?**

Junior answer: "Either works, HAVING is for after grouping."

Principal answer: Any predicate that does not reference an aggregate must go in `WHERE`, because `WHERE` runs *before* grouping and shrinks the input the aggregate has to process, while `HAVING` runs *after* and cannot reduce aggregation work. Only conditions on the aggregates themselves (`HAVING SUM(x) > 1000`, `HAVING COUNT(*) > 5`) belong in `HAVING`. Putting a row-level condition in `HAVING` forces the engine to aggregate every group and then throw most away — wasted work proportional to the full table.

Interviewer follow-up: "Can the planner move a HAVING predicate into WHERE automatically?" — It can in some cases when the predicate is provably row-level and equivalent, but you should not rely on it; write it in `WHERE` yourself for clarity and to guarantee the reduction.

---

### Principal Level

**Q: Explain how PostgreSQL 13 changed HashAggregate behaviour and why a report might get slower after upgrading.**

Junior answer: "PG13 made aggregation faster."

Principal answer: Before PG13, HashAggregate could not spill to disk. If the hash table exceeded `work_mem` at runtime, it would keep allocating and risk exhausting memory — so the planner conservatively avoided HashAggregate whenever it estimated the groups might not fit, often choosing a slower sort-based GroupAggregate defensively. PG13 added hash aggregation spill: when the table exceeds `work_mem`, overflowing groups' input is written to temp files and processed in extra batches (visible as `Batches > 1` and `Disk Usage` in EXPLAIN ANALYZE). The upside is memory safety and the planner more freely choosing HashAggregate. The downside: a query that previously used HashAggregate while quietly overshooting `work_mem` in RAM (fast) may now respect `work_mem` and spill to disk (slower but safe), or a query that previously ran fine now shows disk spill. The fix is to raise `work_mem` for that workload (or `hash_mem_multiplier`) so `Batches` returns to 1, or to pre-aggregate.

Interviewer follow-up: "What is `hash_mem_multiplier` and why introduce it separately from `work_mem`?" — It lets hash-based nodes (hash join, hash aggregate) use a multiple of `work_mem` while leaving sort memory at the base `work_mem`. This is because hash nodes benefit disproportionately from more memory (avoiding spill batches) and you often want to feed them more without inflating memory for every sort in the plan.

---

**Q: A 100M-row aggregation shows a plain `HashAggregate` (no Partial/Finalize) taking 35 seconds on a 16-core box. What is wrong and how do you get it parallel?**

Principal answer: The absence of `Partial HashAggregate` / `Finalize HashAggregate` (and a `Gather`) means parallel aggregation was not used, so a single process is doing all the work. Common causes: `max_parallel_workers_per_gather` is 0 or too low (default 2 is often insufficient); the query contains a parallel-unsafe function (a `VOLATILE` PL/pgSQL function, or `COUNT(DISTINCT)` which restricts parallelism); the table's estimated scan cost is below the parallel threshold; or `max_parallel_workers`/`max_worker_processes` starve the pool. Fixes: `SET max_parallel_workers_per_gather = 4` (or higher), ensure all functions in SELECT/WHERE are `PARALLEL SAFE`, remove or rewrite `COUNT(DISTINCT)`, and confirm cluster-wide worker limits. After that, EXPLAIN should show Partial aggregates in workers feeding a Finalize aggregate. Also budget memory: each worker gets its own `work_mem`.

Interviewer follow-up: "Why does the Partial aggregate emit more rows than the final group count?" — Each worker independently produces its own partial group for the same key (worker A and worker B both emit a partial row for customer 42). The `Finalize Aggregate` then combines same-key partials using the aggregate's combine function to produce one final row per group.

---

**Q: How do indexes make GROUP BY faster, and what is the MIN/MAX special case?**

Principal answer: Three ways. (1) A B-tree on the grouping columns delivers rows in sorted order, letting the planner use GroupAggregate with no Sort node — aggregation for the cost of an index scan, with constant memory and no spill. (2) A covering index (grouping columns plus aggregated columns) enables an Index Only Scan that never touches the heap, provided the visibility map is current. (3) The MIN/MAX special case: `MAX(indexed_col)` with no GROUP BY becomes an index endpoint lookup — the planner rewrites it to `ORDER BY col DESC LIMIT 1` over the index, reading a single entry in O(log N) instead of scanning the table. This only applies to a single MIN or MAX per index and breaks if wrapped in expressions or combined with incompatible grouping.

Interviewer follow-up: "Does `SELECT MIN(x), MAX(x) FROM t` use the index?" — Yes, PostgreSQL can satisfy it with two separate index endpoint lookups (one ascending, one descending) via InitPlans, still O(log N) each, provided an index on `x` exists.

---

## 15. Mental Model Checkpoint

1. You run `SELECT customer_id, SUM(total_amount) FROM orders GROUP BY customer_id` on a 50M-row table with 20M distinct customers and `work_mem = 4MB`. Which strategy does the planner likely choose, will it spill, and what single change most reliably eliminates the spill?

2. Explain why a `LIMIT 10` on a `GROUP BY` query usually does not reduce the aggregation work, and describe the one situation where it can.

3. Your report ran in 2 seconds on PostgreSQL 12 and takes 9 seconds on PostgreSQL 14 with an unchanged query and unchanged data. EXPLAIN now shows `Batches: 6, Disk Usage: 200MB`. What happened, and why is it arguably "correct" behaviour?

4. Why does `COUNT(DISTINCT product_id)` disable parallel aggregation and often spill, while `COUNT(*)` and `SUM(x)` parallelise cleanly? Reason about the aggregate transition/combine state.

5. You have an index on `orders(customer_id, total_amount)`. Explain how the plan for `SELECT customer_id, SUM(total_amount) FROM orders GROUP BY customer_id` differs from the same query with no index, in terms of nodes, memory, and spill risk.

6. A parallel aggregation's `Partial HashAggregate` emits 8M rows but the final result has 2M groups. Explain the discrepancy without using the word "bug."

7. When is it better to maintain a rollup table (incremental `INSERT ... ON CONFLICT DO UPDATE`) versus a materialised view with `REFRESH CONCURRENTLY`, and what property of the data drives that choice?

---

## 16. Quick Reference Card

```sql
-- TWO PHYSICAL STRATEGIES
HashAggregate    -- hash table keyed by GROUP BY cols; 1 pass; mem ∝ #groups
                 -- spills in PG13+ → EXPLAIN shows Batches>1 + Disk Usage
GroupAggregate   -- needs input sorted on GROUP BY cols; O(1) memory
                 -- fed by Index Scan (free) or Sort node (may spill: external merge)

-- PLANNER CHOICE drivers: estimated #groups (n_distinct), work_mem,
-- existence of a usefully-ordered index, cost of the sort.

-- WHERE vs HAVING
WHERE  <row predicate>     -- runs BEFORE grouping; shrinks aggregate input. USE THIS.
HAVING <aggregate predicate>  -- runs AFTER grouping; only for SUM()/COUNT() conditions.

-- work_mem (per node, per worker, per connection — NOT a global cap)
SET LOCAL work_mem = '256MB';        -- scope to the heavy transaction only
SET hash_mem_multiplier = 2.0;       -- PG13+: give hash nodes extra memory vs sorts

-- SPILL SIGNALS in EXPLAIN ANALYZE
HashAggregate ... Batches: N (N>1)  Disk Usage: …   -- hash aggregate spilled (PG13+)
Sort Method: external merge  Disk: …                -- sort feeding GroupAggregate spilled
HashAggregate ... Batches: 1  Memory Usage: …       -- healthy, stayed in RAM

-- PARALLEL AGGREGATION (Partial → Gather → Finalize)
SET max_parallel_workers_per_gather = 4;
-- Finalize HashAggregate → Gather → Partial HashAggregate → Parallel Seq Scan
-- Requires parallel-safe aggregates with a combine function (sum/count/avg/min/max ok).
-- COUNT(DISTINCT) and volatile/unsafe functions DISABLE parallelism.

-- INDEXES FOR GROUP BY
CREATE INDEX ON orders (customer_id);                  -- sorted input → GroupAggregate
CREATE INDEX ON orders (customer_id, total_amount);    -- covering → Index Only Scan
CREATE INDEX ON orders (created_at)
  WHERE status = 'completed';                          -- partial, for filtered aggregation
SELECT MAX(created_at) FROM orders;                    -- index endpoint lookup, O(log N)

-- CHEAP CONDITIONAL AGGREGATION (one pass)
COUNT(*) FILTER (WHERE status = 'completed')

-- AVOID COUNT(DISTINCT) at scale — pre-distinct then count:
SELECT k, COUNT(*) FROM (SELECT DISTINCT k, x FROM t) d GROUP BY k;

-- PRE-AGGREGATION
CREATE MATERIALIZED VIEW ...; REFRESH MATERIALIZED VIEW CONCURRENTLY ...;
INSERT INTO rollup ... GROUP BY ... ON CONFLICT (key) DO UPDATE ...;  -- incremental
-- Aggregate BEFORE joins to avoid fan-out inflation (Topic 11).

-- INTERVIEW ONE-LINERS
-- "HashAggregate = one pass, memory ∝ #groups, spills in PG13+."
-- "GroupAggregate = sorted input, O(1) memory; free when an index provides the order."
-- "Filter in WHERE, not HAVING — WHERE runs before grouping."
-- "work_mem is per node per worker per connection; raise it with SET LOCAL, never globally."
-- "COUNT(DISTINCT) kills parallelism and spills — pre-distinct or approximate."
-- "MIN/MAX on an indexed column is an O(log N) endpoint lookup, not a scan."
```

---

## Connected Topics

- **Topic 20 — GROUP BY Fundamentals**: the logical semantics of grouping; this topic is the physical execution of it.
- **Topic 24 — GROUPING SETS**: multiple grouping levels in one pass; same HashAggregate/spill rules apply, amplified.
- **Topic 26 — Subqueries in Depth** (next): pre-aggregation subqueries and CTEs are a core aggregation-performance tool; how the planner treats them.
- **Topic 11 — INNER JOIN in Depth**: fan-out reasoning — why pre-aggregating before a join reduces both join and aggregate cost.
- **Topic 17 — JOIN Performance Deep Dive**: the same hash-table and `work_mem`/spill machinery used by Hash Join; parallel plans.
- **Topic 07 — NULL in Depth**: NULLs form their own group in `GROUP BY` and are ignored by most aggregates except `COUNT(*)`.
- **Internals — Buffer pool, MVCC, pg_statistic (`n_distinct`), parallel query, external merge sort**: the engine subsystems that govern whether an aggregation stays in memory, parallelises, and estimates group counts correctly.
```
