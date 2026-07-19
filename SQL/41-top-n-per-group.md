# Topic 41 — Top-N Per Group
### SQL Mastery Curriculum — Phase 6: Window Functions — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine a school with 40 classrooms. At the end of term, the principal walks the corridor and stops at each classroom door. In each room she asks: **"Who are your top 3 students by grade? Line them up."** She writes those three names on her clipboard, then moves to the next room. She does this 40 times.

What she is *not* doing is dumping all 1,200 students into the gymnasium, sorting the entire school from best to worst, and taking the first 3 overall — that would give her the top 3 students *in the whole school*, probably all from the same gifted class, and 39 classrooms would go unrepresented.

The difference between those two behaviours is the entire subject of this topic:

- **Global Top-N**: `ORDER BY grade DESC LIMIT 3` — one ranking, one podium, the rest of the school ignored.
- **Top-N per group**: for *each* classroom independently, find that room's own top 3. Forty separate podiums, one per group.

"Top-N per group" (also called "greatest-N-per-group") means: partition the data into groups, rank within each group independently, and keep the top N of *every* group. It is one of the most common — and most subtly mis-solved — problems in all of SQL. It shows up as "the 3 most recent orders per customer," "the highest-paid employee per department," "the best-selling product per category," "the latest login per user." The shape is always the same: **group, rank inside the group, filter to the top slice.**

The reason it earns a whole topic is that SQL gives you at least four genuinely different ways to express it, and they diverge wildly in correctness (how they handle ties), in readability, and — most importantly for a principal engineer — in performance, sometimes by two orders of magnitude on the same data.

---

## 2. Connection to SQL Internals

At the engine level, Top-N per group is a collision of three primitives you have already met, plus one you may not have:

1. **Partitioning** — the `PARTITION BY` clause of a window function (Topic 37) tells the executor to reset its per-group state at every group boundary. Internally, PostgreSQL's `WindowAgg` node consumes an input stream that is *already sorted* by `(partition_key, order_key)` and walks it once, emitting a rank that resets whenever the partition key changes. No hash table of groups is built for the ranking itself — it relies on sort order.

2. **Sorting** — every approach reduces to "sort within each group." The `Sort` node (or an `Index Scan` that supplies pre-sorted rows) is the beating heart of the plan. Whether that sort is a top-N heapsort (bounded, cheap) or a full quicksort/external merge sort (unbounded, expensive) is the single biggest performance lever in this topic.

3. **The B-tree index as a pre-sorted stream** — a composite index on `(group_key, sort_key DESC)` is *physically* a Top-N-per-group answer waiting to be read. Walking that index yields rows already grouped and already ordered; the engine can skip from one group's best row to the next group's best row. PostgreSQL does not have a dedicated "loose index scan" / "skip scan" operator (MySQL 8+ and Oracle do), so we *emulate* it — that emulation is a headline technique of this topic.

4. **`DISTINCT ON`** — a PostgreSQL-specific extension that maps to a special `Unique` node sitting on top of a sort. It reads the sorted stream and emits the first row of each `DISTINCT ON` key group, discarding the rest. It is Top-1-per-group expressed as a scan-and-dedupe rather than a ranking.

5. **`LATERAL`** — a correlated join (Topic 15) where the right-hand subquery re-executes once per left-hand row. Internally this is a **Nested Loop** whose inner side is a parameterised subplan, typically an index scan with a `LIMIT`. This is where the loose-index-scan emulation lives: a tiny table of distinct group keys on the outer side, a `LIMIT N` index probe on the inner side.

The recurring internal theme: **the ranking is free if the rows arrive pre-sorted, and ruinous if they must be materialised and sorted in full.** Everything in the performance section flows from that one sentence.

---

## 3. Logical Execution Order Context

```
FROM / JOIN            ← tables and LATERAL correlations resolved here
WHERE                  ← row-level filters BEFORE any ranking
GROUP BY               ← (only if aggregating; Top-N per group usually does NOT group here)
HAVING
SELECT                 ← window functions (ROW_NUMBER/RANK/DENSE_RANK) computed HERE
DISTINCT / DISTINCT ON ← runs AFTER SELECT-list window functions are computed
ORDER BY
LIMIT
```

The single most important consequence of this ordering, and the number-one Top-N-per-group bug, is:

> **Window functions are computed in the SELECT phase, which runs *after* WHERE. You therefore cannot filter on a window function's result in the same query's WHERE clause.**

```sql
-- ILLEGAL — rn does not exist yet when WHERE runs
SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
FROM orders
WHERE rn <= 3;                       -- ERROR: column "rn" does not exist
```

You must wrap the ranking in a subquery or CTE so the window function is fully computed in the inner query's SELECT, and *then* filter in the outer query's WHERE. That is why the `ROW_NUMBER`-in-a-CTE pattern always has two levels.

`DISTINCT ON` sidesteps this entirely because it is not a window function — it is a post-SELECT deduplication step, so it needs no wrapping. `LATERAL` and the correlated subquery also sidestep it because they push the `LIMIT`/`ORDER BY` into an inner scope that completes before the outer row is emitted. Four approaches, four different relationships to this execution order — that is precisely why they exist.

---

## 4. What Is Top-N Per Group?

Top-N per group returns, for each distinct value of a grouping key, the N rows that rank highest (or lowest) within that group according to some ordering key. When N = 1 it is the "greatest-per-group" problem; when N > 1 it is the general "Top-N per group."

The canonical, tie-safe, N-generalised form uses `ROW_NUMBER()` inside a subquery:

```sql
SELECT customer_id, id, total_amount, created_at
FROM (
  SELECT
    o.*,                                    -- │ carry every column forward for the outer filter
    ROW_NUMBER() OVER (                     -- │ assign a per-row rank within each partition
      PARTITION BY o.customer_id            -- │ └── reset the counter at every new customer_id
      ORDER BY o.created_at DESC,           -- │ └── highest-ranked = most recent order
               o.id DESC                    -- │ └── tie-breaker: guarantees a deterministic 1,2,3
    ) AS rn                                  -- │ the computed rank, 1 = best in its group
  FROM orders o
  WHERE o.status = 'completed'              -- │ pre-ranking filter — runs BEFORE ROW_NUMBER
) ranked                                     -- │ the subquery must exist so rn is materialised
WHERE ranked.rn <= 3                         -- │ └── keep only the top 3 of EACH customer
ORDER BY customer_id, rn;                     -- │ present grouped and ranked
```

Annotated breakdown of the window clause:

```
ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC, id DESC)
│            │     │                         │
│            │     │                         └── ORDER BY: defines "best" within the group;
│            │     │                             the tie-breaker column makes it total-ordered
│            │     └── PARTITION BY: the grouping key; the rank resets to 1 per distinct value
│            └── OVER: opens the window specification
└── ROW_NUMBER(): a rank with NO ties — always 1,2,3,… even when the ORDER BY values are equal
```

The three ranking functions differ *only* in tie behaviour, and choosing the wrong one is a correctness bug, not a style choice:

| Function       | Values 90,90,80,70 rank as | "Top 2" returns | Use when                                        |
|----------------|----------------------------|-----------------|-------------------------------------------------|
| `ROW_NUMBER()` | 1,2,3,4                    | exactly 2 rows  | you need *exactly* N rows, ties broken arbitrarily |
| `RANK()`       | 1,1,3,4                    | 2 rows (both 90s)| you want "the top N *scores*," ties included    |
| `DENSE_RANK()` | 1,1,2,3                    | 3 rows (90,90,80)| you want "the top N *distinct* values"          |

---

## 5. Why Top-N Per Group Mastery Matters in Production

