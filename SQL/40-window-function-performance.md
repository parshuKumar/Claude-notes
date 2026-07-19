# Topic 40 — Window Function Performance
### SQL Mastery Curriculum — Phase 6: Window Functions — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a big regional bake sale. Thousands of pies are laid out on one enormous table. Your boss asks: "For every pie, tell me its rank within its own county by score, and also the running total of revenue as we walk down the table."

You cannot answer either question by looking at a single pie. Rank-within-county needs the pies **grouped by county** and **sorted by score** so you can count "1st, 2nd, 3rd…". The running total needs the pies **in some order** so "running" means something.

So before you can write a single number down, you physically **rearrange the whole table**: first you shove all the county-A pies together, then county-B, then within each county you line them up from highest score to lowest. Only after everything is stacked in the right order do you start walking down the rows, keeping a little counter in your head ("this is the 3rd pie in county-B, running revenue is now $412").

That rearranging step is the expensive part. If someone had *already* handed you the pies pre-stacked by county-then-score (say they came out of the oven that way), you skip the whole shuffle and just walk down counting. That "already stacked" shortcut is exactly what an index gives a window function.

And if there are so many pies they don't fit on the table, you start stacking overflow pies on the **floor** (disk). Walking down a table is fast; bending down to the floor for every county is slow. That floor-spill is `work_mem` overflowing to a temp file.

So the entire performance story of window functions is three sentences:
1. **The sort is the cost** — almost always, the window's price is the Sort feeding it.
2. **An index can pre-sort** — and delete the Sort entirely.
3. **If it doesn't fit in `work_mem`, it spills to disk** — and gets much slower.

Everything below is detail on those three sentences.

---

## 2. Connection to SQL Internals

A window function is executed by a dedicated plan node called **`WindowAgg`** (window aggregate). It is not a scan, not a join, not a `GroupAggregate` — it is its own executor node with its own cost model. Understanding its performance means understanding what that node does mechanically.

**The core mechanic: WindowAgg consumes rows already in partition+order sequence.** `WindowAgg` itself does **not** sort. It *requires* its input to arrive grouped by the `PARTITION BY` columns and ordered by the `ORDER BY` columns. It gets that ordering from exactly one of two places:

1. A **`Sort`** node placed directly beneath it by the planner (the common case), or
2. An **`Index Scan`** (or Index-Only Scan) whose output order already matches `PARTITION BY … ORDER BY …` (the fast case — no Sort needed).

Internally the node keeps a **tuplestore** (an in-memory buffer that overflows to a temp file on disk) holding the current partition's rows, because frame-based functions (`SUM(...) OVER (... ROWS BETWEEN ...)`) and functions like `LAG`/`LEAD`/`LAST_VALUE` need to look **backward and forward** across rows in the same partition. That tuplestore is sized by `work_mem`. When a partition's rows exceed `work_mem`, the tuplestore **spills to a temporary file** in `base/pgsql_tmp` — the single biggest cliff in window performance.

The internals you must name:
- **`Sort` node + `work_mem`**: the sort feeding the window is an external merge sort if it exceeds `work_mem`; you'll see `Sort Method: external merge Disk: NkB`.
- **`WindowAgg` node + tuplestore + `work_mem`**: the per-partition frame buffer, which also spills.
- **B-tree index ordering**: a B-tree stores keys in sorted order, so an Index Scan yields rows pre-sorted — this is how you eliminate the Sort.
- **Buffer pool / heap fetches**: an Index Scan that isn't index-only still visits the heap for each row; a wide `SELECT *` window query pays heap-fetch cost on top.
- **`incremental sort`**: when input is *partially* pre-sorted (index gives partition order but not the full order key), PostgreSQL 13+ can do a cheaper Incremental Sort instead of a full Sort.

The planner cost model treats `WindowAgg` as roughly `input_cost + cpu_operator_cost × rows` for simple ranking functions, plus additional per-row work for frames that must re-scan a moving window. The dominant term in almost every real plan, though, is the Sort beneath it: `O(N log N)` CPU plus disk I/O if it spills.

---

## 3. Logical Execution Order Context

```
FROM / JOIN          ← rows assembled
WHERE                ← row filtering (BEFORE window sees anything)
GROUP BY             ← grouping/aggregation
HAVING               ← group filtering
window functions     ← WindowAgg runs HERE  ◄── this topic
SELECT (projection)
DISTINCT
ORDER BY             ← final presentation order (separate from window ORDER BY)
LIMIT / OFFSET
```

Two performance-critical consequences fall out of this ordering:

**1. `WHERE` runs before the window — so filter first to shrink the sort.** The single most effective window optimisation is reducing the row count *before* `WindowAgg`. A `WHERE created_at >= '2025-01-01'` cutting 100M rows to 2M turns a giant external-merge sort into a small in-memory one. You cannot filter on the window's *output* in the same query level (that requires a subquery/CTE — see Topic 41), but you can and must filter everything the window doesn't need beforehand.

**2. The window's `ORDER BY` is not the query's `ORDER BY`.** These are two different sorts. `OVER (ORDER BY score DESC)` orders rows *for the window computation*; a trailing `ORDER BY created_at` orders the *final result*. If they differ, PostgreSQL may perform **two** sorts — one to feed `WindowAgg`, one for presentation. If they can be made to agree (or a single index satisfies both), you save a sort. This is a recurring optimisation target.

**3. Multiple windows are evaluated in a planner-chosen sequence.** When a query has several `OVER` clauses, the planner evaluates them as a *stack* of `WindowAgg` nodes. If two windows share the same partition+order, they can be evaluated by **one** sort feeding **two** stacked `WindowAgg` nodes. If they differ, each distinct ordering forces its own Sort. Ordering your windows to share sorts is a core technique in Section 6.

---

## 4. What Is Window Function Performance?

Window function performance is the study of the cost of the `WindowAgg` node and, above all, the cost of producing its **required input ordering** (partition + order). Because `WindowAgg` cannot sort itself, its performance is dominated by (a) the `Sort` or `Index Scan` that feeds it, (b) whether that sort fits in `work_mem`, and (c) how many distinct sorts a multi-window query forces.

```sql
SELECT
  employee_id,
  department_id,
  salary,
  RANK() OVER w                              AS dept_rank,
  AVG(salary) OVER w                         AS dept_avg
FROM employees
WINDOW w AS (PARTITION BY department_id ORDER BY salary DESC)
ORDER BY department_id, dept_rank;
```

Annotated breakdown of the performance-relevant parts:

```
SELECT ...
  RANK()  OVER w                      -- window function; contributes WindowAgg CPU per row
  │         └── references named window "w" (WINDOW clause) — see below
  AVG(salary) OVER w                  -- SECOND window function sharing the SAME window "w"
  │         └── shares w ⇒ shares ONE Sort ⇒ ONE WindowAgg, not two
FROM employees
WINDOW w AS (                         -- named WINDOW clause: define the frame ONCE, reuse by name
  │  └── purely a readability/DRY feature; does NOT change the plan by itself,
  │      but makes it OBVIOUS which functions share an ordering (and thus a sort)
  PARTITION BY department_id          -- input must be GROUPED by department_id
  │              └── planner needs rows clustered by this key (Sort or index)
  ORDER BY salary DESC                -- within each partition, rows ordered by salary desc
  │           └── this ordering is what the Sort node (or index) must produce
)
ORDER BY department_id, dept_rank     -- FINAL presentation sort — may or may not be a 2nd sort
                                      -- (here it can often reuse the window's partition order)
```

The key definitional points:
- **`WindowAgg` never sorts.** It relies on a `Sort` node or an ordered `Index Scan` underneath.
- **Each distinct `(PARTITION BY, ORDER BY)` signature = one sort.** Reusing a signature (via a named `WINDOW` or identical `OVER` clauses) reuses the sort.
- **The `WINDOW` clause is DRY, not magic.** Two identical inline `OVER (...)` clauses also share a sort; the `WINDOW` clause just names the definition so the sharing is visible and typo-proof.
- **`work_mem` governs both the Sort and the WindowAgg tuplestore.** Exceed it and either spills to disk.

