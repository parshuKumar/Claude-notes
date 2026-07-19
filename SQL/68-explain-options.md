# Topic 68 — EXPLAIN Options Deep Dive
### SQL Mastery Curriculum — Phase 10: PostgreSQL Specific Power Features

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you hire a courier to deliver a package across a city, and you want to understand the trip.

You can ask three very different questions:

1. **"What's your plan?"** — The courier looks at the map and says: "I'll take the highway, then Main Street, then cut through the park. I estimate about 25 minutes and 12 kilometres." This is a **plan** — it costs nothing, no wheels turn, it's pure prediction. This is plain `EXPLAIN`.

2. **"Actually drive it and tell me exactly what happened."** — Now the courier drives the whole route with a stopwatch and odometer, and comes back saying: "I planned 25 minutes but it actually took 41 because Main Street was jammed. I planned 12 km, drove 12 km." This is `EXPLAIN ANALYZE` — the query **actually runs**, and you get planned-vs-actual side by side.

3. **"And while you drive, count how much fuel you burned, how many times you stopped for gas, and how many potholes you hit."** — Now you're measuring the *physical resources* of the trip, not just the time. In Postgres, "fuel" is **buffers** — how many 8KB pages were read from RAM (fast, `hit`) versus fetched from disk (slow, `read`). This is `EXPLAIN (ANALYZE, BUFFERS)`.

Here is the one warning that this entire topic hangs on: **asking the courier to "actually drive it" means the trip really happens.** If the "trip" is a `DELETE FROM orders`, then running `EXPLAIN ANALYZE` on it **deletes your orders**. The plan is free and safe; the measurement is real and has consequences.

That single distinction — prediction versus execution — is the beating heart of everything in this topic. Master it and you can diagnose any slow query in production. Miss it and you'll one day run `EXPLAIN ANALYZE DELETE ...` on a live table and learn the hard way.

---

## 2. Connection to SQL Internals

`EXPLAIN` is a window directly into the PostgreSQL **query planner/optimizer** and, with `ANALYZE`, into the **executor**. Understanding its options means understanding the machinery underneath.

**The planner (cost-based optimizer).** When you submit a query, Postgres parses it, rewrites it (applying views, rules), and then the planner generates candidate execution plans. Each candidate is assigned a **cost** — an abstract, unitless number derived from `seq_page_cost` (default 1.0), `random_page_cost` (default 4.0), `cpu_tuple_cost` (0.01), `cpu_index_tuple_cost` (0.005), and `cpu_operator_cost` (0.0025). The plan with the lowest total cost wins. Plain `EXPLAIN` shows you these estimated costs and the estimated row counts drawn from `pg_statistic` (populated by `ANALYZE` the *command*, not the option).

**The executor.** `EXPLAIN ANALYZE` runs the chosen plan through the executor node tree — a pull-based (Volcano/iterator) model where each node repeatedly calls its children for the next tuple. Every node is instrumented with a timer and a row counter, so you get `actual time` and `actual rows` per node, per loop.

**The buffer pool (shared_buffers).** `BUFFERS` exposes the interaction with Postgres's page cache. All table and index data lives in 8KB **pages** (blocks). To read a row, the executor requests its page. If the page is already in `shared_buffers`, that's a **hit** (nanoseconds). If not, Postgres asks the OS — which may serve it from the OS page cache or fetch it from physical disk — that's a **read** (microseconds to milliseconds). If the query modifies a page it becomes **dirtied**, and if a backend has to flush an old dirty page to make room, that's **written**. This is exactly how you tell an **I/O-bound** query from a **CPU-bound** one.

**WAL (Write-Ahead Log).** The `WAL` option (PG 13+) reports how many WAL records/bytes a data-modifying statement generated. Every `INSERT`/`UPDATE`/`DELETE` must be logged to WAL before commit (durability). Diagnosing bloated WAL generation — full-page images, over-large transactions — starts here.

**MVCC and heap fetches.** When you see `Heap Fetches` under an Index Only Scan, that's MVCC leaking through: the visibility map wasn't all-visible, so Postgres had to visit the heap page to check tuple visibility. `BUFFERS` and `VERBOSE` make these costs visible.

`EXPLAIN` options are the instrument panel bolted onto every one of these subsystems.

---

## 3. Logical Execution Order Context

`EXPLAIN` is unusual: it does not sit *inside* the logical clause-ordering pipeline (`FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → window → ORDER BY → LIMIT`). Instead, it **wraps** an entire statement and reports how the planner intends to walk that whole pipeline.

```
EXPLAIN [ ( option [, ...] ) ] statement
             │                    │
             │                    └── ANY statement: SELECT, INSERT, UPDATE,
             │                        DELETE, MERGE, VALUES, EXECUTE, DECLARE,
             │                        CREATE TABLE AS, CREATE MATERIALIZED VIEW
             └── zero or more comma-separated options
```

What matters for reading a plan is that the **executor tree is inverted relative to the logical order**. The plan is a tree; execution is depth-first, pull-based, from the **innermost/bottom nodes upward**:

- The **leaf** nodes (Seq Scan, Index Scan) correspond to `FROM` — they produce base rows.
- **Join** nodes sit above them (the `JOIN`/`ON` phase).
- **Filter** lines on scan nodes are `WHERE` predicates pushed down.
- **Aggregate**/**HashAggregate**/**GroupAggregate** nodes are `GROUP BY` + `HAVING`.
- **WindowAgg** is the window phase.
- **Sort** is `ORDER BY` (or a sort feeding a Merge Join / GroupAggregate).
- **Limit** is the top node, `LIMIT`.

So reading a plan means reading the logical execution order **backwards through the tree**: the bottom-most, most-indented node runs first. This is why `LIMIT` appears at the very top yet a `LIMIT` can still make the whole plan fast — it can stop pulling from below early. Remember from Topic 11 that the outermost loop of a Nested Loop is the driving table; that same "outermost = first" reading applies to the entire tree.

The one subtlety: `ANALYZE` executes the statement, so all side effects (triggers, `RETURNING`, modifications) happen in real execution order during the `EXPLAIN` call itself.

---

## 4. What Is EXPLAIN (with options)?

`EXPLAIN` displays the execution plan the planner generates for a statement. Plain `EXPLAIN` shows estimates only (no execution). The **options** in parentheses control what is measured and how it is displayed — most importantly `ANALYZE`, which actually **executes** the statement to collect real runtime measurements.

```sql
EXPLAIN (
  ANALYZE   true,     -- │ actually RUN the statement; report actual time & rows
  VERBOSE   true,     -- │ show output column lists, schema-qualified names, workers
  COSTS     true,     -- │ show estimated startup/total cost & row/width estimates
  SETTINGS  true,     -- │ show non-default planner GUCs that affected the plan
  BUFFERS   true,     -- │ report shared/local/temp buffer hits/reads/dirtied/written
  WAL       true,     -- │ report WAL records/bytes generated (write statements)
  TIMING    true,     -- │ include per-node actual timing (requires ANALYZE)
  SUMMARY   true,     -- │ show Planning Time & Execution Time footer
  FORMAT    TEXT      -- │ output shape: TEXT | XML | JSON | YAML
)
SELECT ...;
```

### Every option, precisely

