# Topic 33 — PARTITION BY and ORDER BY in OVER
### SQL Mastery Curriculum — Phase 6: Window Functions — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a bakery with several shops across a city. At the end of the day, every shop emails you a spreadsheet — one row per sale, all merged into one giant list of thousands of rows. You want to answer two very different kinds of questions, and the answers depend entirely on *how you slice the paper*.

**Question A: "For each sale, what fraction of that shop's daily revenue did it represent?"**

To answer this you cannot just look at one row. You have to first sort the giant list into piles — one pile per shop. Within the "Downtown" pile you add up all the sales to get Downtown's total, then for each Downtown row you divide its amount by the Downtown total. Then you move to the "Airport" pile and do the same, *starting the addition over from zero*. The Airport total has nothing to do with the Downtown total. Each pile is its own little universe.

That act of "make piles, and compute within each pile independently, restarting at each pile boundary" is exactly **`PARTITION BY`**. The pile is called a *partition*. When you cross from the last Downtown row to the first Airport row, the running total resets, the row counter resets, everything resets. `PARTITION BY shop_id` says "make one pile per shop."

**Question B: "Within each shop's pile, rank the sales from largest to smallest, and show a running total as the day progresses."**

Now, within a pile, *order matters*. A "rank" only means something if you've lined the rows up in some sequence. A "running total up to this point in the day" only means something if you know which rows came before "this point." So inside each pile you lay the receipts out in a specific order — say, by time of sale — and now "the rows before me" and "my position in line" are well-defined.

That act of "put the rows in a defined order so that *position* and *before-me* have meaning" is exactly **`ORDER BY` inside `OVER(...)`**. It does *two* jobs at once, and this is the single most misunderstood thing about window functions: it defines the row ordering, **and** it silently defines a *frame* — a sliding sub-slice of the pile that, by default, means "everything from the start of the pile up to and including my row and any rows tied with me."

So the mental picture is:

- **`PARTITION BY`** = which pile am I in? (horizontal slicing into independent groups)
- **`ORDER BY`** inside `OVER` = in what order are the receipts within my pile, and therefore, which receipts count as "before me"? (vertical sequencing + an implicit sliding window)
- **`OVER()`** with nothing inside = one single pile containing the entire list, in no particular order (the whole result set is one partition).

Everything else about window functions — `RANK`, `LAG`, `SUM() OVER`, moving averages, "top N per group" — is built out of these two knobs. Master the two knobs and the rest is vocabulary.

---

## 2. Connection to SQL Internals

A window function does not change how many rows come out of the query — unlike `GROUP BY`, which collapses rows. It runs *after* the rows are already chosen and computes an extra value per row by looking at a set of *peer* rows. That difference drives everything the engine does under the hood.

**The WindowAgg node.** In PostgreSQL, window functions are executed by a plan node called `WindowAgg`. Its input must arrive *already sorted* in a specific way: first by the `PARTITION BY` columns, then by the `ORDER BY` columns. The partition boundary is detected simply by watching for a change in the partition-key columns as sorted rows stream past. This is why, in `EXPLAIN`, you almost always see a `Sort` node (or an ordered `Index Scan`) feeding directly into `WindowAgg`. `PARTITION BY shop_id ORDER BY sold_at` becomes a physical sort on `(shop_id, sold_at)`.

**Why the sort is the whole cost.** The window function itself is cheap — it's a single forward pass over sorted rows maintaining a little state (a running sum, a row counter, a small ring buffer for the frame). The expensive part is getting the rows sorted. If `work_mem` is large enough, the sort is an in-memory quicksort; if not, it spills to disk as an external merge sort, and you'll see `Sort Method: external merge Disk: NNNNkB` in the plan. So optimizing window queries is almost always about *eliminating or cheapening the sort* — usually by providing a B-tree index whose column order matches `(partition_cols, order_cols)`, letting an `Index Scan` supply pre-sorted rows and skip the `Sort` node entirely.

**The frame and the buffer pool.** When `ORDER BY` is present inside `OVER`, the engine maintains a *frame* — the sliding subset of the current partition that the function aggregates over. For the default frame (`RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`), a running aggregate only needs to remember the accumulator, so memory is O(1) per partition. For frames that look both backwards and forwards (`ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING`), the engine must *materialize* a window of tuples in a tuplestore, which can spill to disk exactly like a sort can. This is invisible in the row count but very visible in buffers and timing.

**No MVCC or WAL involvement.** Window functions are pure read-side computation. They don't touch the write-ahead log, don't create tuple versions, and don't take row locks. Everything they do happens after the heap/index access that produced the input rows — the MVCC visibility check already happened during the scan. So the internals that matter here are: the **planner's sort/index decision**, **`work_mem`**, the **tuplestore** for wide frames, and the **`WindowAgg` executor node**. That's the entire surface area.

---

## 3. Logical Execution Order Context

```
FROM / JOIN            ← rows assembled
WHERE                  ← rows filtered (BEFORE windows exist)
GROUP BY               ← rows collapsed into groups
HAVING                 ← groups filtered
--- window functions evaluate HERE ---
SELECT (window funcs)  ← OVER(...) computed over the surviving rows
DISTINCT
ORDER BY               ← final presentation order (separate from OVER's ORDER BY)
LIMIT / OFFSET
```

Three consequences of this position are responsible for most window-function bugs:

**1. You cannot reference a window function in `WHERE`, `GROUP BY`, or `HAVING`.** Those clauses run *before* the window phase. The rows the window sees are the rows that already survived `WHERE`, `GROUP BY`, and `HAVING`.

```sql
-- ERROR: window function not allowed in WHERE
SELECT * FROM orders
WHERE ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at) = 1;
-- ERROR:  window functions are not allowed in WHERE
```

To filter on a window result you must compute it in a subquery/CTE and filter in the *outer* query, where the window value is now an ordinary column:

```sql
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at) AS rn
  FROM orders
) t
WHERE rn = 1;   -- now legal: rn is a plain column
```

**2. `WHERE` shrinks the window's input.** Because filtering happens first, a `WHERE` clause changes what "the whole partition" means. `SUM(amount) OVER (PARTITION BY shop_id)` sums only the rows that passed `WHERE`. If you filter `WHERE status = 'completed'`, the partition total excludes non-completed rows — sometimes what you want, sometimes a silent bug. If you need the *unfiltered* total, you must not filter those rows out before the window (use a `CASE` inside the aggregate, or a conditional frame, instead of `WHERE`).

**3. `ORDER BY` inside `OVER` is unrelated to the query's final `ORDER BY`.** They are two different clauses that happen to share a keyword. The `OVER(... ORDER BY x)` defines the *window's* internal ordering (and frame); the trailing `ORDER BY y` defines the *rows' display order*. You frequently need both, and they can differ:

```sql
SELECT id, customer_id, created_at,
       ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at) AS seq
FROM orders
ORDER BY customer_id, created_at DESC;  -- display order ≠ window order
```

Remember from Topic 32 that window functions are the last major computation before `DISTINCT`/`ORDER BY`/`LIMIT`. They see a filtered, grouped, but not-yet-ordered-for-display set of rows.

---

## 4. What Is PARTITION BY / ORDER BY in OVER?

A window function computes a value for each row based on a *window* of related rows defined by the `OVER` clause. `PARTITION BY` divides the rows into independent groups; `ORDER BY` (inside `OVER`) orders rows within each group and, by default, establishes a running frame up to the current row.

```sql
SELECT
  window_function(args) OVER (
    PARTITION BY  part_col1, part_col2   -- split rows into independent groups
    ORDER BY      sort_col  [ASC|DESC] [NULLS FIRST|LAST]  -- order within group + implicit frame
    [ frame_clause ]                     -- optional: override the default frame
  )
FROM ...
```