---

## 5. Why Window Function Performance Mastery Matters in Production

1. **Window queries are the top cause of "sudden slow report" incidents.** A dashboard query that ran in 200ms on 500K rows silently becomes a 40-second, disk-spilling monster at 5M rows — because the feeding Sort crossed the `work_mem` boundary and went external. The query text didn't change; the data volume did. Recognising the `Sort Method: external merge Disk:` line is the difference between a 5-minute fix and a day of confusion.

2. **The sort dominates, and most engineers optimise the wrong thing.** People add indexes on the `SELECT`ed columns, or rewrite the function, when the entire cost is the `Sort` feeding `WindowAgg`. Knowing that lets you target the one thing that matters: providing pre-sorted input or shrinking the input set.

3. **Multi-window queries multiply sorts invisibly.** A reporting query with `ROW_NUMBER()`, `RANK()`, running `SUM`, and a `LAG` — each with a slightly different `OVER` clause — can force four separate sorts of the same data. Consolidating them to one or two orderings can cut runtime by 3–4×. No error is raised; the query just wastes CPU.

4. **`work_mem` is a loaded gun.** Setting it too low spills every window to disk; setting it globally high risks OOM under concurrency (each sort/hash node can allocate `work_mem` *per node per connection*). The correct move — setting `work_mem` locally for the specific heavy query — requires understanding exactly which node consumes it.

5. **Large partitions and unbounded frames are silent memory bombs.** A `SUM(...) OVER (ORDER BY ts ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` over a 20M-row single partition buffers the whole partition. On a table with one giant partition (e.g. a global running total), this can spill tens of gigabytes and dominate the plan. Knowing which frames buffer and which stream is essential to keep memory bounded.

6. **Indexes for window queries are non-obvious.** The right index is a composite on `(partition_cols, order_cols)` in the exact order and direction the window needs — not an index on the filtered column. Getting this right can turn a Sort + Seq Scan into a plain Index Scan with zero sort cost.

---

## 6. Deep Technical Content

### 6.1 The `WindowAgg` Node and Where Its Cost Comes From

Every window function in a query is executed by one or more `WindowAgg` nodes. A single `WindowAgg` node handles all window functions that **share the same window definition** (same `PARTITION BY` + `ORDER BY` + frame). The node's own per-row cost is small for ranking functions (`ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE`) — essentially a counter update per row. It is larger for:

- **Frame aggregates** (`SUM/AVG/MIN/MAX OVER (... ROWS/RANGE BETWEEN ...)`) that must maintain a moving window,
- **`LAG`/`LEAD`** which must buffer rows to reach across the partition,
- **`FIRST_VALUE`/`LAST_VALUE`/`NTH_VALUE`** which scan within the frame.

But even for these, the `WindowAgg` per-row cost is usually dwarfed by the **Sort** that feeds it. The mental model: `total_cost ≈ cost(Sort) + cost(scan) + small_per_row × rows`.

### 6.2 The Sort Is Almost Always the Bottleneck

Consider:

```sql
SELECT customer_id, order_id, total_amount,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
FROM orders;
```

With no useful index, the plan is:

```
WindowAgg
  └── Sort  (Sort Key: customer_id, created_at DESC)
        └── Seq Scan on orders
```

The `Seq Scan` reads every row. The `Sort` orders all of them by `(customer_id, created_at DESC)` — `O(N log N)` CPU, and if the data exceeds `work_mem`, an external merge sort that writes and re-reads temp files. The `WindowAgg` then walks the sorted stream once, `O(N)`. On 10M rows, the Sort is 90%+ of the runtime. **Optimise the sort, not the function.**

### 6.3 Eliminating the Sort with a Pre-Sorting Index

A B-tree index stores its keys in sorted order. If an index exactly matches the window's required ordering, PostgreSQL can perform an **Index Scan** that emits rows already in `(partition, order)` sequence — deleting the `Sort` node entirely.

```sql
-- Index matching the window's PARTITION BY + ORDER BY, with matching direction
CREATE INDEX idx_orders_cust_created
  ON orders (customer_id, created_at DESC);
```

Now:

```
WindowAgg
  └── Index Scan using idx_orders_cust_created on orders
```

No Sort. The rows come off the index already grouped by `customer_id` and, within each customer, ordered by `created_at DESC`. This is the single most powerful window optimisation. Rules for the index to be usable:

- **Column order must match**: partition columns first, then order columns — `(customer_id, created_at)`, not `(created_at, customer_id)`.
- **Direction matters for multi-column ORDER BY**: `ORDER BY created_at DESC` is satisfied by an index on `created_at DESC` **or** `created_at ASC` scanned backward — PostgreSQL can scan a B-tree in either direction, so a single-column direction is usually fine. But for **mixed directions** (`ORDER BY a ASC, b DESC`) you need the index to encode exactly that mix: `(a ASC, b DESC)`, because a plain backward scan would flip *both*.
- **`PARTITION BY` has no inherent direction**, so `PARTITION BY customer_id` is satisfied by `customer_id` ASC or DESC equally.

### 6.4 Incremental Sort — Partial Pre-Sorting (PostgreSQL 13+)

Sometimes an index provides *some* of the ordering but not all. Suppose you have `CREATE INDEX ON orders (customer_id)` but the window needs `(customer_id, created_at DESC)`. The index gives you `customer_id` order for free but not `created_at` within each customer. PostgreSQL 13+ can use an **Incremental Sort**: it reads rows already grouped by `customer_id` off the index, and only sorts the small `created_at` groups *within* each customer — far cheaper than a full sort, and it never needs to hold more than one partition in memory at a time.

```
WindowAgg
  └── Incremental Sort  (Sort Key: customer_id, created_at DESC;  Presorted Key: customer_id)
        └── Index Scan using idx_orders_customer_id on orders
```

`Presorted Key: customer_id` tells you the index handled the partition boundary; the Incremental Sort only orders `created_at` inside each partition. This bounds sort memory to one partition's worth of rows and is a huge win when full pre-sorting isn't available.

### 6.5 Multiple Windows: When They Share a Sort vs Re-Sort

This is the highest-leverage part of the topic. Each **distinct** window definition `(PARTITION BY, ORDER BY, frame-for-ordering)` requires its input in that specific order. The planner builds a **stack of `WindowAgg` nodes**, and interposes a `Sort` (or Incremental Sort) whenever the next window needs a different ordering than the previous node produced.

**Case A — All windows share one ordering ⇒ ONE sort:**

```sql
SELECT
  department_id, employee_id, salary,
  ROW_NUMBER() OVER w AS rn,
  RANK()       OVER w AS rnk,
  AVG(salary)  OVER w AS dept_avg
FROM employees
WINDOW w AS (PARTITION BY department_id ORDER BY salary DESC);
```

Plan:

```
WindowAgg              -- computes AVG (or all three, stacked)
  WindowAgg
    WindowAgg
      └── Sort  (Sort Key: department_id, salary DESC)   ◄── SINGLE sort
            └── Seq Scan on employees
```

One Sort, three stacked `WindowAgg` nodes reusing the same sorted stream.

**Case B — Different orderings ⇒ multiple sorts:**

```sql
SELECT
  department_id, employee_id, salary, hire_date,
  RANK()       OVER (PARTITION BY department_id ORDER BY salary DESC)     AS salary_rank,
  ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY hire_date ASC)   AS seniority
FROM employees;
```

Plan:

```
WindowAgg               -- seniority: needs (department_id, hire_date)
  └── Sort  (Sort Key: department_id, hire_date)        ◄── SECOND sort
        └── WindowAgg   -- salary_rank: needs (department_id, salary DESC)
              └── Sort  (Sort Key: department_id, salary DESC)   ◄── FIRST sort
                    └── Seq Scan on employees
```

Two distinct orderings ⇒ two Sort nodes ⇒ the data is sorted twice. On large tables this roughly doubles the dominant cost.

