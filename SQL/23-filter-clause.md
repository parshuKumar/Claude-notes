# Topic 23 — FILTER Clause
### SQL Mastery Curriculum — Phase 4: Aggregation and Grouping

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you're a warehouse manager standing at the end of a conveyor belt. Every box that rolls past you is an order. You have a clipboard and you're counting things.

Without any special tricks, you count every box: "1, 2, 3, 4... 500 boxes total." That's `COUNT(*)`.

Now your boss walks up and asks four questions at once:
- "How many boxes total?"
- "How many were shipped?"
- "How many were cancelled?"
- "How many were shipped AND weigh over 10kg?"

You have two ways to answer.

**The old, awkward way (CASE inside aggregate):** For each question, you invent a rule where a box that doesn't qualify becomes "invisible" — you pretend it's a ghost box worth nothing. To count shipped boxes, you say "if this box is shipped, write down a 1, otherwise write down NULL," and then you count only the non-ghost marks. It works, but you're constantly translating "doesn't qualify" into "pretend it's not there." It's mentally noisy.

**The new, clean way (FILTER):** You put on a pair of magic glasses for each question. The magic glasses for "shipped" simply make the non-shipped boxes disappear from view before you count. You count everything you can see. Then you swap glasses for the next question. The box is still on the belt — you just chose not to look at it for that particular tally.

That's the `FILTER` clause. It's a per-count pair of glasses. Each aggregate gets its own `WHERE`-like condition that decides which rows that aggregate is allowed to see — without kicking those rows out of the whole query, and without you having to fake-NULL anything.

```sql
SELECT
  COUNT(*)                                  AS total_boxes,
  COUNT(*) FILTER (WHERE status = 'shipped')   AS shipped_boxes,
  COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled_boxes,
  COUNT(*) FILTER (WHERE status = 'shipped' AND weight_kg > 10) AS heavy_shipped
FROM orders;
```

One pass over the belt. Four different pairs of glasses. Four answers. That's `FILTER`.

---

## 2. Connection to SQL Internals

`FILTER` is not a new execution primitive. It is **syntactic sugar layered on top of the existing aggregate machinery**, and understanding that is the key to reasoning about its performance.

At the physical layer, PostgreSQL executes an aggregate query using an **Aggregate node** (either `Aggregate` for the whole table, `GroupAggregate` after a sort, or `HashAggregate` using an in-memory hash table keyed by the GROUP BY columns). Internally each aggregate maintains a **transition state** — an accumulator that gets updated once per input row via a *transition function* (`sfunc`), then collapsed once at the end via a *final function* (`ffunc`).

For `COUNT(*)`, the transition state is just an `int8` counter. Its transition function is "state = state + 1". For `SUM(x)`, the state is a running total. For `AVG(x)`, the state is a two-element struct `(sum, count)`.

Here is what `FILTER` does at that level: **before the executor calls the transition function for a given aggregate on a given row, it evaluates the FILTER expression. If the expression is not TRUE, the transition function is simply skipped for that aggregate on that row.** The row still flows through the node; it still updates the *other* aggregates whose filters passed (or which have no filter). Only the filtered-out aggregate's accumulator is left untouched.

This is precisely how the SQL standard defines it, and it is why FILTER and the `CASE`-inside-aggregate trick produce identical results — they both amount to "don't feed this row to this accumulator." The difference is that with `CASE ... END` you rely on the aggregate's own NULL-skipping behavior (`COUNT`, `SUM`, `AVG` all ignore NULL inputs), whereas with `FILTER` the executor skips the transition function call outright.

Relevant internal concepts this touches:

- **Single-pass aggregation**: All the aggregates in a SELECT list — filtered or not — are computed in **one scan** of the input rows. The planner does not re-scan the table once per FILTER. This is the entire performance argument for FILTER over `UNION`-ing several separately-filtered queries.
- **Aggregate transition state / `aggtransfn`**: FILTER gates the call to this function.
- **`HashAggregate` vs `GroupAggregate`**: FILTER changes nothing about which grouping strategy the planner picks; it only changes what gets accumulated inside each group's state.
- **Parallel aggregation**: FILTER is fully compatible with PostgreSQL's parallel aggregate (partial aggregates in workers, combined by a `Finalize Aggregate` node). The filter is evaluated inside each worker's partial aggregation.
- **`work_mem`**: For `COUNT(DISTINCT ...) FILTER (...)`, the distinct-tracking still needs memory/sort, and the filter is applied before the distinct set is built.

The mental takeaway: **FILTER is free relative to CASE.** It compiles to the same "skip this row for this accumulator" operation. You are choosing it for readability and correctness clarity, and occasionally for a marginal expression-evaluation win, not for an algorithmic speedup.

---

## 3. Logical Execution Order Context

```
FROM              ← source tables assembled
JOIN ... ON       ← joins resolved
WHERE             ← row-level filter (BEFORE grouping)  ← runs first
GROUP BY          ← rows collapsed into groups
  └─ aggregates evaluated here, and FILTER lives INSIDE this phase
HAVING            ← group-level filter (AFTER aggregation)  ← Topic 22
SELECT            ← output expressions projected
DISTINCT
window functions  ← OVER (...)
ORDER BY
LIMIT
```

`FILTER` is evaluated **during the aggregation phase**, at the exact moment each aggregate consumes its input rows. This placement has several precise consequences:

1. **WHERE runs before FILTER.** The `WHERE` clause removes rows from the entire input *before* any aggregate sees them. FILTER then further restricts, per-aggregate, which of the *surviving* rows each accumulator consumes. A row eliminated by `WHERE` is invisible to every FILTER. A row eliminated by a FILTER is still visible to `WHERE` (it passed) and still counted by other unfiltered aggregates.

2. **FILTER is not HAVING.** This is the single most common conceptual mix-up, so contrast them explicitly (Topic 22 covered HAVING):
   - `WHERE`  → decides which *rows* enter grouping.
   - `FILTER` → decides which rows each *individual aggregate* accumulates.
   - `HAVING` → decides which *groups* survive after aggregation.
   You can have all three in one query, and they run in that order.

3. **FILTER cannot reference the aggregate's own output** — it's a row predicate evaluated per input row, so it may reference the columns available at grouping time (base columns, joined columns), exactly like `WHERE` and `GROUP BY` expressions. It may **not** reference other aggregates or the SELECT-list output aliases.

4. **GROUP BY interaction**: When a GROUP BY is present, each group runs its aggregates independently, and each aggregate's FILTER is applied within that group's rows. So `COUNT(*) FILTER (WHERE ...)` per group counts, within each group, only that group's rows that pass the filter.

Because FILTER lives inside the aggregation phase, it is available anywhere an aggregate is: in a plain `GROUP BY` query, in a whole-table aggregate, and — importantly — on the aggregate form of a **window function** (`SUM(...) FILTER (WHERE ...) OVER (...)`), covered in the deep section.

---

## 4. What Is the FILTER Clause?

The `FILTER` clause is a per-aggregate row predicate. It attaches a `WHERE`-style condition directly to a single aggregate function call, so that the aggregate accumulates only the rows for which the condition evaluates to TRUE. Rows for which it is FALSE or UNKNOWN (NULL) are excluded from that one aggregate — and only that one.

```sql
aggregate_function ( arguments ) FILTER ( WHERE condition )
│                    │                    │      │
│                    │                    │      └── a boolean row predicate, same rules as WHERE:
│                    │                    │           only rows where this is TRUE are accumulated;
│                    │                    │           FALSE and NULL/UNKNOWN rows are skipped
│                    │                    │
│                    │                    └── the FILTER keyword, then a parenthesized WHERE clause;
│                    │                        the parentheses and the WHERE keyword are both REQUIRED
│                    │
│                    └── the normal aggregate argument(s): *, a column, an expression,
│                        or DISTINCT expr — the filter applies BEFORE DISTINCT dedup
│
└── any aggregate: COUNT, SUM, AVG, MIN, MAX, array_agg, string_agg,
    jsonb_agg, bool_and, bool_or, percentile_cont, etc.
```

Optionally combined with `OVER` to filter a window aggregate:

```sql
aggregate_function ( arguments ) FILTER ( WHERE condition ) OVER ( window_spec )
│                                 │                          │
│                                 │                          └── window frame/partition; the FILTER
│                                 │                              restricts which rows contribute to
│                                 │                              the running aggregate over the frame
│                                 └── evaluated per row within the partition, before the window sums
│
└── note: FILTER is allowed on the AGGREGATE form of a window function only,
    NOT on pure window functions like ROW_NUMBER(), RANK(), LEAD(), LAG()
```

### Exact Semantics

Given `orders`:

```
id | status     | total_amount
---+------------+-------------
1  | shipped    | 100
2  | shipped    | 200
3  | cancelled  | 50
4  | pending    | NULL
5  | shipped    | NULL
```