Annotated breakdown of every knob:

```sql
SUM(o.total_amount) OVER (
│   │                └── OVER: "this is a window function; compute over a set of peer rows,
│   │                          do NOT collapse rows the way GROUP BY would"
│   └── the aggregate/window function being applied to each frame
│
    PARTITION BY o.customer_id
    │            └── the grouping key(s). One independent window per distinct value.
    │                Running totals, ranks, row numbers RESET at each partition boundary.
    │                Omit entirely → the ENTIRE result set is one single partition.
    │
    ORDER BY o.created_at ASC NULLS LAST
    │        │            │   └── where NULLs sort. Default: NULLS LAST for ASC, FIRST for DESC.
    │        │            └── direction. Affects rank order AND which rows are "before" current.
    │        └── the column(s) that sequence rows inside each partition.
    │            Presence of ORDER BY changes the DEFAULT FRAME (see below).
    │
    ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW
    │    │       └── frame start ........ frame end ──┘
    │    └── frame mode: ROWS (physical row offsets) | RANGE (value peers) | GROUPS
    └── the frame clause: which subset of the partition this row's function sees.
        If OMITTED and ORDER BY present  → RANGE UNBOUNDED PRECEDING → CURRENT ROW (running)
        If OMITTED and ORDER BY absent   → the WHOLE partition is the frame
)
```

The four canonical shapes of `OVER` you must be able to read instantly:

```sql
-- (a) Empty: one partition = whole result, no ordering, frame = everything.
--     Every row gets the SAME value (a grand total / grand count).
f() OVER ()

-- (b) Partition only: independent group per key, no ordering, frame = whole partition.
--     Every row in a partition gets the SAME value (per-group total).
f() OVER (PARTITION BY k)

-- (c) Order only: one partition (whole result), ordered, DEFAULT running frame.
--     Values accumulate/rank across the entire result set.
f() OVER (ORDER BY c)

-- (d) Both: independent groups, ordered within each, running frame per group.
--     Running totals / ranks that reset at each group boundary. The workhorse.
f() OVER (PARTITION BY k ORDER BY c)
```

The single most important subtlety, restated because it causes more confusion than anything else: **adding `ORDER BY` inside `OVER` silently changes the default frame** from "the whole partition" to "the running range from the start of the partition through the current row's peer group." That is why `SUM(x) OVER (PARTITION BY k)` gives a *constant per-group total*, but `SUM(x) OVER (PARTITION BY k ORDER BY c)` gives a *running total*. Same function, same partition, one extra clause — completely different numbers.

---

## 5. Why PARTITION BY / ORDER BY Mastery Matters in Production

1. **"Top N per group" is a daily requirement and the naive answers are wrong.** "The most recent order per customer," "the top 3 products per category," "the latest status per ticket" — every one of these is a `ROW_NUMBER()/RANK()` over `PARTITION BY group ORDER BY sort`. Developers who don't know windows reach for correlated subqueries or `DISTINCT ON` hacks that are slower and often subtly incorrect on ties.

2. **The `ORDER BY`-changes-the-frame trap silently corrupts reports.** A developer writes `SUM(amount) OVER (PARTITION BY region ORDER BY month)` intending a regional total, gets a running total instead, and ships a dashboard where every row shows a different "total." No error is thrown. The numbers just look plausibly wrong. This is one of the most expensive SQL mistakes because it passes code review and fails silently in production.

3. **Partition size determines whether the query survives at scale.** A window over a partition that turns out to be the entire table (because you forgot `PARTITION BY`, or the partition key has one value) forces a full sort of everything. At 100M rows that's the difference between a 40ms index-ordered scan and a 90-second disk-spilling external sort. Understanding that `PARTITION BY` bounds the sort/frame work is a scaling prerequisite.

4. **Filtering interacts with windows in non-obvious ways.** Because `WHERE` runs before windows (Topic 3), `WHERE status='paid'` changes what a partition total means. Teams routinely produce reports where the "% of total" columns don't add to 100% because the denominator was silently filtered. Knowing the logical order is the fix.

5. **Windows replace whole classes of self-joins and subqueries.** "Compare each row to the previous row," "gap between consecutive events," "difference from the group average" — the pre-window pattern is an expensive self-join or correlated subquery; the window version is a single ordered pass. On large tables this is often a 10–100× speedup and far more readable.

6. **NULL and tie behavior is where correctness lives.** `NULLS FIRST/LAST` in the window `ORDER BY` decides whether the "latest" row is the real latest or a NULL-timestamped junk row. Ties under `RANK` vs `ROW_NUMBER` decide whether "top 3" returns 3 rows or 5. These are the details interviewers probe and production data punishes.

---

## 6. Deep Technical Content

### 6.1 PARTITION BY — Resetting the Window per Group

`PARTITION BY` splits the input rows into groups by the listed columns. Each group is processed as an *independent* window: running aggregates start over, `ROW_NUMBER` restarts at 1, `LAG` cannot see across the boundary, ranks reset. Rows are never combined — every input row still produces exactly one output row.

```sql
-- One running total per customer; resets when customer_id changes
SELECT
  o.customer_id,
  o.id,
  o.created_at,
  o.total_amount,
  SUM(o.total_amount) OVER (
    PARTITION BY o.customer_id
    ORDER BY o.created_at
  ) AS running_customer_spend
FROM orders o;
```

For customer 7 the running total climbs 50 → 130 → 205; when the scan reaches customer 8's first row it resets to that row's amount. The reset is a pure consequence of the partition key changing in the sorted stream.

**Multiple partition columns** define the group by the *combination*:

```sql
PARTITION BY o.customer_id, DATE_TRUNC('month', o.created_at)
-- One window per (customer, month) pair — a customer spans many partitions
```

**No `PARTITION BY`** means one partition containing every row (see 6.4).

**Partitioning is orthogonal to the SELECT list.** You can partition by a column you don't select, and select columns you don't partition by. This is different from `GROUP BY`, where every non-aggregated selected column must be in the `GROUP BY` (see 6.7).

### 6.2 ORDER BY Inside OVER — Two Jobs at Once

`ORDER BY` inside `OVER` does two things simultaneously:

**Job 1 — sequence the rows within each partition.** This is what makes position-based functions meaningful. `ROW_NUMBER`, `RANK`, `LAG(x)`, `FIRST_VALUE`, `NTILE` all require an ordering to have a defined answer. Without `ORDER BY`, "the previous row" and "row number 3" are undefined.

**Job 2 — establish the default frame.** The moment you add `ORDER BY`, the default frame becomes `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. This means aggregate functions (`SUM`, `AVG`, `COUNT`, `MAX`, `MIN`) accumulate from the first row of the partition up to and including the current row's *peer group* (all rows tied on the ORDER BY key). That's why they become *running* aggregates.

```sql
-- Running count of orders per customer, in chronological order
COUNT(*) OVER (PARTITION BY customer_id ORDER BY created_at)
-- Row 1 → 1, row 2 → 2, row 3 → 3 ... within each customer
```

Contrast with no `ORDER BY`:

```sql
-- Total count of orders per customer — SAME value on every row of the partition
COUNT(*) OVER (PARTITION BY customer_id)
```

**Direction affects the answer, not just presentation.** `ORDER BY created_at ASC` makes "before me" mean "earlier"; `DESC` makes "before me" mean "later." A running `SUM` with `DESC` accumulates from newest to oldest. `ROW_NUMBER() ... ORDER BY amount DESC` gives 1 to the biggest.

### 6.3 The Implicit Frame — RANGE vs ROWS and Why Ties Matter

This is the deepest gotcha in window functions. With `ORDER BY` present and no explicit frame, PostgreSQL uses `RANGE UNBOUNDED PRECEDING AND CURRENT ROW`. In `RANGE` mode, "CURRENT ROW" does **not** mean the physical current row — it means "all rows that are *peers* of the current row" (rows with the same `ORDER BY` value). So when there are ties on the ordering column, a running aggregate includes *all tied rows at once*, not one at a time.

```sql
-- Suppose three orders share the exact same created_at within a customer.
SUM(amount) OVER (PARTITION BY customer_id ORDER BY created_at)      -- RANGE (default)
-- All three tied rows show the SAME running sum (the sum through the whole tie group)

