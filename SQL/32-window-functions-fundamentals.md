# Topic 32 — Window Functions Fundamentals
### SQL Mastery Curriculum — Phase 6: Window Functions — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you're a teacher handing back exam papers. You have a stack of papers, one per student, each with a name and a score. You want to do two things at once:

1. **Give every student their own paper back** — nobody's paper disappears, everybody keeps their individual sheet.
2. **Write a little note in the corner of each paper** that says something about the *whole class*: "the class average was 74", "you ranked 3rd", "the running total of scores up to your paper is 512".

Here is the key insight. To write "the class average was 74" on Alice's paper, you had to *look at every other paper in the stack*. To write "you ranked 3rd" you had to *compare Alice against everyone else*. But — and this is the whole trick — **you did not staple the papers together, and you did not throw any away**. Every student still gets exactly one paper back. You just annotated each one using information from the entire pile.

That is a window function.

Contrast this with `GROUP BY`. `GROUP BY` is what happens if the teacher *collects all the papers, shreds them, and hands back a single index card* that says "Class average: 74. Number of students: 30." The individual papers are gone. You collapsed 30 rows into 1.

Window functions are the opposite of that collapse. They let each row *see* the aggregate — the average, the rank, the running total, the value in the row before it — **without losing the row itself**. Every input row produces exactly one output row. The aggregate is computed over a "window" of related rows, and the answer is pasted next to every row in that window.

The mental one-liner you should burn in: **`GROUP BY` folds rows together and destroys detail; a window function keeps every row and adds a column of context computed from a group of related rows.**

---

## 2. Connection to SQL Internals

At the engine level a window function is fundamentally different from an aggregate, and understanding the machinery explains every rule you will later have to memorize.

**What the executor actually does.** PostgreSQL implements window functions with a dedicated plan node called **`WindowAgg`**. This node sits *above* the scan/join/filter/group nodes and *below* the final sort/limit. The node's job is:

1. Receive a stream of rows from its child node.
2. Ensure the stream is ordered correctly — usually by requiring a `Sort` node (or an index that already provides the order) underneath it, matching the `PARTITION BY` and `ORDER BY` in the `OVER` clause.
3. Walk the sorted stream, tracking **partition boundaries** (when the `PARTITION BY` key changes) and **frame boundaries** (the sub-range of the partition visible to the current row).
4. For each row, evaluate the window function over its frame and emit the row *plus* the computed value.

Because the node must see other rows to compute a value for the current row, PostgreSQL **materializes** rows it needs to look back or forward at. For functions like `SUM() OVER (...)` with a running frame it keeps a running state; for functions like `LAG`/`LEAD` or frames with `ROWS BETWEEN ... FOLLOWING` it buffers rows in a **tuplestore** (in `work_mem`, spilling to a temp file on disk if the partition is large). This is the same tuplestore machinery used by CTEs and `Materialize` nodes.

**Why ordering is central.** Ranking (`ROW_NUMBER`, `RANK`), offset (`LAG`, `LEAD`), and running aggregates all depend on a defined row order. The engine cannot compute "the previous row's value" without knowing what "previous" means. That is why a `Sort` node almost always appears directly beneath `WindowAgg` unless a B-tree index already delivers rows in the required order (Topic 33 covers how a matching index eliminates the sort).