```
ANALYZE (default OFF)
  └── Executes the statement and measures reality. Turns "estimated" into
      "estimated vs actual". DANGER: runs INSERT/UPDATE/DELETE for real.

VERBOSE (default OFF)
  └── Adds: Output: column list per node, schema.table.column qualification,
      trigger names, per-worker stats, Settings/JIT detail. More noise, more truth.

COSTS (default ON)
  └── The cost=STARTUP..TOTAL rows=N width=B triplet on every node.
      Turn OFF (COSTS false) for clean diffs / regression test fixtures.

SETTINGS (default OFF)
  └── Lists planner-relevant GUCs that differ from default
      (e.g. work_mem, random_page_cost, enable_seqscan). PG 12+.

GENERIC_PLAN (default OFF, PG 16+)
  └── Plan a parameterized statement ($1,$2) WITHOUT executing —
      shows the generic plan the server would cache. Cannot combine with ANALYZE.

BUFFERS (default ON since PG 18; was OFF and required ANALYZE before)
  └── shared hit/read/dirtied/written, local ..., temp read/written.
      The single most important flag for I/O-vs-CPU diagnosis.

WAL (default OFF, PG 13+)
  └── WAL: records=N fpi=N bytes=N for data-modifying statements.

TIMING (default ON when ANALYZE is on)
  └── Per-node time. Turn OFF (TIMING false) on systems with slow clocks —
      you still get actual rows & buffers, just not per-node timing.

SUMMARY (default ON when ANALYZE is on, else OFF)
  └── The Planning Time / Execution Time footer lines.

MEMORY (default OFF, PG 17+)
  └── Planner memory consumption (used/allocated bytes).

FORMAT { TEXT | XML | JSON | YAML } (default TEXT)
  └── TEXT for humans; JSON/YAML for tools (auto_explain, pgMustard, dashboards).
```

Two syntaxes exist. The **legacy** form `EXPLAIN ANALYZE VERBOSE SELECT ...` works but only for `ANALYZE` and `VERBOSE`. The **parenthesized** form `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...` is the modern, extensible one and the only way to reach `BUFFERS`, `WAL`, `SETTINGS`, `FORMAT`, etc. Options take an optional boolean (`ANALYZE`, `ANALYZE true`, `ANALYZE on` are equivalent; `ANALYZE false` disables). Always use the parenthesized form in production.

---

## 5. Why EXPLAIN Options Mastery Matters in Production

1. **You cannot fix what you cannot see.** A query taking 4 seconds is a symptom. `EXPLAIN (ANALYZE, BUFFERS)` is the only tool that tells you *why* — wrong join algorithm, missing index, disk spill, bad row estimate, or I/O storm. Guessing wastes hours; reading the plan takes minutes.

2. **I/O-bound vs CPU-bound is a `BUFFERS` question.** Two queries with identical `Execution Time` need opposite fixes. One reads 2 million pages from disk (add an index, increase cache, fix bloat). The other reads almost nothing but burns CPU in a nested loop or a sort (restructure the query, add `work_mem`). Without `BUFFERS` you'd apply the wrong fix.

3. **The estimate-vs-actual gap is where bugs hide.** A planner that estimates 10 rows but gets 2 million (`rows=10 ... actual rows=2000000`) has stale or missing statistics, and it likely chose a Nested Loop that's now catastrophic. Only `ANALYZE` reveals this divergence; plain `EXPLAIN` shows the wrong estimate with false confidence.

4. **The `ANALYZE`-executes gotcha has destroyed production data.** Engineers who don't internalize "ANALYZE runs the query" have deleted rows, double-charged payments, and fired triggers while "just checking a plan." Knowing to wrap writes in `BEGIN; ... ROLLBACK;` is not optional knowledge — it's a safety interlock.

5. **`loops` and per-loop numbers cause misdiagnosis.** In a Nested Loop, `actual time=0.05..0.08 rows=1 loops=500000` means the inner node ran 500,000 times, each cheap, but the *total* is 40 seconds. Engineers who multiply-vs-don't-multiply incorrectly chase the wrong node. This single misreading is the most common EXPLAIN mistake in interviews and on the job.

6. **`FORMAT JSON` powers your tooling.** `auto_explain`, monitoring dashboards, and plan-visualizers (pgMustard, explain.dalibo.com, PEV) all consume JSON. Knowing the machine format lets you automate slow-query capture instead of eyeballing logs.

---

## 6. Deep Technical Content

### 6.1 The Anatomy of a Plan Node

Every line in a plan is a **node** with a consistent shape:

```
Node Type  (cost=STARTUP..TOTAL rows=EST_ROWS width=BYTES)
           (actual time=FIRST_ROW..LAST_ROW rows=ACT_ROWS loops=N)
```

- **cost=STARTUP..TOTAL** — `STARTUP` is the cost before the first row can be emitted (e.g., a Sort must consume all input before emitting anything, so its startup cost is high). `TOTAL` is the cost to return all rows. Units are abstract (page-fetch-equivalents), not milliseconds.
- **rows** (estimate) — planner's guess at how many rows this node emits, from `pg_statistic`.
- **width** — estimated average bytes per row. Matters for memory (sorts, hashes) and for whether a Hash spills.
- **actual time=FIRST..LAST** — milliseconds to the first row / to the last row, **per loop** (averaged across loops). This is the trap: it's per-loop, not total.
- **actual rows** — average rows emitted **per loop** (PG rounds; PG 18 shows fractions for sub-1 averages).
- **loops** — how many times this node was executed (re-scanned). Total node rows = `rows × loops`. Total node time ≈ `actual time (last) × loops`.

### 6.2 ANALYZE — Estimated vs Actual

Plain `EXPLAIN` gives you the planner's *beliefs*:

```sql
EXPLAIN
SELECT * FROM orders WHERE status = 'pending';
```
```
Seq Scan on orders  (cost=0.00..2310.00 rows=1200 width=64)
  Filter: (status = 'pending')
```

`EXPLAIN ANALYZE` gives you *reality alongside the belief*:

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'pending';
```
```
Seq Scan on orders  (cost=0.00..2310.00 rows=1200 width=64)
                    (actual time=0.03..48.20 rows=98230 loops=1)
  Filter: (status = 'pending')
  Rows Removed by Filter: 401770
Planning Time: 0.15 ms
Execution Time: 55.10 ms
```

The planner believed `1200` rows; reality was `98230`. That **80× underestimate** is the single most valuable fact on the screen — it explains why the planner might elsewhere pick a Nested Loop that explodes. `Rows Removed by Filter` (only shown by `ANALYZE`) tells you the Seq Scan touched 500,000 rows to emit 98,230 — a filter selectivity problem screaming for a partial or targeted index.

### 6.3 BUFFERS — The I/O vs CPU Diagnostic

`BUFFERS` reports page accounting in units of 8KB blocks, split across three memory regions:

- **shared** — the main buffer pool (`shared_buffers`); regular tables & indexes.
- **local** — temp tables (session-private buffers).
- **temp** — short-lived data for sorts, hashes, materializes that spill.

Each region reports up to four verbs:

```
shared hit=H read=R dirtied=D written=W
       │      │      │         └── dirty pages this backend evicted to disk to free space
       │      │      └── pages modified by this statement (become dirty)
       │      └── pages NOT in shared_buffers; fetched from OS/disk (SLOW)
       └── pages found already in shared_buffers (FAST, in-RAM)
temp read=R written=W
     └── spilled sort/hash data read back / written out (indicates work_mem too small)
```

**The diagnosis rule:**

| Pattern | Meaning | Fix |
|---|---|---|
| High `shared read`, high time | **I/O-bound** — reading from disk | Add index, increase `shared_buffers`, fix bloat, warm cache |
| High `shared hit`, high time | **CPU-bound** — data in RAM but lots of work | Restructure query, better index, raise `work_mem` |
| `temp read/written` non-zero | Sort/hash **spilled to disk** | Raise `work_mem`, reduce data before sort |
| High `dirtied`/`written` on a SELECT | Hint bits / vacuum-on-read | Often benign; first read after bulk load |

Concrete example distinguishing the two:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM orders WHERE created_at >= '2025-01-01';
```
```
Aggregate  (cost=..)(actual time=812.4..812.4 rows=1 loops=1)
  Buffers: shared hit=32 read=88450
  ->  Seq Scan on orders  (actual time=0.4..690.1 rows=2104338 loops=1)
        Filter: (created_at >= '2025-01-01')
        Buffers: shared hit=32 read=88450
Execution Time: 813.0 ms
```

`read=88450` pages × 8KB ≈ **690 MB pulled from disk**. This is unambiguously **I/O-bound**. Compare after adding an index and re-running when cached:

```
Aggregate  (actual time=141.2..141.2 rows=1 loops=1)
  Buffers: shared hit=88482
Execution Time: 141.3 ms
```

Now `hit=88482, read=0` — everything served from RAM. The same 88K pages, but hits instead of reads: **5.7× faster** purely from I/O elimination. `BUFFERS` is what let you prove it was I/O, not CPU.

### 6.4 Reading the Buffers Output — Worked Detail

Buffer counts are **cumulative up the tree**: a parent node's `Buffers` line includes everything its children touched. To find where the I/O actually happened, look at the **leaf** with the largest `read`. Also note buffers appear per node, so in a join you can attribute reads to the specific scan.

```
Hash Join  (actual time=..) Buffers: shared hit=1200 read=45000
  ->  Seq Scan on order_items  Buffers: shared hit=40 read=44800   ← the culprit
  ->  Hash  Buffers: shared hit=1160 read=200
        ->  Seq Scan on products  Buffers: shared hit=1160 read=200
```

The join's `read=45000` is dominated by `order_items` (`read=44800`). That's where to add an index or a filter — not `products`.

### 6.5 loops and Per-Loop vs Total — The #1 Misreading

In a Nested Loop, the inner side is re-executed once per outer row. Its printed `actual time` and `actual rows` are **per-loop averages**, and you must multiply by `loops` to get the true cost.

```
Nested Loop  (actual time=0.09..4210.5 rows=500000 loops=1)
  ->  Seq Scan on orders o  (actual time=0.02..85.4 rows=500000 loops=1)
  ->  Index Scan using customers_pkey on customers c
        (actual time=0.006..0.007 rows=1 loops=500000)   ← per loop!
        Index Cond: (id = o.customer_id)
```

The inner Index Scan looks trivial: `0.007 ms`, `1 row`. But `loops=500000`. True inner cost ≈ `0.007 ms × 500000 = 3.5 seconds`. **That** is where the 4.2 seconds went — not the outer Seq Scan (85 ms). A junior reads "7 microseconds, fast" and misses it entirely. The signal that a Nested Loop is the problem is almost always a **large `loops` count on the inner node**.

Two derived numbers you compute by hand:
- **Total time of a node** ≈ `actual time (last-row) × loops`.
- **Total rows produced** = `actual rows × loops`.

PG shows per-loop rows rounded to the nearest integer, so `rows=0 loops=500000` can still mean hundreds of thousands of total rows (e.g., true average 0.4).

### 6.6 FORMAT — TEXT, JSON, YAML, XML

`TEXT` is for humans. The structured formats expose the same tree as nested objects with explicit numeric fields — ideal for tools.

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM orders WHERE id = 42;
```
```json
[
  {
    "Plan": {
      "Node Type": "Index Scan",
      "Relation Name": "orders",
      "Index Name": "orders_pkey",
      "Startup Cost": 0.43,
      "Total Cost": 8.45,
      "Plan Rows": 1,
      "Plan Width": 64,
      "Actual Startup Time": 0.031,
      "Actual Total Time": 0.033,
      "Actual Rows": 1,
      "Actual Loops": 1,
      "Shared Hit Blocks": 4,
      "Shared Read Blocks": 0
    },
    "Planning Time": 0.102,
    "Execution Time": 0.061
  }
]
```

`YAML` is the same data, indentation-structured; `XML` is verbose and rarely used. Use JSON when feeding `auto_explain` output to a parser or a plan visualizer. In `psql`, wrap with `\pset format` untouched — the JSON comes back as a single text column.

### 6.7 VERBOSE — Output Columns, Qualification, Workers

`VERBOSE` adds detail that's invaluable for wide plans:

```sql
EXPLAIN (VERBOSE, COSTS false)
SELECT c.name, count(*) FROM customers c JOIN orders o ON o.customer_id = c.id GROUP BY c.name;
```
```
HashAggregate
  Output: c.name, count(*)
  Group Key: c.name
  ->  Hash Join
        Output: c.name
        Hash Cond: (o.customer_id = c.id)
        ->  Seq Scan on public.orders o
              Output: o.customer_id
        ->  Hash
              Output: c.name, c.id
              ->  Seq Scan on public.customers c
                    Output: c.name, c.id
```

The `Output:` lines show exactly which columns each node materializes — critical for spotting an accidental `SELECT *` carrying a wide `width`, or a missing covering index. `VERBOSE` also schema-qualifies (`public.orders`), names triggers fired, and under parallel plans shows **per-worker** rows/time.

### 6.8 SETTINGS — Why Did It Choose This Plan?

`SETTINGS` (PG 12+) lists any planner GUC that differs from its default and thus influenced the plan.

```sql
SET work_mem = '256MB';
SET random_page_cost = 1.1;
EXPLAIN (ANALYZE, SETTINGS)
SELECT ...;
```
```
...
Settings: random_page_cost = '1.1', work_mem = '256MB'
```

This is the difference between "the plan is weird" and "the plan is weird **because** someone set `enable_nestloop=off` in this session." Always include `SETTINGS` when a plan on staging differs from production — the GUCs usually explain it.

### 6.9 WAL — Write Amplification for Modifying Statements

`WAL` (PG 13+) quantifies durability cost for `INSERT`/`UPDATE`/`DELETE`.

```sql
BEGIN;
EXPLAIN (ANALYZE, WAL, BUFFERS)
UPDATE orders SET status = 'archived' WHERE created_at < '2020-01-01';
ROLLBACK;
```
```
Update on orders  (actual time=..) rows=0 loops=1
  WAL: records=48210 fpi=1503 bytes=8123400
  ->  Seq Scan on orders ...