1. **It is the most common analytical request that has no keyword.** SQL has `LIMIT` for global Top-N but nothing for per-group Top-N. Every dashboard ("latest 5 events per device"), every feed ("top 3 comments per post"), every leaderboard ("best score per player per level") is this problem. If you cannot write it four ways and pick the right one, you will reach for the wrong one.

2. **The naive solutions are catastrophically slow at scale.** The correlated-subquery form and the "self-join and count" form are O(N²)-flavoured and can turn a 40ms query into a 40-second query on 10M rows. Knowing *which* form the planner can turn into an index-driven plan is the difference between a query that scales and one that pages you at 3am.

3. **Tie handling is a silent correctness bug.** Using `ROW_NUMBER` when the business wants "everyone tied for the top spot" (needs `RANK`) drops legitimate rows with no error. Using `RANK` when a downstream join expects exactly one row per group causes fan-out. These bugs pass code review and unit tests and surface as "why is this customer missing from the report?"

4. **The loose-index-scan emulation is a genuine principal-level lever.** PostgreSQL cannot skip-scan natively, so a `PARTITION BY` over millions of rows re-reads and re-sorts the whole table even when only 3 rows per group survive. The `LATERAL`/recursive-CTE emulation converts that full scan into a handful of index probes — a technique most engineers have never seen and which reliably impresses in interviews and in production.

5. **`DISTINCT ON` is a PostgreSQL superpower that portability-minded developers under-use.** For the extremely common N=1 case ("the latest row per group"), `DISTINCT ON` is the shortest, often the fastest, and the most readable answer — but it is non-standard, so knowing exactly when to reach for it (and when to avoid it for cross-database code) is part of mastery.

---

## 6. Deep Technical Content

### 6.1 Approach A — `ROW_NUMBER()` in a CTE/subquery (the general workhorse)

This is the approach that always works, generalises to any N, and is portable to every modern database (SQL Server, MySQL 8+, Oracle, SQLite 3.25+).

```sql
WITH ranked AS (
  SELECT
    o.*,
    ROW_NUMBER() OVER (
      PARTITION BY o.customer_id
      ORDER BY o.created_at DESC, o.id DESC
    ) AS rn
  FROM orders o
)
SELECT customer_id, id, total_amount, created_at
FROM ranked
WHERE rn <= 3
ORDER BY customer_id, rn;
```

Mechanics:
- The inner query computes `rn` for **every** row in `orders`. This is the key cost: even though you only want 3 rows per customer, the window function must be evaluated over the *entire* input, because it cannot know a row is rank 4 without having ranked rows 1–3 first.
- The executor sorts the whole table by `(customer_id, created_at DESC, id DESC)`, then the `WindowAgg` node walks it once assigning `rn`.
- The outer `WHERE rn <= 3` is a cheap filter applied after ranking.

**When to use it**: N > 1, you need portability, you need exact-N semantics, or you also need `RANK`/`DENSE_RANK` tie semantics (just swap the function). It is the default answer in an interview when you do not yet know the data distribution.

**Its weakness**: it always scans and sorts the full partition set. There is no early exit. On 100M rows with 1M groups, you sort 100M rows to keep 3M. Section 10 shows how bad this gets and when the `LATERAL` form wins instead.

### 6.2 Approach B — `LATERAL` join (the index-driven scaler)

`LATERAL` (Topic 15) lets a subquery on the right of a join reference columns from the left. We drive the query from a small set of **distinct group keys** and, for each one, run a tiny `ORDER BY ... LIMIT N` index scan.

```sql
SELECT c.id AS customer_id, top.id, top.total_amount, top.created_at
FROM customers c
CROSS JOIN LATERAL (
  SELECT o.id, o.total_amount, o.created_at
  FROM orders o
  WHERE o.customer_id = c.id           -- correlation: re-run per customer
    AND o.status = 'completed'
  ORDER BY o.created_at DESC, o.id DESC
  LIMIT 3                              -- the magic: stop after 3 per group
) top;
```

Mechanics:
- The outer side yields one row per customer (the group keys — ideally from a small `customers` table, not a `SELECT DISTINCT` over the big table).
- For each customer, the inner subquery is executed. With an index on `orders(customer_id, created_at DESC, id DESC)`, that inner query is an **Index Scan** that walks straight to this customer's rows in the right order and stops after reading 3. It never touches this customer's 4th+ rows.
- Total work ≈ (number of groups) × (cost of a 3-row index probe). For 1M customers each with 100 orders, this reads ~3M index entries instead of sorting 100M rows.

**When to use it**: many groups, each group large, and you have (or can create) a composite index leading with the group key. This is the loose-index-scan emulation for N > 1. It is the single most important scaling trick in this topic.

**Use `CROSS JOIN LATERAL`** when every group has at least one row and you want to drop empty groups; use `LEFT JOIN LATERAL ... ON true` when you want to keep groups that have zero matching rows (they come back with NULLs).

```sql
SELECT c.id AS customer_id, top.id, top.total_amount
FROM customers c
LEFT JOIN LATERAL (
  SELECT o.id, o.total_amount
  FROM orders o
  WHERE o.customer_id = c.id
  ORDER BY o.created_at DESC
  LIMIT 3
) top ON true;                         -- customers with no orders still appear (NULL id)
```

### 6.3 Approach C — `DISTINCT ON` (PostgreSQL, the N=1 champion)

`DISTINCT ON (expr)` keeps the **first row** of each group defined by `expr`, where "first" is decided by `ORDER BY`. It is the most concise possible spelling of Top-**1**-per-group.

```sql
SELECT DISTINCT ON (o.customer_id)
       o.customer_id, o.id, o.total_amount, o.created_at
FROM orders o
WHERE o.status = 'completed'
ORDER BY o.customer_id,               -- MUST lead with the DISTINCT ON expression(s)
         o.created_at DESC,           -- then the "best row" ordering within the group
         o.id DESC;                   -- tie-breaker for determinism
```

Iron rule: **the `ORDER BY` must begin with exactly the `DISTINCT ON` expressions**, in the same order, before any additional ordering columns. If it does not, PostgreSQL raises `SELECT DISTINCT ON expressions must match initial ORDER BY expressions`.

Mechanics:
- The engine sorts by `(customer_id, created_at DESC, id DESC)`, then a `Unique` node walks the sorted stream emitting the first row of each `customer_id` run and skipping the rest.
- With an index on `(customer_id, created_at DESC)`, the sort disappears entirely and it becomes an index scan feeding a `Unique` node — extremely fast.

**Fundamental limitation**: `DISTINCT ON` gives you **N=1 only**. There is no `DISTINCT ON ... LIMIT 3 per group`. If you need N > 1, you must use Approach A or B. Do not contort `DISTINCT ON` to fake N > 1 — it cannot.

**Ordering of the final result**: because `DISTINCT ON` forces a specific `ORDER BY`, if you want the *output* ordered differently (e.g., by amount) you must wrap it:

```sql
SELECT * FROM (
  SELECT DISTINCT ON (customer_id) customer_id, id, total_amount, created_at
  FROM orders
  ORDER BY customer_id, created_at DESC
) latest
ORDER BY total_amount DESC;           -- re-order the deduped result
```

### 6.4 Approach D — Correlated subquery (the portable Top-1, usually the slow one)

Before window functions existed, greatest-per-group was written with a correlated subquery in `WHERE`:

```sql
-- Latest order per customer, correlated-subquery style
SELECT o.customer_id, o.id, o.total_amount, o.created_at
FROM orders o
WHERE o.created_at = (
  SELECT MAX(o2.created_at)
  FROM orders o2
  WHERE o2.customer_id = o.customer_id   -- correlation
);
```