**Relation to concepts you already know.** The `Sort` here is the same external merge sort used by `ORDER BY` (Topic 25) and by Merge Join (Topic 11's join section). The `work_mem` budget governing spill-to-disk is the same knob that governs hash joins and grouping. MVCC and the heap are untouched — window functions are a pure read-side, post-scan transformation; they never touch the WAL, never take row locks, and never see uncommitted data beyond the snapshot the surrounding query already established. In short: a window function is a **sort + a single sequential pass with per-row state**. Every performance property flows from those two facts.

---

## 3. Logical Execution Order Context

This is the single most important section for interviews on this topic, so read it slowly.

```
FROM / JOIN            ← tables assembled, rows produced
WHERE                  ← row filtering (pre-aggregation)
GROUP BY               ← rows collapsed into groups
HAVING                 ← group filtering
SELECT                 ← expressions evaluated, INCLUDING window functions ★
DISTINCT
ORDER BY               ← final ordering
LIMIT / OFFSET         ← final truncation
```

The star marks the crucial fact: **window functions are evaluated in the `SELECT` phase, after `FROM`, `WHERE`, `GROUP BY`, and `HAVING` have all completed, but before `DISTINCT`, `ORDER BY`, and `LIMIT`.**

Three consequences fall out of this, and every one of them is a classic bug source.

**Consequence 1 — Window functions see the post-WHERE, post-GROUP BY row set.** By the time a window function runs, `WHERE` has already thrown rows away and `GROUP BY` has already collapsed groups. So a running total or a rank is computed over *only the rows that survived filtering*. If you `WHERE status = 'completed'`, your `ROW_NUMBER()` ranks only completed rows — the cancelled ones never entered the window. This is usually what you want, but you must be conscious of it.

**Consequence 2 — You cannot reference a window function in `WHERE`, `GROUP BY`, or `HAVING`.** Those clauses run *before* the window function is computed. The value literally does not exist yet. This:

```sql
SELECT id, ROW_NUMBER() OVER (ORDER BY created_at) AS rn
FROM orders
WHERE rn <= 10;          -- ERROR: column "rn" does not exist
```

fails because `WHERE` executes before `SELECT`, so `rn` has not been produced. Even worse, this doesn't just fail on the alias — you can't inline it either:

```sql
SELECT id
FROM orders
WHERE ROW_NUMBER() OVER (ORDER BY created_at) <= 10;
-- ERROR: window functions are not allowed in WHERE
```

The fix is always the same shape: compute the window value in a subquery or CTE (an "inner" query where `SELECT` runs), then filter in the *outer* query where the value now exists as an ordinary column:

```sql
SELECT id, rn
FROM (
  SELECT id, ROW_NUMBER() OVER (ORDER BY created_at) AS rn
  FROM orders
) t
WHERE rn <= 10;          -- works: rn is a plain column of subquery t
```

**Consequence 3 — Window functions *can* appear in `ORDER BY`,** because `ORDER BY` runs after `SELECT`. You can sort by a rank or a running total directly.

There is one subtle sub-rule worth internalizing: a window function's own `ORDER BY` (inside `OVER (...)`) is independent of the query's final `ORDER BY`. The window's internal ordering decides how the function counts/ranks/accumulates; the query's `ORDER BY` decides the order rows are printed. They are frequently different, and the engine may need two sorts.

---

## 4. What Is a Window Function?

A window function computes a value for each row by looking at a set of rows — the **window frame** — that are somehow related to the current row, while **preserving every input row in the output**. It is defined by attaching an `OVER (...)` clause to a function call.

```sql
SELECT
  employee_id,
  department_id,
  salary,
  AVG(salary) OVER (PARTITION BY department_id) AS dept_avg_salary
FROM employees;
```

Annotated syntax breakdown:

```
AVG(salary) OVER (PARTITION BY department_id ORDER BY hire_date ROWS BETWEEN ...)
│   │       │     │                          │                  │
│   │       │     │                          │                  └── frame clause: which rows
│   │       │     │                          │                      inside the partition are
│   │       │     │                          │                      visible to THIS row
│   │       │     │                          │
│   │       │     │                          └── ORDER BY: defines row order WITHIN the
│   │       │     │                              partition; makes "previous / running /
│   │       │     │                              rank" meaningful. Optional.
│   │       │     │
│   │       │     └── PARTITION BY: split rows into independent groups; the function
│   │       │         restarts for each partition. Optional — omit = one big partition
│   │       │         = all rows.
│   │       │
│   │       └── OVER: the keyword that turns an ordinary function call into a WINDOW
│   │           function. Its presence is what makes this NOT a GROUP BY aggregate.
│   │
│   └── the argument: the column/expression the function operates on (like any aggregate)
│
└── the function: an aggregate (SUM, AVG, COUNT, MIN, MAX) OR a
    window-only function (ROW_NUMBER, RANK, LAG, LEAD, NTILE, FIRST_VALUE, ...)
```

The essential moving parts:

- **`OVER`** — mandatory. Without it, `AVG(salary)` is a `GROUP BY`-style aggregate that collapses rows. With it, `AVG(salary) OVER (...)` is a window function that keeps rows.
- **`PARTITION BY`** — optional. Divides the input into partitions; the calculation runs independently within each. Omitting it means the whole result set is one partition. (Full treatment in Topic 33.)
- **`ORDER BY`** — optional. Orders rows inside each partition; required for ranking and offset functions to be meaningful, and it implicitly defines a default frame for aggregates.
- **The frame clause** (`ROWS`/`RANGE`/`GROUPS BETWEEN ... AND ...`) — optional. Narrows the visible rows further. Defaults are subtle and covered in section 6.

The empty `OVER ()` is legal and meaningful: it makes the window "all rows in the result", so `COUNT(*) OVER ()` gives the grand total row count pasted onto every row.

---

## 5. Why Window Functions Mastery Matters in Production

1. **They replace slow, correlated-subquery patterns.** "For each order, show how it compares to the customer's average order" used to require a correlated subquery that re-scans orders once per row — O(N²). A window function does it in a single sorted pass — O(N log N). At 10M rows this is the difference between a query that returns in 400ms and one that never finishes.

2. **They eliminate self-joins for running totals and comparisons.** Running totals, moving averages, "value vs previous row", "percent of total" — all classically done with self-joins that multiply rows and explode cost. Window functions express them declaratively and let the planner do a single pass.

3. **They are the foundation of every "top-N-per-group" query.** "Latest order per customer", "top 3 products per category", "most recent status per session" — the canonical solution is `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` filtered in an outer query. Getting the execution-order rule wrong here is the #1 window-function bug in real codebases.

4. **They keep detail and summary in one pass.** Reporting queries frequently need both the individual rows *and* their group aggregates (row + "% of department total"). Without windows you either run two queries and join, or you join the detail back to a `GROUP BY` subquery. Windows do it in one scan, one plan node.

5. **Misusing them silently corrupts analytics.** Because window functions run after `WHERE`, a filter you thought applied to the whole dataset actually applied *before* the window computed — so your "running total of all revenue" is really "running total of the filtered subset". This produces plausible-looking but wrong dashboards, which is worse than a crash.

---

## 6. Deep Technical Content

### 6.1 The Fundamental Difference: Window vs GROUP BY

This is the conceptual bedrock. Consider `employees` with 5 rows in department 10 and 3 rows in department 20.

```sql
-- GROUP BY: 8 rows collapse into 2 rows
SELECT department_id, AVG(salary) AS avg_salary
FROM employees
GROUP BY department_id;
--  department_id | avg_salary
--  10            | 72000
--  20            | 55000
--  (2 rows)

-- WINDOW: 8 rows stay 8 rows, avg pasted onto each
SELECT employee_id, department_id, salary,
       AVG(salary) OVER (PARTITION BY department_id) AS avg_salary
FROM employees;
--  employee_id | department_id | salary | avg_salary
--  1           | 10            | 80000  | 72000
--  2           | 10            | 70000  | 72000
--  3           | 10            | 66000  | 72000
--  ... (all 8 rows preserved)
--  (8 rows)
```

Same aggregate value (`72000`), completely different shape. The `GROUP BY` output has 2 rows; the window output has 8. **The presence of `OVER` is the entire difference.** `PARTITION BY department_id` in the window plays the role that `GROUP BY department_id` plays in the aggregate — it defines the grouping — but the rows are *not* collapsed.

You can even mix them, and it's a common real pattern: a query with `GROUP BY` can *also* have a window function that operates over the already-grouped rows.

```sql
-- Per-department totals, PLUS each department's share of company total
SELECT
  department_id,
  SUM(salary) AS dept_payroll,
  SUM(SUM(salary)) OVER () AS company_payroll,
  ROUND(100.0 * SUM(salary) / SUM(SUM(salary)) OVER (), 1) AS pct_of_company
FROM employees
GROUP BY department_id;
```

Read `SUM(SUM(salary)) OVER ()` carefully: the inner `SUM(salary)` is the group aggregate (department payroll, one value per group); the outer `SUM(...) OVER ()` is a window function summing those per-group values across all groups. This is legal precisely because window functions run *after* `GROUP BY` — the window sees the grouped rows, not the raw rows.

### 6.2 The Two Families of Window Functions

**Family A — Aggregate functions used as windows.** Any aggregate (`SUM`, `AVG`, `COUNT`, `MIN`, `MAX`, and user aggregates) can be used with `OVER`. Without a frame, with `ORDER BY` present, they accumulate; without `ORDER BY`, they apply to the whole partition.

**Family B — Window-only functions.** These have no `GROUP BY` meaning and *only* exist with `OVER`:

| Function | Returns | Needs ORDER BY? |
|----------|---------|-----------------|
| `ROW_NUMBER()` | Sequential 1,2,3… with no ties | Yes (to be deterministic) |
| `RANK()` | Rank with gaps after ties (1,1,3) | Yes |
| `DENSE_RANK()` | Rank without gaps (1,1,2) | Yes |
| `PERCENT_RANK()` | (rank-1)/(rows-1), 0..1 | Yes |
| `CUME_DIST()` | Cumulative distribution, 0..1 | Yes |
| `NTILE(n)` | Bucket number 1..n | Yes |
| `LAG(col, off, dflt)` | Value from a preceding row | Yes |
| `LEAD(col, off, dflt)` | Value from a following row | Yes |
| `FIRST_VALUE(col)` | First value in the frame | Yes |
| `LAST_VALUE(col)` | Last value in the frame | Yes (mind the frame!) |
| `NTH_VALUE(col, n)` | Nth value in the frame | Yes |

Sections in Topics 34–36 drill into these individually; here we establish that they are *all* governed by the same `OVER` machinery and the same execution-order rules.

### 6.3 The Frame: What Rows Can a Row See?

Within a partition, the **frame** defines which rows the function actually aggregates for the current row. The frame syntax:

```
{ ROWS | RANGE | GROUPS } BETWEEN <frame_start> AND <frame_end>
```

where each bound is one of:

```
UNBOUNDED PRECEDING     -- start of partition
n PRECEDING             -- n rows/values before current
CURRENT ROW             -- the current row (or peer group under RANGE)
n FOLLOWING             -- n rows/values after current
UNBOUNDED FOLLOWING     -- end of partition
```

**The default frame is the trap.** The rule:

- If `OVER` has **no `ORDER BY`**, the default frame is the **entire partition** (`RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`). So `SUM(x) OVER (PARTITION BY g)` gives the *total for the group* on every row.
- If `OVER` **has `ORDER BY`** but **no explicit frame**, the default frame is `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. So `SUM(x) OVER (ORDER BY t)` gives a **running total** — but note `RANGE ... CURRENT ROW` includes *all peer rows tied on the ORDER BY key*, not just up to the physical current row.

That `RANGE` vs `ROWS` distinction bites people constantly:

```sql
-- Suppose three rows tie on order_date = '2025-03-01' with amounts 10, 20, 30
SUM(amount) OVER (ORDER BY order_date)                              -- RANGE (default)
-- All three tied rows get 60 (10+20+30) — peers share the same running total

SUM(amount) OVER (ORDER BY order_date ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
-- The three rows get 10, 30, 60 — strict physical row-by-row accumulation
```

Rule of thumb: for a *true* row-by-row running total, always spell out `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Rely on the `RANGE` default only when you *want* tied rows to share a value. `GROUPS` mode (Postgres 11+) counts peer groups rather than physical rows and is rarer.

### 6.4 The LAST_VALUE Frame Gotcha

The most infamous frame bug. People write:

```sql
SELECT id, amount,
       LAST_VALUE(amount) OVER (ORDER BY id) AS last_amt
FROM t;
```

expecting `last_amt` to be the final amount in the whole set. Instead every row shows *its own* amount. Why? Because with `ORDER BY` and no explicit frame, the frame defaults to `... UNBOUNDED PRECEDING AND CURRENT ROW`. `LAST_VALUE` returns the last row *of the frame*, and the frame ends at the current row — so the "last" value is always the current row. The fix is to widen the frame to the end of the partition:

```sql
SELECT id, amount,
       LAST_VALUE(amount) OVER (
         ORDER BY id
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS last_amt
FROM t;
```

`FIRST_VALUE` does *not* suffer this because the default frame already starts at `UNBOUNDED PRECEDING`.

### 6.5 Multiple Windows and WINDOW Naming

You can use several different windows in one `SELECT`, and factor out repeated definitions with a named `WINDOW` clause (which runs logically before `SELECT` chooses them):

```sql
SELECT
  employee_id, department_id, salary,
  RANK()       OVER w_dept        AS dept_rank,
  AVG(salary)  OVER w_dept        AS dept_avg,
  ROW_NUMBER() OVER w_company     AS company_row
FROM employees
WINDOW
  w_dept    AS (PARTITION BY department_id ORDER BY salary DESC),
  w_company AS (ORDER BY salary DESC);
```

Named windows are purely a readability/DRY feature — the executor still builds one `WindowAgg` node per *distinct* window specification. Windows that share a partition+order can be evaluated by a single node; differing ones stack into multiple `WindowAgg` nodes (visible in EXPLAIN).

### 6.6 NULL Behaviour

- **In `PARTITION BY`:** all `NULL`s are treated as *equal* to each other, so they form a single partition together (unlike `=`, which treats `NULL` as unknown). Every `NULL` in the partition key lands in one shared partition.
- **In `ORDER BY` (inside OVER):** `NULL`s sort together and you can control their position with `NULLS FIRST` / `NULLS LAST` exactly as in a query-level `ORDER BY`. Default is `NULLS LAST` for `ASC`, `NULLS FIRST` for `DESC`.
- **In aggregate arguments:** `SUM`, `AVG`, `COUNT(col)`, etc. ignore `NULL` inputs, same as ordinary aggregates. `COUNT(*)` counts every row including all-NULL ones.
- **In `LAG`/`LEAD`:** if the offset row's value is `NULL`, you get `NULL` (not skipped). The optional third argument supplies a default when the offset falls outside the partition, not when it's NULL.

### 6.7 Window Functions Cannot Be Nested

You cannot put a window function inside the argument of another window function:

```sql
-- ERROR: window function calls cannot be nested
SUM(ROW_NUMBER() OVER (ORDER BY x)) OVER ()
```

The remedy is layering: compute the inner window in a subquery/CTE, then apply the outer window to that column in an enclosing query. This layering pattern (window → subquery → window) is the workhorse of complex analytics and appears constantly in production.

### 6.8 Interaction with DISTINCT

Because `DISTINCT` runs *after* `SELECT` (and thus after window functions), a window function's output participates in the `DISTINCT` comparison. This surprises people:

```sql
SELECT DISTINCT department_id, COUNT(*) OVER (PARTITION BY department_id) AS dept_size
FROM employees;
```

This works and gives one row per department with its size, because the `(department_id, dept_size)` pairs are identical within a department and `DISTINCT` folds them. But relying on `DISTINCT` to "collapse" window output is a smell — a `GROUP BY` is usually clearer. Also note: `DISTINCT` inside an aggregate window (e.g. `COUNT(DISTINCT x) OVER (...)`) is **not supported** in PostgreSQL and raises an error; you must pre-aggregate.

### 6.9 Filtering With FILTER

Aggregate window functions accept a `FILTER (WHERE ...)` clause, letting you conditionally include rows *inside* the window without a `CASE` expression:

```sql
SELECT
  order_id, status, amount,
  COUNT(*) FILTER (WHERE status = 'completed') OVER () AS completed_count,
  SUM(amount) FILTER (WHERE status = 'completed') OVER () AS completed_revenue
FROM orders;
```

`FILTER` applies to the rows within the frame before the aggregate sees them. It does **not** remove rows from the output — the row still appears, it just doesn't contribute to that particular aggregate. This is distinct from `WHERE`, which removes rows from the entire result before windows run.

---

## 7. EXPLAIN — WindowAgg in the Plan

### Basic window with a required sort

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT employee_id, department_id, salary,
       AVG(salary) OVER (PARTITION BY department_id) AS dept_avg
FROM employees;
```

```
WindowAgg  (cost=164.42..199.42 rows=2000 width=48)
           (actual time=1.83..3.94 rows=2000 loops=1)
  ->  Sort  (cost=164.42..169.42 rows=2000 width=40)
            (actual time=1.80..2.05 rows=2000 loops=1)
        Sort Key: department_id
        Sort Method: quicksort  Memory: 219kB
        ->  Seq Scan on employees  (cost=0.00..54.00 rows=2000 width=40)
                                    (actual time=0.01..0.42 rows=2000 loops=1)
Buffers: shared hit=34
Planning Time: 0.14 ms
Execution Time: 4.19 ms
```

**Reading it:**
- `Seq Scan` reads all 2000 rows (no `WHERE`, so full scan).
- `Sort` orders rows by `department_id` — the partition key. This is what feeds the window node ordered rows so it can detect partition boundaries. `Sort Method: quicksort Memory: 219kB` means it fit in `work_mem` (no disk spill).
- `WindowAgg` walks the sorted stream, computes `AVG(salary)` per partition, and emits all 2000 rows with the extra column. Note `rows=2000` on the WindowAgg — **rows are preserved, not collapsed** (a `GROUP BY` would show a smaller row estimate here).

### Window with ORDER BY (running total) — note the sort key includes order

```sql
EXPLAIN (ANALYZE)
SELECT customer_id, created_at, total_amount,
       SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY created_at
                               ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running
FROM orders;
```

```
WindowAgg  (cost=8391.44..9891.44 rows=100000 width=56)
           (actual time=52.1..131.7 rows=100000 loops=1)
  ->  Sort  (cost=8391.44..8641.44 rows=100000 width=24)
            (actual time=52.0..70.3 rows=100000 loops=1)
        Sort Key: customer_id, created_at
        Sort Method: external merge  Disk: 3520kB
        ->  Seq Scan on orders  (cost=0.00..1943.00 rows=100000 width=24)
                                 (actual time=0.02..12.4 rows=100000 loops=1)
Planning Time: 0.20 ms
Execution Time: 138.9 ms
```

**Reading it:**
- `Sort Key: customer_id, created_at` — both the partition key *and* the window `ORDER BY` appear, in that order. The window node needs rows grouped by partition and ordered within it.
- `Sort Method: external merge Disk: 3520kB` — the sort **spilled to disk** because 100K rows exceeded `work_mem`. This is the #1 window-function performance cost. Raising `work_mem` or adding a covering index `(customer_id, created_at)` removes the sort entirely.

### Sort eliminated by an index

```sql
CREATE INDEX ON orders (customer_id, created_at);

EXPLAIN (ANALYZE)
SELECT customer_id, created_at, total_amount,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at) AS rn
FROM orders;
```

```
WindowAgg  (cost=0.42..6890.42 rows=100000 width=24)
           (actual time=0.05..96.2 rows=100000 loops=1)
  ->  Index Scan using orders_customer_id_created_at_idx on orders
        (cost=0.42..4640.42 rows=100000 width=24)
        (actual time=0.03..40.1 rows=100000 loops=1)
Planning Time: 0.18 ms
Execution Time: 103.4 ms
```

**Reading it:**
- **No `Sort` node.** The B-tree index `(customer_id, created_at)` already returns rows in exactly the order the window needs, so the planner reads them straight from the index. This is the single most effective window-function optimization — turn the sort into an ordered index scan.

### Two stacked WindowAggs

When windows have different partition/order specs, EXPLAIN shows one `WindowAgg` per distinct window, each possibly with its own `Sort`:

```
WindowAgg              -- window 2: OVER (ORDER BY salary)
  ->  Sort  (Sort Key: salary)
        ->  WindowAgg  -- window 1: OVER (PARTITION BY department_id)
              ->  Sort  (Sort Key: department_id)
                    ->  Seq Scan on employees
```

Each layer re-sorts. Minimizing distinct window specs (or ordering them so one sort serves multiple windows) reduces the number of sorts.

### What to look for

| Symptom | Cause | Fix |
|---------|-------|-----|
| `Sort Method: external merge Disk: NkB` under WindowAgg | Partition sort spills; `work_mem` too small | Raise `work_mem`, or add ordered index |
| Separate `Sort` + `WindowAgg` on a hot query | No index matching partition+order | `CREATE INDEX (partition_cols, order_cols)` |
| Multiple stacked `WindowAgg` nodes | Several distinct window specs | Consolidate windows; use a named `WINDOW` |
| WindowAgg `rows` ≈ input rows, huge | Windows never reduce rows | Filter earlier in `WHERE`; window over less data |

---

## 8. Query Examples

### Example 1 — Basic: Average alongside each row

```sql
-- Show every employee's salary next to their department average.
-- Classic "detail + summary in one pass" — impossible with plain GROUP BY.
SELECT
  e.employee_id,
  e.department_id,
  e.salary,
  AVG(e.salary) OVER (PARTITION BY e.department_id) AS dept_avg_salary,      -- group avg, per row
  e.salary - AVG(e.salary) OVER (PARTITION BY e.department_id) AS diff_from_avg
FROM employees e
ORDER BY e.department_id, e.salary DESC;
```

### Example 2 — Intermediate: Running total and row numbering

```sql
-- Per customer, number their orders chronologically and show a running spend total.
SELECT
  o.customer_id,
  o.id                AS order_id,
  o.created_at,
  o.total_amount,
  ROW_NUMBER() OVER w                              AS order_seq,        -- 1st, 2nd, 3rd order...
  SUM(o.total_amount) OVER (
    PARTITION BY o.customer_id
    ORDER BY o.created_at
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW                    -- true running total
  )                                                 AS running_spend,
  o.total_amount
    - LAG(o.total_amount) OVER w                    AS delta_vs_prev    -- change from prior order
FROM orders o
WHERE o.status = 'completed'                         -- NOTE: window sees only completed orders
WINDOW w AS (PARTITION BY o.customer_id ORDER BY o.created_at)
ORDER BY o.customer_id, o.created_at;
```

### Example 3 — Production Grade: Top-3 products per category by revenue

```sql
-- CONTEXT:
--   order_items ~10M rows, orders ~2M rows, products ~50K, categories ~200.
--   Indexes: order_items(order_id), order_items(product_id), orders(status, created_at),
--            products(category_id).
--   GOAL: the 3 highest-revenue products in each category over the last quarter.
--   PERF EXPECTATION: ~300-600ms; dominated by the order_items scan + one sort per stage.
--
-- Window functions cannot be filtered in WHERE, so the rank is computed in an inner
-- query (rev CTE -> ranked CTE) and filtered in the outer query.

WITH product_revenue AS (
  SELECT
    p.category_id,
    p.id            AS product_id,
    p.name          AS product_name,
    SUM(oi.quantity * oi.unit_price) AS revenue
  FROM order_items oi
  JOIN orders   o ON o.id = oi.order_id
  JOIN products p ON p.id = oi.product_id
  WHERE o.status = 'completed'
    AND o.created_at >= DATE_TRUNC('quarter', CURRENT_DATE)
  GROUP BY p.category_id, p.id, p.name
),
ranked AS (
  SELECT
    pr.*,
    ROW_NUMBER() OVER (
      PARTITION BY pr.category_id
      ORDER BY pr.revenue DESC
    ) AS rev_rank                                    -- rank within category
  FROM product_revenue pr
)
SELECT
  c.name AS category_name,
  r.product_name,
  r.revenue,
  r.rev_rank
FROM ranked r
JOIN categories c ON c.id = r.category_id
WHERE r.rev_rank <= 3                                -- filter on window result: OUTER query only
ORDER BY c.name, r.rev_rank;
```

EXPLAIN sketch for Example 3:

```
Sort  (Sort Key: c.name, r.rev_rank)
  ->  Hash Join  (categories x ranked)
        ->  Subquery Scan on r
              Filter: (rev_rank <= 3)
              ->  WindowAgg
                    ->  Sort  (Sort Key: category_id, revenue DESC)
                          ->  Subquery Scan on pr   -- product_revenue CTE
                                ->  HashAggregate    -- GROUP BY category_id, product
                                      ->  Hash Join (order_items x orders x products)
                                            ->  ... Seq/Index Scans with the WHERE filter
```

The `Filter: (rev_rank <= 3)` sits on the `Subquery Scan` *above* the `WindowAgg` — visual proof that the window is computed first and filtered afterward.

---

## 9. Wrong → Right Patterns

### Wrong 1: Filtering a window result in WHERE

```sql
-- WRONG — window functions don't exist yet when WHERE runs
SELECT customer_id, id, created_at,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
FROM orders
WHERE rn = 1;                     -- ERROR: column "rn" does not exist
```

Exact error: `ERROR: column "rn" does not exist`. And inlining the expression instead of the alias gives `ERROR: window functions are not allowed in WHERE`. Both fail for the same reason: `WHERE` runs in the pipeline *before* `SELECT`, so the window value has not been produced.

```sql
-- RIGHT — compute in a subquery, filter in the outer query
SELECT customer_id, id, created_at
FROM (
  SELECT customer_id, id, created_at,
         ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
  FROM orders
) t
WHERE rn = 1;                     -- "latest order per customer" — the canonical pattern
```

### Wrong 2: Expecting the window to ignore the WHERE filter

```sql
-- WRONG expectation: "running total of ALL revenue"
SELECT created_at, amount,
       SUM(amount) OVER (ORDER BY created_at
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM payments
WHERE status = 'completed';
-- The running total is over COMPLETED payments only — WHERE ran first and removed
-- pending/failed rows before the window ever saw them. The number is silently "wrong"
-- relative to what the author imagined.
```

```sql
-- RIGHT — if you truly want the total over all rows but only DISPLAY completed ones,
-- move the filter so the window sees everything, then filter in the outer query:
SELECT created_at, amount, running_total
FROM (
  SELECT created_at, amount, status,
         SUM(amount) OVER (ORDER BY created_at
                           ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
  FROM payments
) t
WHERE status = 'completed';       -- window already accumulated ALL rows; we just hide some
```

### Wrong 3: Using GROUP BY when you needed to keep rows

```sql
-- WRONG — collapses detail; you wanted every order shown with its share
SELECT customer_id, SUM(total_amount) AS total
FROM orders
GROUP BY customer_id;
-- Now you've lost individual orders and can't show "this order = 12% of your spend".
```

```sql
-- RIGHT — keep every order, add the per-customer total as a window
SELECT
  id AS order_id, customer_id, total_amount,
  SUM(total_amount) OVER (PARTITION BY customer_id) AS customer_total,
  ROUND(100.0 * total_amount
        / SUM(total_amount) OVER (PARTITION BY customer_id), 1) AS pct_of_customer
FROM orders;
```

### Wrong 4: LAST_VALUE with the default frame

```sql
-- WRONG — returns each row's own value, not the partition's last value
SELECT customer_id, created_at, status,
       LAST_VALUE(status) OVER (PARTITION BY customer_id ORDER BY created_at) AS latest_status
FROM orders;
-- Default frame ends at CURRENT ROW, so "last of frame" = current row.
```

```sql
-- RIGHT — widen the frame to the whole partition
SELECT customer_id, created_at, status,
       LAST_VALUE(status) OVER (
         PARTITION BY customer_id ORDER BY created_at
         ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
       ) AS latest_status
FROM orders;
-- Or avoid the trap entirely: FIRST_VALUE with ORDER BY created_at DESC.
```

### Wrong 5: COUNT(DISTINCT ...) OVER

```sql
-- WRONG — not supported as a window in PostgreSQL
SELECT category_id,
       COUNT(DISTINCT product_id) OVER (PARTITION BY category_id) AS distinct_products
FROM order_items oi JOIN products p ON p.id = oi.product_id;
-- ERROR: DISTINCT is not implemented for window functions
```

```sql
-- RIGHT — pre-aggregate distinct products, then window over the aggregated set
WITH per_cat AS (
  SELECT p.category_id, COUNT(DISTINCT oi.product_id) AS distinct_products
  FROM order_items oi JOIN products p ON p.id = oi.product_id
  GROUP BY p.category_id
)
SELECT * FROM per_cat;
-- If you need it pasted onto detail rows, join per_cat back to the detail table.
```

---

## 10. Performance Profile

Window functions cost = **sort cost + one linear pass**. Everything below flows from that.

### Scaling by row count

| Rows in partition-sorted set | Dominant cost | Typical time (no index) | With ordered index |
|------------------------------|---------------|--------------------------|--------------------|
| 100K | in-memory quicksort + pass | 50–150 ms | 30–90 ms (sort skipped) |
| 1M | sort likely spills to disk | 300 ms–1.5 s | 150–500 ms |
| 10M | external merge sort, multiple passes | 3–15 s | 1–4 s |
| 100M | huge external sort; consider pre-aggregation | 60 s+ | 15–40 s |

### Memory (`work_mem`)

- The `Sort` beneath `WindowAgg` obeys `work_mem`. Exceed it and you get `Sort Method: external merge Disk: NkB` — disk I/O that can 5–10× the runtime.
- Functions that buffer rows (`LAG`/`LEAD` with large offsets, frames with `FOLLOWING`, `LAST_VALUE` over `UNBOUNDED FOLLOWING`) hold a tuplestore that also draws on `work_mem` and spills to a temp file when large.
- Running aggregates with a `... CURRENT ROW` frame keep only running state (O(1) memory per partition) — cheap.
- Raising `work_mem` is the fastest lever for a spilling window sort, but it's per-sort-node and per-connection; a query with several windows can allocate several multiples. Set it locally: `SET LOCAL work_mem = '256MB';` inside a transaction rather than globally.

### CPU

- Ranking/offset functions are cheap per row (increment a counter, peek a buffered row).
- The sort is the CPU hog: comparison-heavy `O(N log N)`. Wide sort keys (long text, many columns) inflate it.

### The index optimization (the big one)

A B-tree index on `(partition_cols..., order_cols...)` lets the planner feed `WindowAgg` from an ordered `Index Scan` and **eliminate the Sort node entirely**. This is the highest-leverage optimization for window queries:

```sql
-- For:  OVER (PARTITION BY customer_id ORDER BY created_at)
CREATE INDEX ON orders (customer_id, created_at);
```

The column order in the index must match: partition keys first (in any consistent order), then the window `ORDER BY` columns in the same direction (or a consistently reversed direction, which the planner can scan backward). A mismatch (e.g. index `(created_at, customer_id)`) will *not* eliminate the sort.

### Other techniques

- **Filter before windowing.** Push selective `WHERE` predicates so the window operates over fewer rows. Fewer rows → smaller sort.
- **Partition to shrink the sort.** More partitions means many small sorts rather than one giant one; PostgreSQL still sorts the whole set once by `(partition, order)`, but downstream per-partition state stays small.
- **Avoid unnecessary distinct windows.** Each distinct `OVER` spec can add a sort. Reuse a single window (named `WINDOW`) where possible.
- **Consider incremental/materialized approaches at 100M+.** For dashboards, precompute running totals into a summary table or materialized view rather than windowing 100M rows on every request.

---

## 11. Node.js Integration

### 11.1 Latest order per customer (top-1-per-group)

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// The window result is filtered in the OUTER query — remember rn can't go in WHERE
// of the same SELECT that computes it.
async function getLatestOrderPerCustomer(limit = 100) {
  const { rows } = await pool.query(
    `SELECT customer_id, order_id, created_at, total_amount
     FROM (
       SELECT
         customer_id,
         id           AS order_id,
         created_at,
         total_amount,
         ROW_NUMBER() OVER (
           PARTITION BY customer_id
           ORDER BY created_at DESC
         ) AS rn
       FROM orders
       WHERE status = $1
     ) t
     WHERE rn = 1
     ORDER BY created_at DESC
     LIMIT $2`,
    ['completed', limit]
  );
  return rows;
}
```

### 11.2 Running total for a customer's ledger

```javascript
async function getCustomerLedger(customerId) {
  const { rows } = await pool.query(
    `SELECT
       id            AS order_id,
       created_at,
       total_amount,
       SUM(total_amount) OVER (
         ORDER BY created_at
         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_balance
     FROM orders
     WHERE customer_id = $1
       AND status = 'completed'
     ORDER BY created_at`,
    [customerId]
  );
  return rows;   // each row carries its own running_balance
}
```

### 11.3 Paginating stably with ROW_NUMBER (keyset alternative aside)

```javascript
// Attach a stable global sequence, then page over it. For very large offsets,
// prefer keyset pagination (Topic 26); window numbering is fine for bounded sets.
async function pageEmployeesByRank(page, pageSize) {
  const offset = (page - 1) * pageSize;
  const { rows } = await pool.query(
    `SELECT employee_id, department_id, salary, rn
     FROM (
       SELECT
         employee_id, department_id, salary,
         ROW_NUMBER() OVER (ORDER BY salary DESC, employee_id) AS rn
       FROM employees
     ) t
     WHERE rn > $1 AND rn <= $2`,
    [offset, offset + pageSize]
  );
  return rows;
}
```

### 11.4 Setting work_mem locally for a heavy window query

```javascript
// A big analytics window can spill to disk. Bump work_mem for THIS transaction only,
// so you don't blow up memory for every connection.
async function heavyWindowReport() {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query("SET LOCAL work_mem = '256MB'");   // scoped to this txn
    const { rows } = await client.query(
      `SELECT category_id, product_id, revenue, rev_rank
       FROM (
         SELECT category_id, product_id, revenue,
                ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY revenue DESC) AS rev_rank
         FROM product_revenue_mv
       ) t
       WHERE rev_rank <= 10`
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

**Result handling note:** window columns come back as ordinary columns in `rows`. Aggregate windows over `numeric`/`bigint` (like `SUM`) return **strings** from `pg` by default to preserve precision — parse with `Number()` or a custom type parser if you need JS numbers, exactly as with any `SUM`/`COUNT`.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do window functions?** — Not in its type-safe query API. Prisma has no `OVER`, no `ROW_NUMBER`, no `PARTITION BY`. You must drop to raw SQL.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

const latest = await prisma.$queryRaw`
  SELECT customer_id, order_id, created_at, total_amount
  FROM (
    SELECT customer_id, id AS order_id, created_at, total_amount,
           ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
    FROM orders
    WHERE status = 'completed'
  ) t
  WHERE rn = 1
`;
```

**Where it breaks:** the entire feature is absent from the fluent API. `$queryRaw` returns untyped rows; you annotate the generic yourself. **Verdict:** raw SQL only.

### Drizzle ORM

**Can Drizzle do window functions?** — Yes, via helper builders (`sql` plus `window`/`over` helpers in recent versions).

```typescript
import { sql } from 'drizzle-orm';
import { db } from './db';
import { orders } from './schema';

const ranked = db.$with('ranked').as(
  db.select({
    customerId: orders.customerId,
    orderId: orders.id,
    createdAt: orders.createdAt,
    rn: sql<number>`ROW_NUMBER() OVER (
      PARTITION BY ${orders.customerId} ORDER BY ${orders.createdAt} DESC
    )`.as('rn'),
  }).from(orders).where(sql`${orders.status} = 'completed'`)
);

const latest = await db.with(ranked)
  .select().from(ranked)
  .where(sql`${ranked.rn} = 1`);
```

**Where it breaks:** frame clauses and exotic functions still funnel through the `sql` tag. **Verdict:** good — the CTE-plus-window pattern is expressible and typed at the edges.

### Sequelize

**Can Sequelize do window functions?** — Only through `sequelize.literal()` / raw queries; no first-class window API.

```javascript
const { QueryTypes } = require('sequelize');
const rows = await sequelize.query(
  `SELECT customer_id, order_id, created_at
   FROM (
     SELECT customer_id, id AS order_id, created_at,
            ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
     FROM orders WHERE status = :status
   ) t
   WHERE rn = 1`,
  { replacements: { status: 'completed' }, type: QueryTypes.SELECT }
);
```

**Where it breaks:** attempting `attributes: [[sequelize.literal('ROW_NUMBER() OVER (...)'), 'rn']]` works for producing the column but you still cannot filter on it in the model `where` — same execution-order wall — so you end up nesting raw subqueries anyway. **Verdict:** use `sequelize.query()`.

### TypeORM

**Can TypeORM do window functions?** — Partially; QueryBuilder can emit `OVER` via `addSelect` with a raw expression, and you filter in an outer QueryBuilder.

```typescript
const inner = dataSource.getRepository(Order).createQueryBuilder('o')
  .select('o.customer_id', 'customer_id')
  .addSelect('o.id', 'order_id')
  .addSelect('o.created_at', 'created_at')
  .addSelect('ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.created_at DESC)', 'rn')
  .where('o.status = :status', { status: 'completed' });

const rows = await dataSource.createQueryBuilder()
  .select('*').from('(' + inner.getQuery() + ')', 't')
  .setParameters(inner.getParameters())
  .where('t.rn = 1')
  .getRawMany();
```

**Where it breaks:** nesting a QueryBuilder as a subquery is verbose and the window expression is a raw, untyped string. **Verdict:** works but clumsy; raw SQL is often cleaner.

### Knex.js

**Can Knex do window functions?** — Yes, with dedicated helpers: `.rowNumber()`, `.rank()`, `.denseRank()`, `.over()`, `.partitionBy()`, `.orderBy()`.

```javascript
const inner = knex('orders')
  .select('customer_id', 'id as order_id', 'created_at', 'total_amount')
  .rowNumber('rn', function () {
    this.partitionBy('customer_id').orderBy('created_at', 'desc');
  })
  .where('status', 'completed');

const latest = await knex.select('*').from(inner.as('t')).where('rn', 1);

// Aggregate window via raw:
const withPct = await knex('orders')
  .select('id', 'customer_id', 'total_amount')
  .select(knex.raw('SUM(total_amount) OVER (PARTITION BY customer_id) AS customer_total'));
```

**Where it breaks:** frame clauses (`ROWS BETWEEN ...`) and `FILTER` need `knex.raw`. **Verdict:** the most SQL-transparent — named window helpers plus `raw` cover everything.

### ORM summary

| ORM | Native window API? | Filter-on-window story | Verdict |
|-----|--------------------|-----------------------|---------|
| Prisma | No | Raw subquery | `$queryRaw` only |
| Drizzle | Yes (`sql` + CTE) | `.with()` then `.where(sql...)` | Good |
| Sequelize | No | Raw subquery | `sequelize.query()` |
| TypeORM | Partial (`addSelect`) | Nested QueryBuilder | Works, clumsy |
| Knex | Yes (`.rowNumber()`, `.over()`) | Wrap in `.from(inner.as())` | Best transparency |

Across every ORM the same truth holds: because you can't filter a window result in `WHERE`, you always end up with a **two-level query** (inner computes the window, outer filters it). The ORMs differ only in how gracefully they express that nesting.

---

## 13. Practice Exercises

### Exercise 1 — Easy

Given `employees(id, name, salary, department_id)`, write a query that returns every employee with their `name`, `salary`, `department_id`, and a column `dept_avg` showing the average salary of their department — **without collapsing rows** (all employees must appear). Then add a column `above_avg` that is `TRUE` when the employee earns more than their department average.

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines prior topics)

Given `orders(id, customer_id, total_amount, status, created_at)` and `customers(id, name)` (Topic 11 joins), return, for each **completed** order: the customer's `name`, the `order_id`, `total_amount`, a `running_total` of that customer's completed spend ordered by `created_at` (true row-by-row running total), and `order_seq` (1 for their first completed order, 2 for the second, …). Order the output by customer name then `created_at`.

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; naive answer is wrong)

Given `payments(id, user_id, amount, status, created_at)` with ~5M rows, produce a report of, **per user**, only their **single most recent** payment (`status`, `amount`, `created_at`), but the report must also include, for each such row, the user's **all-time total paid amount across every status** (not just the most recent). A naive `WHERE status = ...` or a single window will get one of the two numbers wrong. Explain in a comment which execution-order rule forces a two-level query, and what index makes it fast.

```sql
-- Write your query here
```

### Exercise 4 — Interview Simulation

You are handed this query and told "it errors, fix it and explain why":

```sql
SELECT customer_id, id, created_at,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC) AS rn
FROM orders
WHERE rn = 1
ORDER BY customer_id;
```

Rewrite it so it correctly returns the latest order per customer, and in a comment state (a) the exact error message the original produces, (b) *why* the pipeline rejects it, and (c) whether moving `rn = 1` to a `HAVING` clause would fix it (and why not).

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1 — What is the difference between a window function and a GROUP BY aggregate?

**Junior answer:** "They both compute aggregates like SUM and AVG."

**Principal answer:** They compute the same *values* but produce different *shapes*. `GROUP BY` collapses each group of input rows into a single output row — detail is destroyed. A window function (`AVG(x) OVER (PARTITION BY g)`) keeps every input row and pastes the group's aggregate onto each one — one output row per input row. Mechanically, `GROUP BY` is a `HashAggregate`/`GroupAggregate` node that reduces row count; a window function is a `WindowAgg` node that preserves row count and runs after `GROUP BY` in the logical pipeline. You can even combine them: a `GROUP BY` query can carry a window that operates over the already-grouped rows, e.g. `SUM(SUM(x)) OVER ()` for each group's share of the grand total.

**Interviewer follow-up:** "Show me a case where you'd need both in one query." — Per-department payroll (`GROUP BY department_id`) plus each department's percentage of company payroll (`SUM(SUM(salary)) OVER ()`), computed in a single scan.

### Q2 — Why can't you use a window function in a WHERE clause, and how do you filter on one?

**Junior answer:** "You just can't, it's a SQL rule."

**Principal answer:** It's a consequence of logical execution order. `WHERE` runs in the pipeline *before* `SELECT`, and window functions are evaluated *in* the `SELECT` phase — so at the time `WHERE` executes, the window value has not been produced yet. Referencing its alias gives `column does not exist`; inlining the expression gives `window functions are not allowed in WHERE`. `GROUP BY` and `HAVING` are blocked for the same reason (both precede `SELECT`). The fix is to compute the window in a subquery or CTE, where its `SELECT` runs and materializes the value as an ordinary column, then filter in the enclosing query where it's now a plain column. This two-level pattern is exactly how "top-N-per-group" (`ROW_NUMBER() ... WHERE rn <= N`) is written.

**Interviewer follow-up:** "Where *can* you use a window function directly?" — In `SELECT` (obviously) and in `ORDER BY`, because `ORDER BY` runs after `SELECT`. Not in `WHERE`, `GROUP BY`, or `HAVING`.

### Q3 — Explain the default window frame and why LAST_VALUE often surprises people.

**Junior answer:** "The frame is all the rows in the partition."

**Principal answer:** It depends on whether `OVER` has an `ORDER BY`. With **no `ORDER BY`**, the default frame is the entire partition (`RANGE UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`) — so `SUM(x) OVER (PARTITION BY g)` is the group total. With an `ORDER BY` and no explicit frame, the default becomes `RANGE UNBOUNDED PRECEDING AND CURRENT ROW` — a running aggregate, but `RANGE ... CURRENT ROW` includes all rows *tied* on the order key (peers share the value). `LAST_VALUE` surprises people because "last of the frame" under that default frame is the current row itself — so it returns each row's own value instead of the partition's final value. The fix is to widen the frame to `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`, or use `FIRST_VALUE` with a reversed `ORDER BY`.

**Interviewer follow-up:** "What's the difference between `ROWS` and `RANGE` for a running total?" — `ROWS` counts physical rows (strict row-by-row accumulation); `RANGE` groups peers tied on the order key so they share a cumulative value. For a true per-row running total, always specify `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`.

### Q4 — A window query on 10M rows is slow. EXPLAIN shows a Sort with "external merge Disk". What's happening and how do you fix it?

**Junior answer:** "Add more RAM."

**Principal answer:** The `WindowAgg` requires its input ordered by `(partition_cols, order_cols)`, and the `Sort` node feeding it exceeded `work_mem`, spilling to a temp file (external merge sort) — that disk I/O dominates runtime. Two fixes: (1) raise `work_mem` for that query (`SET LOCAL work_mem` in a transaction) so the sort stays in memory; (2) far better, create a B-tree index on `(partition_cols, order_cols)` matching the window, which lets the planner feed `WindowAgg` from an ordered `Index Scan` and eliminate the `Sort` node entirely — you'll see the `Sort` disappear from EXPLAIN. Also filter earlier with `WHERE` so the window operates over fewer rows, and consider precomputing into a summary table for 100M-row dashboards.

**Interviewer follow-up:** "Does the index column order matter?" — Yes. Partition keys must come first, then the `ORDER BY` columns in matching (or consistently reversed, scannable-backward) direction. An index in the wrong order won't eliminate the sort.

---

## 15. Mental Model Checkpoint

1. You run `SELECT department_id, AVG(salary) OVER (PARTITION BY department_id) FROM employees` on a table with 8 rows across 2 departments. How many rows come back, and why is that different from the `GROUP BY` version?

2. A query has `WHERE status = 'completed'` and a `SUM(amount) OVER (ORDER BY created_at ...)` running total. Over which rows is the running total actually computed, and at what point in the pipeline were the non-completed rows removed?

3. Why does putting `WHERE rn <= 3` in the same `SELECT` that defines `rn = ROW_NUMBER() OVER (...)` fail, while wrapping it in a subquery works? Name the two clauses' pipeline positions.

4. You write `LAST_VALUE(status) OVER (PARTITION BY customer_id ORDER BY created_at)` and every row shows its own status instead of the latest. What is the default frame, and what one change fixes it?

5. When does a window query *not* need a `Sort` node beneath its `WindowAgg` in EXPLAIN? What must exist, and in what column order?

6. Can you nest one window function inside another (e.g. `SUM(ROW_NUMBER() OVER (...)) OVER (...)`)? If not, how do you achieve the effect?

7. You have a `GROUP BY category_id` query and want each category's share of the grand total. Write the window expression conceptually — why is it legal to wrap an aggregate in a window here, given execution order?

---

## 16. Quick Reference Card

```sql
-- ANATOMY
func(arg) OVER ( PARTITION BY p_cols  ORDER BY o_cols  frame_clause )
--         │      │                    │                └ ROWS/RANGE/GROUPS BETWEEN a AND b
--         │      │                    └ order within partition (ranking/offset/running)
--         │      └ split into independent groups (omit = one big partition)
--         └ the keyword that makes it a WINDOW function (keeps all rows)

-- WINDOW vs GROUP BY
GROUP BY g          -> collapses rows (fewer output rows)
OVER (PARTITION g)  -> keeps every row, pastes aggregate on each

-- EXECUTION ORDER (windows run in SELECT phase)
FROM > WHERE > GROUP BY > HAVING > [SELECT: windows here] > DISTINCT > ORDER BY > LIMIT
-- => CANNOT reference a window in WHERE / GROUP BY / HAVING (they run earlier)
-- => CAN reference a window in ORDER BY (runs later)

-- FILTER ON A WINDOW = TWO-LEVEL QUERY
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY k ORDER BY t DESC) AS rn FROM tbl
) s WHERE rn = 1;                 -- top-1-per-group (latest per k)

-- DEFAULT FRAMES
OVER (PARTITION g)                -- whole partition  -> group total on each row
OVER (ORDER BY t)                 -- RANGE ... CURRENT ROW -> running (peers share)
OVER (ORDER BY t ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)  -- strict running total

-- LAST_VALUE FIX
LAST_VALUE(x) OVER (ORDER BY t ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)

-- PERF RULES OF THUMB
-- * cost = SORT + one linear pass
-- * index (partition_cols, order_cols) removes the Sort node  <- biggest win
-- * "external merge Disk" under WindowAgg => raise work_mem or add that index
-- * filter early (WHERE) so the window sorts fewer rows
-- * COUNT(DISTINCT x) OVER (...) is NOT supported -> pre-aggregate

-- INTERVIEW ONE-LINERS
-- "Window keeps rows; GROUP BY collapses them."
-- "Windows run in SELECT, after WHERE/GROUP BY/HAVING, before ORDER BY/LIMIT."
-- "Can't filter a window in WHERE because WHERE runs before SELECT — nest it."
-- "The Sort under WindowAgg is the cost; a matching index eliminates it."
```

---

## Connected Topics

- **Topic 02 — SELECT Execution Order:** the logical pipeline that dictates *why* windows can't appear in `WHERE`.
- **Topic 07 — NULL in Depth:** how `NULL`s group in `PARTITION BY` and sort in the window `ORDER BY`.
- **Topic 11 — INNER JOIN in Depth:** windows frequently run on top of join output; the running-total example replaces a self-join.
- **Topic 20 — GROUP BY Fundamentals:** the collapse-rows counterpart; windows are "GROUP BY that keeps the rows".
- **Topic 25 — ORDER BY and Sorting:** the same `Sort` node and external-merge spill behaviour that sits beneath `WindowAgg`.
- **Topic 31 — Lateral Joins (previous):** an alternative way to compute per-row correlated aggregates; windows are usually cheaper for whole-partition math.
- **Topic 33 — PARTITION BY and ORDER BY in OVER (next):** the deep dive into partitioning, ordering, and how a matching index removes the sort.
- **Topics 34–36 — Ranking, Offset, and Frame Functions:** `ROW_NUMBER`/`RANK`, `LAG`/`LEAD`, and full frame mechanics built on this foundation.