```

- **records** — WAL records generated (roughly per-row-modified plus index updates).
- **fpi** — **full-page images**: entire 8KB pages logged (happens on the first modification of a page after a checkpoint). High `fpi` after checkpoints explains WAL spikes and replication lag.
- **bytes** — total WAL volume. `8.1 MB` for this UPDATE. Multiply by your write rate to predict replication/archive bandwidth.

Note the `BEGIN; ... ROLLBACK;` wrapper — mandatory because `ANALYZE` executes the UPDATE for real. See 6.11.

### 6.10 TIMING, SUMMARY, GENERIC_PLAN, MEMORY

- **`TIMING off`** — keeps `ANALYZE`'s row/buffer measurement but drops per-node timers. Use on hardware with an expensive clock source (`clock_gettime`), where timing overhead itself distorts results — you still get total `Execution Time`. Verify your clock with the `pg_test_timing` utility.
- **`SUMMARY`** — controls the `Planning Time` / `Execution Time` footer. On automatically with `ANALYZE`; you can force it on for plain `EXPLAIN` (`SUMMARY on`) to see planning time without executing.
- **`GENERIC_PLAN`** (PG 16+) — plans a statement containing parameter placeholders (`$1`) **without executing**, showing the generic plan the server would cache for a prepared statement. Cannot be combined with `ANALYZE` (there are no values to run). Invaluable for diagnosing prepared-statement plan regressions without binding values.
- **`MEMORY`** (PG 17+) — reports planner memory (`used`/`allocated`). Relevant for queries with thousands of partitions where planning itself is expensive.

### 6.11 The ANALYZE-Executes Gotcha (The Most Important Subsection)

**`EXPLAIN ANALYZE` executes the statement.** For `SELECT` this is usually harmless (though a `SELECT ... FOR UPDATE` takes real locks, and a `SELECT` calling a volatile function has side effects). For **any data-modifying statement it is fully real**:

```sql
-- THIS DELETES ROWS. The word EXPLAIN does not protect you.
EXPLAIN ANALYZE DELETE FROM orders WHERE status = 'pending';
-- ^ 98,230 orders are gone. Triggers fired. Cascades cascaded.
```

The safe, universal pattern for measuring a write is a transaction you roll back:

```sql
BEGIN;
EXPLAIN (ANALYZE, BUFFERS, WAL)
DELETE FROM orders WHERE status = 'pending';   -- runs for real, inside the txn
ROLLBACK;                                        -- undoes it; measurements already captured
```

Because the plan and all measurements are produced *during* execution, `ROLLBACK` gives you the full `EXPLAIN ANALYZE` output while undoing every row change, trigger effect that's transactional, and WAL that's not yet committed. Caveats to state explicitly:

- **Non-transactional side effects are NOT undone by ROLLBACK**: sequence advances (`nextval`), `dblink`/FDW writes to remote systems, `NOTIFY`, filesystem effects from untrusted functions, and anything a trigger does outside the database.
- **Locks are held** for the duration of the transaction — an `EXPLAIN ANALYZE UPDATE` inside a long `BEGIN` can block other sessions until you `ROLLBACK`.
- **`RETURNING` still returns**, and `ON CONFLICT` still fires, during execution.
- For read-only safety on a whole session, `SET TRANSACTION READ ONLY` will make an accidental modifying `EXPLAIN ANALYZE` error out instead of executing.

The mental rule: **plain `EXPLAIN` is a prediction and always safe; `EXPLAIN ANALYZE` is an execution and safe only for pure reads. Wrap every write in `BEGIN … ROLLBACK`.**

---

## 7. EXPLAIN — Reading a Full Plan with Options

A realistic multi-node plan with every diagnostic flag turned on, then read line by line.

```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS, WAL)
SELECT c.name, sum(o.total_amount) AS revenue
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.created_at >= '2025-01-01'
GROUP BY c.name
ORDER BY revenue DESC
LIMIT 10;
```
```
Limit  (cost=48210.5..48210.6 rows=10 width=40)
       (actual time=612.3..612.4 rows=10 loops=1)
  Output: c.name, (sum(o.total_amount))
  Buffers: shared hit=204 read=88450
  ->  Sort  (cost=48210.5..48310.5 rows=40000 width=40)
            (actual time=612.3..612.3 rows=10 loops=1)
        Output: c.name, (sum(o.total_amount))
        Sort Key: (sum(o.total_amount)) DESC
        Sort Method: top-N heapsort  Memory: 27kB
        Buffers: shared hit=204 read=88450
        ->  HashAggregate  (cost=46800.0..47340.0 rows=40000 width=40)
                           (actual time=598.1..606.7 rows=38210 loops=1)
              Output: c.name, sum(o.total_amount)
              Group Key: c.name
              Batches: 1  Memory Usage: 4113kB
              Buffers: shared hit=204 read=88450
              ->  Hash Join  (cost=1350.0..44100.0 rows=520000 width=40)
                             (actual time=12.4..410.2 rows=515338 loops=1)
                    Output: c.name, o.total_amount
                    Hash Cond: (o.customer_id = c.id)
                    Buffers: shared hit=204 read=88450
                    ->  Seq Scan on public.orders o
                          (cost=0.00..40100.0 rows=520000 width=12)
                          (actual time=0.03..280.5 rows=515338 loops=1)
                          Output: o.customer_id, o.total_amount
                          Filter: (o.created_at >= '2025-01-01')
                          Rows Removed by Filter: 1584662
                          Buffers: shared hit=44 read=88406
                    ->  Hash  (cost=1100.0..1100.0 rows=20000 width=36)
                              (actual time=12.1..12.1 rows=20000 loops=1)
                          Output: c.name, c.id
                          Buckets: 32768  Batches: 1  Memory Usage: 1489kB
                          Buffers: shared hit=160 read=44
                          ->  Seq Scan on public.customers c
                                (cost=0.00..1100.0 rows=20000 width=36)
                                (actual time=0.01..4.2 rows=20000 loops=1)
                                Output: c.name, c.id
                                Buffers: shared hit=160 read=44
Settings: work_mem = '64MB'
Planning Time: 0.42 ms
Execution Time: 613.1 ms
```

**Line-by-line reading (bottom-up = execution order):**

1. **Seq Scan on customers** (leaf, runs first) — 20,000 rows in 4.2 ms, `hit=160 read=44`. Tiny, cached, irrelevant to performance.
2. **Hash** — builds a hash table from customers. `Batches: 1` means it fit in `work_mem` (no spill), `Memory Usage: 1489kB`. Good.
3. **Seq Scan on orders** — the hot spot. `read=88406` pages ≈ **690 MB from disk** → **I/O-bound**. `Rows Removed by Filter: 1584662`: it scanned 2.1M rows to keep 515K. A B-tree index on `orders(created_at)` (or a covering index) would replace this Seq Scan.
4. **Hash Join** — probes the customer hash with each order. `actual rows=515338`, estimate `520000`: **estimate is excellent here**, so the algorithm choice (Hash Join) is sound.
5. **HashAggregate** — groups by `c.name`. `Batches: 1 Memory Usage: 4113kB`: no spill, fits in the 64MB `work_mem` shown in `Settings`.
6. **Sort** — `top-N heapsort Memory: 27kB`: because of the `LIMIT 10`, Postgres uses a bounded top-N sort, not a full sort. Cheap.
7. **Limit** — returns 10 rows. Total `Execution Time: 613 ms`, essentially all of it in the orders Seq Scan I/O.

**Diagnosis:** one fix dominates — index `orders(created_at)` to eliminate the 690 MB read. `Settings` confirms `work_mem` is generous enough that nothing spilled. `WAL` line is absent because this is a read-only `SELECT`. `VERBOSE`'s `Output:` lines confirm we're not carrying unnecessary wide columns.

---

## 8. Query Examples

### Example 1 — Basic: Plan vs Execution

```sql
-- Plain EXPLAIN: prediction only, zero execution, always safe
EXPLAIN
SELECT * FROM users WHERE email = 'a@b.com';
-- Index Scan using users_email_key on users
--   (cost=0.29..8.31 rows=1 width=320)
--   Index Cond: (email = 'a@b.com')

-- Add ANALYZE: now it runs and reports reality
EXPLAIN (ANALYZE)
SELECT * FROM users WHERE email = 'a@b.com';
-- Index Scan using users_email_key on users
--   (cost=0.29..8.31 rows=1 width=320)
--   (actual time=0.028..0.029 rows=1 loops=1)
--   Index Cond: (email = 'a@b.com')
-- Planning Time: 0.09 ms
-- Execution Time: 0.05 ms
```

### Example 2 — Intermediate: I/O vs CPU with BUFFERS

```sql
-- Turn on BUFFERS to classify the bottleneck.
-- First run (cold cache) — watch shared read:
EXPLAIN (ANALYZE, BUFFERS)
SELECT category_id, count(*)
FROM products
GROUP BY category_id;
-- HashAggregate (actual time=95.1..95.3 rows=42 loops=1)
--   Buffers: shared hit=12 read=8330        ← 8330 pages from disk = I/O-bound
--   ->  Seq Scan on products (actual time=0.4..70.2 rows=500000 loops=1)
--         Buffers: shared hit=12 read=8330
-- Execution Time: 96.0 ms

-- Second run (warm cache) — same query, pages now resident:
-- HashAggregate (actual time=61.0..61.2 rows=42 loops=1)
--   Buffers: shared hit=8342 read=0          ← all hits = CPU-bound now
-- Execution Time: 61.4 ms
-- Interpretation: 35ms of the cold run was pure disk I/O.
```

### Example 3 — Production Grade: Measuring a Write Safely

```sql
-- Scenario: archive job UPDATEs ~50K rows nightly on a 2M-row orders table.
-- orders(id PK, customer_id, status, total_amount, created_at)
-- Indexes: orders_pkey, orders_created_at_idx, orders_status_idx
-- We must measure cost WITHOUT actually archiving during business hours.
-- Expectation: index scan on created_at, moderate WAL, no temp spill.

BEGIN;

EXPLAIN (ANALYZE, BUFFERS, WAL, VERBOSE, SETTINGS)
UPDATE orders
SET status = 'archived'
WHERE created_at < NOW() - INTERVAL '2 years'
  AND status = 'completed';