SUM(amount) OVER (PARTITION BY customer_id ORDER BY created_at
                  ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)  -- ROWS
-- The three tied rows show three DIFFERENT increasing sums (row-by-row)
```

Concretely, with tied amounts `[10, 10, 10]`:

| mode | row 1 | row 2 | row 3 |
|------|-------|-------|-------|
| `RANGE` (default) | 30 | 30 | 30 |
| `ROWS` | 10 | 20 | 30 |

**Practical rule:** if your `ORDER BY` column is unique (e.g., a primary key, or a timestamp with no duplicates), `RANGE` and `ROWS` produce identical results and you can ignore the distinction. The moment ties are possible on the ordering column, decide deliberately: use `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW` if you want strict row-by-row accumulation. Most "running total" bugs on non-unique order keys trace to the default `RANGE`. `GROUPS` mode is a third option (frame boundaries counted in whole peer-groups) rarely needed in application code.

### 6.4 The Empty OVER() — The Whole-Result-Set Window

`OVER ()` with nothing inside makes the entire result set a single partition with no ordering, and the frame is the whole partition. Every row therefore receives the same value — a *grand* aggregate computed alongside the detail rows, without collapsing them.

```sql
SELECT
  o.id,
  o.total_amount,
  SUM(o.total_amount)  OVER () AS grand_total,
  COUNT(*)             OVER () AS total_orders,
  AVG(o.total_amount)  OVER () AS avg_order_value,
  ROUND(100.0 * o.total_amount
        / SUM(o.total_amount) OVER (), 2) AS pct_of_grand_total
FROM orders o
WHERE o.status = 'completed';
```

Here every row still exists (unlike `GROUP BY`, which would return one row), but each carries the grand total, letting you compute "this order as a % of all completed revenue" in a single pass. Note the `WHERE status='completed'` shrinks the window's input — `grand_total` is the total of *completed* orders only, because filtering happens before the window (Topic 3).

`OVER ()` is the go-to for "add a column comparing each row to the overall aggregate" without a self-join or a scalar subquery. It reads once; the scalar-subquery alternative (`(SELECT SUM(total_amount) FROM orders WHERE status='completed')`) is often re-planned separately and doesn't share the filtered row set.

### 6.5 PARTITION BY vs GROUP BY — The Fundamental Difference

Both split rows into groups. The difference is what happens after:

| aspect | `GROUP BY` | `PARTITION BY` (window) |
|--------|-----------|-------------------------|
| output rows | one per group (collapses) | one per input row (preserves) |
| detail columns | only grouped cols + aggregates | any column, freely |
| where in logical order | before SELECT | during SELECT (after HAVING) |
| can filter on result | yes, via `HAVING` | no; needs outer query |
| purpose | summarize | annotate each row with a group stat |

```sql
-- GROUP BY: 1 row per customer, loses individual orders
SELECT customer_id, SUM(total_amount) AS spend
FROM orders GROUP BY customer_id;

-- PARTITION BY: every order row kept, each annotated with its customer's total
SELECT id, customer_id, total_amount,
       SUM(total_amount) OVER (PARTITION BY customer_id) AS customer_spend
FROM orders;
```

The window version is what you want when you need *both* the detail and the group aggregate on the same row — e.g., "show each order, and next to it, this customer's total spend and this order's share of it." Doing that with `GROUP BY` requires grouping and then joining back to the detail; the window does it in one pass.

You can also combine them: window functions run *after* `GROUP BY`, so a window can operate over the grouped rows.

```sql
-- Rank monthly revenue within each year — window over already-grouped rows
SELECT
  DATE_TRUNC('year',  month) AS yr,
  month,
  monthly_revenue,
  RANK() OVER (PARTITION BY DATE_TRUNC('year', month)
               ORDER BY monthly_revenue DESC) AS rank_in_year
FROM (
  SELECT DATE_TRUNC('month', created_at) AS month,
         SUM(total_amount) AS monthly_revenue
  FROM orders WHERE status='completed'
  GROUP BY 1
) m;
```

### 6.6 Combinations — The Full Matrix

| `OVER` shape | partitions | ordering | default frame | typical use |
|--------------|-----------|----------|---------------|-------------|
| `OVER ()` | 1 (all rows) | none | whole set | grand total / % of total |
| `OVER (PARTITION BY k)` | per k | none | whole partition | per-group total on each row |
| `OVER (ORDER BY c)` | 1 (all rows) | by c | running to current | global running total / global rank |
| `OVER (PARTITION BY k ORDER BY c)` | per k | by c within k | running within partition | running total / rank per group |
| `... ORDER BY c ROWS BETWEEN n PRECEDING AND n FOLLOWING` | per k | by c | explicit sliding | moving average |
| `... ORDER BY c ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING` | per k | by c | whole partition | per-group total *with* an ordering present |

That last row is a useful trick: if you need an ordering for some columns but want a *whole-partition* aggregate for another (not a running one), give that aggregate an explicit full frame to override the running default.

```sql
SELECT
  customer_id, created_at, total_amount,
  ROW_NUMBER() OVER w                                   AS seq,        -- needs order
  SUM(total_amount) OVER (PARTITION BY customer_id
        ORDER BY created_at
        ROWS BETWEEN UNBOUNDED PRECEDING
                 AND UNBOUNDED FOLLOWING)               AS cust_total  -- full frame
FROM orders
WINDOW w AS (PARTITION BY customer_id ORDER BY created_at);
```

### 6.7 The WINDOW Clause — Naming and Reusing Windows

When several functions share the same `OVER` spec, define it once with a `WINDOW` clause and reference it by name. This avoids copy-paste drift and, importantly, lets the planner recognize the shared window and sort only once.

```sql
SELECT
  customer_id,
  created_at,
  total_amount,
  ROW_NUMBER() OVER w              AS seq,
  SUM(total_amount) OVER w         AS running_spend,
  AVG(total_amount) OVER w         AS running_avg,
  LAG(total_amount) OVER w         AS prev_amount
FROM orders
WINDOW w AS (PARTITION BY customer_id ORDER BY created_at)
ORDER BY customer_id, created_at;
```

You can also base one named window on another (`WINDOW w2 AS (w ORDER BY ...)`), inheriting the partition and adding/refining ordering. Sharing the window matters for performance: four functions over the same `w` require a *single* sort, not four.

### 6.8 NULLS Ordering in the Window ORDER BY

`ORDER BY` inside `OVER` obeys the same NULL-placement rules as a normal `ORDER BY`: `ASC` puts `NULLS LAST` by default, `DESC` puts `NULLS FIRST` by default. This directly changes which row is "first" or "last" in a partition, and therefore what `FIRST_VALUE`, `LAST_VALUE`, `ROW_NUMBER`, and running aggregates return.

```sql
-- "Latest order per customer" — but some rows have NULL created_at (bad data)
ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC)
-- DESC → NULLS FIRST by default → a NULL-timestamp row becomes rank 1 ("latest")!
-- Fix: force NULLs to the bottom
ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC NULLS LAST)
```

This is a classic silent-corruption source: a single NULL timestamp hijacks the "latest" slot. Always be explicit about `NULLS FIRST/LAST` when the ordering column is nullable.

### 6.9 Multiple Different Windows in One Query

A single query can carry many window functions with *different* `OVER` specs. Each distinct window spec is a separate sort/`WindowAgg`, executed in a chain. The planner orders them to reuse sorts when specs share a prefix.

```sql
SELECT
  p.category_id,
  p.id,
  p.price,
  RANK()       OVER (PARTITION BY p.category_id ORDER BY p.price DESC) AS price_rank_in_cat,
  AVG(p.price) OVER (PARTITION BY p.category_id)                       AS cat_avg_price,
  AVG(p.price) OVER ()                                                 AS global_avg_price