**The planner's ordering heuristic:** PostgreSQL orders the `WindowAgg` stack to minimise re-sorting and, when possible, to leave the output in an order that satisfies the final `ORDER BY`. It groups windows with compatible orderings adjacently. But it does **not** rewrite your functions — if you genuinely need two different orderings, two sorts are unavoidable *within a single scan*. Your levers are: (a) make windows share an ordering where the logic permits, (b) provide an index that eliminates at least one sort, or (c) accept the second sort but shrink the input via `WHERE` first.

**Partition prefix sharing:** Even when two windows differ in `ORDER BY`, if they share the `PARTITION BY` prefix, an **Incremental Sort** can reuse the partition ordering and only re-sort the differing tail — cheaper than two full sorts. This is why keeping a common `PARTITION BY` across your windows helps even when their `ORDER BY` differs.

### 6.6 The Named `WINDOW` Clause — Reuse for Clarity (and Correctness)

The `WINDOW` clause lets you define a window specification once and reference it by name:

```sql
SELECT
  product_id,
  SUM(revenue)   OVER monthly       AS running_revenue,
  AVG(revenue)   OVER monthly       AS avg_revenue,
  RANK()         OVER monthly_rank  AS revenue_rank
FROM monthly_sales
WINDOW
  monthly      AS (PARTITION BY product_id ORDER BY month
                   ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW),
  monthly_rank AS (PARTITION BY product_id ORDER BY revenue DESC);
```

Key facts:
- **It is primarily a readability / DRY feature.** Two functions with identical inline `OVER (...)` clauses already share a single sort and `WindowAgg`; the `WINDOW` clause does not change that — it just avoids repeating (and mis-typing) the definition.
- **It prevents "accidental re-sort" bugs.** When you copy-paste an inline `OVER` and change one column by mistake, you silently create a second ordering and a second sort. Naming the window makes the shared definition a single source of truth — if you meant them to share, they provably do.
- **You can extend a named window**: `OVER (monthly ORDER BY ...)` — reference the base and add/override parts. A referencing window may add an `ORDER BY` or a frame to a base that only defined `PARTITION BY`, but it cannot override an already-specified `PARTITION BY` or `ORDER BY`.
- **It has no runtime cost of its own.** The plan for named vs inline-but-identical windows is byte-for-byte the same.

### 6.7 Frame Clauses and Memory: What Buffers vs What Streams

Recall from Topic 39 that the frame (`ROWS`/`RANGE`/`GROUPS BETWEEN ...`) defines which rows each output row aggregates over. Frames have direct memory consequences in `WindowAgg`:

- **`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`** (default running total): streams efficiently — the aggregate accumulates incrementally, only the running state is kept. Cheap.
- **`ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING`**: must see the rest of the partition, so the tuplestore buffers ahead. More memory.
- **`RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`** (whole partition, e.g. `AVG(x) OVER (PARTITION BY k)` with no ORDER BY): buffers the **entire partition** to compute one value applied to all rows. Memory = one partition.
- **Moving frames** (`ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`): PostgreSQL maintains the window incrementally where it can, but for `MAX`/`MIN` over a moving window it may re-scan the frame, adding CPU.
- **`RANGE` with `ORDER BY`** requires peer-group detection (rows with equal order values are "peers") which needs look-ahead buffering to find the peer boundary. `ROWS` needs no peer detection and is cheaper.

The tuplestore that holds these buffered rows is sized by `work_mem`. A large partition with a whole-partition frame is the classic spill scenario (Section 6.9).

### 6.8 `work_mem` — The Governor of Both Sort and WindowAgg

`work_mem` is the per-operation memory budget. Critically, **it applies per node, per connection** — a single query with two sorts and a window can allocate up to `3 × work_mem`, and 50 concurrent connections running it can allocate `150 × work_mem`. This is why you don't just crank `work_mem` globally.

Two places `work_mem` bites in window queries:
1. **The feeding `Sort`**: if the rows to sort exceed `work_mem`, PostgreSQL switches from `quicksort` (in memory) to `external merge` (temp files on disk). You'll see `Sort Method: external merge  Disk: 246880kB`.
2. **The `WindowAgg` tuplestore**: if a partition's buffered rows exceed `work_mem`, the tuplestore spills to a temp file.

The correct pattern for a heavy analytical window query is to raise `work_mem` **for that query/session only**:

```sql
SET LOCAL work_mem = '256MB';   -- inside a transaction, reverts at COMMIT
-- ... run the heavy window query ...
```

`SET LOCAL` scopes the bump to the current transaction so you don't inflate memory for the whole server. Measure the spill size in EXPLAIN (`Disk: NkB`) and set `work_mem` just above it (with headroom) — not arbitrarily high.

### 6.9 Large Partitions and Spilling — Diagnosis and Mitigation

The worst-case window query has a **small number of very large partitions** (or one giant partition) with a **frame that buffers the whole partition**:

```sql
-- One global running total over 50M rows: a single 50M-row partition
SELECT event_id, ts,
       SUM(amount) OVER (ORDER BY ts ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running
FROM audit_logs;
```

Here there is no `PARTITION BY`, so the entire table is one partition. The Sort must order all 50M rows (huge external merge), and the `WindowAgg` walks the whole thing. Mitigations, in order of preference:

1. **Reduce the input with `WHERE` first** — a date range that cuts 50M to 2M turns an external sort into an in-memory one.
2. **Provide a pre-sorting index** (`CREATE INDEX ON audit_logs (ts)`) so the Sort disappears; the running `SUM` streams with `O(1)` state.
3. **Raise `work_mem` for the query** (`SET LOCAL`) so at least the sort stays in memory.
4. **Partition the work** — if a global running total isn't truly required, add a meaningful `PARTITION BY` (per account, per day) so each partition is small and the tuplestore never balloons.
5. **Materialise** — for a repeatedly-run heavy running total, precompute into a summary table (or `MATERIALIZED VIEW`) refreshed on a schedule.

Note the asymmetry: the *running* frame (`UNBOUNDED PRECEDING AND CURRENT ROW`) streams with tiny state; it's the **Sort** that spills, not the tuplestore. But a *whole-partition* frame (`RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`) makes the **tuplestore** spill too, because every row must be re-read after the last row of the partition is seen. Distinguishing "which thing spills" tells you whether the fix is an index (kills the Sort) or a smaller partition (kills the tuplestore growth).

### 6.10 `LIMIT` Does Not Push Through `WindowAgg` for Ranking Output

A subtle trap: adding `LIMIT 10` to a window query does **not** let PostgreSQL skip sorting the rest, because the window function must see the *whole* partition to compute ranks. `ROW_NUMBER() OVER (ORDER BY score)` cannot know row #1 without examining all rows to establish the order. So the full Sort still runs even under a `LIMIT`. (Top-N *per group* has a different, better pattern — see Topic 41.) The lesson for performance: don't expect `LIMIT` to rescue a window query; shrink the input with `WHERE`, or use an index to make the sort free.

### 6.11 Parallelism and Window Functions

`WindowAgg` is **not itself parallel-aware** in the sense of splitting a partition across workers, but the plan *beneath* it can be parallel. A common efficient shape is a Parallel Seq Scan feeding a Gather, then a Sort, then `WindowAgg` — the scan parallelises even though the window computation is serialised after the Gather Merge. When you see a serial Sort + WindowAgg on top of a parallel scan, the scan portion is still using multiple workers. You generally cannot parallelise the window computation itself, which is another reason to minimise the row count reaching it.

---

## 7. EXPLAIN — WindowAgg in the Plan

### 7.1 The Baseline: Sort Feeding WindowAgg (no index)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, order_id, total_amount,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
FROM orders;
```

```
WindowAgg  (cost=1443891.00..1643891.00 rows=8000000 width=40)
           (actual time=6120.4..9880.7 rows=8000000 loops=1)
  ->  Sort  (cost=1443891.00..1463891.00 rows=8000000 width=32)
            (actual time=6120.3..7550.1 rows=8000000 loops=1)
        Sort Key: customer_id, created_at DESC
        Sort Method: external merge  Disk: 262144kB          ◄── SPILLED TO DISK
        ->  Seq Scan on orders  (cost=0.00..179800.00 rows=8000000 width=32)
                                 (actual time=0.02..1210.5 rows=8000000 loops=1)