Or the count-based form that generalises to N (the classic "how many rows are better than me?"):

```sql
-- Top 3 per customer via "count how many are ahead of me"
SELECT o.customer_id, o.id, o.total_amount, o.created_at
FROM orders o
WHERE (
  SELECT COUNT(*)
  FROM orders o3
  WHERE o3.customer_id = o.customer_id
    AND (o3.created_at, o3.id) > (o.created_at, o.id)  -- rows strictly "ahead"
) < 3;                                                  -- fewer than 3 ahead ⇒ I'm top 3
```

Mechanics and hazards:
- The `MAX` form is a clean Top-1 and can be reasonably fast *if* an index on `(customer_id, created_at)` exists — the subquery becomes an index-only `MAX` lookup. But it has two correctness traps: (a) if two rows share the max `created_at`, **both** are returned (unintended fan-out); (b) it re-scans `orders` for the outer and again for the subquery.
- The **count-based N form is a performance trap.** For each of the N outer rows, it runs a correlated `COUNT(*)` that scans the group. This is O(rows × group_size) — quadratic within each group. On large groups it is dramatically slower than every other approach. It exists here so you can *recognise and reject it* in review, and explain *why* it is slow in an interview.

**When to use it**: almost never in modern PostgreSQL — window functions and `DISTINCT ON` dominate. Its value is historical/portability (very old databases without window functions) and as the answer to "why is this legacy query slow?"

### 6.5 Handling ties correctly — the decision that changes the row count

Ties are the correctness heart of this topic. Consider a department where the top two salaries are 90k, 90k, then 80k, and you ask for the "top 2 earners."

```sql
-- ROW_NUMBER: exactly 2 rows, one of the 90k earners dropped ARBITRARILY
ROW_NUMBER() OVER (PARTITION BY department_id ORDER BY salary DESC)  -- 1,2,3 → keep rn<=2

-- RANK: BOTH 90k earners (rank 1,1), 80k (rank 3) excluded → 2 rows, but "fair"
RANK() OVER (PARTITION BY department_id ORDER BY salary DESC)        -- 1,1,3 → keep rank<=2

-- DENSE_RANK: both 90k AND the 80k → 3 rows ("top 2 distinct salary levels")
DENSE_RANK() OVER (PARTITION BY department_id ORDER BY salary DESC)  -- 1,1,2 → keep <=2
```

The rule of thumb:
- Need **exactly N rows**, don't care who wins a tie → `ROW_NUMBER` **with an explicit tie-breaker** in `ORDER BY` (so "arbitrary" becomes "deterministic and reproducible").
- Need **the top N ranks, ties inclusive** → `RANK`.
- Need **the top N distinct values** → `DENSE_RANK`.

**Always add a tie-breaker to `ROW_NUMBER`.** Without one, `ORDER BY salary DESC` alone leaves the 1-vs-2 assignment between the two 90k rows up to the sort's physical whim — which can change between runs, between PostgreSQL versions, and after a `VACUUM`. `ORDER BY salary DESC, id DESC` makes it deterministic.

### 6.6 The empty-group and NULL cases

- **`PARTITION BY` on a NULL group key**: all rows with `NULL` group key form **one** partition (NULLs are considered equal for partitioning, unlike in `=` comparisons). They get ranked together. This surprises people who expect NULLs to each be their own group.
- **NULL in the `ORDER BY` key**: `NULLS LAST`/`NULLS FIRST` controls where they land. `ORDER BY created_at DESC` puts NULLs *first* by default (DESC ⇒ NULLS FIRST), so a row with `created_at = NULL` could grab rank 1. Be explicit: `ORDER BY created_at DESC NULLS LAST`.
- **Empty groups**: a `ROW_NUMBER`/`DISTINCT ON` query simply never emits a row for a group with no rows — the group vanishes. If you must show it (e.g., "all customers, even those with zero orders"), you need `LEFT JOIN LATERAL ... ON true` (6.2) or a `LEFT JOIN` from a group-key table.

### 6.7 N per group where N varies by group

A rare but real requirement: "top 3 for gold customers, top 1 for others." `ROW_NUMBER` handles it with a join to a per-group N table:

```sql
WITH ranked AS (
  SELECT o.*, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY total_amount DESC) AS rn
  FROM orders o
)
SELECT r.*
FROM ranked r
JOIN customers c ON c.id = r.customer_id
WHERE r.rn <= CASE c.tier WHEN 'gold' THEN 3 ELSE 1 END;
```

`DISTINCT ON` and the simple `LATERAL LIMIT` cannot express a variable N without a `CASE` in the `LIMIT` (which `LATERAL` *can* do, since `LIMIT` accepts an expression referencing the outer row) — another reason `LATERAL` is so flexible.

### 6.8 Composite ordering keys and multi-column partitions

Both `PARTITION BY` and `ORDER BY` accept multiple columns. "Best product per (category, region)" partitions by two columns; "most recent, then highest value" orders by two. The index that accelerates it must lead with **all** partition columns, then the order columns: `(category_id, region_id, sold_at DESC)`.

---

## 7. EXPLAIN — Top-N Patterns in the Plan

### 7.1 `ROW_NUMBER` in a CTE — the full-sort plan (no helpful index)

```sql
EXPLAIN (ANALYZE, BUFFERS)
WITH ranked AS (
  SELECT o.*, ROW_NUMBER() OVER (PARTITION BY o.customer_id
                                 ORDER BY o.created_at DESC, o.id DESC) AS rn
  FROM orders o
)
SELECT customer_id, id, total_amount FROM ranked WHERE rn <= 3;
```

```
Subquery Scan on ranked  (cost=1128000..1503000 rows=5000000 width=24)
                         (actual time=6820..9950 rows=3000000 loops=1)
  Filter: (ranked.rn <= 3)
  Rows Removed by Filter: 12000000
  ->  WindowAgg  (cost=1128000..1428000 rows=15000000 width=32)
                 (actual time=6820..9480 rows=15000000 loops=1)
        ->  Sort  (cost=1128000..1165500 rows=15000000 width=24)
                  (actual time=6820..8120 rows=15000000 loops=1)
              Sort Key: o.customer_id, o.created_at DESC, o.id DESC
              Sort Method: external merge  Disk: 512000kB
              ->  Seq Scan on orders o  (cost=0..305000 rows=15000000 width=24)
                                        (actual time=0.02..1420 rows=15000000 loops=1)
Buffers: shared hit=2100 read=178900, temp read=64000 written=64000
Planning Time: 0.3 ms
Execution Time: 10120 ms
```

Line-by-line:
- `Seq Scan on orders` reads all 15M rows — no filter, everything feeds the ranking.
- `Sort` orders all 15M rows by the window's `(partition, order)` key. `Sort Method: external merge Disk: 512000kB` is the alarm: the sort **spilled to disk** (`work_mem` too small for 15M rows), and `temp read/written=64000` blocks of temp I/O confirm it.
- `WindowAgg` assigns `rn` to all 15M rows.
- The outer `Filter: rn <= 3` throws away 12M rows *after* all that work — `Rows Removed by Filter: 12000000`. We sorted 15M rows to keep 3M.
- `Execution Time: 10120 ms` — ten seconds, dominated by the disk sort.

This is the plan to *recognise and fear*: a full Seq Scan feeding a disk-spilling Sort feeding a WindowAgg, with most rows discarded afterward.