FROM products p;
```

Three windows: `(category_id ORDER BY price)`, `(category_id)`, and `()`. The engine can compute the first two from one sort on `category_id` (the second is the first minus the ordering) and the third needs no sort. `EXPLAIN` shows stacked `WindowAgg` nodes.

### 6.10 PARTITION BY / ORDER BY with Expressions

Both clauses accept arbitrary expressions, not just columns. This is how you partition by a derived bucket or order by a computed key.

```sql
-- Rank users within their signup cohort (month), ordered by lifetime value
RANK() OVER (
  PARTITION BY DATE_TRUNC('month', u.created_at)
  ORDER BY u.lifetime_value DESC
)
-- Partition by a CASE bucket
SUM(amount) OVER (
  PARTITION BY CASE WHEN amount >= 1000 THEN 'high' ELSE 'low' END
)
```

Expressions in `PARTITION BY`/`ORDER BY` prevent the planner from using a plain-column index to supply the sort order unless a matching *expression index* exists (e.g., `CREATE INDEX ON users ((DATE_TRUNC('month', created_at)))`). Without it, a `Sort` node is unavoidable.

---

## 7. EXPLAIN — WindowAgg in the Plan

### 7.1 The canonical shape: Sort → WindowAgg

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, created_at, total_amount,
       SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY created_at) AS running
FROM orders;
```

```
WindowAgg  (cost=10847.82..12347.82 rows=100000 width=48)
           (actual time=88.2..151.4 rows=100000 loops=1)
  ->  Sort  (cost=10847.82..11097.82 rows=100000 width=40)
            (actual time=88.1..102.3 rows=100000 loops=1)
        Sort Key: customer_id, created_at
        Sort Method: quicksort  Memory: 11592kB
        ->  Seq Scan on orders  (cost=0.00..2185.00 rows=100000 width=40)
                                (actual time=0.02..21.7 rows=100000 loops=1)
Buffers: shared hit=1185
Planning Time: 0.21 ms
Execution Time: 158.9 ms
```

**Reading it:**
- `Seq Scan` reads all 100K rows (no `WHERE`, so a full scan is correct).
- `Sort Key: customer_id, created_at` — the physical sort that realizes `PARTITION BY customer_id ORDER BY created_at`. Partition columns come first, then order columns. This sort is the entire cost of the query (88ms of the 158ms).
- `Sort Method: quicksort Memory: 11592kB` — the sort fit in `work_mem`. If this said `external merge Disk: 23400kB`, the sort spilled and you'd raise `work_mem` or add an index.
- `WindowAgg` — the single forward pass computing the running sum. Cheap: 151.4 − 88.2 ≈ 63ms mostly emitting rows. Note it does **not** reduce the row count: 100000 in, 100000 out (unlike a `GroupAggregate`).

### 7.2 Eliminating the Sort with an index

```sql
CREATE INDEX idx_orders_cust_created ON orders(customer_id, created_at);

EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, created_at, total_amount,
       SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY created_at) AS running
FROM orders;
```

```
WindowAgg  (cost=0.42..8342.42 rows=100000 width=48)
           (actual time=0.05..96.1 rows=100000 loops=1)
  ->  Index Scan using idx_orders_cust_created on orders
            (cost=0.42..6842.42 rows=100000 width=40)
            (actual time=0.03..41.2 rows=100000 loops=1)
Buffers: shared hit=100812
Planning Time: 0.19 ms
Execution Time: 103.4 ms
```

**Reading it:**
- No `Sort` node. The index `(customer_id, created_at)` already returns rows in exactly the order `WindowAgg` needs, so the planner reads them pre-sorted.
- Execution dropped from 158ms to 103ms *and*, more importantly, uses O(1) memory instead of an 11MB sort buffer — it now scales to arbitrarily large tables without spilling.
- Trade-off: `Buffers: shared hit=100812` is much higher than the Seq Scan's 1185, because an index scan visits the heap in index order (random-ish access). On cold cache this can be slower than sort-then-scan. The planner weighs this; on a table that fits in cache, the index scan wins.

### 7.3 A wide frame that materializes (moving average)

```sql
EXPLAIN (ANALYZE)
SELECT sold_at,
       AVG(amount) OVER (ORDER BY sold_at
                         ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING) AS ma7
FROM daily_sales;
```

```
WindowAgg  (cost=1284.20..1584.20 rows=10000 width=24)
           (actual time=12.1..38.4 rows=10000 loops=1)
  Window: w1 AS (ORDER BY sold_at ROWS BETWEEN 3 PRECEDING AND 3 FOLLOWING)
  ->  Sort  (cost=1284.20..1309.20 rows=10000 width=16)
            (actual time=12.0..13.1 rows=10000 loops=1)
        Sort Key: sold_at
        Sort Method: quicksort  Memory: 853kB
        ->  Seq Scan on daily_sales (actual rows=10000 loops=1)
Execution Time: 41.0 ms
```

**Reading it:** The `Window:` line shows the frame is bidirectional (`3 PRECEDING AND 3 FOLLOWING`). `WindowAgg` must buffer up to 7 rows in a tuplestore as it slides. For small frames this stays in memory; a very wide frame (`1000 FOLLOWING`) over a large partition can spill to a temp file — watch for temp buffer growth. The single partition (no `PARTITION BY`) means one big sort on `sold_at`.

### 7.4 What to look for

| symptom in plan | meaning | fix |
|-----------------|---------|-----|
| `Sort` node feeding `WindowAgg` | no index matches window order | add index on `(partition_cols, order_cols)` |
| `Sort Method: external merge Disk:` | sort spilled to disk | raise `work_mem` or add ordering index |
| stacked `WindowAgg` nodes | multiple distinct windows | share specs via `WINDOW` clause / align order |
| high `Buffers` on `Index Scan` | random heap access in index order | consider whether Seq+Sort is cheaper on cold cache |
| `rows` in ≈ `rows` out at WindowAgg | correct — windows don't collapse | (sanity check vs GroupAggregate) |

---

## 8. Query Examples

### Example 1 — Basic: Per-Group Total on Each Row

```sql
-- Show every completed order alongside its customer's total spend and share.
-- PARTITION BY (no ORDER BY) → whole-partition total, same on every row of the group.
SELECT
  o.customer_id,
  o.id                 AS order_id,
  o.total_amount,
  SUM(o.total_amount) OVER (PARTITION BY o.customer_id)          AS customer_total,
  ROUND(100.0 * o.total_amount
        / SUM(o.total_amount) OVER (PARTITION BY o.customer_id), 2) AS pct_of_customer
FROM orders o
WHERE o.status = 'completed'
ORDER BY o.customer_id, o.total_amount DESC;
```

### Example 2 — Intermediate: Running Total + Latest-Per-Group