Buffers: shared hit=1280 read=98520, temp read=32768 written=32768
Planning Time: 0.20 ms
Execution Time: 10120.8 ms
```

Line-by-line:
- **`Seq Scan on orders`** — full table scan, 8M rows read from the heap.
- **`Sort` … `Sort Method: external merge  Disk: 262144kB`** — the killer. `work_mem` was too small, so the sort spilled ~256MB to a temp file. `temp read/written=32768` (blocks) confirms the disk traffic.
- **`Sort Key: customer_id, created_at DESC`** — exactly the window's `PARTITION BY` + `ORDER BY`.
- **`WindowAgg`** — walks the sorted stream once; its own `actual time` delta over the Sort (7550→9880) is the per-row `ROW_NUMBER` work, cheap relative to the sort+scan.
- **`Execution Time: 10120.8 ms`** — dominated by the spilling sort.

### 7.2 After Adding the Pre-Sorting Index

```sql
CREATE INDEX idx_orders_cust_created ON orders (customer_id, created_at DESC);

EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, order_id, total_amount,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
FROM orders;
```

```
WindowAgg  (cost=0.56..612300.00 rows=8000000 width=40)
           (actual time=0.05..3980.2 rows=8000000 loops=1)
  ->  Index Scan using idx_orders_cust_created on orders
          (cost=0.56..412300.00 rows=8000000 width=32)
          (actual time=0.03..1980.4 rows=8000000 loops=1)
Buffers: shared hit=41200 read=160300
Planning Time: 0.18 ms
Execution Time: 4180.6 ms
```

- **No `Sort` node.** The `Index Scan` emits rows already in `(customer_id, created_at DESC)` order.
- **No `temp read/written`.** No spill.
- **`Execution Time: 4180.6 ms`** — ~2.4× faster, and the sort's disk I/O is gone entirely. (On a smaller/filtered result the win is far larger.)

### 7.3 Multiple Windows Sharing One Sort

```sql
EXPLAIN (ANALYZE)
SELECT department_id, employee_id, salary,
       RANK()      OVER w AS rnk,
       DENSE_RANK() OVER w AS dnk,
       AVG(salary)  OVER w AS dept_avg
FROM employees
WINDOW w AS (PARTITION BY department_id ORDER BY salary DESC);
```

```
WindowAgg  (cost=18500.00..21900.00 rows=200000 width=56)
           (actual time=210.4..340.9 rows=200000 loops=1)
  ->  Sort  (cost=18500.00..19000.00 rows=200000 width=20)
            (actual time=210.3..245.1 rows=200000 loops=1)
        Sort Key: department_id, salary DESC
        Sort Method: quicksort  Memory: 24576kB              ◄── in-memory, no spill
        ->  Seq Scan on employees  (cost=0.00..3600.00 rows=200000 width=20)
Planning Time: 0.22 ms
Execution Time: 358.7 ms
```

Note: **one `Sort`, one `Sort Key`, one `WindowAgg`** even though three window functions are present — because all three share window `w`. (PostgreSQL collapses functions of the same window into a single `WindowAgg`.) `Sort Method: quicksort Memory: 24576kB` — fit in `work_mem`, no disk.

### 7.4 Multiple Windows Forcing Two Sorts

```sql
EXPLAIN (ANALYZE)
SELECT department_id, employee_id, salary, hire_date,
       RANK()       OVER (PARTITION BY department_id ORDER BY salary DESC)   AS salary_rank,
       ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY hire_date)     AS seniority
FROM employees;
```

```
WindowAgg  (cost=40100.00..43600.00 rows=200000 width=64)
           (actual time=520.7..660.2 rows=200000 loops=1)
  ->  Sort  (cost=40100.00..40600.00 rows=200000 width=28)
            (actual time=520.6..556.1 rows=200000 loops=1)
        Sort Key: department_id, hire_date                   ◄── SECOND sort
        Sort Method: quicksort  Memory: 26100kB
        ->  WindowAgg  (cost=18500.00..21500.00 rows=200000 width=28)
                       (actual time=210.5..360.8 rows=200000 loops=1)
              ->  Sort  (cost=18500.00..19000.00 rows=200000 width=20)
                        Sort Key: department_id, salary DESC  ◄── FIRST sort
                        Sort Method: quicksort  Memory: 24576kB
                        ->  Seq Scan on employees
Planning Time: 0.28 ms
Execution Time: 690.4 ms
```

Two `Sort` nodes, two `Sort Key`s — the data is sorted twice because the two windows disagree on `ORDER BY`. Runtime ~2× the shared-sort case. This is exactly the plan shape to hunt for and, where the logic allows, consolidate.

### 7.5 What to Look For in a Window Plan

| Symptom in EXPLAIN | Meaning | Fix |
|---|---|---|
| `Sort Method: external merge  Disk: NkB` | Feeding sort spilled to disk | Raise `work_mem` (SET LOCAL) or add a pre-sorting index |
| Multiple `Sort` nodes with different `Sort Key`s | Windows re-sort the same data | Share window definitions; add index; shrink input |
| `Sort` node directly under `WindowAgg` | No index provides the order | Create index on `(partition_cols, order_cols)` |
| `Incremental Sort … Presorted Key: …` | Index gave partial order (good) | Already efficient; ensure index covers partition prefix |
| `WindowAgg` estimated rows ≠ actual | Stale stats upstream | `ANALYZE` the table |
| Big gap between Sort `actual time` and WindowAgg `actual time` end | Window function itself is expensive (moving frame) | Simplify frame; use `ROWS` not `RANGE` where possible |
| Seq Scan feeding a huge Sort with no WHERE | Filtering could shrink the sort | Add `WHERE`, and index the filter+order columns |

---

## 8. Query Examples

### Example 1 — Basic: Ranking with an Explicit Sort

```sql
-- Rank employees by salary within each department.
-- With no matching index, the plan is Seq Scan -> Sort -> WindowAgg.
SELECT
  department_id,
  employee_id,
  salary,
  RANK() OVER (PARTITION BY department_id ORDER BY salary DESC) AS dept_salary_rank
FROM employees
ORDER BY department_id, dept_salary_rank;   -- final order aligns with the window's partition
```

The window needs `(department_id, salary DESC)`. The final `ORDER BY department_id, dept_salary_rank` is naturally satisfied by the window output order, so PostgreSQL typically needs only the **one** sort that feeds the window — no separate presentation sort.

### Example 2 — Intermediate: Multiple Windows, One Shared Sort

```sql
-- Several metrics per customer, ALL sharing one window => ONE sort.
-- Using a named WINDOW clause makes the sharing explicit and typo-proof.
SELECT
  customer_id,
  order_id,
  total_amount,
  ROW_NUMBER() OVER w                         AS order_seq,        -- 1,2,3... per customer
  SUM(total_amount) OVER w                     AS running_spend,   -- running total per customer
  AVG(total_amount) OVER (PARTITION BY customer_id) AS lifetime_avg -- whole-partition avg
FROM orders
WHERE status = 'completed'                    -- filter BEFORE the window shrinks the sort
WINDOW w AS (
  PARTITION BY customer_id
  ORDER BY created_at
  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
);
```

Note: `order_seq` and `running_spend` share `w` (one sort, one WindowAgg). The `lifetime_avg` uses `PARTITION BY customer_id` with **no ORDER BY** — its ordering requirement (partition only) is a *prefix* of `w`'s, so PostgreSQL can compute it from the same `customer_id`-grouped stream without a second full sort. The `WHERE status = 'completed'` runs first, cutting the row count the sort must handle.

### Example 3 — Production Grade: Large-Table Windowed Report with an Index

**Scenario.** `orders` has 40M rows (~9GB heap). `order_items` has 180M rows. We need, per customer, their most recent 90 days of completed orders annotated with: sequence number, running spend, and rank by order value. Expectation: sub-second for a single customer, a few seconds for a cohort, and **no disk spill** for the common per-customer path.

**Index provided:**

```sql
-- Supports the WHERE (status, created_at) AND the window ordering in one index.
CREATE INDEX idx_orders_cust_status_created
  ON orders (customer_id, status, created_at DESC);
