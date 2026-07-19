# Topic 38 — Aggregate Functions as Window Functions
### SQL Mastery Curriculum — Phase 6: Window Functions — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you're a cashier at the end of a long checkout line, and every customer hands you a receipt as they pass. You have two very different jobs you could be asked to do.

**Job A — the GROUP BY job:** "At the end of the day, tell me the total money the whole line spent." You wait until everyone has passed, add it all up, and write **one** number on a whiteboard. The individual customers vanish from your report — all you keep is the single total. That's a plain aggregate: many rows go in, one row comes out.

**Job B — the window job:** "As each customer passes, stamp their receipt with the *running total so far* — and also with *the total the whole line will spend* — but hand every customer their receipt back." Now nobody vanishes. Every customer keeps their individual receipt (their detail row), but each receipt now also carries extra context: how much had been spent up to and including them, and how much the entire line spent.

That second job is an **aggregate function used as a window function**. The magic word that switches Job A into Job B is `OVER (...)`. The instant you attach `OVER` to `SUM`, `AVG`, `COUNT`, `MIN`, or `MAX`, you are telling the engine: *"Do not collapse the rows. Compute this aggregate against a sliding 'window' of rows around each row, and paste the answer next to every detail row."*

The subtle part — the part that separates people who *use* window functions from people who *understand* them — is the phrase "the window of rows around each row." Around which rows exactly? All of them? Only the ones before me? Only the ones in my department? Only the three nearest? That choice is the **frame**, and it is the entire game. Change the frame and the same `SUM(amount) OVER (...)` turns from a running total into a grand total into a moving average into a per-partition subtotal. Same function, wildly different meaning, decided entirely by what you write inside the parentheses.

---

## 2. Connection to SQL Internals

When PostgreSQL executes a window aggregate, it does something structurally different from a `GROUP BY` aggregate, and understanding the machinery explains every performance characteristic and every gotcha in this topic.

**The WindowAgg node.** A window function shows up in the plan as a `WindowAgg` node sitting *above* the row source. Unlike `HashAggregate` or `GroupAggregate` (which consume N rows and emit fewer), `WindowAgg` consumes N rows and emits **exactly N rows** — it is row-preserving. It buffers rows, computes the aggregate over the relevant frame, and streams each input row back out with the new column attached.

**The mandatory sort.** A window function with `PARTITION BY x ORDER BY y` requires its input to arrive sorted by `(x, y)`. If no index provides that order, the planner inserts a `Sort` node directly beneath the `WindowAgg`. This is the single biggest cost driver: for 10M rows that don't fit in `work_mem`, that Sort becomes an external merge sort spilling to disk (`Sort Method: external merge Disk: ...kB`). The B-tree you learned about in the index topics matters here not for lookups but for *providing pre-sorted input* so the Sort node can be elided entirely.

**The frame and the aggregate state machine.** For a running total (`ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`), PostgreSQL keeps a running aggregate *state* and adds one row's contribution at a time as it walks the partition — this is O(N) per partition. For frames that can shrink from the front (a moving window like `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW`), some aggregates support an **inverse transition function** (`SUM` can subtract the row leaving the window), so the frame slides in O(1) per row instead of recomputing from scratch. `MIN` and `MAX` have *no* inverse — you cannot "un-max" a value — so a sliding-max frame may recompute over the whole frame, which is why `MAX() OVER (... ROWS BETWEEN 100 PRECEDING ...)` can be markedly slower than the equivalent `SUM`.

**RANGE vs ROWS and peer groups.** The frame mode changes what state the engine tracks. `ROWS` counts physical rows. `RANGE` groups rows into **peer groups** — sets of rows with identical `ORDER BY` values — and treats a whole peer group as an atomic unit. This distinction is implemented deep in the executor's frame-boundary logic, and it is the source of the most infamous gotcha in this topic (Section 6.6).

**No MVCC or WAL subtlety.** Window functions are pure read-side computation over an already-materialized row set. There is no MVCC visibility twist beyond the snapshot that produced the input rows, and nothing is written to WAL. The cost is entirely CPU (the aggregate transitions) and memory/IO (the sort and row buffering).

---

## 3. Logical Execution Order Context

```
FROM / JOIN          ← source rows assembled
WHERE                ← row filtering (BEFORE windows exist)
GROUP BY             ← collapses rows into groups
HAVING               ← filters groups
── window functions evaluated HERE ──   ← OVER(...) runs on the post-GROUP-BY row set
SELECT               ← projection (window results become selectable columns)
DISTINCT
ORDER BY             ← final presentation order
LIMIT / OFFSET
```

Two consequences of this ordering dominate everything in this topic:

**1. Window functions run AFTER `WHERE`, `GROUP BY`, and `HAVING`.** This means:
- You **cannot** reference a window function in a `WHERE` clause. `WHERE SUM(x) OVER (...) > 100` is a syntax error — the window hasn't been computed yet when `WHERE` runs. To filter on a window result you must wrap the query in a subquery/CTE and filter in the outer query. This is the number-one beginner mistake and a recurring interview trap.
- The rows the window sees are the rows that *survived* `WHERE`. A running total over "orders in 2025" only sums 2025 rows, because 2024 rows were filtered out before the window ran. Move the date filter and the running total changes.

**2. Window functions run alongside the rest of `SELECT`, but conceptually just before final `ORDER BY`.** The `ORDER BY` *inside* `OVER (...)` is a completely separate thing from the query's final `ORDER BY` — the former defines the window's internal ordering (and thus the frame), the latter defines how the finished result is presented. They can differ, and often should. You can compute a running total in chronological order inside the window while presenting the final result newest-first.

Remember from Topic 20 (GROUP BY Fundamentals): `GROUP BY` *destroys* detail rows. Window functions are the tool you reach for precisely when you need aggregate context **without** destroying the detail — the two are complementary, not competing.

---

## 4. What Is an Aggregate Window Function?

An aggregate window function is one of the standard aggregates — `SUM`, `AVG`, `COUNT`, `MIN`, `MAX` (and the statistical ones like `STDDEV`, `VARIANCE`) — invoked with an `OVER (...)` clause. The `OVER` clause converts it from a **collapsing** aggregate (many rows → one row per group) into a **row-preserving** window computation (N rows → N rows, each annotated with an aggregate computed over a defined window of neighboring rows).

```sql
SELECT
  order_id,
  customer_id,
  amount,
  SUM(amount) OVER (
    PARTITION BY customer_id       -- │ split rows into independent windows, one per customer
    ORDER BY created_at            -- │ order rows WITHIN each partition (defines "so far")
    ROWS BETWEEN UNBOUNDED PRECEDING  -- │ frame START: first row of the partition
             AND CURRENT ROW          -- │ frame END: the current row (running total)
  ) AS running_total
FROM orders;
```

Annotated syntax breakdown:

```
SUM ( amount )  OVER ( PARTITION BY customer_id ORDER BY created_at ROWS BETWEEN ... )
│     │              │            │                   │                 │
│     │              │            │                   │                 └── FRAME: which rows around the
│     │              │            │                   │                     current row feed the aggregate
│     │              │            │                   └── ORDER BY: ordering inside the partition; also
│     │              │            │                       makes the default frame meaningful (running calc)
│     │              │            └── PARTITION BY: divides the result set into groups; the window
│     │              │                resets at each partition boundary (like GROUP BY, but non-collapsing)
│     │              └── OVER: the switch that turns the aggregate into a window function
│     └── the argument being aggregated (an expression, column, or * for COUNT)
└── the aggregate function itself: SUM / AVG / COUNT / MIN / MAX
```

The three pieces inside `OVER` are all optional, and each omission has a precise meaning:

| What you write | What it means |
|---|---|
| `OVER ()` | Empty window = **every row** is in the frame. Aggregate over the whole result set; same value pasted on every row (grand total kept beside detail). |
| `OVER (PARTITION BY x)` | Whole-partition aggregate. Every row in a partition gets that partition's total. No ordering, no running behavior. |
| `OVER (ORDER BY y)` | No partition (one big partition = all rows), but ordered. **Default frame kicks in** → running/cumulative aggregate over the whole set. |
| `OVER (PARTITION BY x ORDER BY y)` | Running aggregate that **resets** at each partition boundary. The workhorse form. |
| `OVER (... ORDER BY y ROWS/RANGE BETWEEN ...)` | Explicit frame — moving averages, bounded windows, RANGE peer-group behavior. |