```sql
-- Per customer: a running chronological spend, plus a flag on the most recent order.
-- PARTITION BY + ORDER BY → running frame (SUM) and row sequencing (ROW_NUMBER).
SELECT
  customer_id,
  order_id,
  created_at,
  total_amount,
  running_spend,
  CASE WHEN recency_rank = 1 THEN 'latest' ELSE '' END AS is_latest
FROM (
  SELECT
    o.customer_id,
    o.id AS order_id,
    o.created_at,
    o.total_amount,
    SUM(o.total_amount) OVER (PARTITION BY o.customer_id
                              ORDER BY o.created_at)          AS running_spend,
    ROW_NUMBER()        OVER (PARTITION BY o.customer_id
                              ORDER BY o.created_at DESC
                              NULLS LAST)                     AS recency_rank
  FROM orders o
  WHERE o.status = 'completed'
) t
ORDER BY customer_id, created_at;
```

### Example 3 — Production Grade: Top 3 Products per Category with Share

```sql
-- Business question: for each category, the 3 highest-revenue products in the last
-- quarter, each product's share of its category's revenue, and its rank.
--
-- Table sizes (assumed):
--   order_items ~ 40M rows, orders ~ 10M, products ~ 200K, categories ~ 2K.
-- Indexes assumed:
--   orders(status, created_at), order_items(order_id), order_items(product_id),
--   products(category_id).
-- Perf expectation: pre-aggregation collapses 40M item rows to ~200K product rows
--   BEFORE the window; the window then sorts ~200K rows partitioned by ~2K categories
--   — a few hundred ms, fully in work_mem. The naive "window over 40M raw rows" would
--   sort 40M rows and spill.
WITH product_rev AS (
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
  GROUP BY p.category_id, p.id, p.name          -- collapse 40M → ~200K
),
ranked AS (
  SELECT
    category_id,
    product_id,
    product_name,
    revenue,
    RANK() OVER (PARTITION BY category_id ORDER BY revenue DESC) AS rev_rank,
    ROUND(100.0 * revenue
          / SUM(revenue) OVER (PARTITION BY category_id), 2)    AS pct_of_category
  FROM product_rev
)
SELECT c.name AS category, r.product_name, r.revenue, r.pct_of_category, r.rev_rank
FROM ranked r
JOIN categories c ON c.id = r.category_id
WHERE r.rev_rank <= 3
ORDER BY c.name, r.rev_rank;
```

```
-- EXPLAIN (ANALYZE) sketch of the ranked/window stage:
Subquery Scan on r  (actual rows=6000 loops=1)
  Filter: (r.rev_rank <= 3)
  Rows Removed by Filter: 194000
  ->  WindowAgg  (actual time=310..355 rows=200000 loops=1)   -- RANK window
        ->  WindowAgg  (actual time=290..305 rows=200000 loops=1) -- SUM per cat
              ->  Sort  (Sort Key: category_id, revenue DESC)
                        Sort Method: quicksort  Memory: 28MB
                        ->  HashAggregate (product_rev)  rows=200000
Execution Time: ~420 ms
```

Two stacked `WindowAgg`s share the `category_id` sort; the `RANK` adds the `revenue DESC` ordering on top of the partition the `SUM` already established.

---

## 9. Wrong → Right Patterns

### Wrong 1: ORDER BY silently turns a group total into a running total

```sql
-- WRONG: intended "each order's share of its customer's TOTAL spend"
SELECT
  o.customer_id, o.id, o.total_amount,
  ROUND(100.0 * o.total_amount
        / SUM(o.total_amount) OVER (PARTITION BY o.customer_id
                                    ORDER BY o.created_at), 2) AS pct
FROM orders o;
-- BUG: the ORDER BY makes SUM a RUNNING total. The denominator grows row by row,
-- so "pct" for the first order is 100%, and it never sums to 100% across the customer.
-- No error — the numbers just look wrong.
```

Why at the execution level: adding `ORDER BY` changes the default frame from the whole partition to `RANGE UNBOUNDED PRECEDING AND CURRENT ROW`. The denominator is the running sum, not the group total.

```sql
-- RIGHT: drop the ORDER BY (whole-partition frame → true group total)
SELECT
  o.customer_id, o.id, o.total_amount,
  ROUND(100.0 * o.total_amount
        / SUM(o.total_amount) OVER (PARTITION BY o.customer_id), 2) AS pct
FROM orders o;
-- Now the denominator is the customer's full total; pct sums to 100% per customer.
```

### Wrong 2: Filtering a window result in WHERE

```sql
-- WRONG: keep only each customer's most recent order
SELECT o.*,
       ROW_NUMBER() OVER (PARTITION BY o.customer_id ORDER BY o.created_at DESC) AS rn
FROM orders o
WHERE rn = 1;
-- ERROR:  column "rn" does not exist
--   (and even aliasing won't help — WHERE runs BEFORE the window is computed)
```

Why: window functions evaluate after `WHERE` in the logical order (Topic 3). `rn` doesn't exist yet when `WHERE` runs.

```sql
-- RIGHT: compute in a subquery, filter in the outer query
SELECT * FROM (
  SELECT o.*,
         ROW_NUMBER() OVER (PARTITION BY o.customer_id
                            ORDER BY o.created_at DESC) AS rn
  FROM orders o
) t
WHERE rn = 1;
```

### Wrong 3: NULLS placement hijacks "latest"

```sql
-- WRONG: some orders have NULL created_at (import bug)
SELECT DISTINCT ON (customer_id) customer_id, id, created_at
FROM (
  SELECT customer_id, id, created_at,
         ROW_NUMBER() OVER (PARTITION BY customer_id
                            ORDER BY created_at DESC) AS rn  -- DESC → NULLS FIRST
  FROM orders
) t WHERE rn = 1;
-- BUG: DESC defaults to NULLS FIRST, so a NULL-timestamp row is ranked "latest".
```

Why: `ORDER BY created_at DESC` places NULLs first by default, so a junk NULL row wins rank 1.

```sql
-- RIGHT: force NULLs to the bottom of the ordering
SELECT * FROM (
  SELECT customer_id, id, created_at,
         ROW_NUMBER() OVER (PARTITION BY customer_id
                            ORDER BY created_at DESC NULLS LAST) AS rn
  FROM orders
) t WHERE rn = 1;
```

### Wrong 4: RANGE default corrupts a running total on tied keys

```sql
-- WRONG: running total ordered by a non-unique day column
SELECT sale_day, amount,
       SUM(amount) OVER (ORDER BY sale_day) AS running   -- default RANGE
FROM daily_sales;
-- BUG: all rows sharing a sale_day get the SAME running value (the sum through the
-- whole day), because RANGE treats tied rows as one peer group. If you expected a
-- strict row-by-row accumulation, it's wrong on every duplicated day.
```

Why: default frame is `RANGE`, whose `CURRENT ROW` includes all peers tied on `sale_day`.

```sql
-- RIGHT: use ROWS for strict per-row accumulation (or add a tiebreaker to ORDER BY)
SELECT sale_day, amount,
       SUM(amount) OVER (ORDER BY sale_day, id
                         ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW) AS running
FROM daily_sales;
```

### Wrong 5: Using WHERE that shrinks the window denominator unintentionally

```sql
-- WRONG: "each order's share of ALL revenue" but filtered to one customer
SELECT o.id, o.total_amount,
       ROUND(100.0*o.total_amount / SUM(o.total_amount) OVER (), 2) AS pct_of_all
FROM orders o
WHERE o.customer_id = 42;
-- BUG: OVER() sums only the rows that passed WHERE — i.e., customer 42's orders,
-- not all revenue. "pct_of_all" is actually pct_of_customer_42.
```

Why: `WHERE` runs before the window, so `OVER ()` only sees customer 42's rows.

```sql
-- RIGHT: keep the global denominator by not pre-filtering it out; filter after,
-- or compute the true total independently.
SELECT id, total_amount, ROUND(100.0*total_amount/global,2) AS pct_of_all
FROM (
  SELECT o.id, o.customer_id, o.total_amount,
         SUM(o.total_amount) OVER () AS global   -- over ALL orders
  FROM orders o
) t
WHERE t.customer_id = 42;                          -- filter AFTER the window
```