ROLLBACK;   -- ← undoes the UPDATE; measurements already captured above
```
```
Update on public.orders  (cost=..)(actual time=..821.4 rows=0 loops=1)
  Buffers: shared hit=51204 read=1840 dirtied=1502 written=310
  WAL: records=49880 fpi=1502 bytes=8340221
  ->  Index Scan using orders_created_at_idx on public.orders
        (actual time=0.05..142.6 rows=49880 loops=1)
        Index Cond: (created_at < (now() - '2 years'::interval))
        Filter: (status = 'completed')
        Rows Removed by Filter: 12010
        Buffers: shared hit=51204 read=1840
Settings: work_mem = '64MB'
Planning Time: 0.30 ms
Execution Time: 838.9 ms
```

Reading it: the Index Scan on `created_at` (not a Seq Scan) confirms the index is used. `dirtied=1502` pages were modified; `WAL: fpi=1502` — one full-page image per dirtied page (this ran right after a checkpoint). `bytes=8.3MB` of WAL is the replication cost per run. No `temp` buffers → no spill. The `ROLLBACK` means production data is untouched. This is exactly how you validate a migration's cost before shipping it.

---

## 9. Wrong → Right Patterns

### Wrong 1: Running EXPLAIN ANALYZE on a modifying statement

```sql
-- WRONG: this DELETES the rows. "EXPLAIN" is not a dry run for ANALYZE.
EXPLAIN ANALYZE
DELETE FROM sessions WHERE expires_at < NOW();
-- RESULT: rows are permanently deleted. Triggers fired. Cascades ran.
-- The plan output is correct — but so was the deletion.
```

**Why it's wrong at the execution level:** `ANALYZE` instructs the executor to *run* the plan and instrument it. There is no separate "dry-run executor." The measurements (`actual rows`, timing, WAL) can only exist because the DELETE actually happened.

```sql
-- RIGHT: wrap in a transaction and roll back
BEGIN;
EXPLAIN (ANALYZE, BUFFERS)
DELETE FROM sessions WHERE expires_at < NOW();
ROLLBACK;   -- rows restored; you still saw the full plan + measurements
```

### Wrong 2: Trusting plain EXPLAIN's row estimate as fact

```sql
-- WRONG: concluding "only 12 rows, a Nested Loop is fine"
EXPLAIN
SELECT * FROM orders o JOIN order_items oi ON oi.order_id = o.id
WHERE o.customer_id = 998;
-- Nested Loop (cost=.. rows=12 ..)   ← estimate. Might be wildly off.
```

**Why it's wrong:** plain `EXPLAIN` shows the planner's *estimate* from `pg_statistic`. If statistics are stale (customer 998 is a wholesale buyer with 40,000 items), the estimate of 12 is fantasy, yet the plan output looks authoritative.

```sql
-- RIGHT: ANALYZE reveals the estimate-vs-actual gap
EXPLAIN (ANALYZE)
SELECT * FROM orders o JOIN order_items oi ON oi.order_id = o.id
WHERE o.customer_id = 998;
-- Nested Loop (cost=.. rows=12 ..)(actual .. rows=40218 loops=1)
--   ← estimate 12, actual 40218: 3350× off. Run ANALYZE (the command)
--     on the table, or expect a bad plan choice here.
```

### Wrong 3: Misreading per-loop numbers as total

```sql
-- WRONG conclusion: "the inner Index Scan is 0.006ms, it's fine"
-- Nested Loop (actual time=..3900.0 rows=800000 loops=1)
--   ->  Seq Scan on orders (actual time=..40.1 rows=800000 loops=1)
--   ->  Index Scan on customers (actual time=0.005..0.006 rows=1 loops=800000)
```

**Why it's wrong:** the inner node's `0.006 ms` is **per loop**, and `loops=800000`. True inner cost ≈ `0.006 × 800000 = 4.8 s`. The bottleneck IS the inner scan being run 800K times — the fix is to avoid the Nested Loop (force a Hash Join by making the join more selective, or add a filter to shrink the outer side), not to "optimize the fast index scan."

```sql
-- RIGHT: read total = per-loop × loops, then attack the loop count
-- Recognize loops=800000 as the smell; a Hash Join here would be O(N+M) once,
-- not 800K index probes. Check why the planner underestimated the outer side.
```

### Wrong 4: Forgetting BUFFERS and guessing the bottleneck

```sql
-- WRONG: "query is slow, must need more work_mem" — pure guess
EXPLAIN ANALYZE SELECT ... ;   -- no BUFFERS
-- Execution Time: 2100 ms   ← you cannot tell WHY from timing alone
```

**Why it's wrong:** without `BUFFERS` you cannot distinguish a query that read 2 GB from disk (needs an index / cache) from one that spilled a sort (needs `work_mem`) from one burning CPU in a nested loop. All three show only "2100 ms."

```sql
-- RIGHT: BUFFERS makes the cause visible
EXPLAIN (ANALYZE, BUFFERS) SELECT ... ;
-- Buffers: shared read=260000 temp written=40000
--   → shared read huge = I/O-bound (index it);
--     temp written = sort spilled (raise work_mem). Now you know.
```

### Wrong 5: Comparing plans across environments without SETTINGS

```sql
-- WRONG: "same query, staging uses an index, prod does a Seq Scan — Postgres bug!"
EXPLAIN SELECT * FROM orders WHERE customer_id = 5 ORDER BY created_at;
```

**Why it's wrong:** the plan difference is almost always a GUC difference (`work_mem`, `random_page_cost`, `enable_indexscan`, or an ORM setting `SET`ting session parameters), not a bug. Without `SETTINGS` you chase a phantom.

```sql
-- RIGHT: SETTINGS surfaces the non-default GUCs on each box
EXPLAIN (ANALYZE, SETTINGS) SELECT ... ;
-- Settings: random_page_cost = '4' (prod)  vs  '1.1' (staging, SSD-tuned)
--   → prod thinks random I/O is 4× costlier, so it avoids the index. Mystery solved.
```

---

## 10. Performance Profile

### Overhead of the options themselves

| Option | Overhead | Notes |
|---|---|---|
| plain `EXPLAIN` | ~planning cost only | No execution; microseconds to low ms |
| `ANALYZE` | runs the query + instrumentation | You pay full execution time |
| `TIMING` (default on w/ ANALYZE) | `clock_gettime` per node per row | Can add 10–30%+ on slow clocks or tight loops |
| `BUFFERS` | negligible | Counters already maintained; essentially free |
| `WAL` | negligible | Reads existing WAL counters |
| `VERBOSE` / `SETTINGS` | negligible | Formatting only |
| `FORMAT JSON` | negligible | Formatting only |

**The TIMING trap at scale.** On a plan with a node executed millions of times (`loops` in the millions), the two `clock_gettime` calls per row can dominate and make `EXPLAIN ANALYZE` report a time *longer* than the real query. If `Execution Time` under `ANALYZE` is much higher than the query's real latency, re-run with `TIMING off` — you keep row counts and buffers, lose per-node timers, and remove the distortion. Check the clock cost with `pg_test_timing`.

### Scaling: what EXPLAIN reveals at 1M / 10M / 100M rows

- **1M rows.** A Seq Scan reads ~8K–90K pages depending on width — often still fast from cache. `BUFFERS` shows whether it's resident. Bad estimates rarely hurt yet.
- **10M rows.** Nested Loops with high `loops` become visibly catastrophic (seconds). `shared read` in the hundreds of thousands means real disk pressure. Sorts start spilling (`temp written` appears) unless `work_mem` scales. This is where reading EXPLAIN pays off most.
- **100M rows.** Estimate errors compound across joins — a 10× error at the leaf becomes a 1000× error three joins up, and the planner picks a Nested Loop that never finishes. `BUFFERS` distinguishes "reading 800 GB from disk" (partition/index the table) from "CPU in an aggregate." Parallel plans appear; `VERBOSE` per-worker rows show whether work is balanced. Planning time itself grows (use `MEMORY` / `GENERIC_PLAN`).

### Optimization workflow driven by EXPLAIN options

1. Run `EXPLAIN (ANALYZE, BUFFERS)`. First question: **I/O-bound or CPU-bound?** (`shared read` vs `shared hit`).
2. Find the node with the largest `actual total time × loops` and the largest `read`. That leaf is your target.
3. Check **estimate vs actual** on that node. Big gap → `ANALYZE tablename;`, raise statistics target, or add extended statistics.
4. Look for **`temp` buffers** → a spill → raise `work_mem` or reduce rows before the sort/hash.
5. Look for **high `loops`** on an inner Nested Loop node → make the join selective or nudge toward Hash Join.
6. Re-measure. The `Execution Time` and `Buffers` lines are your before/after scoreboard.

### Index interactions visible only in the plan

`Index Only Scan` with a non-zero `Heap Fetches` (shown by `ANALYZE`) means the visibility map isn't all-visible — `VACUUM` the table to make the scan truly index-only. `BUFFERS` will show the extra heap page reads. This is a class of problem you literally cannot see without `EXPLAIN (ANALYZE, BUFFERS)`.

---

## 11. Node.js Integration

`EXPLAIN` is just a statement prefix — `pool.query()` runs it like any other SQL, and the plan comes back as result rows.

### 11.1 Capturing a plan as JSON

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function getPlan(sql, params = []) {
  // FORMAT JSON returns one row, one column ("QUERY PLAN"), holding a JS array
  const explainSql = `EXPLAIN (FORMAT JSON) ${sql}`;
  const { rows } = await pool.query(explainSql, params);
  return rows[0]['QUERY PLAN'][0]; // the root plan object
}

const plan = await getPlan(
  'SELECT * FROM orders WHERE customer_id = $1',
  [42]
);
console.log(plan.Plan['Node Type'], plan.Plan['Total Cost']);
```