```

**Query:**

```sql
SELECT
  o.customer_id,
  o.id                                                       AS order_id,
  o.total_amount,
  o.created_at,
  ROW_NUMBER() OVER w                                        AS recency_seq,
  SUM(o.total_amount) OVER w                                 AS running_90d_spend,
  RANK() OVER (PARTITION BY o.customer_id ORDER BY o.total_amount DESC) AS value_rank
FROM orders o
WHERE o.customer_id = $1
  AND o.status = 'completed'
  AND o.created_at >= NOW() - INTERVAL '90 days'
WINDOW w AS (PARTITION BY o.customer_id ORDER BY o.created_at DESC)
ORDER BY o.created_at DESC;
```

**EXPLAIN (single customer):**

```
WindowAgg  (cost=8.90..410.20 rows=120 width=64)
           (actual time=0.08..0.42 rows=118 loops=1)
  ->  WindowAgg  (cost=8.90..360.00 rows=120 width=56)
                 (actual time=0.06..0.30 rows=118 loops=1)
        ->  Sort  (cost=8.90..9.20 rows=120 width=40)
                  Sort Key: total_amount DESC
                  Sort Method: quicksort  Memory: 40kB          ◄── tiny, in-memory
              ->  Index Scan using idx_orders_cust_status_created on orders o
                    (cost=0.56..4.50 rows=120 width=40)
                    Index Cond: (customer_id = $1 AND status = 'completed'
                                 AND created_at >= (now() - '90 days'::interval))
                    (actual time=0.03..0.05 rows=118 loops=1)
Planning Time: 0.30 ms
Execution Time: 0.55 ms
```

Reading it:
- The **Index Scan** satisfies both the `WHERE` (all three predicates via `Index Cond`) **and** the `w` window's `(customer_id, created_at DESC)` ordering — so the running-spend/recency window needs **no Sort**.
- Only the `value_rank` window (`ORDER BY total_amount DESC`) needs a Sort, but it operates on ~118 rows — `Memory: 40kB`, trivially in-memory.
- Sub-millisecond. The index is doing triple duty: filter, partition ordering, and (implicitly) row locality. This is the shape to aim for: push the filter into an index that *also* provides the primary window ordering, so the only remaining sort is on the tiny filtered result.

---

## 9. Wrong → Right Patterns

### Wrong 1: Indexing the SELECTed column instead of the window ordering

```sql
-- WRONG: index chosen because total_amount is "important" in the report
CREATE INDEX idx_orders_amount ON orders (total_amount);

SELECT customer_id, order_id, total_amount,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
FROM orders;
-- The index on total_amount is USELESS here. Plan: Seq Scan -> big Sort -> WindowAgg.
-- The window needs (customer_id, created_at), which this index does not provide.
```

**Why it's wrong at the execution level:** the `WindowAgg` requires input ordered by `(customer_id, created_at DESC)`. An index on `total_amount` provides ordering by `total_amount` — irrelevant. The planner ignores it and sorts from scratch.

```sql
-- RIGHT: index the PARTITION BY + ORDER BY of the window
CREATE INDEX idx_orders_cust_created ON orders (customer_id, created_at DESC);
-- Now: Index Scan -> WindowAgg, no Sort.
```

### Wrong 2: Silently forcing two sorts with near-identical windows

```sql
-- WRONG: two windows that differ only by a copy-paste slip
SELECT
  department_id, employee_id, salary,
  RANK()       OVER (PARTITION BY department_id ORDER BY salary DESC) AS r1,
  DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary)      AS r2  -- missing DESC!
FROM employees;
-- Plan: TWO Sort nodes (salary DESC vs salary ASC) => data sorted twice.
-- Often the ASC was an accident; the intent was one shared window.
```

**Why it's wrong at the execution level:** `salary DESC` and `salary` (ASC) are different orderings, so the planner inserts a second `Sort`. On a large table this doubles the dominant cost — with no error and no visible clue except in EXPLAIN.

```sql
-- RIGHT: share one window definition
SELECT
  department_id, employee_id, salary,
  RANK()       OVER w AS r1,
  DENSE_RANK() OVER w AS r2
FROM employees
WINDOW w AS (PARTITION BY department_id ORDER BY salary DESC);
-- Plan: ONE Sort, ONE WindowAgg.
```

### Wrong 3: No `WHERE`, letting the window sort the entire table

```sql
-- WRONG: compute a rank across all 40M rows, then (elsewhere) look at one month
SELECT customer_id, order_id, created_at,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
FROM orders;                        -- sorts 40M rows, spills to disk
-- Application then filters to last month client-side. Catastrophic.
```

**Why it's wrong:** the window sees 40M rows because `WHERE` (which runs *before* the window) filters nothing. The Sort spills to disk. You paid to rank ancient data you never use.

```sql
-- RIGHT: filter first so the window sorts only what you need
SELECT customer_id, order_id, created_at,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
FROM orders
WHERE created_at >= NOW() - INTERVAL '30 days';   -- sort shrinks by ~40x, stays in memory
```

(If you truly need per-customer *global* ranking but only display recent rows, that is a Top-N-per-group problem — Topic 41 — solved with `LATERAL` + `LIMIT`, not a full-table window.)

### Wrong 4: A whole-partition frame over one giant partition

```sql
-- WRONG: global running balance over 50M rows in a single partition
SELECT id, ts, amount,
       SUM(amount) OVER (ORDER BY ts) AS running_balance   -- default frame + no PARTITION BY
FROM audit_logs;
-- One 50M-row partition. Huge external-merge Sort; WindowAgg walks all 50M.
-- Runs for minutes and spills gigabytes.
```

**Why it's wrong:** with no `PARTITION BY`, the entire table is one partition; the Sort must order all 50M rows on disk. The frame streams, but the Sort is the killer.

```sql
-- RIGHT (option A): pre-sorting index kills the Sort; running SUM streams with O(1) state
CREATE INDEX idx_audit_ts ON audit_logs (ts);
-- Plan: Index Scan -> WindowAgg, no Sort.

-- RIGHT (option B): if a per-account running balance suffices, partition it small
SELECT account_id, id, ts, amount,
       SUM(amount) OVER (PARTITION BY account_id ORDER BY ts) AS running_balance
FROM audit_logs;
-- Each partition is small; the tuplestore never balloons; add index (account_id, ts).
```

### Wrong 5: Expecting `LIMIT` to shortcut the window sort

```sql
-- WRONG assumption: LIMIT 20 makes this cheap
SELECT customer_id, order_id, total_amount,
       RANK() OVER (ORDER BY total_amount DESC) AS overall_rank