---

## 10. Performance Profile

### 10.1 The cost is the sort

A window function is a single linear pass with small per-row state. Its dominant cost is producing input in `(partition_cols, order_cols)` order. So the performance question is always: *sort, or index-provided order?*

| rows | no index (Sort in work_mem) | no index (Sort spills) | index-ordered scan |
|------|-----------------------------|------------------------|--------------------|
| 100K | ~150 ms | — | ~100 ms |
| 1M | ~1.6 s | ~3–5 s if `work_mem` small | ~0.9 s |
| 10M | ~18 s (needs big `work_mem`) | ~40–70 s external merge | ~9 s |
| 100M | usually spills → ~10+ min | ~10–20 min | ~90 s (cache-dependent) |

The index-ordered column trades sort CPU/memory for random heap I/O. On a hot cache it's a clear win and it never spills; on a cold cache with a huge table, sort-then-scan of a sequential read can beat random index-order heap access. Always confirm with `EXPLAIN (ANALYZE, BUFFERS)`.

### 10.2 Partition size drives everything

- **Sort work is roughly O(N log N)** over all rows regardless of partitions (one big sort keyed by partition then order). More, smaller partitions don't reduce sort cost much — but they do bound *frame* memory and enable early emission.
- **A single giant partition** (forgot `PARTITION BY`, or key has one value) is the worst case: a full-table sort plus, for wide frames, a full-partition tuplestore.
- **Very many tiny partitions** are cheap for running aggregates (O(1) state each) but if you `RANK` and filter top-N, you still sort every row first.

### 10.3 Optimization techniques specific to windows

1. **Aggregate before windowing.** As in Example 3, collapse the fact table with `GROUP BY` *before* the window so the window sorts thousands, not millions, of rows. This is the single biggest lever.
2. **Provide an ordering index** on `(partition_cols, order_cols)` to delete the `Sort` node and cap memory.
3. **Share windows** via the `WINDOW` clause so N functions share one sort (Topic 6.7).
4. **Raise `work_mem`** *for the session/query* when a sort is unavoidable and spilling — `SET LOCAL work_mem = '256MB'` inside the transaction. Don't raise it globally; each sort/hash can consume `work_mem`.
5. **Prefer `ROWS` frames over `RANGE`** when both are valid — `ROWS` avoids peer-group scanning and is cheaper, and it sidesteps the tie gotcha.
6. **Push filters that don't affect the denominator into `WHERE`** to shrink the sort input; keep filters that *must not* affect the window out of `WHERE` (use `CASE`/`FILTER` inside the aggregate instead).
7. **Bounded frames don't reduce sort cost** but a very wide bidirectional frame adds tuplestore memory — keep frames as narrow as the requirement allows.

### 10.4 Filtered aggregates without shrinking the window

To include a conditional total without letting `WHERE` shrink the partition, use `FILTER` or `CASE` inside the windowed aggregate:

```sql
SELECT
  customer_id, id, status, total_amount,
  SUM(total_amount) OVER (PARTITION BY customer_id)                              AS all_total,
  SUM(total_amount) FILTER (WHERE status='completed')
                    OVER (PARTITION BY customer_id)                             AS completed_total
FROM orders;
-- Both totals over the SAME (unfiltered) partition; FILTER narrows only one aggregate.
```

---

## 11. Node.js Integration

### 11.1 Basic partitioned window with pg

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Each order with its customer's total and share — parameterized status.
async function ordersWithCustomerShare(status = 'completed') {
  const { rows } = await pool.query(
    `SELECT
       o.customer_id,
       o.id            AS order_id,
       o.total_amount,
       SUM(o.total_amount) OVER (PARTITION BY o.customer_id)          AS customer_total,
       ROUND(100.0 * o.total_amount
             / SUM(o.total_amount) OVER (PARTITION BY o.customer_id), 2) AS pct_of_customer
     FROM orders o
     WHERE o.status = $1
     ORDER BY o.customer_id, o.total_amount DESC`,
    [status]
  );
  return rows;
}
```

### 11.2 Top-N-per-group (window in subquery, filter outside)

```javascript
// "Latest N orders per customer" — the window must be filtered in an outer query.
async function latestOrdersPerCustomer(n = 1) {
  const { rows } = await pool.query(
    `SELECT customer_id, order_id, created_at, total_amount
     FROM (
       SELECT
         o.customer_id,
         o.id AS order_id,
         o.created_at,
         o.total_amount,
         ROW_NUMBER() OVER (PARTITION BY o.customer_id
                            ORDER BY o.created_at DESC NULLS LAST) AS rn
       FROM orders o
     ) t
     WHERE t.rn <= $1
     ORDER BY customer_id, created_at DESC`,
    [n]
  );
  return rows;
}
```

### 11.3 Session-scoped work_mem for a heavy window report

```javascript
// Wrap a heavy windowed report in a transaction and bump work_mem locally,
// so the unavoidable sort stays in memory instead of spilling.
async function heavyWindowReport() {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query(`SET LOCAL work_mem = '256MB'`); // scoped to this txn only
    const { rows } = await client.query(
      `SELECT category_id, product_id, revenue,
              RANK() OVER (PARTITION BY category_id ORDER BY revenue DESC) AS rank
       FROM product_revenue_mv`
    );
    await client.query('COMMIT');
    return rows;
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release(); // work_mem resets automatically — it was SET LOCAL
  }
}
```

### 11.4 Note on ORMs

Numeric window results come back as **strings** from `pg` for `NUMERIC`/`BIGINT` types (to avoid precision loss). Cast in SQL (`::float8`, `::int`) or parse in JS. Also: `ROW_NUMBER`/`RANK` return `bigint` → string. Most query builders (Knex, Drizzle) can express windows via raw fragments; full ORMs generally need raw SQL for the `OVER` clause (see Topic 12).

---

## 12. ORM Comparison

### Prisma

**Can Prisma do window functions?** — No first-class API. Prisma has no `OVER`/`PARTITION BY` builder. You must use `$queryRaw`.

```typescript
const rows = await prisma.$queryRaw`
  SELECT customer_id, id,
         SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY created_at) AS running
  FROM orders
  WHERE status = 'completed'`;
```

**Where it breaks:** everything window-related. `groupBy` collapses rows (like SQL `GROUP BY`) and cannot annotate detail rows.
**Escape hatch:** `$queryRaw` / `$queryRawUnsafe`.
**Verdict:** Raw SQL only. Type the result with a generic on `$queryRaw<T[]>`.

---

### Drizzle ORM

**Can Drizzle do window functions?** — Yes, via `sql` template fragments; there's no dedicated `.over()` builder but the pattern is clean and typed.

```typescript
import { sql } from 'drizzle-orm';

const rows = await db
  .select({
    customerId: orders.customerId,
    id: orders.id,
    running: sql<number>`SUM(${orders.totalAmount}) OVER (
      PARTITION BY ${orders.customerId} ORDER BY ${orders.createdAt})`,
    pct: sql<number>`ROUND(100.0 * ${orders.totalAmount}
      / SUM(${orders.totalAmount}) OVER (PARTITION BY ${orders.customerId}), 2)`,
  })
  .from(orders)
  .where(eq(orders.status, 'completed'));