The critical, non-obvious rule buried in that table: **adding `ORDER BY` inside `OVER` silently changes the default frame** from "the entire partition" to "the start of the partition through the current row." This is why `SUM(x) OVER (PARTITION BY c)` gives a per-partition *total* while `SUM(x) OVER (PARTITION BY c ORDER BY t)` gives a *running total*. Same partition, one word added, completely different answer. Section 6 dissects exactly why.

---

## 5. Why Aggregate Window Function Mastery Matters in Production

1. **They replace whole classes of self-joins and correlated subqueries.** Before window functions, a running total meant a self-join (`o2.created_at <= o1.created_at`) that was O(N²) — a 100K-row table did 10 billion comparisons and timed out. The window version is O(N log N) (one sort, one linear pass). Engineers who don't know windows write the O(N²) version, and it works fine on 500 rows in dev, then melts in production. Mastery is the difference between a 30ms query and a 30-minute query.

2. **"Detail plus subtotal in one pass" is a universal reporting requirement.** Every dashboard that shows line items next to a running balance, every financial ledger, every cohort report showing "this row's revenue and the month's total revenue side by side" needs a window aggregate. Without it you run two queries (detail + aggregate) and stitch them in application code — more round trips, more code, more chances for the two numbers to disagree.

3. **The RANGE-vs-ROWS default is a correctness landmine.** The default frame with `ORDER BY` is `RANGE`, not `ROWS`. When your `ORDER BY` column has duplicate values (two orders at the same timestamp, two rows on the same date), `RANGE` lumps them into one peer group and gives **all of them the same running total** — the total *through the end of the group*, not through each individual row. Developers write a running total, test it on unique-timestamp data, ship it, and then a customer with two same-second orders sees a running balance that jumps by both orders at once on both rows. This is silent, plausible-looking, and wrong. It is the single most important thing in this document.

4. **Frame choice is where "moving average" bugs live.** A 7-day moving average is `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` — but only if there's exactly one row per day. If there are gaps or multiple rows per day, you need `RANGE BETWEEN '6 days' PRECEDING` (a value-based frame) instead. Choosing the wrong frame mode produces an average over the wrong set of rows that still returns a number, so nothing errors — it's just quietly incorrect.

5. **Performance cliffs are predictable once you understand the sort.** Knowing that each distinct `PARTITION BY / ORDER BY` spec forces a sort (unless an index provides the order) lets you (a) add covering indexes to eliminate sorts, (b) reuse a single window definition via `WINDOW` clauses so the engine sorts once, and (c) predict when `work_mem` will be exceeded and the sort spills to disk. This is directly diagnosable in `EXPLAIN` and directly fixable.

---

## 6. Deep Technical Content

### 6.1 The Five Aggregates as Windows — Baseline Semantics

Every standard aggregate works as a window function. Over the empty frame `OVER ()` they simply paste the whole-set aggregate onto every row:

```sql
SELECT
  order_id,
  amount,
  COUNT(*)   OVER () AS total_orders,     -- how many rows total
  SUM(amount) OVER () AS grand_total,      -- sum of every amount
  AVG(amount) OVER () AS avg_amount,       -- mean across all rows
  MIN(amount) OVER () AS cheapest,         -- smallest amount anywhere
  MAX(amount) OVER () AS priciest,         -- largest amount anywhere
  ROUND(amount / SUM(amount) OVER () * 100, 2) AS pct_of_total  -- share of grand total
FROM orders
WHERE status = 'completed';
```

The `pct_of_total` expression is the canonical reason to use `OVER ()`: computing each row's share of the total **without** a self-join or a subquery. The grand total sits right there, usable in arithmetic, on every detail row.

**NULL handling — identical to plain aggregates:**
- `SUM`, `AVG`, `MIN`, `MAX`, and `COUNT(column)` all **ignore NULL** inputs. `AVG` divides by the count of *non-NULL* values, not the row count.
- `COUNT(*)` counts rows including those with NULLs.
- If a frame contains only NULLs (or is empty), `SUM`/`AVG`/`MIN`/`MAX` return **NULL**, and `COUNT` returns **0**. This matters at partition edges — see 6.7.

### 6.2 PARTITION BY — Independent Sub-Windows

`PARTITION BY` slices the rows into groups; the window function restarts at each group boundary and never sees across it. It is the window analog of `GROUP BY`, but detail rows survive.

```sql
-- Each employee's salary next to their department's total/avg/max — detail preserved
SELECT
  e.name,
  e.department_id,
  e.salary,
  SUM(e.salary) OVER (PARTITION BY e.department_id) AS dept_payroll,
  AVG(e.salary) OVER (PARTITION BY e.department_id) AS dept_avg_salary,
  MAX(e.salary) OVER (PARTITION BY e.department_id) AS dept_top_salary,
  e.salary - AVG(e.salary) OVER (PARTITION BY e.department_id) AS delta_from_dept_avg
FROM employees e;
```

Note: **no `ORDER BY` inside `OVER`**, so the frame is the entire partition, giving a per-department *total/average/max* repeated on every employee row. `delta_from_dept_avg` is impossible with plain `GROUP BY` in a single pass because you'd lose the individual `salary`.

You can partition by multiple columns: `PARTITION BY department_id, job_title` creates one window per (department, title) pair.

### 6.3 ORDER BY Inside OVER — The Default Frame Switch

This is the pivotal mechanic. When you add `ORDER BY` inside `OVER` **without** an explicit frame clause, PostgreSQL supplies a default frame of:

```
RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
```

That default turns a whole-partition aggregate into a **cumulative** one — from the partition start up to the current row (and its peers). Compare directly:

```sql
SELECT
  customer_id,
  created_at,
  amount,
  -- NO order by → frame = whole partition → same total on every row
  SUM(amount) OVER (PARTITION BY customer_id) AS customer_lifetime_total,
  -- WITH order by → default frame = start..current → running total
  SUM(amount) OVER (PARTITION BY customer_id ORDER BY created_at) AS running_total
FROM orders;
```

For a customer with orders of 100, 50, 25 (in that time order):

| created_at | amount | customer_lifetime_total | running_total |
|---|---|---|---|
| 09:00 | 100 | 175 | 100 |
| 10:00 | 50 | 175 | 150 |
| 11:00 | 25 | 175 | 175 |

The lifetime total is constant (175 on every row); the running total accumulates (100 → 150 → 175). **One added `ORDER BY` is the entire difference.** This trips up nearly everyone once.

### 6.4 Running Totals, Cumulative Counts, and Cumulative Extremes

The default cumulative frame gives you a family of "so far" metrics for free:

```sql
SELECT
  created_at,
  amount,
  SUM(amount)  OVER w AS running_revenue,      -- cumulative revenue
  COUNT(*)     OVER w AS orders_so_far,          -- cumulative order count
  AVG(amount)  OVER w AS avg_so_far,             -- cumulative mean (careful: RANGE peers, see 6.6)
  MAX(amount)  OVER w AS peak_order_so_far,      -- largest order seen up to here (high-water mark)
  MIN(amount)  OVER w AS smallest_order_so_far   -- running minimum
FROM orders
WHERE status = 'completed'
WINDOW w AS (ORDER BY created_at)               -- named window, defined once, reused 5×
ORDER BY created_at;
```

The `WINDOW w AS (...)` clause (named window) defines the spec once and reuses it — the engine sorts **once** and computes all five aggregates in a single `WindowAgg` pass instead of five. This is both cleaner and faster; use it whenever multiple window columns share a spec.

`MAX(...) OVER (ORDER BY ...)` as a cumulative maximum is the classic "high-water mark" / running peak — extremely useful for drawdown analysis, peak-concurrency tracking, and detecting new records.