### 7.2 `LATERAL` with a supporting index — the index-probe plan

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.id, top.id, top.total_amount
FROM customers c
CROSS JOIN LATERAL (
  SELECT o.id, o.total_amount, o.created_at
  FROM orders o
  WHERE o.customer_id = c.id
  ORDER BY o.created_at DESC, o.id DESC
  LIMIT 3
) top;
-- Index present: CREATE INDEX ON orders (customer_id, created_at DESC, id DESC);
```

```
Nested Loop  (cost=0.56..820000 rows=600000 width=24)
             (actual time=0.03..410 rows=560000 loops=1)
  ->  Seq Scan on customers c  (cost=0..3600 rows=200000 width=8)
                               (actual time=0.01..22 rows=200000 loops=1)
  ->  Limit  (cost=0.56..2.34 rows=3 width=24)
             (actual time=0.0018..0.0019 rows=2.8 loops=200000)
        ->  Index Scan using orders_customer_created_idx on orders o
                 (cost=0.56..44.0 rows=75 width=24)
                 (actual time=0.0016..0.0017 rows=2.8 loops=200000)
              Index Cond: (o.customer_id = c.id)
Buffers: shared hit=1750400
Planning Time: 0.4 ms
Execution Time: 452 ms
```

Line-by-line:
- Outer `Seq Scan on customers` yields 200,000 group keys — cheap, small table.
- For each customer (`loops=200000`), the inner `Index Scan` uses `orders_customer_created_idx` with `Index Cond: customer_id = c.id`, reading rows already ordered by `created_at DESC, id DESC`.
- The `Limit` node stops each inner scan after **3** rows (`actual rows=2.8` avg — some customers have <3 orders). Critically, it reads *only* those rows — no 4th+ row is touched.
- `Execution Time: 452 ms` — **22× faster** than the full-sort CTE on the same data, and it never spills to disk. This is the loose-index-scan emulation paying off.

The tell of a healthy plan here: `Nested Loop → Limit → Index Scan` with tiny `actual rows` per loop and no `Sort` node anywhere.

### 7.3 `DISTINCT ON` with index — the Unique-over-index plan

```sql
EXPLAIN (ANALYZE)
SELECT DISTINCT ON (customer_id) customer_id, id, total_amount
FROM orders
ORDER BY customer_id, created_at DESC;
-- Index present: CREATE INDEX ON orders (customer_id, created_at DESC);
```

```
Unique  (cost=0.56..612000 rows=200000 width=24)
        (actual time=0.04..2110 rows=200000 loops=1)
  ->  Index Scan using orders_customer_created_idx on orders
           (cost=0.56..575000 rows=15000000 width=24)
           (actual time=0.03..1650 rows=15000000 loops=1)
Execution Time: 2170 ms
```

- No `Sort` node — the index already supplies `(customer_id, created_at DESC)` order.
- `Index Scan` streams all 15M rows in order; the `Unique` node emits the first of each `customer_id` run (200,000 rows) and discards the rest.
- Note it *still reads all 15M index entries* — `DISTINCT ON` cannot skip within a group, so for N=1 over huge groups the `LATERAL` form can still beat it. But no sort and no disk spill make it far better than the naive CTE.

### 7.4 What to look for

| Symptom in plan                                   | Meaning                                      | Fix                                                   |
|---------------------------------------------------|----------------------------------------------|-------------------------------------------------------|
| `Sort Method: external merge Disk: …`             | Full sort spilled to disk                    | Add composite index, or raise `work_mem`, or use LATERAL |
| `Rows Removed by Filter` ≫ rows kept              | Ranked everything, threw most away           | Switch to LATERAL/`DISTINCT ON` for early exit        |
| `WindowAgg` over a `Seq Scan`+`Sort`              | Classic full-scan Top-N-per-group            | Index on `(partition, order)`; consider LATERAL       |
| `Nested Loop`+`Limit`+`Index Scan`, tiny per-loop | Healthy loose-index-scan emulation           | Nothing — this is the goal                            |
| `DISTINCT ON` with a `Sort` node present          | No usable index for the ordering             | Index leading with `(distinct_key, order_key)`        |

---

## 8. Query Examples

### Example 1 — Basic: latest order per customer (N=1)

```sql
-- Simplest greatest-per-group: one most-recent completed order per customer.
-- DISTINCT ON is the shortest correct spelling in PostgreSQL.
SELECT DISTINCT ON (o.customer_id)
       o.customer_id,
       o.id            AS order_id,
       o.total_amount,
       o.created_at
FROM orders o
WHERE o.status = 'completed'
ORDER BY o.customer_id,          -- must lead with the DISTINCT ON key
         o.created_at DESC,      -- "latest" wins
         o.id DESC;              -- deterministic tie-break
```

### Example 2 — Intermediate: top 3 products by revenue per category (N=3)

```sql
-- Rank products within each category by revenue, keep the top 3.
-- ROW_NUMBER in a CTE is the portable, exact-N answer.
WITH product_revenue AS (
  SELECT
    p.category_id,
    p.id            AS product_id,
    p.name          AS product_name,
    SUM(oi.quantity * oi.unit_price) AS revenue
  FROM products p
  INNER JOIN order_items oi ON oi.product_id = p.id
  INNER JOIN orders o       ON o.id = oi.order_id
  WHERE o.status = 'completed'
  GROUP BY p.category_id, p.id, p.name
),
ranked AS (
  SELECT
    pr.*,
    ROW_NUMBER() OVER (
      PARTITION BY pr.category_id
      ORDER BY pr.revenue DESC, pr.product_id DESC   -- tie-break for determinism
    ) AS rn
  FROM product_revenue pr
)
SELECT c.name AS category, product_name, revenue, rn AS rank_in_category
FROM ranked r
INNER JOIN categories c ON c.id = r.category_id
WHERE r.rn <= 3
ORDER BY c.name, r.rn;
```

### Example 3 — Production-Grade: 3 most recent completed orders per customer, index-driven

**Scenario.** `orders` has 15,000,000 rows across ~200,000 active customers (avg 75 orders each; power users have thousands). Requirement: for a nightly export, the 3 most recent completed orders per customer. The naive `ROW_NUMBER`-over-full-table CTE (Section 7.1) takes ~10s and spills 512MB to disk. We want sub-second.

**Index (the enabling structure):**

```sql
CREATE INDEX orders_cust_created_idx
  ON orders (customer_id, created_at DESC, id DESC)
  WHERE status = 'completed';           -- partial index: only the rows we ever rank
```

**Query (LATERAL loose-index-scan emulation):**

```sql
SELECT
  c.id                 AS customer_id,
  top.order_id,
  top.total_amount,
  top.created_at,
  top.rn