```sql
SELECT
  COUNT(*)                                        AS all_rows,          -- 5
  COUNT(*) FILTER (WHERE status = 'shipped')      AS shipped_rows,      -- 3 (ids 1,2,5)
  SUM(total_amount)                               AS sum_all,           -- 300 (NULLs ignored)
  SUM(total_amount) FILTER (WHERE status='shipped') AS sum_shipped,     -- 300 (ids 1,2; id5 amt NULL)
  COUNT(total_amount) FILTER (WHERE status='shipped') AS cnt_shipped_amt -- 2 (id5 amt is NULL)
FROM orders;
```

```
all_rows | shipped_rows | sum_all | sum_shipped | cnt_shipped_amt
---------+--------------+---------+-------------+----------------
       5 |            3 |     300 |         300 |               2
```

The complete behaviour:

- **Only TRUE passes.** `WHERE status = 'shipped'` on the `pending` and `cancelled` rows evaluates FALSE → skipped. If the filter column itself were NULL, the comparison would be UNKNOWN → also skipped.
- **FILTER stacks with the aggregate's own NULL handling.** `SUM(total_amount) FILTER (WHERE status='shipped')` first restricts to shipped rows (ids 1, 2, 5), *then* `SUM` ignores the NULL amount of id 5 → `300`. `COUNT(total_amount)` (counting non-NULL amounts) further drops id 5 → `2`. FILTER and the aggregate's argument-NULL rule are independent gates applied in sequence.
- **`COUNT(*) FILTER` counts qualifying rows regardless of any column NULLness** — because `*` has no argument to be NULL. Use `COUNT(*) FILTER (...)` to count rows, `COUNT(col) FILTER (...)` to count rows where both the filter passes *and* `col` is non-NULL.
- **Empty filter result is not NULL for COUNT.** If no row passes the filter, `COUNT(*) FILTER (...)` returns `0`, but `SUM(...) FILTER (...)`, `AVG`, `MIN`, `MAX`, `array_agg` return `NULL` (the standard empty-aggregate behavior). This distinction bites in production — covered in Section 6.

---

## 5. Why FILTER Mastery Matters in Production

1. **It replaces the most error-prone aggregation idiom.** The `CASE WHEN cond THEN 1 END`-inside-`COUNT` and `CASE WHEN cond THEN col END`-inside-`SUM` patterns are everywhere in legacy analytics SQL, and they are subtly bug-prone: developers write `THEN 0 ELSE NULL` (turning a count into a sum of zeros that silently counts *every* row via the wrong aggregate), or `COUNT(CASE WHEN cond THEN 1 ELSE 0 END)` (which counts ALL rows because `0` is non-NULL). FILTER makes the intent unambiguous and removes the class of bug.

2. **Conditional aggregation / pivoting without an extension.** Turning rows into columns ("how many orders in each status, as columns") is a daily reporting need. FILTER does it in standard SQL with no `crosstab`/`tablefunc` extension, no application-side pivoting, and no multiple round-trips.

3. **One-pass dashboards.** A metrics dashboard that shows "total, completed, refunded, this-week, this-month" is naturally several filtered aggregates. Without FILTER (or CASE), naive developers issue N separate `SELECT COUNT(*) WHERE ...` queries — N table scans, N round-trips. FILTER collapses them into a single scan. At 100M rows the difference is minutes vs seconds.