FROM orders
LIMIT 20;
-- The RANK() must see ALL rows to assign ranks, so the FULL sort of 40M rows runs
-- BEFORE LIMIT can take 20. LIMIT does not push through WindowAgg.
```

**Why it's wrong:** ranking is defined over the whole partition; the executor cannot know the top 20 without ordering everything. The `LIMIT` only trims the final output *after* the full Sort + WindowAgg.

```sql
-- RIGHT: if you only want the top 20 by amount, don't use a window at all
SELECT customer_id, order_id, total_amount
FROM orders
ORDER BY total_amount DESC
LIMIT 20;                              -- a Top-N heap sort; with an index, near-instant
-- (If you need the rank VALUE on those 20, compute it in a small subquery over the top slice.)
```

---

## 10. Performance Profile

### 10.1 Cost Composition

For a single-window query, runtime decomposes as:

```
T ≈ T_scan + T_sort + T_windowagg + T_final_sort
```

- **`T_scan`**: reading rows (Seq Scan or Index Scan). Proportional to rows/pages touched.
- **`T_sort`**: `O(N log N)` CPU; **plus disk I/O if it spills** (the cliff). Usually the dominant term.
- **`T_windowagg`**: `O(N)` for ranking; more for moving/whole-partition frames.
- **`T_final_sort`**: a separate presentation sort if the query's `ORDER BY` differs from the window output order.

The optimisation priority is always: **eliminate or shrink `T_sort` first** (index or `WHERE`), then avoid a redundant `T_final_sort`, then simplify the frame if `T_windowagg` is large.

### 10.2 Scaling by Row Count (single partition-heavy window, default `work_mem`=64MB)

| Rows into window | No index (Seq+Sort) | With pre-sort index | Notes |
|---|---|---|---|
| 100K | ~40–80 ms | ~15–30 ms | Sort fits in `work_mem`; index still ~2× faster |
| 1M | ~0.4–0.9 s | ~0.15–0.3 s | Sort still in memory around here |
| 10M | ~6–12 s (external merge) | ~1–3 s | Sort spills without index — the cliff |
| 100M | ~90–180 s, gigabytes of temp | ~15–40 s | Index-driven scan streams; no spill |

The numbers are illustrative but the *shape* is real: without a pre-sorting index, crossing the `work_mem` boundary causes a super-linear jump as the sort goes external; with the index, cost grows roughly linearly with rows scanned.

### 10.3 Multiple Windows

- **N windows, all sharing one ordering**: cost ≈ single-sort cost + N × (cheap per-row). Nearly free to add more functions to a shared window.
- **N windows, K distinct orderings**: roughly K sorts. Each new *distinct* ordering adds a full sort. Consolidating from K=4 to K=1 can cut runtime ~3–4×.
- **Partition-prefix-shared orderings**: Incremental Sort reduces the extra sorts' cost to per-partition work — cheaper than K full sorts.

### 10.4 Index Interactions

- **Best index**: `(partition_cols…, order_cols… [directions])`. Eliminates the feeding Sort entirely.
- **Filter + window in one index**: put equality-filter columns first, then the window's order column, e.g. `(customer_id, status, created_at DESC)` for `WHERE customer_id=? AND status=? … OVER (PARTITION BY customer_id ORDER BY created_at DESC)`. Equality predicates on leading columns don't disturb the ordering of trailing columns.
- **Covering (INCLUDE) indexes**: `CREATE INDEX … (customer_id, created_at DESC) INCLUDE (total_amount)` can make the scan **Index-Only**, avoiding heap fetches for wide projections — helpful when the window query is I/O-bound.
- **Mixed-direction ORDER BY** needs the exact directions baked in: `ORDER BY dept ASC, salary DESC` ⇒ index `(dept ASC, salary DESC)`.

### 10.5 Memory Tuning Rules of Thumb

- Read the spill size from EXPLAIN (`Disk: NkB`) and set `SET LOCAL work_mem` a bit above it for that query.
- Remember `work_mem` is **per node per connection** — don't set it high globally under concurrency; scope it with `SET LOCAL`.
- Whole-partition frames buffer a partition — size `work_mem` to the *largest partition*, or make partitions smaller.
- Running frames (`UNBOUNDED PRECEDING … CURRENT ROW`) barely use the tuplestore — for these, focus `work_mem` on the feeding Sort.

### 10.6 CPU vs I/O

- **I/O-bound**: large Seq Scan + external-merge Sort writing temp files. Fix with index (removes Sort I/O) and/or covering index (removes heap I/O).
- **CPU-bound**: in-memory quicksort of many rows, or an expensive moving-frame `MAX/MIN` re-scan. Fix by shrinking input, using `ROWS` over `RANGE`, or pre-sorting so only a cheap incremental sort remains.

---

## 11. Node.js Integration

### 11.1 Basic windowed query with `pg`

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Per-customer recency sequence + running spend, filtered first to keep the sort small.
async function getCustomerOrderTimeline(customerId, days = 90) {
  const { rows } = await pool.query(
    `SELECT
       order_id,
       total_amount,
       created_at,
       ROW_NUMBER() OVER w                 AS recency_seq,
       SUM(total_amount) OVER w            AS running_spend
     FROM orders
     WHERE customer_id = $1
       AND status = 'completed'
       AND created_at >= NOW() - ($2 || ' days')::INTERVAL
     WINDOW w AS (PARTITION BY customer_id ORDER BY created_at DESC)
     ORDER BY created_at DESC`,
    [customerId, days]
  );
  return rows;
}
```

`$1`/`$2` are bound server-side (no string interpolation). The `WHERE` shrinks the input before `WindowAgg`, and the `WINDOW w` clause keeps both window functions on one sort.

### 11.2 Raising `work_mem` for a single heavy report

```javascript
// A heavy analytical window query: bump work_mem for THIS transaction only.
async function runHeavyCohortReport(cohortStart, cohortEnd) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query("SET LOCAL work_mem = '256MB'");  // reverts at COMMIT/ROLLBACK
    const { rows } = await client.query(
      `SELECT customer_id, order_id, total_amount,
              RANK() OVER w AS value_rank,
              SUM(total_amount) OVER w AS running_spend
       FROM orders
       WHERE created_at BETWEEN $1 AND $2
       WINDOW w AS (PARTITION BY customer_id ORDER BY total_amount DESC)`,
      [cohortStart, cohortEnd]
    );
    await client.query('COMMIT');
    return rows;
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();  // connection returns to pool with work_mem back to default
  }
}
```

`SET LOCAL` is the correct scope: it applies only inside the transaction and never leaks to other pooled queries. Using a dedicated `client` (not `pool.query`) guarantees the `SET LOCAL` and the query run on the *same* connection.

### 11.3 Diagnosing spills programmatically

```javascript
// Pull the plan and warn if the window's feeding sort spilled to disk.
async function explainWindowQuery(sql, params) {
  const { rows } = await pool.query(`EXPLAIN (ANALYZE, FORMAT JSON) ${sql}`, params);
  const planText = JSON.stringify(rows[0]['QUERY PLAN']);
  if (planText.includes('external merge')) {
    console.warn('[perf] window feeding Sort spilled to disk — raise work_mem or add index');
  }
  const sortCount = (planText.match(/"Node Type":"Sort"/g) || []).length;
  if (sortCount > 1) {
    console.warn(`[perf] ${sortCount} Sort nodes — windows may be re-sorting; consider a shared WINDOW`);
  }
  return rows[0]['QUERY PLAN'];
}
```

### 11.4 Streaming large windowed results with a cursor

```javascript
import QueryStream from 'pg-query-stream';

// For a large windowed export, stream rows so Node doesn't buffer the whole result set.
async function streamRankedOrders(onRow) {
  const client = await pool.connect();
  try {
    const query = new QueryStream(
      `SELECT customer_id, order_id, total_amount,
              ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
       FROM orders
       WHERE created_at >= NOW() - INTERVAL '30 days'`
    );
    const stream = client.query(query);
    for await (const row of stream) onRow(row);   // backpressure-aware
  } finally {
    client.release();
  }
}
```

Note: the *database* still fully sorts and computes the window before streaming begins (the window needs the whole partition). Streaming only bounds Node's memory, not Postgres's — the `WHERE`/index still govern DB-side cost.

---

## 12. ORM Comparison

Window functions are, across the board, weakly supported by ORMs — most cannot express `OVER (...)`, named `WINDOW` clauses, or frames in their fluent API, and none give you control over the *sort/index* strategy that governs performance. Raw SQL is the norm for anything beyond a single simple window.

### Prisma

**Can it?** — Not in the fluent API. Prisma has no `OVER`, no `PARTITION BY`, no `WINDOW` clause. Window functions require `$queryRaw`.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

const rows = await prisma.$queryRaw<Array<{ order_id: number; rn: number; running_spend: string }>>`
  SELECT order_id,
         ROW_NUMBER()      OVER w AS rn,
         SUM(total_amount) OVER w AS running_spend
  FROM orders
  WHERE customer_id = ${customerId} AND status = 'completed'
  WINDOW w AS (PARTITION BY customer_id ORDER BY created_at DESC)
`;
```

**Where it breaks:** no way to influence the plan; you must add the pre-sorting index via a migration yourself. `$queryRaw` returns loosely-typed results (numerics come back as strings). **Verdict:** raw SQL only; use the tagged template for safe parameterisation, and manage indexes in migrations.

### Drizzle ORM

**Can it?** — Partially. Drizzle has no first-class `WINDOW`/`OVER` builder, but its `sql` template composes window expressions cleanly and typed.

```typescript
import { sql } from 'drizzle-orm';
import { orders } from './schema';