### 6.5 Explicit Frames — ROWS, RANGE, GROUPS

The frame clause has the form:

```
{ ROWS | RANGE | GROUPS }  BETWEEN <frame_start> AND <frame_end>
                           [ EXCLUDE { CURRENT ROW | GROUP | TIES | NO OTHERS } ]
```

Frame boundaries (`<frame_start>` / `<frame_end>`):

```
UNBOUNDED PRECEDING     -- start of the partition
<n> PRECEDING           -- n rows before (ROWS) / n units of value before (RANGE) / n peer-groups before (GROUPS)
CURRENT ROW             -- the current row (ROWS) or the current row's peer group (RANGE/GROUPS)
<n> FOLLOWING           -- n rows/units/groups after
UNBOUNDED FOLLOWING     -- end of the partition
```

The three frame **modes**:

- **`ROWS`** — counts *physical rows*. `ROWS BETWEEN 6 PRECEDING AND CURRENT ROW` = this row plus the 6 physically-preceding rows, regardless of their `ORDER BY` values. Deterministic count, ignores ties.
- **`RANGE`** — counts by *value offset* relative to the `ORDER BY` value, and treats **peer groups** (rows with equal `ORDER BY` value) as one unit. `RANGE BETWEEN '7 days' PRECEDING AND CURRENT ROW` = all rows whose `ORDER BY` value is within 7 days behind the current value. Requires a single `ORDER BY` column of a type that supports subtraction (numeric, date/time) when using offset bounds.
- **`GROUPS`** (PostgreSQL 11+) — counts *peer groups*. `GROUPS BETWEEN 2 PRECEDING AND CURRENT ROW` = the current peer group plus the 2 preceding peer groups (however many rows each contains).

Common frame recipes:

```sql
-- Running total (start → current):
ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW

-- 7-row trailing moving average (this row + 6 before):
ROWS BETWEEN 6 PRECEDING AND CURRENT ROW

-- Centered 5-row moving average (2 before, self, 2 after):
ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING

-- Reverse running total (current → end of partition):
ROWS BETWEEN CURRENT ROW AND UNBOUNDED FOLLOWING

-- Whole partition explicitly (equivalent to no ORDER BY):
ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING

-- 30-day trailing sum by actual calendar distance (handles gaps & dupes):
RANGE BETWEEN INTERVAL '30 days' PRECEDING AND CURRENT ROW
```

### 6.6 THE Big Gotcha — RANGE vs ROWS With Duplicate ORDER BY Values

This is the section to reread. The default frame that `ORDER BY` gives you is `RANGE`, not `ROWS`. When your `ORDER BY` column has **duplicate values**, `RANGE` and `ROWS` diverge, and the difference silently corrupts running totals.

Consider a customer with these orders, two of which land at the **same timestamp** `10:00`:

```
created_at | amount
-----------+-------
09:00      | 100
10:00      |  50    ← peer
10:00      |  30    ← peer  (same ORDER BY value as the row above)
11:00      |  20
```

```sql
SELECT created_at, amount,
  SUM(amount) OVER (ORDER BY created_at) AS range_default,      -- RANGE (the default!)
  SUM(amount) OVER (ORDER BY created_at
                    ROWS BETWEEN UNBOUNDED PRECEDING
                             AND CURRENT ROW) AS rows_explicit  -- ROWS
FROM orders;
```

Result:

| created_at | amount | range_default | rows_explicit |
|---|---|---|---|
| 09:00 | 100 | 100 | 100 |
| 10:00 | 50 | **180** | 150 |
| 10:00 | 30 | **180** | 180 |
| 11:00 | 20 | 200 | 200 |

Look at the two `10:00` rows. With **`ROWS`** the running total is a clean per-row accumulation: 100 → 150 → 180. With **`RANGE`** (the default), both `10:00` rows are **peers** — the frame `RANGE ... CURRENT ROW` includes the *entire current peer group*, so both peer rows get the same value `180` (100 + 50 + 30), the total *through the end of the tie group*.

Why does this happen? Under `RANGE`, "CURRENT ROW" as a frame boundary does not mean the physical current row — it means "up to and including all rows that are **peers** of the current row" (rows with an equal `ORDER BY` value). Peers are indistinguishable to `RANGE`, so they share a frame and share a result.

**Why it's dangerous:** A running account balance computed with the default frame will show the same balance on two same-timestamp transactions — a balance that already includes *both* transactions on *each* row. Reconciliation breaks. Nobody notices until same-timestamp rows appear, which they eventually will (batch imports, sub-second collisions truncated to seconds, same-day date columns).

**The fix — two options:**

1. **Use `ROWS` explicitly** whenever you want a true per-row running total:
   ```sql
   SUM(amount) OVER (ORDER BY created_at ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
   ```
2. **Make the `ORDER BY` unique** by adding a tiebreaker (e.g. the primary key), so no peer groups exist and `RANGE` and `ROWS` coincide:
   ```sql
   SUM(amount) OVER (ORDER BY created_at, order_id)   -- order_id breaks ties → no peers
   ```

Both work. `ROWS` is the more explicit and self-documenting choice for running totals; the unique-tiebreaker approach is essential when you *also* need a deterministic row ordering (e.g., ranking). Best practice: for running totals, **always write `ROWS ... CURRENT ROW` explicitly** and never rely on the default frame.

Note the flip side: for a genuine *cumulative distribution* or "total through this value" metric (e.g., "cumulative revenue up to and including this price point"), the `RANGE` peer-group behavior is exactly what you want. The default isn't wrong — it's just rarely what you mean when you say "running total."

### 6.7 Frame Edge Behavior — Empty and Partial Frames

At partition boundaries, frames can be partial or empty, and each aggregate has a defined answer:

```sql
SELECT
  created_at, amount,
  -- "previous 2 rows only, EXCLUDING current" → empty for the first row
  SUM(amount) OVER (ORDER BY created_at
                    ROWS BETWEEN 2 PRECEDING AND 1 PRECEDING) AS prev_two_sum,
  AVG(amount) OVER (ORDER BY created_at
                    ROWS BETWEEN 2 PRECEDING AND 1 PRECEDING) AS prev_two_avg
FROM orders;
```

For the **first** row of a partition, the frame `2 PRECEDING AND 1 PRECEDING` contains **no rows**:
- `SUM` → `NULL` (not 0!)
- `AVG` → `NULL`
- `COUNT` → `0`
- `MIN` / `MAX` → `NULL`

The `SUM → NULL` (not zero) surprises people building "sum of the previous N" columns. Wrap in `COALESCE(..., 0)` if a numeric zero is what you want downstream.

`EXCLUDE CURRENT ROW` / `EXCLUDE TIES` / `EXCLUDE GROUP` let you carve the current row (or its peers) out of an otherwise-inclusive frame — useful for "average of everyone *except me* in my department":

```sql
AVG(salary) OVER (PARTITION BY department_id
                  ORDER BY salary
                  RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING
                  EXCLUDE CURRENT ROW) AS peers_avg_excluding_me
```

### 6.8 Moving Averages — ROWS vs RANGE for Time Series

A "7-day moving average" is ambiguous until you decide whether you mean *7 rows* or *7 days*.

```sql
-- daily_sales(sale_date, revenue) — exactly one row per day, no gaps
-- 7-row trailing average (correct ONLY if one row per day, no gaps):
SELECT sale_date, revenue,
  AVG(revenue) OVER (ORDER BY sale_date
                     ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7_rows
FROM daily_sales;

-- 7-DAY trailing average by calendar distance (correct even with gaps/dupes):
SELECT sale_date, revenue,
  AVG(revenue) OVER (ORDER BY sale_date
                     RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW) AS ma7_days
FROM daily_sales;
```

If a weekend is missing (a gap), `ROWS BETWEEN 6 PRECEDING` reaches back to whatever 6 rows physically precede — possibly spanning 9 calendar days. `RANGE BETWEEN INTERVAL '6 days' PRECEDING` always covers exactly the trailing 6-day window regardless of how many rows fall in it. **Use `RANGE` with an interval for time-based windows; use `ROWS` for count-based windows.** Getting this wrong yields a plausible but subtly-wrong average.