FROM customers c
CROSS JOIN LATERAL (
  SELECT
    o.id            AS order_id,
    o.total_amount,
    o.created_at,
    ROW_NUMBER() OVER (ORDER BY o.created_at DESC, o.id DESC) AS rn  -- rank within this customer only
  FROM orders o
  WHERE o.customer_id = c.id
    AND o.status = 'completed'
  ORDER BY o.created_at DESC, o.id DESC
  LIMIT 3
) top
ORDER BY c.id, top.rn;
```

**Performance expectation.** Each inner scan is a 3-row probe of the partial index → ~200,000 probes × ~3 index rows = well under a second, zero disk spill. Measured ~450ms vs ~10,100ms for the CTE — a 22× improvement.

**Its EXPLAIN** (matching Section 7.2): `Nested Loop → Limit → Index Scan using orders_cust_created_idx`, `actual rows≈2.8 per loop`, `loops=200000`, no `Sort`, no temp I/O.

---

## 9. Wrong → Right Patterns

### Wrong 1: Filtering on the window function in the same WHERE

```sql
-- WRONG — window functions are computed in SELECT, AFTER WHERE runs
SELECT customer_id, id, total_amount,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
FROM orders
WHERE rn <= 3;
-- ERROR:  column "rn" does not exist
--   (rn is not visible in WHERE; the ranking hasn't been computed yet — see Section 3)
```

Why it's wrong at the execution level: `WHERE` is evaluated in the logical phase *before* the SELECT list where `ROW_NUMBER` lives. The identifier `rn` simply does not exist at that point. This is not a scoping quirk you can annotate around — it is the execution order.

```sql
-- RIGHT — materialise the rank in a subquery/CTE, filter in the OUTER query
SELECT customer_id, id, total_amount
FROM (
  SELECT customer_id, id, total_amount,
         ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC, id DESC) AS rn
  FROM orders
) ranked
WHERE rn <= 3;
```

### Wrong 2: Using `RANK` (or no tie-breaker) when you need exactly N rows

```sql
-- WRONG — a downstream job expects exactly ONE latest order per customer,
-- but two orders share the max created_at, so RANK gives both rank 1.
SELECT customer_id, id
FROM (
  SELECT customer_id, id,
         RANK() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rk
  FROM orders
) r
WHERE rk = 1;
-- BUG: customers with a same-timestamp tie return 2+ rows → fan-out downstream
```

Why it's wrong: `RANK` is tie-*inclusive* by design. Combined with a non-unique `ORDER BY` key, "rank 1" is not guaranteed to be a single row. The job that assumed one-row-per-customer now double-counts.

```sql
-- RIGHT — ROW_NUMBER with a unique tie-breaker guarantees exactly one row
SELECT customer_id, id
FROM (
  SELECT customer_id, id,
         ROW_NUMBER() OVER (PARTITION BY customer_id
                            ORDER BY created_at DESC, id DESC) AS rn  -- id breaks ties
  FROM orders
) r
WHERE rn = 1;
```

### Wrong 3: The count-based correlated subquery on large groups

```sql
-- WRONG (performance) — quadratic within each group
SELECT o.customer_id, o.id
FROM orders o
WHERE (
  SELECT COUNT(*) FROM orders o2
  WHERE o2.customer_id = o.customer_id
    AND o2.created_at > o.created_at
) < 3;
-- On customers with thousands of orders this runs a COUNT scan per row →
-- O(group_size²). A 15M-row table can take minutes and thrash the buffer pool.
```

Why it's wrong at the execution level: the subquery re-executes once per outer row and scans that row's entire group each time. For a group of size G it does ~G² comparisons; summed over all groups this dwarfs a single sort.

```sql
-- RIGHT — LATERAL turns it into G index probes of 3 rows each (Section 6.2)
SELECT c.id AS customer_id, top.id
FROM customers c
CROSS JOIN LATERAL (
  SELECT o.id FROM orders o
  WHERE o.customer_id = c.id
  ORDER BY o.created_at DESC, o.id DESC
  LIMIT 3
) top;
```

### Wrong 4: `DISTINCT ON` without matching `ORDER BY`, or used for N>1

```sql
-- WRONG — ORDER BY does not start with the DISTINCT ON key
SELECT DISTINCT ON (customer_id) customer_id, id, created_at
FROM orders
ORDER BY created_at DESC;
-- ERROR: SELECT DISTINCT ON expressions must match initial ORDER BY expressions

-- ALSO WRONG — trying to get "top 3 per customer" from DISTINCT ON; it only ever gives 1
SELECT DISTINCT ON (customer_id) ... ;   -- there is no per-group LIMIT here
```

Why it's wrong: `DISTINCT ON` deduplicates to the *first* row per key in the sorted stream — structurally N=1. And PostgreSQL requires the sort to lead with the `DISTINCT ON` expressions so that "first per group" is well-defined.

```sql
-- RIGHT — fix the ORDER BY prefix (N=1) ...
SELECT DISTINCT ON (customer_id) customer_id, id, created_at
FROM orders
ORDER BY customer_id, created_at DESC, id DESC;

-- ... and use ROW_NUMBER / LATERAL when you actually need N>1.
```

### Wrong 5: NULL ordering key silently grabbing rank 1

```sql
-- WRONG — some orders have created_at IS NULL; DESC puts NULLS FIRST,
-- so a null-dated order can win "most recent"
SELECT DISTINCT ON (customer_id) customer_id, id, created_at
FROM orders
ORDER BY customer_id, created_at DESC;      -- NULLS FIRST under DESC → NULL ranks first!

-- RIGHT — be explicit about NULL placement
SELECT DISTINCT ON (customer_id) customer_id, id, created_at
FROM orders
ORDER BY customer_id, created_at DESC NULLS LAST, id DESC;
```

---

## 10. Performance Profile

### 10.1 Cost model of each approach

| Approach                     | Dominant cost                          | Early exit? | Scales with          |
|------------------------------|----------------------------------------|-------------|----------------------|
| `ROW_NUMBER` in CTE          | Full sort of all rows: O(R log R)      | No          | total rows R         |
| `LATERAL` + `LIMIT N` (indexed) | G × (index probe of N rows): O(G·N·log R) | Yes (per group) | number of groups G |
| `DISTINCT ON` (indexed)      | One index scan of all rows: O(R)       | No (reads all, emits G) | total rows R  |
| Correlated `MAX` (indexed)   | R outer × index MAX probe: O(R·log R)  | Partial     | total rows R         |
| Correlated `COUNT` (N-form)  | Σ over groups of G²: O(Σ Gᵢ²)          | No          | group size² — avoid  |

The crossover intuition: **`LATERAL` wins when groups are large and few relative to total rows** (few probes, each stopping early). **`ROW_NUMBER`/`DISTINCT ON` win when groups are tiny and numerous** (nearly every row survives anyway, so a single streaming sort/scan beats millions of index probes). When almost every row is in the Top-N, there is no early exit to exploit and one linear pass is cheapest.

### 10.2 Scaling at 1M / 10M / 100M rows (3 most recent per customer)

Assume ~200k customers throughout; group size grows with the table.

| Rows  | `ROW_NUMBER` CTE (no index)       | `ROW_NUMBER` CTE (indexed)    | `LATERAL` (indexed)      |
|-------|-----------------------------------|-------------------------------|--------------------------|
| 1M    | ~600ms, in-memory sort            | ~350ms (index avoids sort)    | ~90ms                    |
| 10M   | ~6–8s, sort spills to disk        | ~2.5s                         | ~300ms                   |
| 100M  | ~90s+, heavy external merge sort  | ~20s                          | ~1.5s                    |

The `ROW_NUMBER` CTE degrades roughly linearly-plus-log and hits a cliff when the sort no longer fits in `work_mem` (disk spill = 5–10× slowdown). `LATERAL` degrades far more slowly because per-probe cost grows only with the *depth* of the index (log R), not the table size — the number of probes (G) is fixed.

### 10.3 The index that makes or breaks it

The single most important structure for every non-CTE approach:

```sql
-- Covers PARTITION/ORDER for LATERAL, DISTINCT ON, and correlated MAX
CREATE INDEX ON orders (customer_id, created_at DESC, id DESC);

-- Even better for a filtered workload — partial index shrinks the tree
CREATE INDEX ON orders (customer_id, created_at DESC, id DESC)
  WHERE status = 'completed';

-- Covering index: add INCLUDE columns so the heap is never visited (index-only scan)
CREATE INDEX ON orders (customer_id, created_at DESC, id DESC)
  INCLUDE (total_amount);