```

**Where it breaks:** the window body is a raw `sql` string — no type-checking of the `OVER` internals; you own its correctness. Column interpolation is safe against injection.
**Escape hatch:** `db.execute(sql\`...\`)` for anything complex.
**Verdict:** Best of the ORMs for windows — column references stay typed, syntax stays close to SQL.

---

### Sequelize

**Can Sequelize do window functions?** — Only via `sequelize.literal()` embedded in attributes, which is essentially raw SQL with a wrapper.

```javascript
const rows = await Order.findAll({
  attributes: [
    'customerId', 'id',
    [sequelize.literal(
      `SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY created_at)`
    ), 'running'],
  ],
  where: { status: 'completed' },
  raw: true,
});
```

**Where it breaks:** `literal()` is unchecked text; column names must match the DB, not the model. Ordering/limit interactions with `literal` windows are fragile.
**Escape hatch:** `sequelize.query(sql, { type: QueryTypes.SELECT })`.
**Verdict:** Use `sequelize.query()` for anything nontrivial; `literal()` only for one-off columns.

---

### TypeORM

**Can TypeORM do window functions?** — Via `QueryBuilder.addSelect()` with a raw expression, retrieved with `getRawMany()`.

```typescript
const rows = await dataSource
  .getRepository(Order)
  .createQueryBuilder('o')
  .select('o.customer_id', 'customerId')
  .addSelect('o.id', 'id')
  .addSelect(
    `SUM(o.total_amount) OVER (PARTITION BY o.customer_id ORDER BY o.created_at)`,
    'running'
  )
  .where('o.status = :status', { status: 'completed' })
  .getRawMany();
```

**Where it breaks:** window columns are raw strings (no type safety), and they only appear in `getRawMany()` output, not mapped entities.
**Escape hatch:** `dataSource.query(sql, params)`.
**Verdict:** Workable via `addSelect` + `getRawMany`; drop to `.query()` for CTE-based top-N.

---

### Knex.js

**Can Knex do window functions?** — Yes, with first-class helpers: `.rowNumber()`, `.rank()`, `.sum().over()`, `.partitionBy()`, `.orderBy()` on the analytic builder (Knex ≥ 0.21), plus `knex.raw` for anything else.

```javascript
const rows = await knex('orders')
  .select('customer_id', 'id')
  .sum('total_amount')
  .over(qb => qb.partitionBy('customer_id').orderBy('created_at'))
  .as('running')                    // some builds; else use knex.raw
  .where('status', 'completed');

// Most portable: raw fragment
const rows2 = await knex('orders')
  .select('customer_id', 'id',
    knex.raw(`SUM(total_amount) OVER (PARTITION BY customer_id ORDER BY created_at) AS running`))
  .where('status', 'completed');
```

**Where it breaks:** the fluent `.over()` API coverage varies by Knex version and dialect; frame clauses (`ROWS BETWEEN ...`) usually need `knex.raw`.
**Escape hatch:** `knex.raw()`.
**Verdict:** Most SQL-transparent; use `knex.raw` for frames and NULLS ordering.

---

### ORM Summary Table

| ORM | Window support | Partition/Order | Frame clause | Verdict |
|-----|----------------|-----------------|--------------|---------|
| Prisma | None (raw only) | `$queryRaw` | `$queryRaw` | Raw SQL only |
| Drizzle | `sql` fragments | typed column refs | raw in `sql` | Best typed option |
| Sequelize | `literal()` | raw text | raw text | Prefer `sequelize.query()` |
| TypeORM | `addSelect` raw | raw text | raw text | `getRawMany()` / `.query()` |
| Knex | fluent + raw | `.partitionBy/.orderBy` | `knex.raw` | Most transparent |

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given `employees(id, name, salary, department_id)`:

Write a query returning every employee with `name`, `salary`, `department_id`, their department's **average salary**, and the **difference** between their salary and that average. Use a window function — every employee row must be preserved (do not collapse with `GROUP BY`).

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines GROUP BY + window)

Given `orders(id, customer_id, total_amount, status, created_at)`:

First aggregate completed orders into **monthly revenue** (`DATE_TRUNC('month', created_at)`). Then, using a window over that aggregated set, add for each month: the **running year-to-date revenue** (resetting each calendar year) and the **month-over-month change** (this month minus previous month, within the same year). Order the output chronologically.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; naive answer is wrong)

Given `orders(id, customer_id, total_amount, status, created_at)` where some rows have `created_at IS NULL` and some customers have several orders at the exact same `created_at`:

Return exactly **one row per customer** — their single most recent *completed* order (`id`, `created_at`, `total_amount`). Requirements the naive answer gets wrong: (a) NULL timestamps must never count as "most recent"; (b) on an exact timestamp tie, break the tie deterministically by highest `id`; (c) customers with no completed orders must not appear. State why `ROW_NUMBER` (not `RANK`) is required here.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

You are given `payments(id, order_id, amount, status, created_at)`. Produce a report with one row per payment showing: the payment, the **running total of settled payments** (`status='settled'`) per `order_id` in chronological order, and each payment's **percentage of that order's total settled amount** (the full-order denominator, not the running one). Payments that are not settled must still appear in the output but must not contribute to either the running total or the denominator. Explain, in a comment, why you cannot achieve this with a plain `WHERE status='settled'`.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1. What is the difference between `PARTITION BY` and `GROUP BY`?

**Junior answer:** They both group rows by a column.