`RANGE` with an `INTERVAL` offset requires the `ORDER BY` to be a single date/timestamp column (the type must support `+`/`-` with the offset).

### 6.9 Combining Window Aggregates With GROUP BY

Window functions run *after* `GROUP BY`, so you can aggregate first, then window over the grouped rows — a two-level aggregation in one query:

```sql
-- Monthly revenue AND its running cumulative total for the year
SELECT
  DATE_TRUNC('month', created_at) AS month,
  SUM(amount)                     AS monthly_revenue,          -- plain aggregate (GROUP BY)
  SUM(SUM(amount)) OVER (                                       -- window OVER the grouped sums
    ORDER BY DATE_TRUNC('month', created_at)
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS cumulative_revenue
FROM orders
WHERE status = 'completed'
  AND created_at >= '2025-01-01'
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY month;
```

The `SUM(SUM(amount))` reads oddly but is correct and important: the **inner** `SUM(amount)` is the `GROUP BY` aggregate (monthly revenue); the **outer** `SUM(...) OVER (...)` is the window running-total over those monthly sums. The window operates on the *output* of `GROUP BY`. This nested-aggregate pattern is a favorite interview question (Section 14).

### 6.10 Window Aggregates Cannot Be Filtered in WHERE/HAVING

Because windows run after `WHERE`/`HAVING`, you cannot filter on them there:

```sql
-- ILLEGAL — window function in WHERE:
SELECT customer_id, amount,
       SUM(amount) OVER (PARTITION BY customer_id) AS total
FROM orders
WHERE SUM(amount) OVER (PARTITION BY customer_id) > 1000;  -- ERROR
```

```
ERROR:  window functions are not allowed in WHERE
LINE 5: WHERE SUM(amount) OVER (PARTITION BY customer_id) > 1000;
```

The fix is to compute the window in a subquery/CTE and filter in the outer query:

```sql
SELECT * FROM (
  SELECT customer_id, amount,
         SUM(amount) OVER (PARTITION BY customer_id) AS total
  FROM orders
) s
WHERE s.total > 1000;
```

### 6.11 DISTINCT Inside Window Aggregates — Not Allowed

`COUNT(DISTINCT x) OVER (...)` and `SUM(DISTINCT x) OVER (...)` are **not supported** in PostgreSQL:

```
ERROR:  DISTINCT is not implemented for window functions
```

To count distinct values within a window you must work around it — a common approach is `DENSE_RANK`-based tricks or pre-aggregating in a subquery. For a distinct count over a whole partition, aggregate in a CTE grouped by the partition key and join back, or use the `dense_rank() + dense_rank() desc - 1` trick over the partition. This limitation surprises people migrating GROUP-BY logic to windows.

### 6.12 FILTER Clause With Window Aggregates

The `FILTER (WHERE ...)` clause works with window aggregates, letting you conditionally include rows in the aggregate without splitting into separate windows:

```sql
SELECT
  created_at,
  status,
  amount,
  -- running total of ONLY completed orders, evaluated over all rows in time order
  SUM(amount) FILTER (WHERE status = 'completed')
    OVER (ORDER BY created_at ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
    AS running_completed_revenue,
  COUNT(*) FILTER (WHERE status = 'refunded')
    OVER (ORDER BY created_at ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
    AS refunds_so_far
FROM orders;
```

`FILTER` is cleaner than `SUM(CASE WHEN status='completed' THEN amount END)` and communicates intent better. It is fully supported with `OVER`.

---

## 7. EXPLAIN — Window Aggregates in the Plan