```

The leading column **must** be the partition/group key; the ordering columns follow in the exact direction of the `ORDER BY`. If the index direction matches the query, PostgreSQL walks it forward with no `Sort` node. `INCLUDE` turns the plan into an **index-only scan**, eliminating heap fetches — the last big win for wide result columns.

### 10.4 `work_mem`, memory, and the disk-spill cliff

- The `ROW_NUMBER` CTE sorts the *entire* input. If that exceeds `work_mem`, PostgreSQL switches to an external merge sort using temp files — the `Disk: NkB` line in EXPLAIN. This is the dominant cost in Section 7.1.
- Raising `work_mem` for that one query (`SET LOCAL work_mem = '512MB'`) can keep the sort in memory and cut time 5–10×, but it is a band-aid: it consumes memory per sort per connection and does nothing about the fundamental "sort everything to keep 3 per group" waste.
- `LATERAL` and indexed `DISTINCT ON` use **no sort memory at all** — they stream from the index. This is why they are the durable fix, not a tuning knob.

### 10.5 CPU and buffer behaviour

- `ROW_NUMBER` CTE: CPU-bound on the sort comparator; buffer-bound reading the whole heap (Seq Scan). High temp I/O when spilling.
- `LATERAL`: buffer-bound on random index+heap access, but touches *far* fewer pages (only Top-N rows per group). Excellent buffer-cache hit ratio on hot customers; can incur random I/O on cold data.
- `DISTINCT ON`: reads the whole index in order (sequential-ish index I/O), light CPU (`Unique` is a cheap comparison). Good middle ground for N=1.

---

## 11. Node.js Integration

### 11.1 `LATERAL` Top-N with `pg`

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// N most recent completed orders per customer, index-driven (Section 8, Example 3).
// N is parameterised into the LATERAL LIMIT — pg binds it as $1.
async function recentOrdersPerCustomer(n = 3) {
  const { rows } = await pool.query(
    `SELECT c.id AS customer_id, top.order_id, top.total_amount, top.created_at
     FROM customers c
     CROSS JOIN LATERAL (
       SELECT o.id AS order_id, o.total_amount, o.created_at
       FROM orders o
       WHERE o.customer_id = c.id
         AND o.status = 'completed'
       ORDER BY o.created_at DESC, o.id DESC
       LIMIT $1
     ) top
     ORDER BY c.id, top.created_at DESC`,
    [n]
  );
  return rows;
}
```

Note: `LIMIT $1` works because PostgreSQL accepts an expression in `LIMIT`, and inside `LATERAL` it can even reference the outer row (e.g. `LIMIT c.top_n`) for per-group variable N (Section 6.7).

### 11.2 `ROW_NUMBER` CTE with an outer rank filter

```javascript
// Portable exact-N (works if you ever migrate off Postgres). Filter rn in the outer query.
async function topProductsPerCategory(topN = 3) {
  const { rows } = await pool.query(
    `WITH ranked AS (
       SELECT p.category_id, p.id AS product_id, p.name,
              ROW_NUMBER() OVER (PARTITION BY p.category_id
                                 ORDER BY p.sales DESC, p.id DESC) AS rn
       FROM products p
     )
     SELECT category_id, product_id, name, rn
     FROM ranked
     WHERE rn <= $1
     ORDER BY category_id, rn`,
    [topN]
  );
  return rows;
}
```

### 11.3 `DISTINCT ON` for the latest-row-per-group case

```javascript
// The single latest completed order per customer — shortest correct query in Postgres.
async function latestOrderPerCustomer() {
  const { rows } = await pool.query(
    `SELECT DISTINCT ON (o.customer_id)
            o.customer_id, o.id AS order_id, o.total_amount, o.created_at
     FROM orders o
     WHERE o.status = 'completed'
     ORDER BY o.customer_id, o.created_at DESC, o.id DESC`
  );
  return rows;
}
```

### 11.4 A note on ORMs

Every mainstream Node ORM can express the `ROW_NUMBER` CTE only through raw SQL (window functions are largely unsupported in their fluent APIs), and `DISTINCT ON` / `LATERAL` are PostgreSQL-specific extensions that most ORMs cannot generate at all. For Top-N per group you will almost always drop to the raw-SQL escape hatch — `pool.query`, `prisma.$queryRaw`, `db.execute(sql\`…\`)`, `knex.raw`. Treat this as expected, not a failure of the ORM. Section 12 covers exactly where each one breaks.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do Top-N per group?** — Not in the fluent API. Prisma has no window functions, no `DISTINCT ON`, and no `LATERAL`. Its `distinct` option deduplicates in the client layer by key but cannot order-then-pick-first per group the way `DISTINCT ON` does, and it offers no per-group `take`.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

// The ONLY reliable way: raw SQL. Use LATERAL for scale.
const topOrders = await prisma.$queryRaw<
  { customer_id: number; order_id: number; total_amount: string }[]
>(Prisma.sql`
  SELECT c.id AS customer_id, top.order_id, top.total_amount
  FROM customers c
  CROSS JOIN LATERAL (
    SELECT o.id AS order_id, o.total_amount
    FROM orders o
    WHERE o.customer_id = c.id AND o.status = 'completed'
    ORDER BY o.created_at DESC, o.id DESC
    LIMIT ${3}
  ) top
`);
```

**Where it breaks**: `findMany({ distinct: ['customerId'], orderBy: … , take: … })` looks like it should give latest-per-customer, but `take` limits the *total* result set, not per group — a classic silent bug. **Verdict**: use `$queryRaw` with `LATERAL` or `DISTINCT ON`; do not fight the fluent API.

---

### Drizzle ORM

**Can Drizzle do Top-N per group?** — Partially. Drizzle can express `ROW_NUMBER()` via its window-function helpers and can subquery, so the CTE approach is buildable in typed code. `LATERAL` and `DISTINCT ON` need the `sql` escape or `.$dynamic()` in most versions.

```typescript
import { db } from './db';
import { orders } from './schema';
import { sql, desc } from 'drizzle-orm';

const ranked = db.$with('ranked').as(
  db.select({
    customerId: orders.customerId,
    id: orders.id,
    totalAmount: orders.totalAmount,
    rn: sql<number>`ROW_NUMBER() OVER (
      PARTITION BY ${orders.customerId}
      ORDER BY ${orders.createdAt} DESC, ${orders.id} DESC)`.as('rn'),
  }).from(orders)
);

const result = await db.with(ranked)
  .select().from(ranked)
  .where(sql`${ranked.rn} <= 3`);
```

**Where it breaks**: the window expression itself is a `sql` template — no type-checking of the `PARTITION BY`/`ORDER BY` columns. `LATERAL` requires raw `sql`. **Verdict**: best typed story of the group for the `ROW_NUMBER` CTE; still `sql` for `LATERAL`/`DISTINCT ON`.

---

### Sequelize

**Can Sequelize do Top-N per group?** — No first-class support. No window functions, no `DISTINCT ON`, no `LATERAL` in the query API. `sequelize.literal` can inject a window expression as a virtual attribute, but you still cannot filter on it without a raw subquery.

```javascript
const [rows] = await sequelize.query(
  `SELECT DISTINCT ON (customer_id) customer_id, id, total_amount, created_at
   FROM orders
   WHERE status = 'completed'
   ORDER BY customer_id, created_at DESC, id DESC`,
  { type: sequelize.QueryTypes.SELECT }
);
```

**Where it breaks**: `limit` on a `findAll` with an `include` limits total rows, never per group — the same trap as Prisma. **Verdict**: use `sequelize.query()` raw; `DISTINCT ON` for N=1, `LATERAL` for N>1 at scale.

---

### TypeORM

**Can TypeORM do Top-N per group?** — Only via QueryBuilder with raw fragments. You can put `ROW_NUMBER()` in `addSelect` and wrap in a second QueryBuilder, but it is verbose; most teams use `.query()`.

```typescript
const rows = await dataSource.query(
  `SELECT c.id AS customer_id, top.id, top.total_amount
   FROM customers c
   CROSS JOIN LATERAL (
     SELECT o.id, o.total_amount
     FROM orders o
     WHERE o.customer_id = c.id
     ORDER BY o.created_at DESC, o.id DESC
     LIMIT $1
   ) top`,
  [3]
);
```

**Where it breaks**: entity hydration does not understand the `LATERAL`-derived columns, so you get raw rows, not managed entities. Filtering on a windowed `addSelect` alias in `.where()` fails for the same execution-order reason as raw SQL (Section 9, Wrong 1). **Verdict**: `dataSource.query()` with `LATERAL`/`DISTINCT ON`.

---

### Knex.js

**Can Knex do Top-N per group?** — Closest of all to raw SQL. `knex.raw` for the window expression, real subquery/CTE support via `.with()`, and `.joinRaw`/`crossJoin` for `LATERAL`.

```javascript
// ROW_NUMBER CTE, fluent-ish
const ranked = knex('orders')
  .select('customer_id', 'id', 'total_amount')
  .select(knex.raw(
    `ROW_NUMBER() OVER (PARTITION BY customer_id
                        ORDER BY created_at DESC, id DESC) AS rn`));