**Principal answer:** `GROUP BY` *collapses* each group into a single output row and only lets you select grouped columns and aggregates; it runs before `SELECT` in the logical order. `PARTITION BY` *preserves* every input row and annotates each with an aggregate computed over its group; it runs during the `SELECT` phase, after `HAVING`, and you can select any column freely. Use `GROUP BY` to summarize; use `PARTITION BY` when you need the detail row *and* a group-level statistic on the same row (e.g., each order plus its customer's total). They also compose: a window can run over already-grouped rows.

**Follow-up the interviewer asks:** "Can you reproduce a `PARTITION BY` result using `GROUP BY`?" — Yes, by grouping to get the aggregate and joining it back to the detail table on the group key, but that's two passes and a join; the window does it in one ordered pass.

---

### Q2. What does adding `ORDER BY` inside `OVER` change, besides ordering?

**Junior answer:** It just sorts the rows in the window.

**Principal answer:** It does two things. First, it sequences rows within each partition so that position-based functions (`ROW_NUMBER`, `RANK`, `LAG`, `FIRST_VALUE`) have a defined answer. Second — the subtle part — it **changes the default frame** from the whole partition to `RANGE BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`. That converts aggregates like `SUM`/`COUNT`/`AVG` from a constant per-group total into a *running* aggregate. So `SUM(x) OVER (PARTITION BY k)` and `SUM(x) OVER (PARTITION BY k ORDER BY c)` return completely different numbers — group total vs running total — from one extra clause.

**Follow-up:** "And what's the difference between the default `RANGE` frame and `ROWS`?" — In `RANGE`, `CURRENT ROW` includes all rows tied on the `ORDER BY` value (the peer group), so tied rows share one running value; `ROWS` counts physical rows, giving strict row-by-row accumulation. They differ only when the ordering key has duplicates.

---

### Q3. Why can't you put a window function in the `WHERE` clause, and how do you filter on one?

**Junior answer:** SQL just doesn't allow it.

**Principal answer:** Because of logical evaluation order: `WHERE` runs before the window phase (windows evaluate after `WHERE`/`GROUP BY`/`HAVING`, during `SELECT`). At `WHERE` time the window value doesn't exist yet, and even the row set the window will see isn't finalized. To filter on a window result you compute it in a subquery or CTE, exposing it as an ordinary column, then filter in the outer query — the classic `WHERE rn = 1` top-N-per-group pattern. A corollary: `WHERE` shrinks the window's *input*, so a filter changes what a partition total means.

**Follow-up:** "If I `WHERE status='completed'`, what happens to `SUM(amount) OVER ()`?" — It sums only completed rows, because filtering precedes the window. To keep an unfiltered denominator, don't filter those rows out before the window; use `FILTER (WHERE ...)` inside the aggregate or filter in an outer query.

---

### Q4. A running total ordered by a non-unique date column gives duplicate running values on the same date. Why, and how do you fix it?

**Junior answer:** Maybe the data is wrong.

**Principal answer:** The default frame with `ORDER BY` present is `RANGE`, and in `RANGE` mode `CURRENT ROW` means "all peer rows tied on the ordering value." Rows sharing a date are one peer group, so they all get the sum through the end of that group — identical values. To get strict row-by-row accumulation, either switch to `ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW`, or add a unique tiebreaker to the `ORDER BY` (e.g., `ORDER BY sale_day, id`) so no ties remain.

**Follow-up:** "Which is better?" — Adding a deterministic tiebreaker also makes `ROW_NUMBER`/`LAG` deterministic, so it's usually the better fix; `ROWS` alone fixes the aggregate but leaves ranking functions nondeterministic on ties.

---

### Q5. How do you make a `PARTITION BY ... ORDER BY ...` window fast on a 50M-row table?

**Junior answer:** Add an index.

**Principal answer:** The cost is the sort into `(partition_cols, order_cols)` order. Three levers: (1) **Pre-aggregate** before the window if possible, so it sorts thousands not millions of rows — the biggest win. (2) **Create a composite index** on `(partition_cols, order_cols)` so an ordered `Index Scan` supplies pre-sorted input and the `Sort` node disappears, capping memory at O(1) and eliminating disk spills — verify in `EXPLAIN` that the `Sort` node is gone. (3) If a sort is unavoidable, raise `work_mem` with `SET LOCAL` for that query so the sort stays in memory instead of doing an external merge to disk. Also **share windows** via the `WINDOW` clause so multiple functions reuse one sort. Confirm everything with `EXPLAIN (ANALYZE, BUFFERS)` — watch for `Sort Method: external merge Disk:`.

**Follow-up:** "The index scan made it slower — why?" — Index-ordered access reads the heap in random order; on a cold cache over a huge table that random I/O can cost more than a sequential Seq Scan plus an in-memory sort. It's a cache/selectivity trade-off; measure both.

---

## 15. Mental Model Checkpoint

1. You have `SUM(amount) OVER (PARTITION BY region)` and `SUM(amount) OVER (PARTITION BY region ORDER BY month)`. Describe, in words, how the two output columns differ for a region with three months of data.

2. A query has `WHERE status = 'paid'` and a column `SUM(amount) OVER ()`. A colleague expects that column to be total revenue across all statuses. Why are they wrong, and what would you change to give them the all-status total while still showing only paid rows?

3. Your ordering column `event_time` has occasional exact duplicates. You write a running `COUNT(*) OVER (ORDER BY event_time)`. Two rows share a timestamp and both show count 7. Explain why, and give two different fixes.

4. Explain why you can select `product_name` in a `PARTITION BY category_id` window query but cannot select `product_name` in a `GROUP BY category_id` query (without aggregating or grouping it).

5. A "latest order per customer" query using `ORDER BY created_at DESC` returns a row with `created_at = NULL` as the latest for one customer. What single keyword pair fixes it, and why did the NULL win by default?

6. You need each order row to show (a) its running spend within the customer and (b) the customer's grand total — both in one query. Sketch the two `OVER` clauses and explain why (b) needs an explicit frame or no `ORDER BY`.

7. In `EXPLAIN` you see `Sort` feeding `WindowAgg` with `Sort Method: external merge Disk: 240MB`. Give two structurally different ways to eliminate the disk spill and the trade-off of each.

---

## 16. Quick Reference Card

```sql
-- ANATOMY
f() OVER (
  PARTITION BY k1, k2      -- independent group per key combo; resets everything
  ORDER BY c [DESC] [NULLS LAST]   -- sequences rows AND sets default running frame
  [ROWS|RANGE|GROUPS BETWEEN ... AND ...]  -- override frame
)

-- THE FOUR SHAPES
f() OVER ()                              -- whole result = 1 partition (grand total)
f() OVER (PARTITION BY k)                -- per-group total, same on every group row
f() OVER (ORDER BY c)                    -- global running / global rank
f() OVER (PARTITION BY k ORDER BY c)     -- running/rank per group (the workhorse)

-- THE #1 GOTCHA: ORDER BY changes the default frame
SUM(x) OVER (PARTITION BY k)             -- CONSTANT group total
SUM(x) OVER (PARTITION BY k ORDER BY c)  -- RUNNING total (default RANGE→current row)

-- RANGE vs ROWS (matters only with TIES on the order key)
... ORDER BY c                                     -- default RANGE: tied rows share value
... ORDER BY c ROWS BETWEEN UNBOUNDED PRECEDING
                           AND CURRENT ROW          -- strict row-by-row

-- FILTER ON A WINDOW RESULT → outer query (WHERE runs before windows)
SELECT * FROM (SELECT *, ROW_NUMBER() OVER (...) rn FROM t) s WHERE rn = 1;

-- NULLS in the window ORDER BY (default: ASC→LAST, DESC→FIRST)
ORDER BY created_at DESC NULLS LAST      -- keep NULL timestamps out of "latest"

-- WHOLE-partition aggregate WHILE an ordering exists (override running default)
SUM(x) OVER (PARTITION BY k ORDER BY c
             ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)

-- CONDITIONAL total without shrinking the partition
SUM(x) FILTER (WHERE cond) OVER (PARTITION BY k)

-- SHARE a window (one sort for many functions)
SELECT ROW_NUMBER() OVER w, SUM(x) OVER w
FROM t WINDOW w AS (PARTITION BY k ORDER BY c);

-- PERF RULES OF THUMB
-- • cost = the sort into (partition_cols, order_cols); function pass is cheap
-- • index on (partition_cols, order_cols) → deletes Sort node, O(1) memory
-- • aggregate BEFORE windowing to sort thousands not millions of rows
-- • Sort Method: external merge Disk → raise work_mem (SET LOCAL) or add index
-- • windows NEVER reduce row count (rows_in == rows_out at WindowAgg)

-- INTERVIEW ONE-LINERS
-- PARTITION BY = per-group reset, keeps all rows; GROUP BY collapses.
-- ORDER BY in OVER = ordering + implicit running frame (the silent trap).
-- Default frame with ORDER BY = RANGE UNBOUNDED PRECEDING → CURRENT ROW.
-- OVER() = one partition = the whole (filtered) result set.
-- Can't filter a window in WHERE — wrap it and filter outside.
```

---

## Connected Topics

- **Topic 32 — Window Functions Fundamentals**: the `OVER` clause, how windows preserve rows, and why they run after `GROUP BY` — the foundation this topic refines into `PARTITION BY`/`ORDER BY` mechanics.
- **Topic 34 — Ranking Functions**: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `NTILE` — all built on the ordered partitions defined here; tie behavior is the through-line.
- **Topic 35 — Frame Clauses (ROWS / RANGE / GROUPS)**: the full treatment of the frame the `ORDER BY` here implicitly creates — moving averages, sliding windows, `RANGE` peer groups.
- **Topic 36 — LAG / LEAD / FIRST_VALUE / LAST_VALUE**: offset and boundary functions that depend entirely on the partition ordering established here.
- **Topic 20 — GROUP BY Fundamentals**: the collapsing counterpart to `PARTITION BY`; the two are frequently composed (window over grouped rows).
- **Topic 07 — NULL in Depth**: why `NULLS FIRST/LAST` in the window `ORDER BY` decides which row is "first"/"latest," and how three-valued logic shapes ordering.
- **Internals — Sort & work_mem**: the `WindowAgg`-fed sort is the cost center; `work_mem`, external merge spills, and ordering indexes are the levers.