Note `EXPLAIN (FORMAT JSON)` (no `ANALYZE`) is safe with any statement — it does not execute. The pg driver parses the JSON column automatically into a JS object.

### 11.2 Safely measuring a write with ANALYZE + ROLLBACK

```javascript
// Measure a modifying statement without persisting it. Use a dedicated client
// so BEGIN/ROLLBACK stay on one connection (pool.query may use different ones).
async function explainAnalyzeWrite(sql, params = []) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const { rows } = await client.query(
      `EXPLAIN (ANALYZE, BUFFERS, WAL, FORMAT JSON) ${sql}`,
      params
    );
    return rows[0]['QUERY PLAN'][0];
  } finally {
    await client.query('ROLLBACK'); // undo the executed write, always
    client.release();
  }
}

const plan = await explainAnalyzeWrite(
  `DELETE FROM sessions WHERE expires_at < NOW()`
);
console.log('would have deleted rows:',
  plan.Plan['Actual Rows'],
  'WAL bytes:', plan.Plan['WAL Bytes']);
```

The `ROLLBACK` in `finally` is the safety interlock: even if reading the plan throws, the write is undone.

### 11.3 A slow-query guard: warn on Seq Scans over a threshold

```javascript
// Lightweight CI/dev check: fail if a query's plan does a big Seq Scan
async function assertNoBigSeqScan(sql, params = [], maxCost = 10000) {
  const root = (await pool.query(
    `EXPLAIN (FORMAT JSON) ${sql}`, params
  )).rows[0]['QUERY PLAN'][0].Plan;

  const walk = (node) => {
    if (node['Node Type'] === 'Seq Scan' && node['Total Cost'] > maxCost) {
      throw new Error(
        `Expensive Seq Scan on ${node['Relation Name']} ` +
        `(cost ${node['Total Cost']}) — add an index?`
      );
    }
    (node.Plans || []).forEach(walk);
  };
  walk(root);
}
```

### 11.4 Never build EXPLAIN by string-concatenating user input

```javascript
// You CANNOT parameterize the statement text of EXPLAIN itself, but you CAN
// (and must) parameterize the values inside the wrapped query with $1, $2, ...
// WRONG: `EXPLAIN SELECT ... WHERE id = ${userInput}`  → SQL injection
// RIGHT: pass params through to the inner query placeholders:
await pool.query('EXPLAIN (ANALYZE) SELECT * FROM users WHERE id = $1', [userInput]);
```

**Do ORMs handle EXPLAIN?** Mostly no, or only partially. Prisma, Sequelize, TypeORM, and Knex expose raw-query escape hatches you use to prepend `EXPLAIN (...)`. None generate `EXPLAIN` from their fluent API, and none automatically wrap `ANALYZE` writes in a transaction — that safety is on you. See Section 12.

---

## 12. ORM Comparison

The key question for every ORM: **can I get the plan for the exact SQL the ORM generated?** The universal escape hatch is a raw query with an `EXPLAIN (...)` prefix. The hard part is capturing the *ORM's* generated SQL (with correctly bound params) rather than hand-writing it.

### Prisma

**Can it?** Partially. Prisma has no `.explain()` on the query API. You either use `$queryRaw` with a hand-written `EXPLAIN`, or turn on query logging to capture the generated SQL and explain it separately.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient({ log: ['query'] }); // logs generated SQL + params

// Explain a raw statement (FORMAT JSON → returns rows of JSON)
const plan = await prisma.$queryRawUnsafe(
  `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
   SELECT * FROM orders WHERE customer_id = $1`,
  42
);
console.log((plan as any)[0]['QUERY PLAN'][0].Plan['Node Type']);
```

**Where it breaks:** you cannot pass a Prisma `findMany({...})` object to EXPLAIN. You must copy the SQL Prisma logged (which includes Prisma's own parameter placeholders) and explain that. For safe `ANALYZE` on writes, wrap manually: `prisma.$transaction([...])` won't roll back automatically for measurement — use interactive `prisma.$transaction(async tx => { ...; throw })` and catch, or a raw `BEGIN/ROLLBACK`.

**Verdict:** raw-SQL only. Good enough for spot-checking; use the query log to get exact SQL.

### Drizzle ORM

**Can it?** Yes — Drizzle is the best here. `.toSQL()` gives you the exact SQL + params from a built query, so you can explain precisely what runs.

```typescript
import { db } from './db';
import { orders } from './schema';
import { eq, sql } from 'drizzle-orm';

const q = db.select().from(orders).where(eq(orders.customerId, 42));
const { sql: text, params } = q.toSQL();       // exact generated SQL + params

const plan = await db.execute(
  sql.raw(`EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${text}`)
  // bind params via db.execute with placeholders in real code
);
```

**Where it breaks:** you still assemble the `EXPLAIN` string yourself; no first-class `.explain()`. But `.toSQL()` removes all guesswork about what SQL is generated.

**Verdict:** best-in-class for plan capture thanks to `.toSQL()`.

### Sequelize

**Can it?** Partially, via the `benchmark`/logging options and raw queries. No native `.explain()`.

```javascript
const { QueryTypes } = require('sequelize');
// Capture generated SQL via logging, then explain a raw statement:
const plan = await sequelize.query(
  `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
   SELECT * FROM orders WHERE customer_id = :cid`,
  { replacements: { cid: 42 }, type: QueryTypes.SELECT }
);
console.log(plan[0]['QUERY PLAN'][0].Plan['Total Cost']);
```

**Where it breaks:** Sequelize's generated SQL (from `findAll` with includes) is often complex; capturing it requires the `logging` callback. No auto transaction wrapping for `ANALYZE` writes — pass `{ transaction: t }` and roll back yourself.

**Verdict:** raw query escape hatch works; use `logging` to see the real SQL.

### TypeORM

**Can it?** Yes — uniquely, TypeORM's QueryBuilder has `.getQueryAndParameters()`, and you can run `EXPLAIN` via `.query()`.

```typescript
const qb = dataSource.getRepository(Order)
  .createQueryBuilder('o')
  .where('o.customer_id = :cid', { cid: 42 });