const rows = await knex.with('ranked', ranked)
  .select('*').from('ranked').where('rn', '<=', 3);

// LATERAL via joinRaw
const lateral = await knex('customers as c')
  .joinRaw(
    `CROSS JOIN LATERAL (
       SELECT o.id, o.total_amount FROM orders o
       WHERE o.customer_id = c.id
       ORDER BY o.created_at DESC, o.id DESC
       LIMIT ?
     ) top`, [3])
  .select('c.id as customer_id', 'top.id', 'top.total_amount');
```

**Where it breaks**: nothing fundamental — Knex is a query builder, not an ORM, so it maps cleanly. You just write the window/`LATERAL`/`DISTINCT ON` as raw fragments. **Verdict**: the most transparent; use `.with()`+`knex.raw` for the CTE, `.joinRaw` for `LATERAL`.

---

### ORM Summary Table

| ORM       | `ROW_NUMBER` CTE        | `DISTINCT ON` | `LATERAL`     | Per-group `LIMIT` in fluent API | Verdict                          |
|-----------|-------------------------|---------------|---------------|---------------------------------|----------------------------------|
| Prisma    | `$queryRaw`             | `$queryRaw`   | `$queryRaw`   | No (`take` is global — trap)    | Raw SQL only                     |
| Drizzle   | Typed via `sql` window  | `sql`         | `sql`         | No                              | Best typed CTE; raw for the rest |
| Sequelize | `.query()` raw          | `.query()`    | `.query()`    | No (`limit`+`include` is global)| Raw only                         |
| TypeORM   | QueryBuilder + raw frag | `.query()`    | `.query()`    | No                              | `.query()` recommended           |
| Knex      | `.with()`+`knex.raw`    | `knex.raw`    | `.joinRaw`    | No, but raw is clean            | Most transparent                 |

The universal lesson: **Top-N per group is a raw-SQL feature in the Node ecosystem.** Every fluent `limit`/`take` limits the *total* rows, never per group — memorise that trap.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Tables:
- `employees(id, name, salary, department_id, hired_at)`
- `departments(id, name)`

Write a query returning **the single highest-paid employee in each department** — `department name`, `employee name`, `salary`. Break ties by earliest `hired_at`, then lowest `id`, so exactly one employee per department is returned. Use `DISTINCT ON`.

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines Topics 06 aggregation + 11 INNER JOIN + this topic)

Tables:
- `orders(id, customer_id, total_amount, status, created_at)`
- `order_items(id, order_id, product_id, quantity, unit_price)`
- `products(id, name, category_id)`

For each `category_id`, return the **top 3 products by total units sold** across all completed orders in the last 12 months: `category_id`, `product_name`, `units_sold`, and `rank_in_category` (1–3). Use a `ROW_NUMBER` CTE. Break ties by product name ascending.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

`events(id, device_id, event_type, occurred_at)` has 80,000,000 rows across 500,000 devices. You need the **5 most recent events per device**. A colleague submits:

```sql
WITH r AS (
  SELECT e.*, ROW_NUMBER() OVER (PARTITION BY device_id ORDER BY occurred_at DESC) rn
  FROM events e
)
SELECT * FROM r WHERE rn <= 5;
```

It runs for 90 seconds and spills 3GB to disk in staging.

1. Explain, from the plan's perspective, exactly why it is slow (name the node that dominates).
2. Rewrite it to run in ~1–2 seconds using the loose-index-scan emulation.
3. State the exact index you would create, including direction and any partial/`INCLUDE` clause, and why.
4. What breaks if `occurred_at` can be NULL, and how do you guard it?

```sql
-- Write your rewrite here
-- Write your CREATE INDEX here
```

---

### Exercise 4 — Interview Simulation

You are asked, live: *"Return the 2 most recent orders per customer. Now handle the case where a customer's two most recent orders share the exact same `created_at`. Now make it fast on 50M rows. Now do it a second way that is portable to a database without `DISTINCT ON`."*

Write all pieces: (a) the tie-safe query, (b) the fast index-driven version, (c) the portable version, and name which of your queries you would ship and why.

```sql
-- (a) tie-safe:
-- (b) fast / index-driven:
-- (c) portable:
```

---

## 14. Interview Questions

### Q1 — "Give me the four ways to get the top 3 orders per customer, and tell me which you'd ship."

**Junior answer**: "Use `ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC)` in a subquery and filter `rn <= 3`."

**Principal answer**: "Four approaches: (1) `ROW_NUMBER` in a CTE — portable, exact-N, but sorts the entire table and has no early exit; (2) `LATERAL` with `ORDER BY … LIMIT 3` and an index on `(customer_id, created_at DESC, id DESC)` — emulates a loose index scan, does ~G small index probes, best at scale with large groups; (3) `DISTINCT ON` — only N=1, but the shortest and often fastest for latest-per-group; (4) correlated subquery — legacy, and the count-based N form is quadratic per group, so I reject it. What I ship depends on data shape: for N=1 I use `DISTINCT ON`; for N>1 on a large table with many rows per group I use `LATERAL` with the composite index; the `ROW_NUMBER` CTE is my portable default when groups are small or I need `RANK`/`DENSE_RANK` tie semantics."

**Interviewer follow-up**: "When would `ROW_NUMBER` actually beat `LATERAL`?" → When groups are tiny/numerous so almost every row is in the Top-N: there is no early exit for `LATERAL` to exploit, and one streaming sort beats millions of index probes.

---

### Q2 — "Why can't I write `WHERE rn <= 3` in the same query as the `ROW_NUMBER`?"

**Junior answer**: "SQL doesn't let you use a column alias in WHERE."

**Principal answer**: "It's the logical execution order, not an alias rule. `WHERE` is evaluated before the SELECT list, and window functions are computed *in* the SELECT phase — so `rn` genuinely does not exist yet when `WHERE` runs. You must materialise the ranking in a subquery or CTE whose SELECT completes first, then filter in the outer query where `rn` is now a real column. `DISTINCT ON`, `LATERAL`, and the correlated subquery avoid the wrapper because each pushes the selection into a scope that finishes before the outer row is produced."

**Interviewer follow-up**: "Does `HAVING` help?" → No — `HAVING` filters groups after `GROUP BY`, still before the SELECT-list window functions; it can't see `rn` either. The subquery/CTE wrap is the only route.

---

### Q3 — "Top 2 earners per department. Two people tie for first. What does your query return, and is that correct?"

**Junior answer**: "It returns 2 rows, the top 2 salaries."

**Principal answer**: "Depends entirely on the ranking function, and the *right* choice is a business decision. `ROW_NUMBER` returns exactly 2 rows and arbitrarily drops one of the tied top earners — wrong if the business means 'everyone who is a top-2 earner.' `RANK` returns both tied top earners (ranks 1,1) and excludes the next salary — correct for 'top 2 pay levels, ties included,' but it can return more than 2 rows. `DENSE_RANK` would include the two 90k earners *and* the 80k earner as the second distinct level. I'd clarify the requirement first; if they truly want exactly N rows I use `ROW_NUMBER` **with a deterministic tie-breaker** like `id` so results are reproducible across runs and after VACUUM."

**Interviewer follow-up**: "Why does the tie-breaker matter beyond aesthetics?" → Without it, which tied row gets rank 1 is decided by physical sort order, which can change between executions, PG versions, and after table reorganisation — making a supposedly deterministic report non-reproducible and breaking any downstream diff/idempotency assumption.

---

### Q4 — "This greatest-per-group query does a full Seq Scan and a disk-spilling Sort on 50M rows. Fix it."

**Junior answer**: "Increase `work_mem` so the sort fits in memory."

**Principal answer**: "Raising `work_mem` only moves the sort off disk — I'm still sorting 50M rows to keep 3 per group, which is fundamentally wasteful. The real fix is to give the query an ordered access path and an early exit: create `(group_key, order_key DESC, tiebreak DESC)`, ideally partial on the status filter, then rewrite to `LATERAL` driven off the small distinct-group table so each group does a 3-row `LIMIT` index probe. That converts the plan from `Seq Scan → Sort → WindowAgg` into `Nested Loop → Limit → Index Scan` — no sort node, no temp files, and it reads only Top-N rows per group. For N=1 I'd instead use `DISTINCT ON` over the same index, which drops the sort entirely."

**Interviewer follow-up**: "What if there's no small table of distinct group keys?" → Derive them cheaply (a `SELECT DISTINCT group_key` can itself use a loose-index-scan emulation via a recursive CTE that skips from one key to the next), or accept `DISTINCT ON` if N=1. If the distinct-key set is nearly as large as the table, groups are tiny and the `ROW_NUMBER` CTE is fine.

---

## 15. Mental Model Checkpoint

1. A table has 1,000,000 rows across 1,000 groups (1,000 rows each). You want the top 3 per group. Which approach reads the fewest rows, and roughly how many rows does it read versus the `ROW_NUMBER` CTE?

2. You write `ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC)` with no tie-breaker. Two orders share a `created_at`. Is your `WHERE rn = 1` result deterministic across runs? Why or why not?

3. Explain why `DISTINCT ON` cannot return "top 3 per group" no matter how you write the `ORDER BY`.

4. A group's ordering column is entirely NULL for one group. Under `ORDER BY col DESC`, which row of that group becomes rank 1, and how do you control it?

5. You have `LEFT JOIN LATERAL (…) ON true` versus `CROSS JOIN LATERAL (…)`. A customer has zero orders. What does each return for that customer, and when would you choose one over the other?

6. Your `LATERAL` Top-N query is slow and EXPLAIN shows a `Sort` node inside the inner subquery. What is missing, and what exact index removes that sort?

7. When does the `ROW_NUMBER` CTE genuinely outperform the `LATERAL` emulation? Describe the data shape and the reason there is no early exit to exploit.

---

## 16. Quick Reference Card

```sql
-- ========== APPROACH A: ROW_NUMBER in CTE (portable, exact-N, any N) ==========
WITH ranked AS (
  SELECT t.*, ROW_NUMBER() OVER (PARTITION BY grp ORDER BY sort_key DESC, id DESC) AS rn
  FROM t
)
SELECT * FROM ranked WHERE rn <= N;          -- filter in OUTER query (execution order!)