### Running total with a required sort

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, created_at, amount,
       SUM(amount) OVER (PARTITION BY customer_id ORDER BY created_at
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM orders;
```

```
WindowAgg  (cost=15420.82..18170.82 rows=100000 width=52)
           (actual time=185.3..312.7 rows=100000 loops=1)
  ->  Sort  (cost=15420.82..15670.82 rows=100000 width=20)
            (actual time=185.2..201.4 rows=100000 loops=1)
        Sort Key: customer_id, created_at
        Sort Method: external merge  Disk: 3928kB
        ->  Seq Scan on orders  (cost=0.00..2810.00 rows=100000 width=20)
                                 (actual time=0.02..18.9 rows=100000 loops=1)
Buffers: shared hit=1810, temp read=491 written=492
Planning Time: 0.14 ms
Execution Time: 322.1 ms
```

**Reading it:**
- `WindowAgg` sits at the top and emits `rows=100000` — same as input. **Row-preserving**, confirming this is a window, not a `GROUP BY`.
- The `Sort` node beneath it is the cost center: `Sort Key: customer_id, created_at` matches the `PARTITION BY / ORDER BY`. The window *needs* its input in this order.
- `Sort Method: external merge Disk: 3928kB` — the sort **spilled to disk** because it exceeded `work_mem`. This is the classic window-function performance cliff. `temp read=491 written=492` confirms temp-file IO.
- The whole thing is a `Seq Scan` feeding a `Sort` feeding a `WindowAgg`. No index is helping.

### Same query with a covering index — sort eliminated

```sql
CREATE INDEX ON orders (customer_id, created_at) INCLUDE (amount);

EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, created_at, amount,
       SUM(amount) OVER (PARTITION BY customer_id ORDER BY created_at
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM orders;
```

```
WindowAgg  (cost=0.42..6540.42 rows=100000 width=52)
           (actual time=0.05..88.6 rows=100000 loops=1)
  ->  Index Only Scan using orders_customer_id_created_at_amount_idx on orders
        (cost=0.42..4290.42 rows=100000 width=20)
        (actual time=0.03..40.1 rows=100000 loops=1)
        Heap Fetches: 0
Buffers: shared hit=402
Planning Time: 0.20 ms
Execution Time: 96.4 ms
```

**Reading it:**
- **No `Sort` node.** The index `(customer_id, created_at)` already delivers rows in exactly the order the `WindowAgg` needs, so the planner skips the sort entirely.
- `Index Only Scan` with `Heap Fetches: 0` — the `INCLUDE (amount)` makes it covering; no heap access needed.
- Execution dropped from **322ms to 96ms** (~3.3×) purely by removing the sort. On 10M rows the gap is far larger because the eliminated sort would have been a large multi-pass external merge.
- Buffers fell from `1810 + temp IO` to `402 hit, 0 temp` — no disk spill.

### Multiple windows sharing one spec (named WINDOW → one sort)

```sql
EXPLAIN (ANALYZE)
SELECT created_at, amount,
       SUM(amount)   OVER w,
       AVG(amount)   OVER w,
       MAX(amount)   OVER w
FROM orders
WINDOW w AS (ORDER BY created_at ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW);
```

```
WindowAgg  (cost=12080.42..14330.42 rows=100000 width=44)
           (actual time=142.1..205.3 rows=100000 loops=1)
  ->  Sort  (cost=12080.42..12330.42 rows=100000 width=12)
        Sort Key: created_at
        Sort Method: quicksort  Memory: 8704kB
        ->  Seq Scan on orders (cost=0.00..2810.00 rows=100000 width=12)
Execution Time: 210.7 ms
```

**Reading it:** All three aggregates (`SUM`, `AVG`, `MAX`) share the single window `w`, so there is exactly **one** `WindowAgg` and **one** `Sort`. Had you written three different `OVER (...)` specs (even identical ones written out inline), the planner may stack multiple `WindowAgg` nodes each with its own sort. The `WINDOW` clause guarantees the single-sort plan.

### EXPLAIN signals table

| Symptom in plan | Meaning | Fix |
|---|---|---|
| `Sort` node under `WindowAgg` with `Disk:` | Sort spilled — `work_mem` too small | Raise `work_mem`, or add an index matching PARTITION BY, ORDER BY |
| Multiple stacked `WindowAgg` nodes | Multiple distinct window specs, multiple sorts | Consolidate with a `WINDOW` clause; order specs to share sorts |
| `WindowAgg rows` ≫ downstream `rows` after a filter | Windowing a huge set then discarding most | Filter (WHERE) before the window when semantically valid |
| No `Sort` but wrong results | Relying on default `RANGE` frame with duplicate ORDER BY | Switch to explicit `ROWS` frame |

---

## 8. Query Examples

### Example 1 — Basic: Percent of Total (empty frame)

```sql
-- Each product category's revenue and its share of total revenue, detail preserved
SELECT
  category_id,
  SUM(revenue)                                   AS category_revenue,
  SUM(SUM(revenue)) OVER ()                       AS total_revenue,   -- grand total on every row
  ROUND(
    SUM(revenue) / SUM(SUM(revenue)) OVER () * 100, 2
  )                                               AS pct_of_total
FROM (
  SELECT p.category_id, oi.quantity * oi.unit_price AS revenue
  FROM order_items oi
  JOIN products p ON p.id = oi.product_id
) sales
GROUP BY category_id
ORDER BY category_revenue DESC;
```

The `SUM(SUM(revenue)) OVER ()` computes the grand total of the per-category sums and pastes it beside each category so the percentage is computable in one pass.

### Example 2 — Intermediate: Per-Customer Running Balance (ROWS, tie-safe)

```sql
-- Running account balance per customer, correct even with same-timestamp orders
SELECT
  customer_id,
  order_id,
  created_at,
  amount,
  SUM(amount) OVER (
    PARTITION BY customer_id
    ORDER BY created_at, order_id                 -- order_id breaks timestamp ties
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW  -- explicit ROWS → true per-row running total
  ) AS running_balance,
  COUNT(*) OVER (
    PARTITION BY customer_id
    ORDER BY created_at, order_id
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS order_sequence_number                       -- 1,2,3,... per customer
FROM orders
WHERE status = 'completed'
ORDER BY customer_id, created_at, order_id;
```

Both the tiebreaker (`, order_id`) and the explicit `ROWS` frame defend against the RANGE-duplicate gotcha.

### Example 3 — Production Grade: 7-Day Moving Average + Cumulative + Per-Region Share

**Context:** `orders` table, ~50M rows, indexed on `(region_id, created_at)`. Reporting query for a regional revenue dashboard, run daily. Expectation: sub-second per region with the index eliminating the sort; a few seconds for a full multi-region scan.

```sql
WITH daily AS (
  SELECT
    region_id,
    DATE_TRUNC('day', created_at)::date AS day,
    SUM(amount)                          AS daily_revenue
  FROM orders
  WHERE status = 'completed'
    AND created_at >= CURRENT_DATE - INTERVAL '90 days'
  GROUP BY region_id, DATE_TRUNC('day', created_at)::date
)
SELECT
  region_id,
  day,
  daily_revenue,
  -- 7-day trailing moving average BY CALENDAR DISTANCE (gap-safe → RANGE + interval)
  ROUND(AVG(daily_revenue) OVER (
    PARTITION BY region_id
    ORDER BY day
    RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW
  ), 2) AS revenue_ma7,
  -- cumulative revenue for the 90-day window (explicit ROWS running total)
  SUM(daily_revenue) OVER (
    PARTITION BY region_id
    ORDER BY day
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
  ) AS cumulative_revenue,
  -- this day's share of the region's 90-day total (whole-partition frame)
  ROUND(daily_revenue / SUM(daily_revenue) OVER (PARTITION BY region_id) * 100, 3)
    AS pct_of_region_total
FROM daily
ORDER BY region_id, day;
```

```
EXPLAIN (ANALYZE, BUFFERS) ...

WindowAgg  (cost=..)  (actual time=.. rows=8100 loops=1)             -- pct_of_region_total window
  ->  WindowAgg  (actual time=.. rows=8100 loops=1)                   -- cumulative_revenue window
      ->  WindowAgg  (actual time=.. rows=8100 loops=1)               -- revenue_ma7 window
          ->  Sort  (actual time=.. rows=8100 loops=1)
                Sort Key: daily.region_id, daily.day
                Sort Method: quicksort  Memory: 812kB
                ->  Subquery Scan on daily  (rows=8100 loops=1)
                      ->  HashAggregate  (actual time=.. rows=8100 loops=1)
                            Group Key: region_id, (date_trunc('day', created_at))
                            ->  Index Scan using orders_region_created_idx on orders
                                  Index Cond: (created_at >= CURRENT_DATE - '90 days')
                                  Filter: (status = 'completed')
Execution Time: 340 ms
```

Note the CTE collapses 50M raw rows to ~8,100 daily-per-region rows *before* the windows run — so the three stacked `WindowAgg` nodes operate on 8,100 rows, not 50M. **Aggregate first, window second** is the key production pattern for scaling window functions: shrink the row set with `GROUP BY` before the window touches it.

---

## 9. Wrong → Right Patterns

### Wrong 1: Default frame running total with duplicate ORDER BY values

```sql
-- WRONG: relies on default RANGE frame; ties share a running total
SELECT created_at, amount,
       SUM(amount) OVER (ORDER BY created_at) AS running_total
FROM orders;
-- For two rows at the same created_at, BOTH show the total THROUGH the tie group,
-- e.g. both same-second rows read 180 instead of 150 and 180.
```

Why wrong at the execution level: with no explicit frame, `ORDER BY` defaults to `RANGE ... CURRENT ROW`, and `RANGE`'s "current row" boundary includes the entire peer group of equal `ORDER BY` values. Peers are inseparable.

```sql
-- RIGHT: explicit ROWS frame (per-row accumulation), plus a unique tiebreaker
SELECT created_at, amount,
       SUM(amount) OVER (ORDER BY created_at, id
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running_total
FROM orders;
```

### Wrong 2: Filtering on a window result in WHERE

```sql
-- WRONG: window functions don't exist yet when WHERE runs
SELECT customer_id, amount,
       SUM(amount) OVER (PARTITION BY customer_id) AS total
FROM orders
WHERE SUM(amount) OVER (PARTITION BY customer_id) > 5000;
-- ERROR: window functions are not allowed in WHERE
```

Why wrong: logical execution order — `WHERE` runs before window evaluation. The window column literally does not exist at that phase.

```sql
-- RIGHT: window in a subquery/CTE, filter in the outer query
SELECT * FROM (
  SELECT customer_id, amount,
         SUM(amount) OVER (PARTITION BY customer_id) AS total
  FROM orders
) s
WHERE s.total > 5000;
```

### Wrong 3: ROWS moving average on gappy time series

```sql
-- WRONG: "7-day" average using ROWS on data with missing days
SELECT sale_date, revenue,
       AVG(revenue) OVER (ORDER BY sale_date
                          ROWS BETWEEN 6 PRECEDING AND CURRENT ROW) AS ma7
FROM daily_sales;   -- weekends missing → "7 rows" spans 9+ calendar days
```

Why wrong: `ROWS` counts physical rows, not calendar days. With gaps, the 7 preceding rows can reach far further back in time than 7 days, so the "7-day" average silently averages the wrong span.

```sql
-- RIGHT: value-based RANGE frame measured in actual days
SELECT sale_date, revenue,
       AVG(revenue) OVER (ORDER BY sale_date
                          RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW) AS ma7
FROM daily_sales;
```

### Wrong 4: Expecting COUNT to return 0 at an empty frame edge

```sql
-- WRONG assumption: prev_sum is 0 on the first row
SELECT created_at, amount,
       SUM(amount) OVER (ORDER BY created_at
                         ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING) AS prev_amount
FROM orders;
-- On the first row the frame is EMPTY → SUM returns NULL, not 0
```

Why wrong: an empty frame makes `SUM`/`AVG`/`MIN`/`MAX` return `NULL` (only `COUNT` returns 0). Downstream arithmetic like `amount - prev_amount` then yields `NULL`.

```sql
-- RIGHT: coalesce to the intended default
SELECT created_at, amount,
       COALESCE(SUM(amount) OVER (ORDER BY created_at
                  ROWS BETWEEN 1 PRECEDING AND 1 PRECEDING), 0) AS prev_amount
FROM orders;
```

### Wrong 5: COUNT(DISTINCT) inside a window

```sql
-- WRONG: DISTINCT not supported in window aggregates
SELECT region_id, created_at,
       COUNT(DISTINCT customer_id) OVER (PARTITION BY region_id) AS unique_customers
FROM orders;
-- ERROR: DISTINCT is not implemented for window functions
```

Why wrong: PostgreSQL's window executor has no `DISTINCT` support for aggregates.

```sql
-- RIGHT: pre-aggregate distinct counts in a subquery, then join/window
SELECT o.region_id, o.created_at, uc.unique_customers
FROM orders o
JOIN (
  SELECT region_id, COUNT(DISTINCT customer_id) AS unique_customers
  FROM orders GROUP BY region_id
) uc ON uc.region_id = o.region_id;
```

---

## 10. Performance Profile

**The cost model in one sentence:** a window aggregate costs one sort (unless an index provides the order) plus one linear pass per distinct window spec.

| Rows | No index (Seq Scan + Sort) | Index provides order (no Sort) |
|---|---|---|
| 100K | Quicksort in `work_mem`, ~100–300ms | Index scan + linear pass, ~50–100ms |
| 1M | Sort likely spills to disk, ~1–3s | ~0.4–1s, no spill |
| 10M | External merge sort, multi-pass, ~15–40s | ~4–10s, IO-bound on the scan |
| 100M | Large external sort dominates, minutes | Depends on index scan IO; often the only viable path |

**Memory:**
- The sort uses up to `work_mem`; beyond that it spills (`Sort Method: external merge`). Each concurrent connection running a big window query needs its own `work_mem` — 100 connections × 256MB `work_mem` = 25.6GB potential, a common OOM cause.
- The `WindowAgg` itself also buffers rows for frames that look *forward* (`FOLLOWING`) or need the whole partition — a `RANGE BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` must hold an entire partition in memory/tuplestore before it can emit the first row of that partition. Cumulative `... AND CURRENT ROW` frames can stream and are cheaper.

**CPU:**
- `SUM`/`COUNT`/`AVG` over a sliding window use an **inverse transition function** — as the window slides one row forward, the leaving row is *subtracted* — giving O(1) per step, O(N) total.
- `MIN`/`MAX` have **no inverse** (you can't un-max), so a shrinking-from-the-front frame may rescan the frame. `MAX() OVER (... ROWS BETWEEN 1000 PRECEDING AND CURRENT ROW)` can be O(N × frame_size). Prefer bounded frames or, for pure running max (`UNBOUNDED PRECEDING`), it stays O(N) because the frame only grows.

**Optimization techniques specific to this topic:**
1. **Index to eliminate the sort.** Build a B-tree on `(partition_cols, order_cols)` — ideally covering with `INCLUDE (aggregated_cols)` for an Index-Only Scan. This is the single highest-leverage fix (Section 7 showed 3.3× on 100K, far more at 10M).
2. **Share one sort across windows.** Use a named `WINDOW w AS (...)` clause so multiple aggregates reuse one `WindowAgg`/`Sort`. Order multiple different specs so compatible ones are adjacent (the planner can sometimes reuse a sort for a spec that is a prefix of another).
3. **Aggregate before windowing.** Collapse rows with `GROUP BY` in a CTE first (Example 3): windowing 8K grouped rows instead of 50M raw rows is thousands of times cheaper.
4. **Bump `work_mem` for the session** running heavy window reports (`SET LOCAL work_mem = '256MB'`) to keep the sort in memory — but scope it locally, never globally.
5. **Prefer `ROWS` over `RANGE` when both are correct** — `ROWS` frame boundaries are cheaper to compute (no peer-group scanning) and avoid the duplicate-value gotcha.

---

## 11. Node.js Integration

### 11.1 Running total with pg

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function getCustomerRunningBalance(customerId) {
  const { rows } = await pool.query(
    `SELECT
       order_id,
       created_at,
       amount,
       SUM(amount) OVER (
         ORDER BY created_at, order_id
         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
       ) AS running_balance
     FROM orders
     WHERE customer_id = $1
       AND status = 'completed'
     ORDER BY created_at, order_id`,
    [customerId]
  );
  // pg returns NUMERIC columns as STRINGS to avoid float precision loss.
  // running_balance arrives as e.g. "175.00" — parse deliberately.
  return rows.map(r => ({
    orderId: r.order_id,
    createdAt: r.created_at,
    amount: Number(r.amount),
    runningBalance: Number(r.running_balance),
  }));
}
```

Key gotcha: `SUM(NUMERIC)` returns a `numeric`, which `node-postgres` maps to a JS **string** (not a number) to preserve precision. Always `Number(...)` (or use a decimal library for money) — silently comparing the string `"175.00"` to a number is a bug.

### 11.2 Parameterized moving average window size

```javascript
async function movingAverage(regionId, windowDays) {
  // Window frame bounds CANNOT be parameterized as $-placeholders in the frame clause;
  // the interval literal must be built safely. windowDays is validated as an integer first.
  if (!Number.isInteger(windowDays) || windowDays < 1 || windowDays > 365) {
    throw new Error('windowDays must be an integer between 1 and 365');
  }
  const { rows } = await pool.query(
    `SELECT day, daily_revenue,
            ROUND(AVG(daily_revenue) OVER (
              ORDER BY day
              RANGE BETWEEN ($2::int - 1) * INTERVAL '1 day' PRECEDING AND CURRENT ROW
            ), 2) AS moving_avg
     FROM (
       SELECT DATE_TRUNC('day', created_at)::date AS day, SUM(amount) AS daily_revenue
       FROM orders
       WHERE region_id = $1 AND status = 'completed'
       GROUP BY 1
     ) d
     ORDER BY day`,
    [regionId, windowDays]
  );
  return rows;
}
```

Note: you *can* parameterize the numeric multiplier of an `INTERVAL` (`$2::int * INTERVAL '1 day'`), which is how you make the frame width dynamic without string-concatenating SQL. The frame *keywords* (`ROWS`/`RANGE`, `PRECEDING`) cannot be parameters.

### 11.3 Streaming large window result sets

```javascript
import QueryStream from 'pg-query-stream';

async function streamCumulativeRevenue(onRow) {
  const client = await pool.connect();
  try {
    const query = new QueryStream(
      `SELECT day,
              SUM(daily_revenue) OVER (ORDER BY day
                ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative
       FROM daily_revenue_mv
       ORDER BY day`
    );
    const stream = client.query(query);
    for await (const row of stream) {
      onRow(row);  // process one row at a time — avoids buffering millions of rows in Node heap
    }
  } finally {
    client.release();
  }
}
```

For large windowed reports, stream results rather than materializing the entire `rows` array in Node — the window has already been computed server-side; streaming just controls client memory.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do window aggregates?** — Not through its typed query API. Prisma has `groupBy` and `aggregate`, but **no window function support** at all. Any `OVER (...)` requires `$queryRaw`.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

const rows = await prisma.$queryRaw<Array<{ order_id: bigint; running_total: string }>>(
  Prisma.sql`
    SELECT order_id, amount,
           SUM(amount) OVER (
             PARTITION BY customer_id
             ORDER BY created_at, order_id
             ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
           ) AS running_total
    FROM orders
    WHERE customer_id = ${customerId}
    ORDER BY created_at, order_id
  `
);
```

**Where it breaks:** everything window-related. **Verdict:** raw SQL only. Use `Prisma.sql` tagged templates for safe interpolation.

### Drizzle ORM

**Can Drizzle do window aggregates?** — Yes, since it exposes a `sql` template and window helpers. It has first-class-ish window support via `.over()` on aggregate helpers in recent versions, and always the `sql` escape hatch.

```typescript
import { sql } from 'drizzle-orm';
import { orders } from './schema';

const rows = await db
  .select({
    orderId: orders.id,
    amount: orders.amount,
    runningTotal: sql<string>`
      SUM(${orders.amount}) OVER (
        PARTITION BY ${orders.customerId}
        ORDER BY ${orders.createdAt}, ${orders.id}
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
      )`.as('running_total'),
  })
  .from(orders)
  .where(sql`${orders.status} = 'completed'`)
  .orderBy(orders.createdAt, orders.id);
```

**Where it breaks:** complex frame clauses are written inside the `sql` template regardless of helper sugar, so you still hand-write the frame. **Verdict:** best of the ORMs for windows — typed columns interpolate into the window spec, close to raw SQL.

### Sequelize

**Can Sequelize do window aggregates?** — Only via `sequelize.literal()` inside attributes, or a full raw query. No structured window API.

```javascript
const rows = await Order.findAll({
  attributes: [
    'id', 'amount',
    [sequelize.literal(`
      SUM(amount) OVER (
        PARTITION BY customer_id
        ORDER BY created_at, id
        ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
      )`), 'running_total'],
  ],
  where: { status: 'completed' },
  order: [['createdAt', 'ASC'], ['id', 'ASC']],
});
```

**Where it breaks:** `literal()` is an unparameterized string — never interpolate user input into it. For dynamic values, use a full `sequelize.query(sql, { bind: {...} })`. **Verdict:** works but clumsy; prefer `sequelize.query` with bind params for anything nontrivial.

### TypeORM

**Can TypeORM do window aggregates?** — Via QueryBuilder `addSelect` with a raw window expression, retrieved with `getRawMany()`.

```typescript
const rows = await dataSource
  .getRepository(Order)
  .createQueryBuilder('o')
  .select('o.id', 'id')
  .addSelect('o.amount', 'amount')
  .addSelect(`SUM(o.amount) OVER (
      PARTITION BY o.customer_id
      ORDER BY o.created_at, o.id
      ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)`, 'running_total')
  .where('o.status = :status', { status: 'completed' })
  .orderBy('o.created_at', 'ASC')
  .addOrderBy('o.id', 'ASC')
  .getRawMany();
```

**Where it breaks:** window results are raw (not mapped to entities), so `getRawMany()` returns plain objects; `getMany()` won't include the window column. **Verdict:** usable through `addSelect` + `getRawMany`, no type safety on the window expression.

### Knex.js

**Can Knex do window aggregates?** — Yes; recent Knex has `.sum(...).over(...)` partial support, and `knex.raw` always works.

```javascript
const rows = await knex('orders')
  .select('id', 'amount')
  .select(knex.raw(`
    SUM(amount) OVER (
      PARTITION BY customer_id
      ORDER BY created_at, id
      ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    ) AS running_total`))
  .where('status', 'completed')
  .orderBy([{ column: 'created_at' }, { column: 'id' }]);
```

**Where it breaks:** the structured `.over()` builder covers only simple `PARTITION BY`/`ORDER BY`; explicit frame clauses (`ROWS BETWEEN ...`) need `knex.raw`. **Verdict:** most SQL-transparent; use `knex.raw` for the frame.

### ORM Summary Table

| ORM | Window support | Frame clauses | Verdict |
|---|---|---|---|
| Prisma | None (raw only) | Raw only | `$queryRaw` for all windows |
| Drizzle | `sql` template, typed interpolation | Hand-written in `sql` | Best typed option |
| Sequelize | `literal()` only | Raw string | Clumsy; prefer `sequelize.query` |
| TypeORM | `addSelect` + `getRawMany` | Raw string | Works, no type safety |
| Knex | `.over()` (basic) + `raw` | `knex.raw` | Most transparent |

Universal truth: **no ORM gives full typed support for frame clauses.** Every one drops to raw SQL for `ROWS/RANGE BETWEEN`. Learn the SQL — the ORM is just a delivery mechanism here.

---

## 13. Practice Exercises

### Exercise 1 — Easy

Given `payments(id, user_id, amount, paid_at)`, write a query returning each payment with a **running total of that user's payments** in chronological order, plus a **sequence number** (1, 2, 3, …) for each user's payments. Ensure two payments at the same `paid_at` produce distinct running totals (no tie sharing).

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines Topic 20 GROUP BY + windows)

Given `orders(id, customer_id, amount, status, created_at)`, produce a **monthly report** for completed orders in 2025 with: `month`, `monthly_revenue`, `cumulative_revenue` (running total of monthly revenue across the year), and `pct_of_year` (each month's share of the full-year total). Use a single query with `GROUP BY` feeding window functions.

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation, naive answer is wrong)

Given `daily_active_users(product_id, day, dau)` where **some days are missing** for some products, compute a **7-day moving average of DAU** per product that is correct with respect to *calendar days* (a gap must not pull in older rows to fill the count). Also add `dau_vs_ma7_pct` = how far today's DAU deviates from its 7-day moving average, as a signed percentage. Explain in a comment why `ROWS BETWEEN 6 PRECEDING` would be the wrong choice here.

```sql
-- Write your query here
```

### Exercise 4 — Interview Simulation

You are given `orders(id, customer_id, amount, created_at)`. Write a single query that returns, for each customer, only the orders where the **running total of their spending first crossed 1,000** — i.e., the specific order at which cumulative spend went from below 1,000 to ≥ 1,000. (Hint: you'll need the running total *and* the previous running total, then filter in an outer query since you can't filter a window in `WHERE`.)

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What is the difference between `SUM(x)` with `GROUP BY` and `SUM(x) OVER (...)`?**

A: `SUM(x)` with `GROUP BY` collapses each group into a single output row — you lose the individual rows and keep only the group total. `SUM(x) OVER (...)` is a window function: it keeps **every** input row and pastes an aggregate value next to each one. So `GROUP BY` is many-rows-in, fewer-rows-out; the window version is N-rows-in, N-rows-out. Use the window when you need both the detail row and an aggregate (like each order's amount alongside the customer's total).

**Follow-up the interviewer asks:** *"If I write `SUM(x) OVER (PARTITION BY c)` versus `SUM(x) OVER (PARTITION BY c ORDER BY t)`, do I get the same answer?"*
No. Without `ORDER BY`, the frame is the whole partition, so every row gets the partition total. **Adding `ORDER BY` changes the default frame** to "start of partition through the current row," making it a running (cumulative) total. One clause flips a grand total into a running total.

---

### Junior/Mid Level

**Q: Why can't I put a window function in a `WHERE` clause?**

A: Because of logical execution order. `WHERE` runs before window functions are evaluated — the window column doesn't exist yet at that phase. Window functions run after `WHERE`, `GROUP BY`, and `HAVING`, right before the final `SELECT`/`ORDER BY`. To filter on a window result, compute it in a subquery or CTE and filter in the outer query.

**Follow-up:** *"Where in the logical order do window functions run relative to `GROUP BY`?"*
After `GROUP BY` and `HAVING`. That's why you can nest `SUM(SUM(x)) OVER (...)` — the inner `SUM` is the `GROUP BY` aggregate, and the outer windowed `SUM` operates on those already-grouped rows.

---

### Principal Level

**Q: A developer reports that a running-total query gives "wrong" numbers — two rows with the same timestamp show identical running totals that already include both rows. What's happening and how do you fix it?**

A: They're hitting the `RANGE`-vs-`ROWS` default-frame gotcha. When you write `OVER (ORDER BY created_at)` with no explicit frame, PostgreSQL supplies `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. Under `RANGE`, "CURRENT ROW" as a boundary doesn't mean the physical row — it means the entire **peer group** of rows sharing the current `ORDER BY` value. Two orders at the same timestamp are peers, so the frame includes both, and both rows get the total *through the end of the tie group* rather than a per-row accumulation. The fix is either (1) use an explicit `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` frame, which counts physical rows and accumulates per-row, or (2) add a unique tiebreaker to the `ORDER BY` (e.g. `ORDER BY created_at, id`) so there are no peer groups and `RANGE` coincides with `ROWS`. Best practice: write `ROWS` explicitly for running totals and never rely on the default.

**Follow-up:** *"When would the `RANGE` peer behavior actually be the correct choice?"*
For a true cumulative-distribution or "total through this value" metric — e.g., cumulative revenue up to and including a given price point, where all rows at the same price *should* share the cumulative value. `RANGE` is correct for value-based accumulation; `ROWS` is correct for row-sequence accumulation.

---

**Q: You have a 50M-row table and need a per-customer running total. `EXPLAIN` shows a `Sort` node with `Sort Method: external merge Disk: ...`. Walk me through your optimization.**

A: The `WindowAgg` requires its input ordered by `(customer_id, created_at)`. With no matching index, the planner sorts 50M rows, exceeding `work_mem`, so the sort spills to disk as an external merge — the dominant cost. Steps: (1) Build a B-tree index on `(customer_id, created_at)`, ideally `INCLUDE (amount)` to make it covering — the planner can then feed the `WindowAgg` directly from an Index-Only Scan and **eliminate the Sort node entirely**, which is the biggest win. (2) If the report reads all 50M rows anyway, also consider whether I can pre-aggregate with `GROUP BY` in a CTE before windowing to shrink the row set. (3) For a one-off heavy report, `SET LOCAL work_mem` higher so any remaining sort stays in memory. (4) Verify with `EXPLAIN (ANALYZE, BUFFERS)` that the `Sort` node is gone and `Heap Fetches: 0` on the index-only scan. Order-of-magnitude: removing a spilling external sort on tens of millions of rows typically turns tens of seconds into a few seconds.

**Follow-up:** *"Why does `MAX() OVER (... ROWS BETWEEN 500 PRECEDING AND CURRENT ROW)` sometimes run much slower than the same query with `SUM`?"*
Because `SUM`/`COUNT`/`AVG` have an inverse transition function — as the sliding frame advances, the engine subtracts the row leaving the window, so each step is O(1). `MIN`/`MAX` have no inverse (you can't "un-max" a value), so when the frame's trailing edge moves past the current max, the engine may have to rescan the frame to find the new max — up to O(frame_size) per row. For pure running max with `UNBOUNDED PRECEDING`, it stays O(N) because the frame only grows; the cost appears specifically with bounded, sliding frames.

---

## 15. Mental Model Checkpoint

1. You write `SUM(amount) OVER (PARTITION BY customer_id)` and then add `ORDER BY created_at` inside the same `OVER`. The numbers change completely. Explain precisely what changed and why — name the default frame.

2. A table has three rows all with `created_at = '2025-01-01 10:00:00'` and amounts 10, 20, 30. Using the default `OVER (ORDER BY created_at)`, what running total does each of the three rows show, and why are they identical? What would `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` show instead?

3. You need a "7-day moving average" over a table with occasional missing days. Which frame mode do you choose and what exactly goes wrong if you pick the other one?

4. Why is `SELECT ... WHERE SUM(x) OVER (...) > 100` illegal, and what is the minimal rewrite that makes it work?

5. On the first row of a partition, a frame of `ROWS BETWEEN 3 PRECEDING AND 1 PRECEDING` is empty. What do `SUM`, `AVG`, `COUNT`, and `MAX` each return, and which one differs from the others?

6. `EXPLAIN` shows three stacked `WindowAgg` nodes each with its own `Sort` for a query with three window columns. How do you collapse this to one sort, and what SQL construct do you use?

7. You want a running total that is genuinely per-row correct even when timestamps collide, *and* you also need the rows numbered 1..N deterministically. What single change to the `ORDER BY` inside `OVER` solves both at once, and why does it also cure the RANGE gotcha?

---

## 16. Quick Reference Card

```sql
-- ── Empty window: grand total on every row (percent-of-total) ──
SUM(x) OVER ()
ROUND(x / SUM(x) OVER () * 100, 2)      -- share of total, no self-join

-- ── Whole-partition total (no ORDER BY = whole partition frame) ──
SUM(x) OVER (PARTITION BY g)            -- same total repeated per group row

-- ── Running total (ALWAYS write ROWS explicitly!) ──
SUM(x) OVER (PARTITION BY g ORDER BY t
             ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

-- ── Cumulative count / running max (high-water mark) ──
COUNT(*) OVER (ORDER BY t ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)
MAX(x)   OVER (ORDER BY t)              -- running peak

-- ── Trailing moving average: ROWS (count) vs RANGE (calendar) ──
AVG(x) OVER (ORDER BY t ROWS  BETWEEN 6 PRECEDING AND CURRENT ROW)          -- 7 ROWS
AVG(x) OVER (ORDER BY d RANGE BETWEEN INTERVAL '6 days' PRECEDING AND CURRENT ROW) -- 7 DAYS

-- ── Centered moving average ──
AVG(x) OVER (ORDER BY t ROWS BETWEEN 2 PRECEDING AND 2 FOLLOWING)

-- ── Nested: monthly total + cumulative (window over GROUP BY) ──
SELECT month, SUM(x) AS monthly,
       SUM(SUM(x)) OVER (ORDER BY month
         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS cumulative
FROM t GROUP BY month;

-- ── Share one sort across many windows ──
SELECT SUM(x) OVER w, AVG(x) OVER w, MAX(x) OVER w
FROM t WINDOW w AS (ORDER BY t ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW);

-- ── Conditional window aggregate ──
SUM(x) FILTER (WHERE status='completed') OVER (ORDER BY t ROWS ...)

-- ── Filter on a window result (must wrap) ──
SELECT * FROM (SELECT *, SUM(x) OVER (...) AS s FROM t) q WHERE q.s > 100;
```

**Perf rules of thumb:**
- Each distinct window spec = one sort unless an index `(partition_cols, order_cols)` provides the order. Add `INCLUDE (agg_cols)` for Index-Only Scan.
- `Sort Method: external merge Disk:` in EXPLAIN = spilled sort; add the index or raise `work_mem`.
- Aggregate with `GROUP BY` **before** windowing to shrink the row set (Example 3).
- `SUM/COUNT/AVG` slide in O(1) (inverse transition); `MIN/MAX` on bounded sliding frames can be O(frame_size) — no inverse.

**Interview one-liners:**
- "`OVER` turns a collapsing aggregate into a row-preserving one — N in, N out."
- "Adding `ORDER BY` inside `OVER` silently switches the default frame to `RANGE UNBOUNDED PRECEDING → CURRENT ROW`, which makes it cumulative."
- "The default frame is `RANGE`, not `ROWS`; with duplicate `ORDER BY` values, `RANGE` shares the total across the whole peer group — always write `ROWS` for running totals."
- "You can't filter a window in `WHERE` — it runs after `WHERE`; wrap it in a subquery."
- "`RANGE` with an `INTERVAL` for time windows, `ROWS` for count windows."

---

## Connected Topics

- **Topic 20 — GROUP BY Fundamentals**: window functions preserve detail where `GROUP BY` collapses it; the two are complementary and combine via `SUM(SUM(x)) OVER (...)`.
- **Topic 07 — NULL in Depth**: window aggregates ignore NULL inputs (except `COUNT(*)`), and empty frames return NULL — the three-valued logic surfaces at frame edges.
- **Topic 37 — FIRST_VALUE / LAST_VALUE / NTH_VALUE** (previous): navigation functions that share the exact same `OVER (... frame ...)` machinery — and where `LAST_VALUE` bites you precisely because of the default `CURRENT ROW` frame you learned here.
- **Topic 39 — Frame Clauses in Depth** (next): the full formal treatment of `ROWS`/`RANGE`/`GROUPS`, `EXCLUDE`, and multi-column frame semantics — this topic is the applied preview, Topic 39 is the exhaustive reference.
- **Topic 36 — ROW_NUMBER / RANK / DENSE_RANK**: ranking functions built on `PARTITION BY / ORDER BY`; the tiebreaker discipline here is the same discipline that makes ranks deterministic.
- **Internals — B-tree indexes & external sort**: the sort beneath every `WindowAgg` is the cost center; a covering `(partition, order)` index eliminates it (Topics on Indexing).
- **Internals — work_mem and external merge sort**: window sorts spill to disk when they exceed `work_mem`; the same tuning that governs `ORDER BY` and hash joins governs window performance.