const rows = await db
  .select({
    orderId: orders.id,
    rn: sql<number>`ROW_NUMBER() OVER (PARTITION BY ${orders.customerId} ORDER BY ${orders.createdAt} DESC)`,
    runningSpend: sql<string>`SUM(${orders.totalAmount}) OVER (PARTITION BY ${orders.customerId} ORDER BY ${orders.createdAt} DESC)`,
  })
  .from(orders)
  .where(sql`${orders.status} = 'completed'`);
```

**Where it breaks:** you must repeat the `OVER (...)` per column (no named-window support), which risks the "accidental re-sort" bug from Section 9 — keep the `OVER` text identical so Postgres shares one sort. **Verdict:** best fluent-ish option; drop to a full `sql` string for a named `WINDOW` clause when you have several windows.

### Sequelize

**Can it?** — Not natively. Only via `sequelize.literal()` inside attributes, or a full raw query.

```javascript
const { QueryTypes } = require('sequelize');

const rows = await sequelize.query(
  `SELECT order_id,
          RANK() OVER w AS value_rank,
          SUM(total_amount) OVER w AS running_spend
   FROM orders
   WHERE customer_id = :cid
   WINDOW w AS (PARTITION BY customer_id ORDER BY total_amount DESC)`,
  { replacements: { cid: customerId }, type: QueryTypes.SELECT }
);
```

**Where it breaks:** `literal()` window expressions are opaque strings with no type safety and no plan control; combining them with Sequelize's own generated `GROUP BY`/`ORDER BY` frequently produces surprising extra sorts. **Verdict:** use `sequelize.query()` raw for windows.

### TypeORM

**Can it?** — Via QueryBuilder `addSelect` with a raw window expression, or `query()` raw.

```typescript
const rows = await dataSource
  .getRepository(Order)
  .createQueryBuilder('o')
  .select('o.id', 'order_id')
  .addSelect('ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.created_at DESC)', 'rn')
  .addSelect('SUM(o.total_amount) OVER (PARTITION BY o.customer_id ORDER BY o.created_at DESC)', 'running_spend')
  .where('o.status = :status', { status: 'completed' })
  .getRawMany();
```

**Where it breaks:** no named `WINDOW` support, so you repeat `OVER (...)`; results only come back via `getRawMany()` (no entity hydration for computed window columns). **Verdict:** workable through raw `addSelect`; use `dataSource.query()` for a named `WINDOW` clause.

### Knex.js

**Can it?** — Yes, Knex has explicit window helpers (`rowNumber`, `rank`, `denseRank`, `.over()`, `partitionBy`, `orderBy`) plus `knex.raw()` for anything else.

```javascript
const rows = await knex('orders')
  .select('order_id', 'total_amount')
  .rowNumber('rn', function () {
    this.partitionBy('customer_id').orderBy('created_at', 'desc');
  })
  .select(knex.raw(
    `SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS running_spend`
  ))
  .where('status', 'completed');
```

**Where it breaks:** the fluent window helpers cover ranking well but frames (`ROWS BETWEEN …`) and named `WINDOW` clauses still need `knex.raw()`. **Verdict:** the most window-capable builder; still reach for `knex.raw()` for frames and shared named windows.

### ORM Summary Table

| ORM | Native window support | Named `WINDOW` clause | Frame (`ROWS/RANGE`) | Plan/index control | Verdict |
|---|---|---|---|---|---|
| Prisma | None (`$queryRaw` only) | Raw only | Raw only | None (migrations for index) | Raw SQL |
| Drizzle | `sql` template | No (repeat OVER) | Via `sql` | None | Best typed; watch repeated OVER |
| Sequelize | `literal()` / raw | Raw only | Raw only | None | Raw query |
| TypeORM | Raw `addSelect` | No | Raw | None | Raw addSelect / query |
| Knex | Fluent helpers | `raw` only | `raw` only | None | Most capable builder |

Across all of them: **no ORM lets you control the sort/index strategy** that determines window performance. You must create the pre-sorting index yourself and verify the plan with EXPLAIN regardless of ORM.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given `employees(id, name, salary, department_id, hire_date)`:

Write a query that returns each employee's `name`, `department_id`, `salary`, and their salary rank within their department (highest salary = rank 1). Then state:
1. Which single index would let PostgreSQL run this **without a Sort** node?
2. What `Sort Key` would appear in EXPLAIN if that index did *not* exist?

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines Topics 20, 39, 40)

Given `orders(id, customer_id, total_amount, status, created_at)`:

For **completed** orders in the last 12 months, return per customer: `customer_id`, `order_id`, `created_at`, a running total of `total_amount` ordered oldest-to-newest (`running_spend`), and the 3-order moving average of `total_amount` (`ROWS BETWEEN 2 PRECEDING AND CURRENT ROW`, `moving_avg`). Both window functions must share a single sort. Use a named `WINDOW` clause. Explain in one sentence why the moving average and the running total can share one `WindowAgg`.

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; the naive answer spills)

`audit_logs(id, account_id, amount, ts)` has 60M rows across ~200K accounts, but one system account (`account_id = 0`) owns 25M of those rows. You must produce, per account, a running balance ordered by `ts`.

1. Write the straightforward windowed query.
2. Explain precisely why, even with `PARTITION BY account_id`, this query can still spill for `account_id = 0`, and *which* structure spills (Sort vs tuplestore).
3. Propose the index that removes the Sort, and explain whether it also fixes the tuplestore concern for the giant partition.
4. Propose a mitigation if a single account's partition is genuinely too large to buffer.

```sql
-- Write your query here
```

### Exercise 4 — Interview Simulation

A teammate's reporting query has **four** window functions and EXPLAIN shows **three** distinct `Sort` nodes; the query takes 22 seconds on 8M rows and shows `Sort Method: external merge Disk: 380000kB`.

1. Explain, step by step, how you'd cut this to (ideally) one sort with no disk spill.
2. Which changes are query rewrites, which are indexes, and which are session settings?
3. What EXPLAIN evidence would confirm each fix worked?

```sql
-- Describe your optimisation plan; include any rewritten SQL and DDL.
```

---

## 14. Interview Questions

### Q1. "Where does the cost of a window function actually come from, and how do you reduce it?"

**Junior answer:** "Window functions are slow because they process every row and do calculations like ranking, so you should avoid them on big tables."

**Principal answer:** "The `WindowAgg` node itself is cheap for ranking functions — roughly a counter per row. The cost is almost entirely the **Sort** that feeds it, because `WindowAgg` requires its input already grouped by `PARTITION BY` and ordered by `ORDER BY`, and it cannot sort itself. That sort is `O(N log N)` and, crucially, spills to disk as an external merge sort when the data exceeds `work_mem` — that's the cliff. So I reduce cost in three ways, in order: (1) filter with `WHERE` before the window, since `WHERE` runs first and shrinks what gets sorted; (2) create a B-tree index on `(partition_cols, order_cols)` in matching order and direction, which lets an Index Scan supply pre-sorted input and **removes the Sort node entirely**; and (3) if a sort is unavoidable, raise `work_mem` for that query with `SET LOCAL` so it stays in memory. I confirm each in EXPLAIN — no `Sort` node, or `Sort Method: quicksort` instead of `external merge Disk:`."

**Interviewer follow-up:** "Your index removed the Sort for the main window, but there's a second window with a different `ORDER BY` that still sorts. What now?" *(Expected: accept that distinct orderings require distinct sorts within one scan; try to share `PARTITION BY` so an Incremental Sort handles only the tail; or shrink input first; or split into CTEs if one window is far cheaper on a filtered subset.)*

### Q2. "You have five window functions in one query. How many times does the data get sorted, and can you control it?"

**Junior answer:** "Five functions, so probably five times — that's why it's slow."

**Principal answer:** "It's not about the *number* of functions — it's the number of **distinct window definitions** `(PARTITION BY, ORDER BY, frame)`. All functions that share one definition are computed by a single `WindowAgg` on a single sorted stream, so five functions sharing one window means **one** sort. Five functions with five different orderings means five sorts, stacked as `WindowAgg` nodes each with its own `Sort` beneath. I control it by consolidating windows onto a shared definition wherever the logic allows — and I use a named `WINDOW` clause so the sharing is explicit and I don't accidentally introduce a second ordering with a copy-paste typo like a missing `DESC`. Even when two windows must differ in `ORDER BY`, keeping a common `PARTITION BY` lets PostgreSQL use an **Incremental Sort** that reuses the partition ordering and only re-sorts within partitions. I verify by counting `Sort` nodes and distinct `Sort Key`s in EXPLAIN."

**Interviewer follow-up:** "Does the `WINDOW` clause itself make the query faster?" *(Expected: no — two identical inline `OVER` clauses already share a sort; the `WINDOW` clause is DRY/readability and bug-prevention, plan-identical to identical inline windows.)*

### Q3. "A window query was fast for months, then suddenly takes 40 seconds. Nothing in the code changed. What happened and how do you confirm it?"

**Junior answer:** "Maybe the server is under load, or the table needs `VACUUM`."

**Principal answer:** "The most likely cause is that the data volume crossed the `work_mem` boundary, so the feeding Sort flipped from an in-memory `quicksort` to an external merge sort writing temp files — a step-change, not gradual. I confirm with `EXPLAIN (ANALYZE, BUFFERS)`: I look for `Sort Method: external merge Disk: NkB` and `temp read/written` in the Buffers line. If that's it, I have three fixes: add a pre-sorting index on `(partition, order)` to remove the Sort entirely; add or tighten a `WHERE` filter so fewer rows reach the sort; or raise `work_mem` for this query via `SET LOCAL` to keep it in memory — I size it just above the reported `Disk:` value. A secondary possibility is stale statistics causing a bad row estimate and a worse plan, which I'd rule out with `ANALYZE` and by comparing estimated vs actual rows. I'd also check whether a previously-used index became unusable (e.g., a new mixed-direction `ORDER BY`)."

**Interviewer follow-up:** "You raised `work_mem` to 512MB and it's fast now. Any risk?" *(Expected: `work_mem` is per node per connection; under concurrency N connections × multiple sort/hash nodes can exhaust RAM and cause OOM — so scope it with `SET LOCAL` to the heavy query rather than raising it globally.)*

---

## 15. Mental Model Checkpoint

1. A query has `ROW_NUMBER() OVER (PARTITION BY a ORDER BY b)` and no index. Describe the three plan nodes from the bottom up and say which one dominates runtime at 10M rows.

2. You add `CREATE INDEX ON t (a, b)`. What node disappears from the plan, and why can an Index Scan legally replace it?

3. Two window functions use `OVER (PARTITION BY a ORDER BY b DESC)` and `OVER (PARTITION BY a ORDER BY b)` respectively. How many Sort nodes appear, and what one-character change would collapse it to one?

4. Explain why adding `LIMIT 10` to a `RANK() OVER (ORDER BY score)` query does **not** avoid sorting the whole table.

5. A running total `SUM(x) OVER (ORDER BY ts ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)` over 50M rows spills. Is it the Sort or the WindowAgg tuplestore that spills? How does the answer change for `RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`?

6. Why is `work_mem` dangerous to raise globally, and what is the correct scope for bumping it for one heavy window report?

7. You have `WHERE status = 'completed'` and `OVER (PARTITION BY customer_id ORDER BY created_at DESC)`. What single composite index serves both the filter and the window ordering, and in what column order?

---

## 16. Quick Reference Card

```sql
-- ── The one law of window performance ──────────────────────────────
-- WindowAgg NEVER sorts. Its cost = the Sort (or Index Scan) feeding it.
-- Optimise the SORT, not the function.