4. **Correctness under GROUP BY fan-out.** After a one-to-many JOIN (Topic 11's fan-out problem), you often need "count of orders in status X per customer." FILTER expresses this precisely per group and composes with `COUNT(DISTINCT ...)` to handle multiplication.

5. **Readability at scale.** A SELECT list of ten `CASE WHEN` expressions is unreadable in code review. Ten `FILTER (WHERE ...)` clauses read like English. On a team, the maintainability difference is real: reviewers catch filter-logic bugs faster.

6. **It's a standard, portable feature.** `FILTER (WHERE ...)` is ANSI SQL:2003. It works in PostgreSQL, and the CASE fallback works everywhere else — knowing both means you can write the clean version where supported and the portable version where not (MySQL, older SQLite).

Without FILTER mastery you get: multiple redundant scans, silently-wrong counts from `ELSE 0`, unreadable CASE soup, and reliance on the `tablefunc` extension for pivots you could have written in plain SQL.

---

## 6. Deep Technical Content

### 6.1 FILTER vs CASE-Inside-Aggregate — Exact Equivalence

Every `FILTER` can be rewritten as a `CASE` inside the aggregate, and vice versa. The equivalences:

```sql
-- COUNT of rows matching a condition
COUNT(*) FILTER (WHERE cond)
COUNT(CASE WHEN cond THEN 1 END)        -- note: NO "ELSE 0" — that would count everything
COUNT(*)      FILTER == COUNT(1) FILTER -- 1 is non-NULL, same as *

-- SUM restricted to matching rows
SUM(amount) FILTER (WHERE cond)
SUM(CASE WHEN cond THEN amount END)      -- non-matching rows become NULL, SUM ignores them
SUM(CASE WHEN cond THEN amount ELSE 0 END) -- ALSO correct for SUM (0 is additive identity)

-- AVG restricted to matching rows
AVG(amount) FILTER (WHERE cond)
AVG(CASE WHEN cond THEN amount END)      -- CORRECT: non-matching → NULL → excluded from AVG
AVG(CASE WHEN cond THEN amount ELSE 0 END) -- WRONG: 0s are averaged in, dragging AVG down!
```

The `AVG` row exposes exactly why FILTER is safer. With FILTER, non-matching rows *do not exist* for that aggregate — there is no "else value" to accidentally include. With CASE, you must remember that `ELSE 0`:
- is **harmless for SUM** (adds zero),
- is **catastrophic for AVG** (adds a data point of value 0 to the denominator and numerator),
- is **catastrophic for COUNT** (0 is non-NULL, so `COUNT` counts it → counts every row),
- is **catastrophic for MIN** (0 may become the new minimum).

FILTER eliminates the entire `ELSE` decision. That is the readability-and-correctness argument in one sentence: **FILTER has no ELSE branch to get wrong.**

### 6.2 The COUNT-returns-0, SUM-returns-NULL Asymmetry

When *no* row passes the filter:

```sql
SELECT
  COUNT(*)   FILTER (WHERE status='unicorn')      AS c,   -- 0
  SUM(amount) FILTER (WHERE status='unicorn')     AS s,   -- NULL
  AVG(amount) FILTER (WHERE status='unicorn')     AS a,   -- NULL
  MAX(amount) FILTER (WHERE status='unicorn')     AS m,   -- NULL
  array_agg(id) FILTER (WHERE status='unicorn')   AS arr  -- NULL (not empty array!)
FROM orders;
```

This is not FILTER-specific — it is the universal empty-input aggregate rule — but FILTER makes empty inputs *common* (a filter that matches nothing per group). Two production consequences:

```sql
-- Guard SUM/AVG so an empty filter yields 0 instead of NULL for display:
COALESCE(SUM(amount) FILTER (WHERE status='shipped'), 0) AS shipped_revenue

-- Guard division that uses a filtered denominator:
ROUND(
  100.0 * COUNT(*) FILTER (WHERE status='refunded')
        / NULLIF(COUNT(*), 0)
, 2) AS refund_rate_pct
```

`array_agg ... FILTER` returning `NULL` (not `{}`) when nothing matches is a frequent surprise when building JSON payloads — wrap with `COALESCE(array_agg(...) FILTER(...), '{}')` if you need an empty array.

### 6.3 Conditional Aggregation — Pivoting Rows into Columns

The canonical use: turn a categorical column's values into separate columns of counts or sums. This is a "pivot" or "crosstab," and FILTER does it natively.

```sql
-- Order counts by status, as columns (one output row for the whole table)
SELECT
  COUNT(*)                                     AS total,
  COUNT(*) FILTER (WHERE status = 'pending')   AS pending,
  COUNT(*) FILTER (WHERE status = 'shipped')   AS shipped,
  COUNT(*) FILTER (WHERE status = 'delivered') AS delivered,
  COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled,
  COUNT(*) FILTER (WHERE status = 'refunded')  AS refunded
FROM orders;
```

Compared with the `tablefunc` extension's `crosstab()`:

| Aspect | `FILTER` pivot | `crosstab()` |
|--------|----------------|--------------|
| Extension required | None (core SQL) | `CREATE EXTENSION tablefunc` |
| Column list | Explicit, statically known | Must match a column definition list |
| Dynamic categories | Needs dynamic SQL either way | Needs dynamic SQL either way |
| Readability | High | Low (string-y, positional) |
| Mixed aggregates (count + sum + avg per category) | Trivial — just add more filtered aggregates | Awkward |
| NULL handling | Explicit and standard | Fiddly |

FILTER wins for every case where the set of pivot columns is known at query-authoring time (the overwhelming majority of dashboards). `crosstab` retains a niche only for truly dynamic column sets, and even then dynamic SQL that generates FILTER clauses is usually cleaner.

### 6.4 Multi-Metric Pivots — Count AND Sum AND Avg per Category

FILTER's real power over `crosstab` is mixing aggregate types in one pass:

```sql
SELECT
  -- counts
  COUNT(*) FILTER (WHERE status='shipped')                 AS shipped_orders,
  COUNT(*) FILTER (WHERE status='refunded')                AS refunded_orders,
  -- sums per category
  SUM(total_amount) FILTER (WHERE status='shipped')        AS shipped_revenue,
  SUM(total_amount) FILTER (WHERE status='refunded')       AS refunded_amount,
  -- averages per category
  AVG(total_amount) FILTER (WHERE status='shipped')        AS avg_shipped_order,
  -- distinct counts per category
  COUNT(DISTINCT customer_id) FILTER (WHERE status='shipped') AS shipped_customers
FROM orders
WHERE created_at >= DATE_TRUNC('month', CURRENT_DATE);
```

One scan, six metrics, three aggregate families, five different implicit sub-populations. Expressing this with `crosstab` would be impractical; expressing it with CASE would be six `CASE` expressions with three different correctness rules for the ELSE branch.

### 6.5 FILTER with GROUP BY

FILTER composes with GROUP BY: each group computes its filtered aggregates over that group's rows.

```sql
-- Per-customer status breakdown
SELECT
  customer_id,
  COUNT(*)                                        AS total_orders,
  COUNT(*) FILTER (WHERE status = 'delivered')    AS delivered,
  COUNT(*) FILTER (WHERE status = 'cancelled')    AS cancelled,
  SUM(total_amount) FILTER (WHERE status='delivered') AS delivered_revenue,
  ROUND(
    100.0 * COUNT(*) FILTER (WHERE status='cancelled')
          / NULLIF(COUNT(*), 0)
  , 1) AS cancel_rate_pct
FROM orders
GROUP BY customer_id
HAVING COUNT(*) >= 5                    -- Topic 22: group-level filter, runs AFTER aggregation
ORDER BY cancel_rate_pct DESC;
```

Note the three-layer filtering in one query:
- (no `WHERE` here, but there could be — pre-aggregation row filter)
- `FILTER` — per-aggregate, per-group
- `HAVING COUNT(*) >= 5` — drops groups (customers) with fewer than 5 orders *after* aggregating

### 6.6 FILTER with COUNT(DISTINCT ...)

FILTER applies **before** the DISTINCT deduplication inside the aggregate:

```sql
-- Number of distinct products sold, but only in completed orders
SELECT
  COUNT(DISTINCT oi.product_id)
    FILTER (WHERE o.status = 'completed') AS distinct_completed_products
FROM order_items oi
INNER JOIN orders o ON o.id = oi.order_id;
```

Semantics: first restrict to rows where `o.status='completed'`, then collect the distinct `product_id` values from those rows, then count them. The order matters — a product appearing only in cancelled orders is excluded entirely. This is exactly the fan-out-safe pattern from Topic 11, now made conditional.

### 6.7 FILTER on Non-COUNT Aggregates

FILTER works on *any* aggregate, which unlocks patterns awkward to express otherwise:

```sql
SELECT
  -- most recent delivered order date per customer
  MAX(created_at) FILTER (WHERE status='delivered')       AS last_delivered_at,
  -- min order value among high-value tier
  MIN(total_amount) FILTER (WHERE total_amount > 500)     AS cheapest_big_order,
  -- concatenate only the SKUs that shipped
  string_agg(sku, ',') FILTER (WHERE status='shipped')    AS shipped_skus,
  -- collect refunded order ids into an array
  array_agg(id) FILTER (WHERE status='refunded')          AS refunded_ids,
  -- build a JSON list of pending orders
  jsonb_agg(jsonb_build_object('id', id, 'amt', total_amount))
    FILTER (WHERE status='pending')                       AS pending_json,
  -- median order value among completed only
  percentile_cont(0.5) WITHIN GROUP (ORDER BY total_amount)
    FILTER (WHERE status='completed')                     AS median_completed,
  -- did ANY order exceed the limit?
  bool_or(total_amount > 10000)                           AS has_whale_order,
  -- were ALL orders paid?
  bool_and(paid)                                          AS all_paid
FROM orders
GROUP BY customer_id;
```

Note `percentile_cont(...) WITHIN GROUP (...) FILTER (...)` — the ordered-set aggregate takes both a `WITHIN GROUP` ordering clause and a `FILTER` clause; the FILTER goes *after* `WITHIN GROUP`.

### 6.8 FILTER on Window (Aggregate) Functions

FILTER is legal on the **aggregate form** of a window function:

```sql
SELECT
  order_id,
  created_at,
  status,
  total_amount,
  -- running total of ONLY completed order amounts, over time
  SUM(total_amount) FILTER (WHERE status='completed')
    OVER (ORDER BY created_at)                       AS running_completed_revenue,
  -- count of refunds so far in this customer's history
  COUNT(*) FILTER (WHERE status='refunded')
    OVER (PARTITION BY customer_id ORDER BY created_at) AS refunds_to_date
FROM orders;
```

The FILTER restricts which rows within the window frame contribute to the running aggregate. Rows that fail the filter still appear in the output (windows don't drop rows) but don't add to the accumulator.

**Restriction:** FILTER is **not** allowed on pure window/ranking functions — `ROW_NUMBER()`, `RANK()`, `DENSE_RANK()`, `NTILE()`, `LAG()`, `LEAD()`, `FIRST_VALUE()`, `LAST_VALUE()`. These are not aggregates and have no transition state to gate.

```sql
-- ERROR: FILTER not allowed on ROW_NUMBER
ROW_NUMBER() FILTER (WHERE status='shipped') OVER (...)   -- ✗ syntax/semantic error
```

If you need a "rank among only shipped rows," you filter in a subquery/CTE first, then rank.

### 6.9 What FILTER Cannot Do

- **Cannot reference an aggregate or another aggregate's result.** `COUNT(*) FILTER (WHERE SUM(x) > 5)` is illegal — the filter is a per-row predicate, not a group predicate. Use `HAVING` for group predicates.
- **Cannot reference SELECT-list aliases.** Like WHERE, the filter is evaluated before the SELECT projection.
- **Cannot be used on a non-aggregate scalar function.** `LENGTH(x) FILTER (...)` is nonsense — FILTER attaches only to aggregates (including their window form).
- **Cannot filter on a window function's result inside the same expression.**
- **The `WHERE` keyword is mandatory.** `COUNT(*) FILTER (status='x')` is a syntax error; it must be `FILTER (WHERE status='x')`.

### 6.10 FILTER and Multiple Conditions / Complex Predicates

The filter predicate is a full boolean expression — anything legal in a `WHERE` clause (except aggregates and subqueries that reference the outer aggregate context; scalar subqueries are technically allowed but rare and usually a smell):

```sql
COUNT(*) FILTER (
  WHERE status = 'shipped'
    AND created_at >= CURRENT_DATE - INTERVAL '7 days'
    AND total_amount BETWEEN 50 AND 500
    AND channel IN ('web','mobile')
) AS recent_midvalue_shipped
```

### 6.11 FILTER vs WHERE — When the Same Result, Which to Use

If you need **only one** sub-population and nothing else, `WHERE` is simpler and often faster (it removes rows before aggregation, shrinking the input):

```sql
-- If shipped_count is the ONLY thing you need:
SELECT COUNT(*) FROM orders WHERE status = 'shipped';   -- simplest, can use a partial/normal index

-- FILTER only earns its keep when you need MULTIPLE sub-populations in one pass:
SELECT
  COUNT(*)                                    AS total,
  COUNT(*) FILTER (WHERE status='shipped')    AS shipped
FROM orders;   -- WHERE cannot give you both total and shipped in one row
```

Rule of thumb: **one condition, one number → WHERE. Several conditions, several numbers, one scan → FILTER.**

### 6.12 NULL and Three-Valued Logic in the FILTER Predicate

The FILTER predicate follows the same three-valued logic as WHERE (Topic 07): only **TRUE** admits the row. **FALSE** and **UNKNOWN (NULL)** both skip it.

```
-- If some rows have status = NULL:
COUNT(*) FILTER (WHERE status = 'shipped')       -- NULL-status rows: NULL='shipped' → UNKNOWN → skipped
COUNT(*) FILTER (WHERE status IS DISTINCT FROM 'shipped')  -- NULL rows: TRUE → counted (NULL ≠ 'shipped')
COUNT(*) FILTER (WHERE status <> 'shipped')      -- NULL rows: UNKNOWN → skipped (misses NULLs!)
```

If you intend "everything that is not 'shipped', including NULLs," you must use `IS DISTINCT FROM` — the plain `<>` silently drops NULL-status rows, a classic bug.

---

## 7. EXPLAIN — FILTER in the Plan

FILTER does not create its own plan node. It appears *inside* the Aggregate node's output expressions. The key thing EXPLAIN shows is that **all filtered aggregates share a single Aggregate node over a single scan** — proving the one-pass property.

### Whole-table filtered aggregation (single Aggregate node)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT
  COUNT(*)                                     AS total,
  COUNT(*) FILTER (WHERE status='shipped')     AS shipped,
  SUM(total_amount) FILTER (WHERE status='refunded') AS refunded_amt
FROM orders;
```

```
Aggregate  (cost=20834.00..20834.01 rows=1 width=24)
           (actual time=182.4..182.4 rows=1 loops=1)
  ->  Seq Scan on orders  (cost=0.00..15834.00 rows=1000000 width=14)
                          (actual time=0.02..71.9 rows=1000000 loops=1)
  Buffers: shared hit=64 read=5770
Planning Time: 0.11 ms
Execution Time: 182.5 ms
```

**Reading it:**
- A **single `Aggregate` node** sits over a **single `Seq Scan`**. All three aggregates (total, shipped, refunded_amt) are computed in that one node during that one scan. There is no repeated scan per filter — this is the one-pass guarantee, visible.
- `rows=1000000` scanned once; the filters are evaluated as part of the aggregate's per-row transition, not as a separate `Filter:` line (a WHERE would show a `Filter:` line and reduce the row count into the aggregate).
- Contrast: three separate `SELECT COUNT(*) WHERE ...` queries would produce three plans, three Seq Scans, ~3× the buffers and time.

### FILTER using a partial index

If you have a partial index matching a filter, the planner may use it — but only when the *entire query* can be satisfied by that predicate, typically when there's also a matching WHERE, or in an index-only aggregate path. A partial index does not automatically accelerate a FILTER that coexists with unfiltered aggregates (those still need every row). Example where it does help — a query that is effectively one filtered count:

```sql
CREATE INDEX orders_shipped_idx ON orders (created_at) WHERE status = 'shipped';

EXPLAIN (ANALYZE, BUFFERS)
SELECT COUNT(*) FILTER (WHERE status='shipped') AS shipped
FROM orders
WHERE status = 'shipped';     -- WHERE lets the partial index apply
```

```
Aggregate  (cost=4210.00..4210.01 rows=1 width=8)
           (actual time=41.2..41.2 rows=1 loops=1)
  ->  Index Only Scan using orders_shipped_idx on orders
        (cost=0.42..3960.00 rows=200000 width=0)
        (actual time=0.03..24.1 rows=200000 loops=1)
        Heap Fetches: 0
  Buffers: shared hit=552
Planning Time: 0.14 ms
Execution Time: 41.3 ms
```

**Reading it:** Here the `WHERE status='shipped'` shrank the input via the partial index (200K rows, not 1M), and the redundant FILTER is now a no-op TRUE for every scanned row. Buffers dropped from ~5800 to 552. **Lesson:** if you only need the filtered population, push the predicate to WHERE so an index can help; FILTER alongside unfiltered aggregates cannot skip the full scan because the other aggregates need all rows.

### GROUP BY with FILTER (HashAggregate)

```sql
EXPLAIN (ANALYZE)
SELECT customer_id,
       COUNT(*) FILTER (WHERE status='delivered') AS delivered,
       COUNT(*) FILTER (WHERE status='cancelled') AS cancelled
FROM orders
GROUP BY customer_id;
```

```
HashAggregate  (cost=18334.00..18584.00 rows=20000 width=24)
               (actual time=201.7..214.3 rows=20000 loops=1)
  Group Key: customer_id
  Batches: 1  Memory Usage: 2833kB
  ->  Seq Scan on orders  (cost=0.00..15834.00 rows=1000000 width=10)
                          (actual time=0.01..70.2 rows=1000000 loops=1)
Planning Time: 0.12 ms
Execution Time: 215.9 ms
```

**Reading it:**
- `HashAggregate` with `Group Key: customer_id` — one hash bucket per customer, each holding two counter states (delivered, cancelled). The FILTER gates which counter increments per row.
- `Batches: 1` — the hash table of group states fit in `work_mem`. If it spilled, you'd see `Batches: > 1` and disk I/O — FILTER doesn't change this; the number of *groups* and their state size does.
- The filters add negligible per-row cost (two boolean comparisons) versus the scan and hashing.

### Parallel filtered aggregate

```
Finalize Aggregate  (cost=10841.55..10841.56 rows=1 width=16)
  ->  Gather  (cost=10841.33..10841.54 rows=2 width=16)
        Workers Planned: 2
        ->  Partial Aggregate  (cost=9841.33..9841.34 rows=1 width=16)
              ->  Parallel Seq Scan on orders
                    (actual rows=333333 loops=3)
```

**Reading it:** Each worker computes a `Partial Aggregate` — including the FILTER — over its slice, then `Finalize Aggregate` combines them. FILTER is parallel-safe; the per-worker partial states are merged by the combine function exactly as unfiltered aggregates are.

---

## 8. Query Examples

### Example 1 — Basic: Status Breakdown in One Row

```sql
-- Single-row dashboard summary of the whole orders table.
-- One scan, several sub-populations as columns.
SELECT
  COUNT(*)                                        AS total_orders,
  COUNT(*) FILTER (WHERE status = 'pending')      AS pending,
  COUNT(*) FILTER (WHERE status = 'shipped')      AS shipped,
  COUNT(*) FILTER (WHERE status = 'delivered')    AS delivered,
  COUNT(*) FILTER (WHERE status = 'cancelled')    AS cancelled,
  -- revenue only from money-realizing statuses
  COALESCE(SUM(total_amount) FILTER (WHERE status IN ('shipped','delivered')), 0)
                                                  AS realized_revenue
FROM orders;
```

### Example 2 — Intermediate: Per-Customer Conditional Metrics with Rates

```sql
-- Per-customer behavioral profile in a single pass.
-- Combines FILTER (per-aggregate), GROUP BY (per-customer),
-- and HAVING (drop low-volume customers).
SELECT
  c.id                                            AS customer_id,
  c.name,
  COUNT(o.id)                                     AS total_orders,
  COUNT(o.id) FILTER (WHERE o.status='delivered') AS delivered,
  COUNT(o.id) FILTER (WHERE o.status='refunded')  AS refunded,
  COALESCE(SUM(o.total_amount)
    FILTER (WHERE o.status='delivered'), 0)       AS delivered_revenue,
  ROUND(
    100.0 * COUNT(o.id) FILTER (WHERE o.status='refunded')
          / NULLIF(COUNT(o.id), 0)
  , 2)                                            AS refund_rate_pct,
  MAX(o.created_at) FILTER (WHERE o.status='delivered')
                                                  AS last_delivery_at
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id
WHERE o.created_at >= CURRENT_DATE - INTERVAL '1 year'
GROUP BY c.id, c.name
HAVING COUNT(o.id) >= 3                            -- ignore customers with <3 orders
ORDER BY refund_rate_pct DESC NULLS LAST
LIMIT 100;
```

### Example 3 — Production Grade: Daily Funnel + Revenue Pivot

```sql
-- Daily operational report over a partitioned/large orders table.
--
-- Table sizes:  orders ~ 40M rows (5 years), order_items ~ 180M rows.
-- Indexes:      orders(created_at) btree; orders(created_at) WHERE status='refunded' partial;
--               order_items(order_id) btree.
-- Perf target:  < 500ms for a 30-day window on warm cache.
-- Why FILTER:   one scan of the 30-day slice yields ~8 metrics; the alternative
--               (8 separate WHERE queries) would scan the slice 8 times.
WITH day_orders AS (
  SELECT
    DATE_TRUNC('day', o.created_at)::date         AS day,
    o.id,
    o.customer_id,
    o.status,
    o.total_amount,
    o.channel
  FROM orders o
  WHERE o.created_at >= CURRENT_DATE - INTERVAL '30 days'
    AND o.created_at <  CURRENT_DATE
)
SELECT
  day,
  COUNT(*)                                              AS orders_created,
  COUNT(DISTINCT customer_id)                           AS unique_customers,
  -- funnel stages as columns
  COUNT(*) FILTER (WHERE status='paid')                 AS paid,
  COUNT(*) FILTER (WHERE status='shipped')              AS shipped,
  COUNT(*) FILTER (WHERE status='delivered')            AS delivered,
  COUNT(*) FILTER (WHERE status='cancelled')            AS cancelled,
  COUNT(*) FILTER (WHERE status='refunded')             AS refunded,
  -- revenue realized vs lost
  COALESCE(SUM(total_amount) FILTER (WHERE status IN ('shipped','delivered')), 0)
                                                        AS realized_revenue,
  COALESCE(SUM(total_amount) FILTER (WHERE status='refunded'), 0)
                                                        AS refunded_revenue,
  -- channel split of delivered orders
  COUNT(*) FILTER (WHERE status='delivered' AND channel='web')    AS delivered_web,
  COUNT(*) FILTER (WHERE status='delivered' AND channel='mobile') AS delivered_mobile,
  -- conversion + refund rates
  ROUND(100.0 * COUNT(*) FILTER (WHERE status='delivered')
              / NULLIF(COUNT(*), 0), 2)                 AS delivery_rate_pct,
  ROUND(100.0 * COUNT(*) FILTER (WHERE status='refunded')
              / NULLIF(COUNT(*) FILTER (WHERE status IN ('shipped','delivered')), 0), 2)
                                                        AS refund_of_fulfilled_pct
FROM day_orders
GROUP BY day
ORDER BY day;
```

```sql
-- EXPLAIN (ANALYZE, BUFFERS) shape for the 30-day window:
--
-- GroupAggregate  (cost=0.56..48210.00 rows=30 width=...)
--                 (actual time=12.1..268.4 rows=30 loops=1)
--   Group Key: (date_trunc('day', o.created_at))
--   ->  Index Scan using orders_created_at_idx on orders o
--         Index Cond: (created_at >= ... AND created_at < ...)
--         (actual rows=~650000 loops=1)
--   Buffers: shared hit=9800 read=1400
-- Execution Time: ~290 ms
--
-- Reading it: the created_at index restricts the scan to ~650K rows (30 days),
-- and ALL the FILTER metrics are computed in ONE GroupAggregate pass over those
-- rows. Meets the <500ms target. Doing 8 separate WHERE-count queries over the
-- same slice would multiply the ~650K-row scan by 8.
```

---

## 9. Wrong → Right Patterns

### Wrong 1: `COUNT(CASE WHEN cond THEN 1 ELSE 0 END)` counts everything

```sql
-- WRONG: intended to count shipped orders
SELECT COUNT(CASE WHEN status='shipped' THEN 1 ELSE 0 END) AS shipped
FROM orders;
-- RESULT: equals COUNT(*) — every row! Because 0 is NON-NULL, and COUNT counts
-- every non-NULL value. The ELSE 0 defeats the whole purpose.
```

Why it's wrong at the execution level: `COUNT(expr)` increments its counter for every row where `expr IS NOT NULL`. The `CASE` returns `1` for shipped and `0` for everything else — both non-NULL — so the counter increments on every row.

```sql
-- RIGHT (CASE form): drop the ELSE so non-matching rows become NULL
SELECT COUNT(CASE WHEN status='shipped' THEN 1 END) AS shipped FROM orders;

-- RIGHT (FILTER form): no ELSE branch to get wrong
SELECT COUNT(*) FILTER (WHERE status='shipped') AS shipped FROM orders;
```

### Wrong 2: `AVG(CASE ... ELSE 0)` corrupts the average

```sql
-- WRONG: intended average order value among shipped orders
SELECT AVG(CASE WHEN status='shipped' THEN total_amount ELSE 0 END) AS avg_shipped
FROM orders;
-- BUG: every non-shipped order contributes a value of 0 to the average.
-- If 20% of orders are shipped, the "average" is diluted ~5× toward zero.
```

Why: `AVG` = sum / count-of-non-NULL. `ELSE 0` makes every row non-NULL with value 0, so the denominator is *all* rows and the numerator gains a pile of zeros.

```sql
-- RIGHT (CASE form): ELSE must be omitted so non-shipped → NULL → excluded
SELECT AVG(CASE WHEN status='shipped' THEN total_amount END) AS avg_shipped FROM orders;

-- RIGHT (FILTER form): unambiguous, no ELSE to reason about
SELECT AVG(total_amount) FILTER (WHERE status='shipped') AS avg_shipped FROM orders;
```

### Wrong 3: Using `<>` in a FILTER and silently dropping NULLs

```sql
-- WRONG: intended "count of orders that are NOT delivered", including NULL-status ones
SELECT COUNT(*) FILTER (WHERE status <> 'delivered') AS not_delivered
FROM orders;
-- BUG: rows with status = NULL evaluate NULL <> 'delivered' → UNKNOWN → skipped.
-- Those undelivered-and-unknown rows vanish from the count.
```

Why: three-valued logic (Topic 07). `<>` against a NULL yields UNKNOWN, and FILTER admits only TRUE.

```sql
-- RIGHT: IS DISTINCT FROM treats NULL as a real, distinct value
SELECT COUNT(*) FILTER (WHERE status IS DISTINCT FROM 'delivered') AS not_delivered
FROM orders;
```

### Wrong 4: Forgetting `SUM ... FILTER` returns NULL, then dividing/adding

```sql
-- WRONG: a customer with zero refunds gets NULL, breaking downstream math
SELECT
  customer_id,
  SUM(total_amount) FILTER (WHERE status='refunded') AS refund_total,
  SUM(total_amount) - SUM(total_amount) FILTER (WHERE status='refunded') AS net
FROM orders
GROUP BY customer_id;
-- BUG: for customers with no refunds, refund_total is NULL, so
-- (SUM - NULL) = NULL → net becomes NULL for exactly the good customers.
```

```sql
-- RIGHT: COALESCE the filtered sum to 0
SELECT
  customer_id,
  COALESCE(SUM(total_amount) FILTER (WHERE status='refunded'), 0) AS refund_total,
  SUM(total_amount)
    - COALESCE(SUM(total_amount) FILTER (WHERE status='refunded'), 0) AS net
FROM orders
GROUP BY customer_id;
```

### Wrong 5: Putting the condition in HAVING instead of FILTER

```sql
-- WRONG: trying to get per-status counts by filtering in HAVING
SELECT customer_id, COUNT(*) AS shipped
FROM orders
GROUP BY customer_id
HAVING status = 'shipped';           -- ERROR: "status" must appear in GROUP BY or aggregate
-- Even if you group by status too, HAVING filters GROUPS, giving a different shape
-- (one row per customer×status), not one shipped-count column per customer.
```

Why: `HAVING` filters whole groups after aggregation; `status` isn't a grouping key, so it's not even referenceable. The intent — "count only shipped rows into this one column" — is per-aggregate, which is FILTER's job.

```sql
-- RIGHT: FILTER puts the per-row condition on the aggregate
SELECT customer_id, COUNT(*) FILTER (WHERE status='shipped') AS shipped
FROM orders
GROUP BY customer_id;
```

### Wrong 6: FILTER on a ranking window function

```sql
-- WRONG: FILTER is not allowed on ROW_NUMBER / RANK
SELECT
  id,
  ROW_NUMBER() FILTER (WHERE status='shipped') OVER (ORDER BY created_at) AS rn
FROM orders;
-- ERROR: FILTER is not implemented for non-aggregate window functions
```

```sql
-- RIGHT: filter first (CTE/subquery), then rank the survivors
WITH shipped AS (
  SELECT * FROM orders WHERE status='shipped'
)
SELECT id, ROW_NUMBER() OVER (ORDER BY created_at) AS rn
FROM shipped;
```

---

## 10. Performance Profile

### The Core Performance Fact

FILTER's cost is **one boolean expression evaluation per (row × filtered-aggregate)**, added on top of an aggregation that was going to scan the rows anyway. It does **not**:
- add a scan,
- add a sort,
- add a plan node,
- change the grouping strategy (Hash vs Group aggregate),
- change parallelizability.

So the relevant comparison is never "FILTER vs no aggregation." It is:

| Alternative for N sub-populations | Scans | Relative cost |
|-----------------------------------|-------|---------------|
| N filtered aggregates in one SELECT (FILTER) | 1 | **baseline (best)** |
| N `CASE`-inside-aggregate in one SELECT | 1 | ~same as FILTER (identical work) |
| N separate `SELECT COUNT(*) WHERE ...` | N | ~N× the scan cost |
| N queries UNION ALL'd | N | ~N× |

### Scaling with row count (single-pass filtered aggregate over `orders`)

| Rows | Seq Scan aggregate | With covering/partial index on the population | Notes |
|------|--------------------|-----------------------------------------------|-------|
| 1M | ~150–200 ms | ~30–50 ms (index-only for a single pushed filter) | filters are cheap vs scan |
| 10M | ~1.5–2.5 s | ~0.3–0.6 s | parallel workers help ~linearly |
| 100M | ~15–30 s (parallelized: ~4–8 s) | ~2–5 s if the WHERE window is selective | must restrict via WHERE first |

The dominant cost at scale is **reading the rows**, not evaluating the filters. Therefore the optimization playbook for filtered aggregation is the *same* as for any aggregation:

1. **Restrict the input with WHERE first.** If every metric is confined to, say, the last 30 days, a `WHERE created_at >= ...` with an index turns a 100M-row scan into a 650K-row index range scan. The FILTERs then run over far fewer rows. This is the single biggest lever.
2. **Partial indexes** help only when a filter's predicate coincides with a WHERE that the planner can use for access (see Section 7). A partial index cannot make a FILTER skip rows that unfiltered aggregates in the same query still need.
3. **Parallelism**: filtered aggregates are parallel-safe. Ensure `max_parallel_workers_per_gather > 0` and the table is large enough for the planner to parallelize; each worker evaluates filters on its slice.
4. **`work_mem` for grouped filtered aggregation**: memory is driven by the number of groups × per-group state size, not by the filters. Adding more FILTER aggregates adds a few bytes of state per group (an extra counter/accumulator). Watch `Batches > 1` in EXPLAIN for HashAggregate spill and raise `work_mem` if needed.
5. **`COUNT(DISTINCT ...) FILTER`** is the expensive case: DISTINCT tracking needs a sort or hash set per group. The FILTER shrinks the input to the distinct-builder but the distinct itself dominates. Consider `approx_count_distinct` (via extensions) or pre-aggregation if this is hot.

### CPU and memory summary

- **CPU**: +1 predicate evaluation per row per filtered aggregate. Negligible unless the predicate itself is expensive (e.g., a regex or a function call) — in that case the predicate cost, not FILTER, is the issue.
- **Memory**: per-group state grows by a small constant per filtered aggregate. Immaterial except with millions of groups and dozens of aggregates.
- **I/O**: unchanged from the equivalent unfiltered aggregation — same rows read.

### When FILTER can be *slower* than expected

- **A FILTER coexisting with an unfiltered aggregate forces a full scan.** If you write `COUNT(*)` and `COUNT(*) FILTER(WHERE status='rare')`, the `COUNT(*)` needs every row, so no partial index can prune. If you only cared about the rare population, drop the total and push to WHERE.
- **Expensive predicates × many rows.** `COUNT(*) FILTER (WHERE my_expensive_udf(x))` evaluates the UDF per row. FILTER didn't make it slow; the UDF did.

---

## 11. Node.js Integration

### 11.1 Single-row dashboard summary

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function getOrderSummary() {
  const { rows } = await pool.query(
    `SELECT
       COUNT(*)                                     AS total,
       COUNT(*) FILTER (WHERE status = 'pending')   AS pending,
       COUNT(*) FILTER (WHERE status = 'shipped')   AS shipped,
       COUNT(*) FILTER (WHERE status = 'delivered') AS delivered,
       COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled,
       COALESCE(SUM(total_amount)
         FILTER (WHERE status IN ('shipped','delivered')), 0) AS realized_revenue
     FROM orders`
  );
  // NOTE: COUNT(*) comes back as a STRING (int8 → JS string) from node-postgres.
  // Parse the numeric fields you need as numbers.
  const r = rows[0];
  return {
    total: Number(r.total),
    pending: Number(r.pending),
    shipped: Number(r.shipped),
    delivered: Number(r.delivered),
    cancelled: Number(r.cancelled),
    realizedRevenue: Number(r.realized_revenue),
  };
}
```

### 11.2 Parameterized filter thresholds with `$1`

```javascript
// The FILTER predicate can reference bound parameters, just like WHERE.
async function getRevenueBands(highValueThreshold, sinceDays) {
  const { rows } = await pool.query(
    `SELECT
       COUNT(*)                                            AS orders,
       COUNT(*) FILTER (WHERE total_amount >= $1)          AS high_value_orders,
       COALESCE(SUM(total_amount) FILTER (WHERE total_amount >= $1), 0) AS high_value_revenue,
       COALESCE(AVG(total_amount) FILTER (WHERE total_amount <  $1), 0) AS avg_low_value
     FROM orders
     WHERE created_at >= NOW() - ($2 || ' days')::interval`,
    [highValueThreshold, sinceDays]   // $1 reused in three filters, $2 in WHERE
  );
  return rows[0];
}
```

### 11.3 Per-group pivot for a table/chart response

```javascript
async function getDailyStatusPivot(days = 30) {
  const { rows } = await pool.query(
    `SELECT
       DATE_TRUNC('day', created_at)::date          AS day,
       COUNT(*)                                     AS total,
       COUNT(*) FILTER (WHERE status='delivered')   AS delivered,
       COUNT(*) FILTER (WHERE status='cancelled')   AS cancelled,
       COUNT(*) FILTER (WHERE status='refunded')    AS refunded
     FROM orders
     WHERE created_at >= CURRENT_DATE - ($1 || ' days')::interval
     GROUP BY 1
     ORDER BY 1`,
    [days]
  );
  // Each row is already a "pivoted" record — ideal for a stacked bar chart.
  return rows.map(r => ({
    day: r.day,
    total: Number(r.total),
    delivered: Number(r.delivered),
    cancelled: Number(r.cancelled),
    refunded: Number(r.refunded),
  }));
}
```

### 11.4 A note on int8 parsing

`COUNT(...)` returns PostgreSQL `bigint` (int8), which node-postgres returns as a **string** to avoid precision loss beyond `Number.MAX_SAFE_INTEGER`. Always `Number(...)` (or `BigInt(...)` for truly huge counts) the count columns. `SUM` of an `integer` column returns `bigint` too (string); `SUM` of `numeric` returns a string as well. Configure a type parser (`pg.types.setTypeParser`) globally if you prefer numbers, but be mindful of precision.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do FILTER?** — Not natively. Prisma's `groupBy`/`aggregate` API has no per-aggregate conditional clause. You can approximate a *single* filtered count with a filtered `count` relation or a `where`, but not multiple sub-populations in one pass.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// Prisma can count with a where, but that's ONE population per query:
const shipped = await prisma.order.count({ where: { status: 'shipped' } });
const total   = await prisma.order.count();
// ^ two round-trips, two scans — exactly what FILTER avoids.

// Real FILTER pivots require raw SQL:
const rows = await prisma.$queryRaw<Array<{
  day: Date; total: bigint; delivered: bigint; cancelled: bigint;
}>>`
  SELECT DATE_TRUNC('day', created_at)::date AS day,
         COUNT(*)                                   AS total,
         COUNT(*) FILTER (WHERE status='delivered') AS delivered,
         COUNT(*) FILTER (WHERE status='cancelled') AS cancelled
  FROM orders
  WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
  GROUP BY 1
  ORDER BY 1
`;
// Note: bigint results — convert with Number(...) at the boundary.
```

**Where it breaks**: no FILTER expression in the query API; multiple filtered metrics = multiple queries or raw SQL. **Verdict**: use `$queryRaw` for any real conditional-aggregation/pivot.

---

### Drizzle ORM

**Can Drizzle do FILTER?** — Yes, cleanly, via the `sql` template or the `filter()` helper on aggregate builders in recent versions.

```typescript
import { db } from './db';
import { orders } from './schema';
import { sql, gte } from 'drizzle-orm';

// Using the sql template (works on every version):
const rows = await db
  .select({
    total: sql<number>`count(*)`.mapWith(Number),
    delivered: sql<number>`count(*) filter (where ${orders.status} = 'delivered')`.mapWith(Number),
    cancelled: sql<number>`count(*) filter (where ${orders.status} = 'cancelled')`.mapWith(Number),
    revenue: sql<number>`coalesce(sum(${orders.totalAmount})
                          filter (where ${orders.status} in ('shipped','delivered')), 0)`.mapWith(Number),
  })
  .from(orders)
  .where(gte(orders.createdAt, sql`current_date - interval '30 days'`));
```

**Where it breaks**: nothing structural — the `sql` tag passes FILTER straight through, and `.mapWith(Number)` handles the int8→string issue. Type inference on the raw `sql` fragment is manual (`sql<number>`). **Verdict**: Drizzle is the best fit — near-SQL, typed, no escape hatch needed.

---

### Sequelize

**Can Sequelize do FILTER?** — Only via `sequelize.literal()` inside an attribute; the query builder has no first-class FILTER.

```javascript
const { fn, col, literal } = require('sequelize');

const rows = await Order.findAll({
  attributes: [
    [fn('COUNT', col('id')), 'total'],
    [literal(`COUNT(*) FILTER (WHERE status = 'delivered')`), 'delivered'],
    [literal(`COUNT(*) FILTER (WHERE status = 'cancelled')`), 'cancelled'],
    [literal(`COALESCE(SUM(total_amount) FILTER (WHERE status IN ('shipped','delivered')),0)`), 'revenue'],
  ],
  raw: true,
});
```

**Where it breaks**: `literal()` is an unparameterized raw string — you must hand-escape any dynamic values (SQL-injection risk if you interpolate user input). No type safety. For grouped pivots you also add `group: [...]`. **Verdict**: workable but ugly; for anything nontrivial use `sequelize.query()` with bind parameters.

---

### TypeORM

**Can TypeORM do FILTER?** — Via `addSelect()` with a raw expression in the QueryBuilder; results come through `getRawMany()`.

```typescript
const rows = await dataSource
  .getRepository(Order)
  .createQueryBuilder('o')
  .select(`DATE_TRUNC('day', o.created_at)`, 'day')
  .addSelect(`COUNT(*)`, 'total')
  .addSelect(`COUNT(*) FILTER (WHERE o.status = 'delivered')`, 'delivered')
  .addSelect(`COUNT(*) FILTER (WHERE o.status = 'cancelled')`, 'cancelled')
  .where('o.created_at >= :since', { since: sinceDate })
  .groupBy('day')
  .orderBy('day')
  .getRawMany();
```

**Where it breaks**: the FILTER expression is a raw string (no column-type checking); you must use `getRawMany()` (entity hydration can't represent computed pivot columns). Bind params work in `where`, not inside the raw `addSelect` string — keep dynamic filter values out of it or bind carefully. **Verdict**: fine for reporting endpoints; treat it as raw SQL with a builder wrapper.

---

### Knex.js

**Can Knex do FILTER?** — Yes, via `knex.raw()` in a select; Knex is SQL-transparent.

```javascript
const rows = await knex('orders')
  .select(knex.raw(`DATE_TRUNC('day', created_at)::date AS day`))
  .select(knex.raw(`COUNT(*) AS total`))
  .select(knex.raw(`COUNT(*) FILTER (WHERE status = 'delivered') AS delivered`))
  .select(knex.raw(`COUNT(*) FILTER (WHERE status = 'cancelled') AS cancelled`))
  .select(knex.raw(
    `COUNT(*) FILTER (WHERE total_amount >= ?) AS high_value`, [threshold]  // bound param!
  ))
  .where('created_at', '>=', sinceDate)
  .groupByRaw('1')
  .orderByRaw('1');
```

**Where it breaks**: nothing — `knex.raw` supports `?` bind parameters *inside* the FILTER predicate, so dynamic thresholds are safe. Return values are strings for int8; convert at the boundary. **Verdict**: excellent; the raw-with-bindings form is both safe and readable.

---

### ORM Summary Table

| ORM | FILTER support | How | Bind params in filter | Verdict |
|-----|----------------|-----|-----------------------|---------|
| Prisma | None | `$queryRaw` only | via tagged template | Raw SQL for all pivots |
| Drizzle | Yes | `sql` template / `filter()` | yes (`${}`) | Best fit, typed |
| Sequelize | Raw only | `literal()` | no (hand-escape) | Ugly; prefer `.query()` |
| TypeORM | Raw only | `addSelect(raw)` + `getRawMany` | in `where` only | OK for reports |
| Knex | Yes | `knex.raw` with `?` | yes (`?`) | Excellent, safe |

Across the board: **FILTER is a raw-SQL feature for ORMs.** Only Drizzle and Knex let you express it comfortably with parameter binding; the others force `literal`/`$queryRaw`. This is a strong argument for keeping analytics queries in raw SQL regardless of ORM.

---

## 13. Practice Exercises

### Exercise 1 — Easy

Given `orders(id, customer_id, status, total_amount, created_at)`:

Write a single query returning one row with:
- `total_orders` — all orders
- `paid_orders` — orders with `status = 'paid'`
- `paid_revenue` — sum of `total_amount` for paid orders, shown as `0` (not NULL) if there are none
- `avg_paid_order` — average `total_amount` among paid orders

Do it in a single table scan (no subqueries, no UNION).

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines Topics 11 + 22 + 23)

Given `customers(id, name)`, `orders(id, customer_id, status, total_amount, created_at)`:

For each customer, over the last 12 months, return:
- `customer_id`, `name`
- `orders` — total order count
- `delivered` — count of delivered orders
- `refunded` — count of refunded orders
- `refund_rate_pct` — refunded / orders as a percentage, 1 dp, safe against divide-by-zero
- `net_revenue` — sum of delivered `total_amount` minus sum of refunded `total_amount`, never NULL

Only include customers with at least 5 orders in the window. Order by `net_revenue` descending.

(Recall Topic 11: the JOIN; Topic 22: HAVING for the ≥5 filter.)

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation, naive answer is wrong)

Given `order_items(id, order_id, product_id, quantity, unit_price)` and `orders(id, status, created_at)`:

Produce, per `product_id`, for orders created this year:
- `units_completed` — total quantity in **completed** orders
- `units_refunded` — total quantity in **refunded** orders
- `distinct_completed_customers` — number of distinct customers (from `orders.customer_id`) who bought this product in a completed order
- `net_units` — completed minus refunded

The naive answer uses `SUM(quantity)` with a status column that doesn't exist on `order_items` (it's on `orders`), and mishandles the fan-out for the distinct-customer count. Write the correct query. State which join you need and why `COUNT(DISTINCT ...) FILTER (...)` is required.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

You're handed this query in a code review:

```sql
SELECT
  seller_id,
  COUNT(CASE WHEN status='shipped' THEN 1 ELSE 0 END)    AS shipped,
  AVG(CASE WHEN status='shipped' THEN total_amount ELSE 0 END) AS avg_shipped,
  COUNT(*) FILTER (WHERE status <> 'cancelled')          AS non_cancelled
FROM orders
GROUP BY seller_id;
```

1. Identify the three bugs (one per output column) and explain each at the execution level.
2. Rewrite all three metrics using FILTER, correctly.
3. The reviewer asks: "Would `WHERE status='shipped'` give the same `shipped` count more cheaply?" Under what circumstance is that true, and when is it wrong?

```sql
-- Write your corrected query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What does the FILTER clause do, and how is it different from WHERE?**

A: `WHERE` filters rows for the *entire* query before grouping — every aggregate then works on the same surviving rows. `FILTER` attaches a condition to *one specific aggregate*, so that aggregate accumulates only the rows matching its own condition, while other aggregates in the same SELECT are unaffected. `WHERE` gives you one filtered population; `FILTER` lets each aggregate have its own population, computed in a single pass. Example: `COUNT(*)` and `COUNT(*) FILTER (WHERE status='shipped')` side by side give total and shipped counts from one scan — `WHERE` alone can't produce both.

**Follow-up the interviewer asks:** *"If you only needed the shipped count, would you use FILTER or WHERE?"* — WHERE, because it removes non-shipped rows before aggregation and can use an index/partial index to avoid scanning them at all. FILTER only earns its place when you need multiple sub-populations in one query.

---

### Junior Level

**Q: Why is `COUNT(CASE WHEN cond THEN 1 ELSE 0 END)` a bug, and what's the fix?**

A: `COUNT(expr)` counts every row where `expr` is non-NULL. `ELSE 0` returns `0` — which is non-NULL — for non-matching rows, so the counter increments on every row and you get `COUNT(*)`, not the conditional count. The fix is to omit the ELSE so non-matching rows become NULL (`COUNT(CASE WHEN cond THEN 1 END)`), or better, use `COUNT(*) FILTER (WHERE cond)`, which has no ELSE branch to get wrong.

**Follow-up:** *"Is `SUM(CASE WHEN cond THEN x ELSE 0 END)` also buggy?"* — No, for SUM the `ELSE 0` is harmless because adding zero doesn't change the sum. But for AVG it *is* buggy (the zeros become data points that drag the average down and inflate the denominator). That inconsistency across aggregate types is exactly why FILTER is safer — it removes the ELSE decision entirely.

---

### Principal Level

**Q: A dashboard endpoint issues eight separate `SELECT COUNT(*) FROM orders WHERE status = 'X'` queries and is slow on a 50M-row table. Walk me through the fix and the internals.**

A: Each of those queries is a full aggregation over (potentially) the whole table — eight scans, eight round-trips. Collapse them into one query with eight `COUNT(*) FILTER (WHERE status='X')` aggregates. At the engine level, all eight filtered aggregates are computed inside a **single Aggregate node during one scan**: the executor evaluates each filter as a gate on that aggregate's transition function per row, so the row is read once and fans out into whichever counters it qualifies for. That's ~8× less I/O immediately. Next, if all eight metrics are confined to a time window, add `WHERE created_at >= ...` with an index so the single scan becomes an index range scan over just the relevant rows — the biggest lever, since the cost is dominated by reading rows, not evaluating filters. Finally, verify in EXPLAIN that it's one Aggregate over one scan (no repeated scans), and that parallel workers kick in for the 50M-row case (`Partial Aggregate` under `Gather`) — FILTER is parallel-safe.

**Follow-up:** *"Could a partial index on `status='shipped'` make the shipped count skip rows even in the combined query?"* — Not if the combined query also has an unfiltered `COUNT(*)` or filters for other statuses, because those aggregates need every row, forcing a full scan regardless. A partial index only helps when the query's access path can be restricted to that predicate (e.g., a lone filtered count with a matching WHERE). Combined multi-status pivots inherently read all rows once.

---

### Principal Level

**Q: Explain the empty-set behavior of filtered aggregates and where it causes production bugs.**

A: When no row passes a filter, `COUNT(*) FILTER (...)` returns `0`, but `SUM`, `AVG`, `MIN`, `MAX`, and `array_agg` return `NULL` — the standard empty-input rule. FILTER makes empty inputs common (a per-group filter that matches nothing for some groups). The bugs: (1) arithmetic on a filtered `SUM` — e.g. `total - SUM(x) FILTER(...)` becomes NULL for exactly the groups with no matching x, silently nulling out good rows; fix with `COALESCE(..., 0)`. (2) Divide-by-filtered-denominator producing NULL or division errors — guard with `NULLIF(denominator, 0)` and often `COALESCE` the whole ratio. (3) `array_agg ... FILTER` returning NULL instead of `{}` when building JSON payloads — wrap with `COALESCE(array_agg(...) FILTER(...), '{}')`. The principle: distinguish "counted zero rows" (0) from "aggregated over zero rows" (NULL) and coalesce wherever downstream math or serialization assumes a value.

**Follow-up:** *"Does `COUNT(col) FILTER (...)` ever return NULL?"* — No. `COUNT` always returns a non-NULL bigint (0 when nothing qualifies). Only value-producing aggregates return NULL on empty input.

---

### Principal Level

**Q: When would you reach for the `tablefunc` `crosstab()` over a FILTER-based pivot, and vice versa?**

A: Use FILTER for essentially all pivots where the set of output columns is known when you write the query — it's core SQL (no extension), reads clearly, mixes aggregate types trivially (count + sum + avg per category in one pass), and handles NULLs with standard semantics. Reach for `crosstab()` only when the pivot columns are genuinely dynamic *and* you specifically want the tablefunc machinery — but even then, generating a FILTER-based SELECT via dynamic SQL is usually cleaner and avoids `crosstab`'s positional column-definition-list fragility. In modern PostgreSQL, FILTER has largely obsoleted `crosstab` for static pivots; `crosstab` survives mainly in legacy code and truly dynamic edge cases.

**Follow-up:** *"How do you pivot when the category values aren't known ahead of time?"* — Either build the FILTER-per-category SELECT string dynamically in application code / a PL/pgSQL function from a `SELECT DISTINCT category`, or return the data un-pivoted (long format, `GROUP BY category`) and pivot in the application/BI layer. Truly dynamic column sets can't be typed statically in SQL either way.

---

## 15. Mental Model Checkpoint

1. **A table has 1,000 rows; 300 have `status='shipped'`. In one SELECT you write `COUNT(*)` and `COUNT(*) FILTER (WHERE status='shipped')`. How many times is the table scanned, and what are the two results?**

2. **`AVG(total_amount) FILTER (WHERE status='shipped')` vs `AVG(CASE WHEN status='shipped' THEN total_amount ELSE 0 END)` — do they return the same value? If not, which is correct and why?**

3. **For a group where no row matches the filter, what does `COUNT(*) FILTER (...)` return? What does `SUM(x) FILTER (...)` return? Why the difference?**

4. **You want "count of orders that are not delivered, including rows where status is NULL." Why does `FILTER (WHERE status <> 'delivered')` get this wrong, and what's the correct predicate?**

5. **Can a FILTER predicate reference another aggregate, e.g. `COUNT(*) FILTER (WHERE SUM(x) > 100)`? If not, which clause expresses a per-group condition?**

6. **You add `COUNT(*) FILTER (WHERE status='rare')` next to a plain `COUNT(*)` and hope a partial index on `status='rare'` will speed it up. Why doesn't the partial index prune the scan here?**

7. **Is `SUM(amount) FILTER (WHERE status='completed') OVER (ORDER BY created_at)` legal? Is `ROW_NUMBER() FILTER (WHERE status='completed') OVER (...)` legal? Explain the difference.**

---

## 16. Quick Reference Card

```sql
-- SYNTAX ---------------------------------------------------------------
aggregate(args) FILTER (WHERE predicate)              -- WHERE keyword is REQUIRED
aggregate(args) FILTER (WHERE predicate) OVER (...)   -- aggregate window form only

-- ONLY TRUE PASSES (three-valued logic, like WHERE) --------------------
FILTER (WHERE status = 'x')            -- NULL-status rows: UNKNOWN → skipped
FILTER (WHERE status <> 'x')           -- NULL-status rows: UNKNOWN → skipped (misses NULLs!)
FILTER (WHERE status IS DISTINCT FROM 'x')  -- NULL rows: TRUE → included

-- EMPTY-SET RESULTS ---------------------------------------------------
COUNT(*)  FILTER (...)  -> 0     when nothing matches
SUM/AVG/MIN/MAX/array_agg FILTER (...) -> NULL   when nothing matches
COALESCE(SUM(x) FILTER (...), 0)                 -- guard for math/display
COALESCE(array_agg(x) FILTER (...), '{}')        -- guard for empty array

-- CONDITIONAL PIVOT (no tablefunc needed) -----------------------------
SELECT
  COUNT(*) FILTER (WHERE status='a') AS a,
  COUNT(*) FILTER (WHERE status='b') AS b,
  SUM(amt) FILTER (WHERE status='a') AS a_rev
FROM t;

-- FILTER vs CASE equivalences -----------------------------------------
COUNT(*) FILTER (WHERE c)      == COUNT(CASE WHEN c THEN 1 END)  -- NOT "ELSE 0"
SUM(x)   FILTER (WHERE c)      == SUM(CASE WHEN c THEN x END)    -- ELSE 0 ok for SUM only
AVG(x)   FILTER (WHERE c)      == AVG(CASE WHEN c THEN x END)    -- ELSE 0 CORRUPTS avg

-- RATE PATTERN (safe division) ----------------------------------------
ROUND(100.0 * COUNT(*) FILTER (WHERE c) / NULLIF(COUNT(*), 0), 2) AS pct

-- COMBINE WITH GROUP BY / HAVING --------------------------------------
GROUP BY g
HAVING COUNT(*) >= 5           -- HAVING = group filter (AFTER agg); FILTER = per-agg

-- PERF RULES OF THUMB -------------------------------------------------
-- 1 population, 1 number  -> use WHERE (indexable, prunes rows)
-- N populations, 1 pass   -> use FILTER (beats N separate WHERE queries by ~N×)
-- FILTER adds no scan/sort/node; cost = 1 bool eval per row per filtered agg
-- Restrict input with WHERE first; partial index only helps a lone pushed filter
-- COUNT(DISTINCT ...) FILTER (...) is the expensive case (distinct dominates)

-- NOT ALLOWED ---------------------------------------------------------
ROW_NUMBER()/RANK()/LAG()/LEAD() FILTER (...)   -- ✗ non-aggregate window funcs
FILTER (WHERE <aggregate>)                       -- ✗ use HAVING for group predicates
FILTER (status='x')                              -- ✗ WHERE keyword missing

-- INTERVIEW ONE-LINERS ------------------------------------------------
-- "FILTER is a per-aggregate WHERE; all filtered aggregates share one scan."
-- "FILTER has no ELSE branch to get wrong — that's the correctness argument."
-- "COUNT FILTER -> 0 on empty; SUM/AVG FILTER -> NULL on empty. COALESCE it."
-- "One condition/one number = WHERE. Many conditions/one pass = FILTER."
```

---

## Connected Topics

- **Topic 07 — NULL in Depth**: Three-valued logic governs the FILTER predicate — only TRUE admits a row; `<>` silently drops NULLs where `IS DISTINCT FROM` keeps them.
- **Topic 11 — INNER JOIN in Depth**: Fan-out from one-to-many joins is why `COUNT(DISTINCT ...) FILTER (...)` is often needed for correct conditional counts after a join.
- **Topic 20 — GROUP BY Fundamentals**: FILTER lives inside the aggregation phase; grouping defines the populations each filtered aggregate accumulates over.
- **Topic 21 — Aggregate Functions**: The transition-state machinery (`sfunc`/`ffunc`) that FILTER gates; the empty-set NULL-vs-0 behavior originates here.
- **Topic 22 — HAVING**: The neighbor that filters *groups after* aggregation, versus FILTER which filters *rows per aggregate during* aggregation — the two are complementary, not interchangeable.
- **Topic 24 — GROUPING SETS, ROLLUP, CUBE**: The next step in multi-dimensional aggregation; FILTER pivots columns while GROUPING SETS pivots/rolls up rows — they combine for rich reports.
- **Topic 25/26 — Window Functions**: FILTER attaches to the aggregate form of window functions (`SUM(...) FILTER(...) OVER (...)`) but not to ranking functions like `ROW_NUMBER()`.