const [text, params] = qb.getQueryAndParameters(); // exact SQL + params array
const plan = await dataSource.query(
  `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${text}`, params
);
console.log(plan[0]['QUERY PLAN'][0].Plan['Node Type']);
```

**Where it breaks:** entity-manager methods (`find`, `findOne`) don't expose their SQL as cleanly as QueryBuilder; prefer QueryBuilder when you need to explain. Wrap `ANALYZE` writes with `dataSource.transaction` and throw to roll back.

**Verdict:** good — `getQueryAndParameters()` plus `.query()` is a clean path.

### Knex.js

**Can it?** Yes, and it's the most transparent. `.toSQL()` / `.toString()` expose the SQL, and you can prefix `EXPLAIN` on a raw call.

```javascript
const q = knex('orders').where('customer_id', 42).select('*');
const { sql, bindings } = q.toSQL();  // exact SQL + bindings

const plan = await knex.raw(
  `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sql}`, bindings
);
console.log(plan.rows[0]['QUERY PLAN'][0].Plan['Total Cost']);
```

**Where it breaks:** nothing structural — Knex is a thin builder. For `ANALYZE` writes, use `knex.transaction(async trx => { ...; throw })` to roll back.

**Verdict:** most SQL-transparent; `.toSQL()` + `knex.raw('EXPLAIN ...')` is idiomatic.

### ORM Summary Table

| ORM | Native EXPLAIN? | Get exact SQL | ANALYZE-write safety | Verdict |
|---|---|---|---|---|
| Prisma | No | `log: ['query']` | Manual `$transaction`+throw | Raw SQL only |
| Drizzle | No | `.toSQL()` ✓ | Manual txn | Best capture via `.toSQL()` |
| Sequelize | No | `logging` cb | `{ transaction }` + rollback | Raw query works |
| TypeORM | No | `.getQueryAndParameters()` ✓ | `dataSource.transaction`+throw | Clean via QueryBuilder |
| Knex | No | `.toSQL()` ✓ | `knex.transaction`+throw | Most transparent |

**Universal rule:** no ORM auto-wraps `EXPLAIN ANALYZE` on writes in a rollback. You must do it. And no ORM generates `EXPLAIN` from the fluent API — always a raw escape hatch.

---

## 13. Practice Exercises

### Exercise 1 — Easy

You run this and see the output below:

```sql
EXPLAIN
SELECT * FROM products WHERE category_id = 7;
```
```
Seq Scan on products  (cost=0.00..9800.00 rows=340 width=88)
  Filter: (category_id = 7)
```

1. Did this query execute? How can you tell from the output alone?
2. What does `rows=340` represent, and where did that number come from?
3. Rewrite the command so it reports the *actual* number of rows and the execution time.

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines Topic 11 JOINs)

Given `orders` and `customers` (recall the Hash Join reading from Topic 11), you capture:

```
Hash Join  (cost=..)(actual time=8.1..920.4 rows=515000 loops=1)
  Buffers: shared hit=210 read=90000
  ->  Seq Scan on orders o  (actual time=..810.2 rows=515000 loops=1)
        Buffers: shared hit=40 read=89960
  ->  Hash  ...
        ->  Seq Scan on customers c  (actual .. rows=20000 loops=1)
              Buffers: shared hit=170 read=40
```

1. Is this query I/O-bound or CPU-bound? Cite the exact numbers that prove it.
2. Which scan is responsible for almost all the cost, and what single change would you test first?
3. The `BUFFERS` option isn't shown as being requested in the snippet — what full `EXPLAIN (...)` command produced this, and would it have executed the query?

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

A teammate wants to know how expensive this cleanup is before running it in prod:

```sql
DELETE FROM audit_logs WHERE created_at < NOW() - INTERVAL '1 year';
```

They run `EXPLAIN ANALYZE DELETE FROM audit_logs WHERE created_at < NOW() - INTERVAL '1 year';` on the **production** database to "just check the plan."

1. What exactly happened to the production `audit_logs` table? Why?
2. Write the correct command sequence that measures the plan, buffers, and WAL cost **without** deleting anything.
3. Even with your safe wrapper, name two side effects that a `ROLLBACK` would **not** undo.
4. The table has 200M rows. What in the plan output would tell you whether the DELETE is I/O-bound, and what would tell you the WAL/replication impact?

```sql
-- Write your query here
```

### Exercise 4 — Interview Simulation

An interviewer shows you:

```
Nested Loop  (cost=0.43..251000.0 rows=9 width=40)
             (actual time=0.06..8400.2 rows=9 loops=1)
  ->  Seq Scan on orders o  (actual time=0.02..410.5 rows=1000000 loops=1)
        Filter: (status = 'flagged')
        Rows Removed by Filter: 0
  ->  Index Scan using customers_pkey on customers c
        (actual time=0.007..0.008 rows=0 loops=1000000)
        Index Cond: (id = o.customer_id)