-- Plan shape (no index):     WindowAgg -> Sort -> Seq Scan
-- Plan shape (with index):   WindowAgg -> Index Scan            (no Sort!)

-- ── Kill the Sort with a pre-sorting index ─────────────────────────
CREATE INDEX ON t (partition_col, order_col DESC);   -- match cols + direction
--   PARTITION BY cols first, then ORDER BY cols, matching directions.
--   Mixed directions need exact encoding: (a ASC, b DESC).

-- ── Filter first: WHERE runs BEFORE the window ─────────────────────
SELECT ..., ROW_NUMBER() OVER (PARTITION BY a ORDER BY b)
FROM t
WHERE created_at >= now() - interval '30 days';   -- shrinks the sort input

-- ── Multiple windows: 1 sort per DISTINCT (PARTITION BY, ORDER BY) ──
-- SHARE a definition => ONE sort. Named WINDOW makes sharing explicit:
SELECT RANK() OVER w, AVG(x) OVER w, SUM(x) OVER w
FROM t
WINDOW w AS (PARTITION BY a ORDER BY b DESC);        -- one Sort, one WindowAgg
-- Different ORDER BY => extra Sort. Same PARTITION BY => Incremental Sort helps.

-- ── work_mem: the spill governor (per node, per connection!) ───────
SET LOCAL work_mem = '256MB';   -- scope to ONE transaction; size just above Disk: N

-- ── EXPLAIN red flags ──────────────────────────────────────────────
-- Sort Method: external merge  Disk: NkB   -> raise work_mem OR add index
-- multiple Sort nodes, different Sort Keys  -> consolidate windows / index
-- Sort directly under WindowAgg             -> missing (partition, order) index
-- Incremental Sort ... Presorted Key: ...   -> good: index gave partial order

-- ── Memory by frame ────────────────────────────────────────────────
-- ROWS UNBOUNDED PRECEDING..CURRENT ROW   -> streams, O(1) state (cheap)
-- RANGE UNBOUNDED PRECEDING..UNBOUNDED FOLLOWING -> buffers WHOLE partition
-- Big single partition + whole-partition frame -> tuplestore spills

-- ── Gotchas ────────────────────────────────────────────────────────
-- LIMIT does NOT push through WindowAgg (ranks need the whole partition).
-- WINDOW clause = DRY only; plan-identical to identical inline OVER clauses.
-- Index on SELECTed column ≠ index on window ORDER BY column (useless).
```

**Interview one-liners:**
- "The window's cost is the sort feeding it — `WindowAgg` can't sort itself."
- "One sort per *distinct* window definition; sharing a definition shares the sort."
- "A `(partition, order)` index deletes the Sort node; that's the biggest win."
- "`WHERE` runs before the window — filter first to shrink the sort."
- "`external merge Disk:` in EXPLAIN means the sort spilled — index it or `SET LOCAL work_mem`."
- "`work_mem` is per node per connection; scope bumps with `SET LOCAL`, never globally under concurrency."
- "`LIMIT` doesn't rescue a ranking window — the full partition must be ordered first."

---

## Connected Topics

- **Topic 39 — Frame Clauses** (previous): `ROWS`/`RANGE`/`GROUPS BETWEEN` semantics — the frame directly determines what the `WindowAgg` tuplestore must buffer and therefore whether it spills.
- **Topic 41 — Top-N Per Group** (next): when you want only the top rows per partition, a full-table window is the wrong tool — `LATERAL` + `LIMIT` or a filtered `ROW_NUMBER()` in a subquery avoids sorting everything.
- **Topic 17 — JOIN Performance Deep Dive**: the same `work_mem` / spill mechanics govern Hash Joins and Sort-Merge Joins; the external-merge-sort cliff is identical.
- **Topic 20 — GROUP BY Fundamentals**: `GroupAggregate` also relies on sorted input and can share the same pre-sorting index strategy; contrast with `WindowAgg` which keeps every row.
- **Topic 30-ish — Indexing / B-tree internals**: why a B-tree yields rows in sorted order and how column order and direction in a composite index decide whether the Sort can be eliminated.
- **Internals — `work_mem`, tuplestore, external merge sort**: the shared machinery beneath sorts, hashes, and window buffering; understanding it explains every spill in this topic.
- **Topic 07 — NULL in Depth**: `NULLS FIRST/LAST` ordering in the window `ORDER BY` must match the index's null ordering for the index to eliminate the Sort.