-- ========== APPROACH B: LATERAL (index-driven, scales, any N) ==========
SELECT g.grp, top.*
FROM groups g
CROSS JOIN LATERAL (                          -- LEFT JOIN LATERAL ... ON true to keep empty groups
  SELECT * FROM t WHERE t.grp = g.grp
  ORDER BY sort_key DESC, id DESC LIMIT N
) top;
-- Needs: CREATE INDEX ON t (grp, sort_key DESC, id DESC);

-- ========== APPROACH C: DISTINCT ON (Postgres, N=1 ONLY) ==========
SELECT DISTINCT ON (grp) *
FROM t
ORDER BY grp, sort_key DESC, id DESC;         -- ORDER BY MUST start with grp

-- ========== APPROACH D: correlated subquery (legacy; count-form is O(G^2) — avoid) ==========
SELECT * FROM t WHERE sort_key = (SELECT MAX(sort_key) FROM t t2 WHERE t2.grp = t.grp);

-- ========== TIE FUNCTION CHEAT SHEET (values 90,90,80) ==========
-- ROW_NUMBER → 1,2,3   exactly N rows, arbitrary tie loser (ADD a tie-breaker!)
-- RANK       → 1,1,3   top-N ranks, ties included, may exceed N rows
-- DENSE_RANK → 1,1,2   top-N distinct values

-- ========== PERF RULES OF THUMB ==========
-- Many large groups + composite index → LATERAL (early exit, no sort)
-- Tiny numerous groups (most rows survive) → ROW_NUMBER CTE (one streaming pass)
-- N=1 → DISTINCT ON (shortest, no sort with index)
-- Full Seq Scan + disk-spilling Sort + rn<=N filter = the anti-pattern to kill
-- ORM fluent limit/take = GLOBAL limit, NEVER per-group → always raw SQL

-- ========== NULL / DETERMINISM GUARDS ==========
ORDER BY sort_key DESC NULLS LAST, id DESC     -- NULL won't steal rank 1; id makes it deterministic
```

**Interview one-liners:**
- "You can't filter a window function in the same WHERE — it's computed in SELECT, after WHERE."
- "`DISTINCT ON` is Top-1-per-group; for N>1 use `ROW_NUMBER` or `LATERAL`."
- "`LATERAL` + `LIMIT N` + a `(grp, order_key DESC)` index is PostgreSQL's loose-index-scan emulation."
- "Always give `ROW_NUMBER` a tie-breaker, or your 'deterministic' report isn't."
- "An ORM's `limit`/`take` limits total rows, never per group."

---

## Connected Topics

- **Topic 37 — Window Functions Fundamentals**: `PARTITION BY`/`OVER` and the `WindowAgg` node that computes the rank; the foundation of Approach A.
- **Topic 38 — RANK, DENSE_RANK, ROW_NUMBER**: the tie-behaviour trio whose correct selection is the correctness core of this topic.
- **Topic 40 — Window Function Performance** *(previous)*: why the full-partition sort spills to disk, and the `work_mem`/index levers reused here.
- **Topic 42 — Index Types in PostgreSQL** *(next)*: the B-tree composite/partial/covering indexes that power the `LATERAL` and `DISTINCT ON` plans.
- **Topic 15 — LATERAL Joins**: the correlated-subquery-as-join mechanics that make Approach B's index-probe emulation possible.
- **Topic 11 — INNER JOIN in Depth**: `CROSS JOIN LATERAL` vs `LEFT JOIN LATERAL ON true` for dropping vs keeping empty groups.
- **Topic 27 — EXISTS and NOT EXISTS**: the semi-join relative of the correlated-subquery approach and its early-exit behaviour.
- **Topic 07 — NULL in Depth**: why NULL group keys form one partition and NULL order keys can grab rank 1 under DESC.
- **Internals — B-tree ordered access & loose index scan**: the pre-sorted-stream property that makes an index a Top-N-per-group answer waiting to be read.