```

Explain, out loud, (a) why this query is slow despite returning only 9 rows, (b) what the planner got wrong, and (c) two concrete fixes. Then write the `EXPLAIN` command you'd run next to confirm your hypothesis about I/O.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1 — What is the difference between `EXPLAIN` and `EXPLAIN ANALYZE`, and what's the danger?

**Junior answer:** "`EXPLAIN` shows the plan and `EXPLAIN ANALYZE` shows more detail like timing."

**Principal answer:** "Plain `EXPLAIN` asks the planner for the chosen plan and prints *estimated* costs and row counts from `pg_statistic` — nothing executes, so it's always safe. `EXPLAIN ANALYZE` **actually executes the statement**, instruments every node, and prints *actual* time, rows, and loops alongside the estimates, plus the `Planning Time`/`Execution Time` footer. The critical operational danger: because it executes, running `EXPLAIN ANALYZE` on an `INSERT`/`UPDATE`/`DELETE`/`MERGE` performs the modification for real — fires triggers, cascades, advances sequences. The safe pattern for measuring a write is `BEGIN; EXPLAIN (ANALYZE, BUFFERS, WAL) <write>; ROLLBACK;` — the plan and measurements are captured during execution, and the rollback undoes the transactional effects."

**Interviewer follow-up:** "Your ROLLBACK undid the DELETE — is everything undone?" → *No: `nextval` sequence advances, `NOTIFY`, FDW/dblink writes to remote systems, and non-transactional side effects inside functions persist. Locks were also held for the transaction's duration.*

### Q2 — How do you tell whether a slow query is I/O-bound or CPU-bound?

**Junior answer:** "Look at the execution time, and maybe add more RAM or a faster CPU."

**Principal answer:** "`Execution Time` alone can't distinguish them — you add `BUFFERS`. `EXPLAIN (ANALYZE, BUFFERS)` reports, per node, `shared hit` (pages served from `shared_buffers`, i.e., RAM — fast) versus `shared read` (pages fetched from the OS/disk — slow). A large `shared read` with high time means **I/O-bound**: the fix is an index to touch fewer pages, more cache, or fixing bloat. A large `shared hit` with high time means the data's already in RAM and the time is CPU — a nested loop, a sort, an aggregate — so you **CPU-bound** fixes: restructure the query, add a better index, or raise `work_mem`. Separately, non-zero `temp read/written` means a sort or hash spilled to disk — raise `work_mem`. I locate the specific culprit by finding the leaf node with the largest `read`, since buffer counts are cumulative up the tree."

**Interviewer follow-up:** "The same query is slow cold but fast warm — what does that prove?" → *It was I/O-bound; the cold run's `shared read` became `shared hit` once pages were cached. The delta is your disk-I/O cost.*

### Q3 — Explain `loops` in a plan and the mistake people make reading it.

**Junior answer:** "Loops is how many rows there are."

**Principal answer:** "`loops` is how many times a node was executed — in a Nested Loop, the inner node runs once per outer row, so `loops` equals the outer row count. The trap is that a node's printed `actual time` and `actual rows` are **per-loop averages**, not totals. So `Index Scan ... (actual time=0.006..0.007 rows=1 loops=500000)` is not 7 microseconds of work — it's `0.007 ms × 500000 ≈ 3.5 seconds`, and it produced `1 × 500000 = 500000` rows total. The single most common EXPLAIN misreading is seeing a small per-loop time on the inner node and declaring it fine, missing that a huge `loops` count makes it the bottleneck. When I see a large `loops` on a Nested Loop inner node, that's the signal to check whether a Hash Join (O(N+M) once) would beat 500K index probes, which usually means the planner underestimated the outer side."

**Interviewer follow-up:** "How would you push the planner off the Nested Loop?" → *Fix the row estimate (ANALYZE the table, raise statistics target / extended statistics), make the outer filter more selective, or, as a diagnostic, `SET enable_nestloop = off` and compare — then include `SETTINGS` so the change is visible in the plan.*

### Q4 — What do `BUFFERS`, `WAL`, and `SETTINGS` each add, and when do you reach for them?

**Junior answer:** "They add more information to the output."

**Principal answer:** "`BUFFERS` (default on in PG 18, previously required requesting and `ANALYZE`) gives page accounting — `shared/local/temp` `hit/read/dirtied/written` — the primary I/O-vs-CPU diagnostic and the way to spot sort/hash spills via `temp`. `WAL` (PG 13+) reports `records`, `fpi` (full-page images), and `bytes` for modifying statements — I use it to predict replication lag and archive volume, and `fpi` spikes tell me a write hit pages fresh after a checkpoint. `SETTINGS` (PG 12+) lists non-default planner GUCs (`work_mem`, `random_page_cost`, `enable_*`) that shaped the plan — indispensable when a plan differs between staging and production, because the cause is almost always a GUC difference, not a bug. My default diagnostic command is `EXPLAIN (ANALYZE, BUFFERS, SETTINGS)`, adding `WAL` for writes and `VERBOSE` for wide/parallel plans."

**Interviewer follow-up:** "When would you turn `TIMING off`?" → *On systems with an expensive clock source, or plans with millions of loops where the two `clock_gettime` calls per row dominate and inflate `Execution Time`; you keep rows and buffers, drop per-node timers. Verify clock cost with `pg_test_timing`.*

---

## 15. Mental Model Checkpoint

1. You run `EXPLAIN UPDATE orders SET status='x' WHERE id=1;` (no `ANALYZE`). Did any row change? Now you run `EXPLAIN ANALYZE UPDATE ...` — did any row change? What single wrapper makes the second one safe, and what does it fail to undo?

2. A plan node reads `(actual time=0.05..0.09 rows=2 loops=300000)`. How much total time did this node consume, and how many rows did it emit in total? Why can't you just read "0.09 ms" off the line?

3. Two queries both report `Execution Time: 1800 ms`. One shows `Buffers: shared read=220000`, the other `Buffers: shared hit=220000 temp written=15000`. Which is I/O-bound, which needs more `work_mem`, and how did you decide?

4. You see `WAL: records=60000 fpi=1500 bytes=9MB` on an UPDATE right after a checkpoint, then a much lower `fpi` when you re-run it minutes later. Explain the `fpi` difference and why it matters for replication.

5. A `SELECT` runs an `Index Only Scan` but `ANALYZE` shows `Heap Fetches: 480000`. Why isn't it truly index-only, and what maintenance operation would reduce the heap fetches?

6. On staging a query uses an Index Scan; on production the identical query uses a Seq Scan. You suspect it's not a bug. Which single `EXPLAIN` option would most likely reveal the cause, and name two GUCs that commonly differ.

7. Why does `EXPLAIN (ANALYZE)` sometimes report a *longer* `Execution Time` than the query's real latency in your application, and what option removes that distortion without losing row/buffer data?

---

## 16. Quick Reference Card

```sql
-- SYNTAX (modern, parenthesized — the only form that reaches BUFFERS/WAL/FORMAT)
EXPLAIN (option [, ...]) statement;

-- THE OPTIONS
ANALYZE      -- EXECUTE + measure actual time/rows. DANGER: runs writes for real
BUFFERS      -- shared/local/temp hit/read/dirtied/written  (I/O vs CPU); default ON in PG18
WAL          -- records / fpi / bytes  (write statements; PG13+)
VERBOSE      -- Output: columns, schema-qualified names, per-worker, triggers
SETTINGS     -- non-default planner GUCs that shaped the plan (PG12+)
TIMING       -- per-node timers (default ON w/ ANALYZE; TIMING off if clock is slow)
SUMMARY      -- Planning Time / Execution Time footer
COSTS        -- cost=..  rows=..  width=..  (default ON; COSTS false for clean diffs)
GENERIC_PLAN -- plan a $1/$2 query WITHOUT executing (PG16+; not with ANALYZE)
MEMORY       -- planner memory used/allocated (PG17+)
FORMAT       -- TEXT (humans) | JSON | YAML | XML (tools)

-- READING A NODE
Node  (cost=STARTUP..TOTAL rows=EST width=B)(actual time=FIRST..LAST rows=ACT loops=N)
--   total time  ≈ LAST * loops        total rows = ACT * loops   (PER-LOOP numbers!)
--   cost is abstract page-units, NOT milliseconds

-- I/O vs CPU (the core diagnosis)
shared read  big + slow  → I/O-bound   → index / cache / fix bloat
shared hit   big + slow  → CPU-bound   → restructure / better index / work_mem
temp written non-zero    → sort/hash spilled → raise work_mem
high loops on NL inner   → avoid Nested Loop → fix estimate / force Hash Join

-- SAFE MEASUREMENT OF A WRITE  (mandatory)
BEGIN;
EXPLAIN (ANALYZE, BUFFERS, WAL) DELETE FROM t WHERE ...;
ROLLBACK;   -- undoes rows; sequences/NOTIFY/FDW writes/locks-during NOT undone

-- MY DEFAULT COMMANDS
EXPLAIN (ANALYZE, BUFFERS, SETTINGS) SELECT ...;      -- reads
EXPLAIN (ANALYZE, BUFFERS, WAL) <write inside BEGIN/ROLLBACK>;  -- writes
EXPLAIN (FORMAT JSON) SELECT ...;                      -- safe, no execution, for tools
```

**Interview one-liners:**
- "Plain EXPLAIN predicts; EXPLAIN ANALYZE executes — wrap writes in BEGIN/ROLLBACK."
- "BUFFERS separates I/O-bound (`read`) from CPU-bound (`hit`); nothing else does."
- "Per-loop numbers × `loops` = the truth; a big `loops` is the Nested Loop smell."
- "`temp` buffers = a spill = raise `work_mem`."
- "Plans differ across environments? `SETTINGS` — it's a GUC, not a bug."
- "`WAL fpi` spikes after checkpoints drive replication lag."

---

## Connected Topics

- **Topic 11 — INNER JOIN in Depth**: the Hash/Nested Loop/Merge Join reading in Section 7 builds directly on the join-algorithm foundations there.
- **Topic 17 — JOIN Performance Deep Dive**: EXPLAIN is the instrument you use to diagnose which join algorithm the planner chose and why.
- **Topic 67 — Materialized Views** (previous): `EXPLAIN ANALYZE` on the `REFRESH` and on queries against the matview tells you whether the pre-computation is actually paying off.
- **Topic 69 — pg_stat_statements** (next): where EXPLAIN inspects one query in depth, `pg_stat_statements` aggregates *all* queries over time — you use it to find the slow query, then EXPLAIN to fix it.
- **Internals — The Cost-Based Planner**: `seq_page_cost`, `random_page_cost`, `cpu_*_cost` and `pg_statistic` are the machinery whose output plain EXPLAIN prints.
- **Internals — shared_buffers & the Page Cache**: the hit/read distinction `BUFFERS` reports is a direct window into the buffer pool.
- **Internals — WAL & MVCC**: the `WAL` option and `Heap Fetches`/visibility-map behavior are MVCC and write-ahead-logging leaking into the plan.
- **Tooling — auto_explain**: logs plans (optionally with `ANALYZE`, `BUFFERS`) for slow statements in production using exactly these options in `FORMAT JSON`.

