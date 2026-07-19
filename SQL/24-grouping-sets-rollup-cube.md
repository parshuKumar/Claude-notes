# Topic 24 — GROUPING SETS, ROLLUP, CUBE
### SQL Mastery Curriculum — Phase 4: Aggregation and Grouping

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a chain of coffee shops and at the end of the month your regional manager drops a spreadsheet on your desk and says: "I want the numbers."

You ask: "Which numbers?"

She says: "All of them. Sales per city. Sales per product. Sales per city *and* product. And the grand total at the bottom. On one page."

Now, the naive way to do this is to run four separate reports:
- One that groups by city
- One that groups by product
- One that groups by (city, product) together
- One with no grouping at all — the single grand total

Then you'd staple the four printouts together. Four passes over the same pile of receipts. Four trips through the shoebox.

`GROUPING SETS`, `ROLLUP`, and `CUBE` are the database saying: "Give me the shoebox once. I'll count everything you asked for in a single pass and hand you one combined sheet." You describe *which* combinations of grouping you want, and the engine scans the data one time, computing every requested level of aggregation together.

- **GROUPING SETS** — "Here is the exact list of groupings I want." You spell out each combination by hand.
- **ROLLUP** — "Give me the hierarchy: country, then country+state, then country+state+city, then the grand total." Subtotals that roll *up* a hierarchy, like collapsing an outline.
- **CUBE** — "Give me *every possible* combination of these columns." All corners of the box.

The one subtlety — and the whole reason section 6 exists — is this: when the engine prints the "sales per product across all cities" row, the *city* column has to hold *something*. It puts a `NULL` there, meaning "this column was not part of this particular grouping." But what if you genuinely have a city named `NULL` (a real missing value in the data)? Now you have two kinds of `NULL` in the same column that mean completely different things. The `GROUPING()` function is the flashlight that tells them apart.

That's the whole topic. One pass, many groupings, and a function to read the subtotal markers.

---

## 2. Connection to SQL Internals

At the engine level, a plain `GROUP BY city` is executed by one of two physical operators:

1. **HashAggregate** — build a hash table keyed by `city`, and for each incoming row, find-or-create the bucket and fold the row into the running aggregate (SUM, COUNT, etc.). O(N) time, O(distinct groups) memory.
2. **GroupAggregate** — require the input to be sorted on the grouping key (often via an index or an explicit Sort node), then walk the sorted stream and emit an aggregate each time the key changes. O(N) after the sort, O(1) memory beyond the current group.

`GROUPING SETS`, `ROLLUP`, and `CUBE` are *not* new physical operators. They are a **planner-level expansion** into multiple grouping keys that share a single scan of the input. PostgreSQL implements them with two strategies:

- **MixedAggregate** (the `GroupAggregate` family extended) — when at least one grouping set benefits from sorted input, Postgres sorts once and computes multiple grouping sets from that single sorted stream, resetting aggregates at the appropriate key boundaries. You will see a node literally named `MixedAggregate` or `GroupAggregate` with a `Group Key` line plus `Hash Key` lines.
- **HashAggregate with multiple hash tables** — Postgres can maintain several hash tables simultaneously in one pass, one per grouping set, each keyed on a different subset of columns. You will see a `HashAggregate` node with multiple `Hash Key:` lines. This is the feature that makes `GROUPING SETS` genuinely one-pass: N grouping sets, one scan, N hash tables living in `work_mem` at once.

The critical internal consequence: **memory pressure multiplies**. A single `GROUP BY` needs one hash table sized to its distinct-group count. A `CUBE` of 4 columns needs up to `2^4 = 16` hash tables coexisting in `work_mem`. When they don't all fit, Postgres spills partitions to disk (`Batches > 1`) exactly as it does for a large hash join — and now you are paying I/O for the convenience of one query.

The `GROUPING()` function itself is computed for free during aggregation: it is just a bitmask the executor already tracks internally to know which grouping set produced each output row. It costs nothing extra to select it.

This topic builds directly on Topic 20 (GROUP BY Fundamentals) and Topic 22 (aggregate functions). It is the multi-key generalization of everything a single `GROUP BY` does.

---

## 3. Logical Execution Order Context

```
FROM users / orders / ...
JOIN ...                     ← rows assembled
WHERE ...                    ← row filter (BEFORE grouping)
GROUP BY GROUPING SETS (...) ← grouping happens HERE — multiple groupings at once
HAVING ...                   ← filter on aggregated groups (AFTER grouping)
SELECT ..., GROUPING(col)    ← GROUPING() readable here
DISTINCT
ORDER BY ...                 ← runs last; sees the subtotal NULLs
LIMIT ...
```

The key facts about where this fits:

- **WHERE runs before grouping** — it filters raw rows. It cannot reference an aggregate and it cannot reference `GROUPING()`. If you want "only rows where amount > 0" before subtotals are computed, that goes in WHERE.
- **The grouping sets are expanded in the GROUP BY phase** — this is the moment the engine decides "I will compute these 5 groupings." All the subtotal rows and the grand-total row are *born* here.
- **HAVING runs after grouping** — it can filter out entire subtotal rows. `HAVING GROUPING(city) = 0` removes the "all cities" subtotal rows. This is a common and important pattern.
- **SELECT can reference `GROUPING(col)`** — because grouping is already done. You surface the bitmask here.
- **ORDER BY runs last and sees the NULL subtotal markers** — this matters enormously for report layout. Subtotal rows have `NULL` in the rolled-up columns, and `NULL` sorts last (ASC) or first (DESC) by default. To force subtotals to the bottom of their section, you order by `GROUPING(col)` first (covered in section 6).

Remember from Topic 20: the golden rule is that every non-aggregated column in SELECT must appear in GROUP BY. With grouping sets, the rule bends — a column may be in *some* grouping sets and not others, and in the sets where it is absent, the engine substitutes `NULL`. This is legal and is the entire point.

---

## 4. What Is GROUPING SETS / ROLLUP / CUBE?

`GROUPING SETS` is an extension of `GROUP BY` that computes **multiple independent groupings in a single query and single pass**, unioning their results into one output. `ROLLUP` and `CUBE` are shorthand macros that expand into specific, common patterns of grouping sets — `ROLLUP` for hierarchical subtotals, `CUBE` for all possible combinations. The `GROUPING()` function returns a bit that is 1 when a column was *not* part of the grouping set that produced the current row (i.e., the NULL in that column is a subtotal marker, not real data).

```sql
SELECT
  region,                              -- may be NULL in higher-level subtotals
  product,                             -- may be NULL in higher-level subtotals
  SUM(amount) AS revenue,
  GROUPING(region)  AS g_region,       -- 1 = "region rolled up" (subtotal), 0 = real region
  GROUPING(product) AS g_product       -- 1 = "product rolled up" (subtotal), 0 = real product
FROM orders
GROUP BY GROUPING SETS (
  (region, product),   -- │ leaf level: one row per (region, product) combination
  (region),            -- │ subtotal per region across all products
  (product),           -- │ subtotal per product across all regions
  ()                   -- │ grand total: one row, everything aggregated
);
--     │              └── each parenthesized tuple is ONE grouping set
--     └── GROUPING SETS takes a comma-separated list of column-tuples
```

### The three constructs annotated

```sql
GROUP BY GROUPING SETS ( (a, b), (a), () )
--                       │        │    └── empty set () = grand total (no grouping)
--                       │        └── single-column set = subtotal per a
--                       └── multi-column set = one row per distinct (a, b)
--       └── explicit: YOU list every grouping you want, exactly

GROUP BY ROLLUP (a, b, c)
--              │
--              └── EXPANDS TO: (a,b,c), (a,b), (a), ()
--                  a hierarchy collapsing left-to-right — N+1 grouping sets for N columns
--                  Use when columns form a drill-down hierarchy: year → quarter → month

GROUP BY CUBE (a, b, c)
--            │
--            └── EXPANDS TO all 2^N subsets:
--                (a,b,c), (a,b), (a,c), (b,c), (a), (b), (c), ()
--                every possible combination — 2^N grouping sets for N columns
--                Use when you want every cross-tabulation, no hierarchy assumed
```

### The GROUPING function annotated

```sql
GROUPING(col)          -- returns 0 or 1 for a SINGLE column
--       │
--       └── 0 → this column IS in the current row's grouping set (value is real)
--           1 → this column is NOT in the set (value is a subtotal-marker NULL)

GROUPING(a, b, c)      -- returns an INTEGER bitmask across MULTIPLE columns
--       │
--       └── treats the columns as bits, leftmost = most significant:
--           a in set, b in set, c in set     → 000 = 0
--           a in set, b in set, c rolled up  → 001 = 1
--           a rolled up, b,c in set          → 100 = 4
--           all rolled up (grand total)      → 111 = 7
```

`GROUPING SETS`, `ROLLUP`, and `CUBE` can also be **combined and nested** inside a single `GROUP BY` clause — covered in section 6.5.

---

## 5. Why GROUPING SETS Mastery Matters in Production

1. **One pass instead of N passes.** The classic "wrong" way to build a dashboard with subtotals is to run one query per grouping level and `UNION ALL` them, or worse, to run them as separate round-trips from the application. Each is a full scan of a large fact table. A report with leaf rows + 2 subtotal levels + grand total is 4 scans of a 50M-row `order_items` table. `GROUPING SETS` collapses that to one scan. On big tables this is the difference between a 200ms dashboard and an 800ms one.

2. **Reports genuinely need this shape.** Finance, analytics, and BI reports almost always want subtotals and grand totals interleaved with detail rows: revenue by (region, product), then per-region subtotals, then a company-wide total. Without `ROLLUP`, developers reinvent it with fragile `UNION ALL` stacks that drift out of sync when a filter changes in one branch but not another.

3. **The NULL-ambiguity bug is silent and corrupting.** If your data legitimately contains `NULL` in a grouped column (an order with no assigned region), a naive `ROLLUP` report will show *two* rows with `region = NULL`: one for real null-region orders, one for the "all regions" subtotal. A reader — or a downstream aggregation — cannot tell them apart. Without `GROUPING()` to disambiguate, totals get double-counted or mislabeled. This is a data-integrity bug that passes every test until someone has a NULL in production.

4. **Performance cliffs from memory.** A `CUBE` over many high-cardinality columns can explode into dozens of simultaneous hash tables and spill to disk. Knowing that `CUBE(a,b,c,d)` is 16 grouping sets — and that each is a hash table — lets you predict and avoid the cliff, or reach for `ROLLUP` (N+1 sets) when a full cube is not actually needed.

5. **Interview signal.** Being able to explain why a `ROLLUP` report shows two `NULL` regions, and fixing it with `GROUPING()`, is a reliable senior-vs-junior discriminator. It tests whether you understand three-valued logic (Topic 07), grouping semantics (Topic 20), and report requirements at once.

---

## 6. Deep Technical Content

### 6.1 GROUPING SETS — The Foundation

Everything else is sugar over `GROUPING SETS`. A grouping set is a tuple of columns to group by. `GROUPING SETS` takes a list of them and computes each grouping independently, then unions the results.

```sql
SELECT region, product, SUM(amount) AS revenue
FROM orders
GROUP BY GROUPING SETS ((region, product), (region), ());
```

This is exactly equivalent to:

```sql
SELECT region, product, SUM(amount) AS revenue
FROM orders GROUP BY region, product
UNION ALL
SELECT region, NULL, SUM(amount) FROM orders GROUP BY region
UNION ALL
SELECT NULL, NULL, SUM(amount) FROM orders;
```

...except the `GROUPING SETS` version scans `orders` **once**, while the `UNION ALL` version scans it **three times**. The result rows are identical (modulo the NULL-ambiguity issue below), but the execution profile is completely different.

The empty grouping set `()` is the grand total — it groups the entire table into one row. It is legal and common. `GROUPING SETS (())` alone is equivalent to a query with aggregates and no `GROUP BY` at all.

A single-column grouping set produces subtotals for that column with all *other* grouped columns set to `NULL`.

### 6.2 ROLLUP — Hierarchical Subtotals

`ROLLUP(a, b, c)` expands to a **prefix hierarchy**, N+1 grouping sets for N columns:

```
ROLLUP(a, b, c)  ≡  GROUPING SETS ( (a,b,c), (a,b), (a), () )
```

Note it drops columns **from the right**. It never produces `(a, c)` or `(b, c)`. This is exactly what you want when the columns form a drill-down hierarchy where each level is meaningful only within its parent:

```sql
-- Time hierarchy: year → quarter → month, with subtotals at each level
SELECT
  EXTRACT(YEAR    FROM created_at)::int AS yr,
  EXTRACT(QUARTER FROM created_at)::int AS qtr,
  EXTRACT(MONTH   FROM created_at)::int AS mon,
  SUM(total_amount) AS revenue
FROM orders
WHERE status = 'completed'
GROUP BY ROLLUP (
  EXTRACT(YEAR    FROM created_at),
  EXTRACT(QUARTER FROM created_at),
  EXTRACT(MONTH   FROM created_at)
)
ORDER BY yr, qtr, mon;
```

Output shape (subtotal rows carry NULLs in the rolled-up columns):

```
 yr  | qtr | mon | revenue
-----+-----+-----+---------
2025 |   1 |   1 |   12000   ← leaf: Jan 2025
2025 |   1 |   2 |   14500   ← leaf: Feb 2025
2025 |   1 |   3 |   13000   ← leaf: Mar 2025
2025 |   1 |NULL |   39500   ← subtotal: Q1 2025 (month rolled up)
2025 |   2 |   4 |   15000
...
2025 |NULL |NULL |  158000   ← subtotal: full year 2025 (quarter+month rolled up)
NULL |NULL |NULL |  310000   ← grand total (everything rolled up)
```

Because a rollup is a prefix of a hierarchy, it is also the cheapest of the three macros: only N+1 grouping sets. Prefer `ROLLUP` whenever your columns have a genuine parent→child relationship.

### 6.3 CUBE — All Combinations

`CUBE(a, b, c)` expands to **all 2^N subsets** of the columns:

```
CUBE(a, b, c) ≡ GROUPING SETS (
  (a,b,c),
  (a,b), (a,c), (b,c),
  (a),   (b),   (c),
  ()
)
```

Use `CUBE` when you want every cross-tabulation and the columns are *independent dimensions*, not a hierarchy. Example: revenue broken down by every combination of region, payment method, and product category — because an analyst might slice by any of them.

```sql
SELECT
  region,
  payment_method,
  category,
  SUM(amount) AS revenue,
  COUNT(*)    AS n
FROM orders o
JOIN payments p  ON p.order_id = o.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products pr ON pr.id = oi.product_id
JOIN categories c ON c.id = pr.category_id
GROUP BY CUBE (region, payment_method, category);
```

The cost warning: `CUBE` of N columns is `2^N` grouping sets, each a potential hash table. `CUBE(a,b,c,d,e)` is 32 grouping sets. On high-cardinality columns this both explodes the output row count and blows `work_mem`. Reach for `CUBE` only when you truly need every combination; otherwise enumerate the handful you want with explicit `GROUPING SETS`.

### 6.4 The GROUPING() Function — Reading the NULL Markers

This is the crux of the topic. Consider a `ROLLUP(region)` where the data itself contains rows with `region IS NULL` (orders never assigned a region):

```sql
SELECT region, SUM(amount) AS revenue
FROM orders
GROUP BY ROLLUP (region)
ORDER BY region;
```

```
 region  | revenue
---------+---------
 East    |  50000
 West    |  70000
 NULL    |   8000   ← is this "orders with no region" OR "grand total"??
 NULL    | 128000   ← ...and here's the other NULL row. Which is which?
```

Two rows have `region = NULL` and they mean utterly different things. One is real data (unassigned orders totaling 8000), one is the grand total (128000). A human might guess from the magnitude; a program cannot. `GROUPING()` resolves it deterministically:

```sql
SELECT
  region,
  SUM(amount) AS revenue,
  GROUPING(region) AS is_subtotal   -- 0 = real region value, 1 = rolled-up subtotal
FROM orders
GROUP BY ROLLUP (region)
ORDER BY GROUPING(region), region;
```

```
 region  | revenue | is_subtotal
---------+---------+-------------
 East    |  50000  |     0
 West    |  70000  |     0
 NULL    |   8000  |     0        ← is_subtotal=0 → REAL null-region data
 NULL    | 128000  |     1        ← is_subtotal=1 → grand-total subtotal marker
```

Now they are distinguishable. The standard production idiom is to replace the marker NULL with a human label while preserving real NULLs:

```sql
SELECT
  CASE WHEN GROUPING(region) = 1 THEN 'ALL REGIONS'
       ELSE COALESCE(region, '(no region)')     -- real NULL → its own label
  END AS region_label,
  SUM(amount) AS revenue
FROM orders
GROUP BY ROLLUP (region)
ORDER BY GROUPING(region), region;
```

```
 region_label | revenue
--------------+---------
 East         |  50000
 West         |  70000
 (no region)  |   8000    ← real missing-region orders
 ALL REGIONS  | 128000    ← the grand total, clearly labeled
```

The lesson: **any time you `ROLLUP`/`CUBE`/`GROUPING SETS` over a column that can contain NULL, you MUST use `GROUPING()` to disambiguate, or your report is ambiguous and potentially wrong.**

### 6.5 GROUPING() with Multiple Columns — The Bitmask

`GROUPING(a, b, c)` returns an integer bitmask, treating each column as a bit (leftmost = most significant). Bit is 1 when the column is rolled up.

```sql
SELECT
  region, product,
  SUM(amount) AS revenue,
  GROUPING(region, product) AS gbits   -- integer 0..3
FROM orders
GROUP BY CUBE (region, product)
ORDER BY gbits;
```

```
 region | product | revenue | gbits    interpretation
--------+---------+---------+------   ---------------------------------
 East   | Widget  |  10000  |  0       00 → both real (leaf row)
 West   | Gadget  |  25000  |  0       00 → both real (leaf row)
 East   | NULL    |  30000  |  1       01 → product rolled up (per-region subtotal)
 West   | NULL    |  45000  |  1       01 → product rolled up
 NULL   | Widget  |  18000  |  2       10 → region rolled up (per-product subtotal)
 NULL   | Gadget  |  40000  |  2       10 → region rolled up
 NULL   | NULL    |  75000  |  3       11 → both rolled up (grand total)
```

The bitmask lets you classify a row's level with a single comparison. `gbits = 0` → leaf detail. `gbits = 3` → grand total. `gbits = 1` → per-region subtotal. This is invaluable for `HAVING` filtering and for `ORDER BY` layout. It maps directly to the executor's internal grouping-set identifier.

### 6.6 Filtering Grouping Sets with HAVING

Because `HAVING` runs after grouping, you can keep or discard entire subtotal levels using `GROUPING()`:

```sql
-- Show ONLY per-region subtotals and the grand total; drop the leaf detail rows
SELECT region, product, SUM(amount) AS revenue,
       GROUPING(region, product) AS gbits
FROM orders
GROUP BY CUBE (region, product)
HAVING GROUPING(product) = 1        -- keep only rows where product is rolled up
ORDER BY gbits;
```

```sql
-- Show leaf rows and grand total, but suppress the intermediate subtotals
... HAVING GROUPING(region, product) IN (0, 3)
```

This is how you build a report that has detail rows and a grand total but *not* the middle subtotal noise — without a second query.

### 6.7 Combining and Nesting Grouping Constructs

You can put multiple constructs in one `GROUP BY`, and they multiply (a Cartesian product of grouping sets):

```sql
-- ROLLUP over time, crossed with every payment method
GROUP BY payment_method, ROLLUP (year, month)
-- expands to, for each payment_method:
--   (payment_method, year, month), (payment_method, year), (payment_method)
```

```sql
-- Two grouping sets crossed
GROUP BY GROUPING SETS ((a), (b)), GROUPING SETS ((c), (d))
-- expands to (a,c), (a,d), (b,c), (b,d)
```

```sql
-- ROLLUP and CUBE together
GROUP BY ROLLUP(region), CUBE(product, channel)
```

This composability is powerful but easy to over-use — the grouping-set count is the product of the parts, and each is a hash table. Always reason about how many grouping sets your combination produces before running it on a large table.

### 6.8 Ordering Subtotals Correctly

By default, subtotal rows (NULL in rolled-up columns) sort to the end in `ASC` (NULLs last) or to the front in `DESC` (NULLs first). For a clean report where each subtotal appears *below* its detail block, order by the grouping bit first, then the column:

```sql
ORDER BY
  region,                 -- group regions together
  GROUPING(product),      -- within a region: detail rows (0) before the subtotal (1)
  product;                -- alphabetize the detail rows
```

Without the `GROUPING(product)` term, the per-region subtotal (product = NULL) floats to the bottom of the *whole* result or top of the region depending on NULLS ordering, and the report looks broken. This ordering trick is a hallmark of someone who has actually shipped a subtotal report.

### 6.9 GROUPING SETS with Empty Set vs No Rows

Edge case: if the table (after WHERE) has **zero rows**, behavior differs by grouping set:

- `GROUP BY GROUPING SETS ((region), ())` on an empty table: the `(region)` set produces **no rows** (nothing to group), but the `()` grand-total set produces **exactly one row** with `SUM = NULL`, `COUNT = 0`. The grand total always emits one row even on an empty input, just as `SELECT COUNT(*) FROM empty` returns one row containing 0.

This means a `ROLLUP` report on a filtered-to-empty dataset still shows a grand-total line (with NULL/0 aggregates), which is usually the desired behavior for a report footer.

### 6.10 DISTINCT and ORDER BY Inside Aggregates Still Work

Grouping sets compose with everything from Topic 22. `SUM(DISTINCT x)`, `COUNT(DISTINCT x)`, `STRING_AGG(x, ',' ORDER BY x)`, and `FILTER` (Topic 23) all work per grouping set:

```sql
SELECT
  region,
  COUNT(DISTINCT customer_id) AS unique_customers,
  SUM(amount) FILTER (WHERE status = 'completed') AS completed_rev,
  GROUPING(region) AS g
FROM orders
GROUP BY ROLLUP (region);
```

Each grouping set gets its own independent `COUNT(DISTINCT ...)` and its own `FILTER` evaluation. The grand-total row's `unique_customers` is the count of distinct customers across the *entire* table — correctly de-duplicated, not the sum of per-region distinct counts.

---

## 7. EXPLAIN — Grouping Sets in the Plan

### MixedAggregate (sorted strategy, ROLLUP)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT region, product, SUM(amount) AS revenue
FROM orders
GROUP BY ROLLUP (region, product);
```

```
MixedAggregate  (cost=8214.00..9102.00 rows=3201 width=52)
                (actual time=61.2..74.8 rows=3187 loops=1)
  Hash Key: region, product
  Hash Key: region
  Group Key: ()
  Batches: 1  Memory Usage: 4096kB
  ->  Seq Scan on orders  (cost=0.00..6714.00 rows=200000 width=44)
                          (actual time=0.02..18.3 rows=200000 loops=1)
  Buffers: shared hit=4214
Planning Time: 0.28 ms
Execution Time: 76.1 ms
```

**Reading it**:
- `MixedAggregate` — Postgres computes several grouping sets in one node. Note there is **one** `Seq Scan` feeding it — a single pass over `orders`, which is the entire point.
- `Hash Key: region, product` and `Hash Key: region` — these two grouping sets are computed via in-memory hash tables.
- `Group Key: ()` — the grand total is computed as a plain aggregate (no key). Postgres mixes hash and sorted/group strategies in one node, hence the name.
- `Batches: 1` — all hash tables fit in `work_mem`. If this were `Batches: 4`, the grouping sets spilled to disk — the signal to raise `work_mem`.
- `rows=3187 actual` — the leaf rows plus every subtotal plus the grand total, all emitted from one node.

### HashAggregate with multiple hash keys (CUBE)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT region, payment_method, SUM(amount)
FROM orders o JOIN payments p ON p.order_id = o.id
GROUP BY CUBE (region, payment_method);
```

```
HashAggregate  (cost=12050.00..12420.00 rows=124 width=48)
               (actual time=98.4..98.9 rows=118 loops=1)
  Hash Key: o.region, p.payment_method
  Hash Key: o.region
  Hash Key: p.payment_method
  Group Key: ()
  Batches: 1  Memory Usage: 1536kB
  ->  Hash Join  (cost=3400.00..10100.00 rows=200000 width=40)
        Hash Cond: (p.order_id = o.id)
        ->  Seq Scan on payments p  (actual rows=200000 loops=1)
        ->  Hash  (actual rows=50000 loops=1)
              ->  Seq Scan on orders o  (actual rows=50000 loops=1)
Buffers: shared hit=8100
Execution Time: 99.6 ms
```

**Reading it**:
- Four grouping sets for `CUBE(region, payment_method)`: `(region, pm)`, `(region)`, `(pm)`, `()` — visible as three `Hash Key` lines plus `Group Key: ()`.
- The join runs **once**; its output feeds all four grouping sets simultaneously. No repeated scanning.
- `Memory Usage: 1536kB` — the combined footprint of all hash tables. For a `CUBE` over high-cardinality columns this number balloons; watch it against `work_mem`.

### The spill-to-disk warning sign

```
HashAggregate  (actual time=1204..3890 rows=1900000 loops=1)
  Hash Key: a, b, c
  Hash Key: a, b
  ...
  Batches: 8  Memory Usage: 262144kB  Disk Usage: 512480kB   ← SPILLED
```

`Batches > 1` plus `Disk Usage` means the simultaneous hash tables blew `work_mem` and Postgres partitioned to disk. Fixes: raise `work_mem` for the session, reduce the number of grouping sets (use `ROLLUP` instead of `CUBE`, or explicit `GROUPING SETS` with only the levels you need), or pre-aggregate.

### What to look for

| Symptom | Meaning | Fix |
|---------|---------|-----|
| One `Seq Scan` feeding `MixedAggregate`/`HashAggregate` with multiple keys | Correct — single-pass grouping sets | nothing, this is the goal |
| `Batches > 1`, `Disk Usage` present | Hash tables spilled to disk | raise `work_mem`, fewer grouping sets |
| Separate scans per grouping level | You wrote `UNION ALL`, not `GROUPING SETS` | rewrite with GROUPING SETS |
| Huge `actual rows` from the aggregate | `CUBE` over high-cardinality cols exploded output | narrow to needed grouping sets |
| `Sort` node before the aggregate | Sorted (GroupAggregate) strategy chosen | fine if input already indexed; else memory sort cost |

---

## 8. Query Examples

### Example 1 — Basic: ROLLUP for a Two-Level Subtotal Report

```sql
-- Revenue by category, with a subtotal per category and a grand total.
-- Single pass over the join; ROLLUP gives us category subtotals + grand total.
SELECT
  c.name                         AS category,
  SUM(oi.quantity * oi.unit_price) AS revenue,
  GROUPING(c.name)               AS is_grand_total   -- 1 on the grand-total row
FROM order_items oi
JOIN products p   ON p.id = oi.product_id
JOIN categories c ON c.id = p.category_id
GROUP BY ROLLUP (c.name)
ORDER BY GROUPING(c.name), revenue DESC;   -- detail rows first, grand total last
```

### Example 2 — Intermediate: CUBE with GROUPING() Disambiguation

```sql
-- Cross-tab of revenue by region and payment method, all combinations.
-- Labels every subtotal cleanly and preserves any real NULLs in region.
SELECT
  CASE WHEN GROUPING(o.region) = 1 THEN 'ALL REGIONS'
       ELSE COALESCE(o.region, '(unassigned)') END        AS region,
  CASE WHEN GROUPING(p.payment_method) = 1 THEN 'ALL METHODS'
       ELSE COALESCE(p.payment_method, '(unknown)') END   AS payment_method,
  SUM(p.amount)                          AS revenue,
  COUNT(DISTINCT o.id)                   AS orders,
  GROUPING(o.region, p.payment_method)   AS level         -- 0/1/2/3 bitmask
FROM orders o
JOIN payments p ON p.order_id = o.id
WHERE o.status = 'completed'
GROUP BY CUBE (o.region, p.payment_method)
ORDER BY level, region, payment_method;
```

### Example 3 — Production Grade: Monthly Sales Hierarchy Dashboard

```sql
-- CONTEXT:
--   orders: ~40M rows, order_items: ~180M rows (partitioned by month)
--   Index: order_items(order_id), orders(created_at, status)
--   Goal: a finance dashboard showing revenue by year → quarter → month,
--         plus category breakdown, with subtotals at each hierarchy level.
--   Expectation: single scan of the last-12-months partition slice, ~300-600ms.
--   Using ROLLUP (not CUBE) because time is a strict hierarchy — no need for
--   the (quarter, month-without-year) combinations CUBE would generate.

WITH line_revenue AS (
  -- Pre-aggregate to line level first to keep the fact table narrow before rollup.
  SELECT
    o.id                              AS order_id,
    EXTRACT(YEAR    FROM o.created_at)::int AS yr,
    EXTRACT(QUARTER FROM o.created_at)::int AS qtr,
    EXTRACT(MONTH   FROM o.created_at)::int AS mon,
    c.name                            AS category,
    oi.quantity * oi.unit_price       AS line_amount
  FROM orders o
  JOIN order_items oi ON oi.order_id = o.id
  JOIN products p     ON p.id = oi.product_id
  JOIN categories c   ON c.id = p.category_id
  WHERE o.status = 'completed'
    AND o.created_at >= DATE_TRUNC('month', CURRENT_DATE) - INTERVAL '12 months'
)
SELECT
  yr, qtr, mon,
  CASE WHEN GROUPING(category) = 1 THEN 'ALL CATEGORIES' ELSE category END AS category,
  SUM(line_amount)                                   AS revenue,
  -- classify each output row for the front-end to render indentation/bolding
  GROUPING(yr, qtr, mon, category)                   AS level_bits
FROM line_revenue
GROUP BY ROLLUP (yr, qtr, mon), ROLLUP (category)
ORDER BY
  yr, qtr, mon,
  GROUPING(category), category;
```

```
EXPLAIN (ANALYZE, BUFFERS) [abridged]:

MixedAggregate  (cost=? rows=? width=?) (actual time=? rows=48213 loops=1)
  Hash Key: yr, qtr, mon, category
  Hash Key: yr, qtr, mon
  Hash Key: yr, qtr, category
  Hash Key: yr, qtr
  ... (ROLLUP(time) × ROLLUP(category) = 4 × 2 = 8 grouping sets)
  Batches: 1  Memory Usage: 18432kB
  ->  Subquery Scan on line_revenue (actual rows=2140000 loops=1)
        ->  ... single scan of the 12-month partition slice ...
Execution Time: ~470 ms
```

Note `ROLLUP(time) , ROLLUP(category)` produces 4 × 2 = 8 grouping sets — a deliberate, bounded choice, not the 32 a `CUBE(yr,qtr,mon,category)` would produce.

---

## 9. Wrong → Right Patterns

### Wrong 1: UNION ALL instead of GROUPING SETS (N scans)

```sql
-- WRONG: three full scans of a huge table to build a subtotal report
SELECT region, product, SUM(amount) FROM orders GROUP BY region, product
UNION ALL
SELECT region, NULL,   SUM(amount) FROM orders GROUP BY region
UNION ALL
SELECT NULL,   NULL,   SUM(amount) FROM orders;
-- Scans orders 3×. On 50M rows this is 3× the I/O and 3× the CPU.
-- Also: the branches can silently drift if one gets a WHERE the others miss.
```

```sql
-- RIGHT: one scan, one query, impossible to drift out of sync
SELECT region, product, SUM(amount),
       GROUPING(region, product) AS g
FROM orders
GROUP BY GROUPING SETS ((region, product), (region), ());
```

**Why it's wrong at the execution level**: each `UNION ALL` branch is planned as an independent aggregate with its own `Seq Scan`. `GROUPING SETS` shares one scan across all levels via `MixedAggregate`/`HashAggregate`.

### Wrong 2: NULL ambiguity — real NULLs vs subtotal NULLs

```sql
-- WRONG: some orders have region = NULL in the data
SELECT region, SUM(amount) AS revenue
FROM orders
GROUP BY ROLLUP (region);
-- Result has TWO rows with region = NULL: unassigned-region orders AND the grand
-- total. A downstream consumer cannot tell them apart. If code does
-- WHERE region IS NULL to "find unassigned orders", it also grabs the grand total.
```

```sql
-- RIGHT: disambiguate with GROUPING(), label the subtotal
SELECT
  CASE WHEN GROUPING(region) = 1 THEN 'ALL REGIONS'
       ELSE COALESCE(region, '(unassigned)') END AS region,
  SUM(amount) AS revenue
FROM orders
GROUP BY ROLLUP (region)
ORDER BY GROUPING(region), region;
```

**Why it's wrong**: `ROLLUP` uses NULL as its subtotal marker in the rolled-up column. That marker collides with genuine NULL data. Only `GROUPING()` distinguishes them — no amount of `IS NULL` checking can.

### Wrong 3: CUBE where ROLLUP was meant (exploded output + memory)

```sql
-- WRONG: a strict time hierarchy grouped with CUBE
SELECT yr, qtr, mon, SUM(amount)
FROM orders
GROUP BY CUBE (yr, qtr, mon);
-- Produces nonsense grouping sets like (qtr, mon) and (mon) — "month 3 across
-- all years and quarters", which is meaningless for a fiscal report, plus 8
-- grouping sets instead of 4, doubling memory and output rows.
```

```sql
-- RIGHT: ROLLUP respects the hierarchy — only prefixes
SELECT yr, qtr, mon, SUM(amount)
FROM orders
GROUP BY ROLLUP (yr, qtr, mon);   -- 4 sets: (yr,qtr,mon),(yr,qtr),(yr),()
```

**Why it's wrong**: `CUBE` generates every subset including hierarchically meaningless ones. For nested dimensions (year⊃quarter⊃month) only the prefix sets make sense, and `ROLLUP` gives exactly those at half the cost.

### Wrong 4: Filtering subtotals in WHERE instead of HAVING

```sql
-- WRONG: trying to keep only subtotal rows via WHERE — errors or does nothing useful
SELECT region, product, SUM(amount)
FROM orders
WHERE GROUPING(product) = 1        -- ERROR: GROUPING/aggregates not allowed in WHERE
GROUP BY CUBE (region, product);
```

```sql
-- RIGHT: GROUPING() lives in HAVING (post-grouping)
SELECT region, product, SUM(amount)
FROM orders
GROUP BY CUBE (region, product)
HAVING GROUPING(product) = 1;      -- keep only per-region subtotals
```

**Why it's wrong**: `WHERE` runs before grouping (section 3), so grouping sets do not exist yet and `GROUPING()` is undefined there. Filtering on grouping level is a post-aggregation operation → `HAVING`.

### Wrong 5: Selecting a column not in any grouping set and expecting a value

```sql
-- WRONG expectation: customer_id will show the real customer on subtotal rows
SELECT region, customer_id, SUM(amount)
FROM orders
GROUP BY ROLLUP (region);
-- ERROR: column "customer_id" must appear in GROUP BY or be used in an aggregate.
-- customer_id is in NO grouping set, so it has no defined value per group.
```

```sql
-- RIGHT: either aggregate it or add it to the grouping sets
SELECT region, COUNT(DISTINCT customer_id) AS customers, SUM(amount)
FROM orders
GROUP BY ROLLUP (region);
```

**Why it's wrong**: the Topic 20 rule still holds — every non-aggregated SELECT column must be part of the grouping. A column absent from all grouping sets has no single value per group and is rejected.

---

## 10. Performance Profile

### Cost model

| Construct | Grouping sets | Simultaneous hash tables (worst case) |
|-----------|--------------|----------------------------------------|
| `GROUP BY a` | 1 | 1 |
| `GROUPING SETS ((a),(b))` | 2 | 2 |
| `ROLLUP (a,b,c)` | N+1 = 4 | up to 4 |
| `CUBE (a,b,c)` | 2^N = 8 | up to 8 |
| `CUBE (a,b,c,d,e)` | 2^N = 32 | up to 32 |
| `ROLLUP(a,b), CUBE(c,d)` | 3 × 4 = 12 | up to 12 |

The single most important performance fact: **grouping sets share one scan of the input, but each grouping set needs its own aggregate state (hash table) in memory concurrently.** Time scales with input rows (one pass); memory scales with the number and cardinality of grouping sets.

### Scaling at 1M / 10M / 100M rows

| Rows | `GROUP BY a` | `ROLLUP(a,b)` (3 sets) | `CUBE(a,b,c)` (8 sets) |
|------|-------------|------------------------|------------------------|
| 1M | ~80ms, in-memory | ~120ms, in-memory | ~200ms, in-memory |
| 10M | ~800ms | ~1.2s | ~2.5s, may spill if high-cardinality |
| 100M | ~8s, likely spill | ~12s, spill likely | ~30s+, heavy spill, reconsider design |

The numbers are dominated by (a) the single scan cost — same order as a plain `GROUP BY` — plus (b) hash-table maintenance for every grouping set. When the combined hash tables exceed `work_mem`, Postgres spills (`Batches > 1`) and time jumps 3–10×.

### Optimization techniques specific to grouping sets

1. **Prefer ROLLUP over CUBE** when the columns are hierarchical — N+1 sets vs 2^N. Massive difference at N ≥ 4.
2. **Enumerate only the levels you need with explicit GROUPING SETS.** If a report needs leaf + grand total but not the middle subtotal, write `GROUPING SETS ((a,b), ())` — 2 sets, not the 4 `ROLLUP` gives.
3. **Raise `work_mem` for the session** running the report, not globally: `SET LOCAL work_mem = '256MB';` inside the transaction. This keeps all hash tables resident and avoids spilling.
4. **Pre-aggregate in a CTE** to shrink the fact table before rolling up (see Example 3). Rolling up 2M pre-aggregated rows is far cheaper than 180M line items.
5. **Filter early in WHERE.** WHERE runs before grouping — the fewer rows enter the aggregate, the fewer any hash table holds. Partition pruning on a time column is the biggest lever on large fact tables.
6. **Watch output cardinality with CUBE on high-distinct columns.** `CUBE(customer_id, product_id)` on millions of distinct values produces an enormous result set — the output itself becomes the bottleneck.
7. **Materialize heavy dashboards.** If the same rollup runs every page load, compute it into a summary table (or `MATERIALIZED VIEW`) on a schedule and query that.

### Index interactions

Grouping sets can use the **sorted (GroupAggregate/MixedAggregate)** strategy when an index provides the needed order, avoiding an explicit sort and reducing memory (sorted aggregation holds only the current group, not a full hash table). An index on the leading rollup columns — e.g. `(region, product)` — can feed a `MixedAggregate` without a separate `Sort` node. But for hash-based grouping sets, indexes on the grouping columns rarely help; the operation is scan-bound. The win from an index is mostly in the `WHERE` filter that reduces input rows.

---

## 11. Node.js Integration

### 11.1 Basic ROLLUP report with the pg driver

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function revenueByCategoryWithTotal() {
  const { rows } = await pool.query(
    `SELECT
       CASE WHEN GROUPING(c.name) = 1 THEN 'ALL CATEGORIES'
            ELSE c.name END                        AS category,
       SUM(oi.quantity * oi.unit_price)            AS revenue,
       GROUPING(c.name)                            AS is_total
     FROM order_items oi
     JOIN products p   ON p.id = oi.product_id
     JOIN categories c ON c.id = p.category_id
     GROUP BY ROLLUP (c.name)
     ORDER BY GROUPING(c.name), revenue DESC`
  );
  // is_total comes back as a number (0 or 1). revenue is a string (numeric) — parse it.
  return rows.map(r => ({
    category: r.category,
    revenue: Number(r.revenue),
    isGrandTotal: r.is_total === 1,
  }));
}
```

### 11.2 Parameterized CUBE with GROUPING() classification

```javascript
// Cross-tab revenue by region × payment method for a date window.
// $1/$2 are the window bounds — never string-interpolate dates into SQL.
async function revenueCrossTab(fromDate, toDate) {
  const { rows } = await pool.query(
    `SELECT
       CASE WHEN GROUPING(o.region) = 1 THEN 'ALL'
            ELSE COALESCE(o.region, '(unassigned)') END      AS region,
       CASE WHEN GROUPING(p.payment_method) = 1 THEN 'ALL'
            ELSE COALESCE(p.payment_method, '(unknown)') END AS payment_method,
       SUM(p.amount)                        AS revenue,
       GROUPING(o.region, p.payment_method) AS level_bits
     FROM orders o
     JOIN payments p ON p.order_id = o.id
     WHERE o.status = 'completed'
       AND o.created_at >= $1 AND o.created_at < $2
     GROUP BY CUBE (o.region, p.payment_method)
     ORDER BY level_bits, region, payment_method`,
    [fromDate, toDate]
  );
  // Split the flat result into detail vs subtotal buckets for the UI.
  return {
    detail:       rows.filter(r => r.level_bits === 0),
    regionTotals: rows.filter(r => r.level_bits === 1),
    methodTotals: rows.filter(r => r.level_bits === 2),
    grandTotal:   rows.find(r => r.level_bits === 3) ?? null,
  };
}
```

### 11.3 Raising work_mem for a heavy rollup within a transaction

```javascript
// SET LOCAL confines the work_mem bump to this transaction only — safe under a pool.
async function heavyRollup() {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query("SET LOCAL work_mem = '256MB'");   // avoid grouping-set spill
    const { rows } = await client.query(
      `SELECT yr, qtr, mon, SUM(amount) AS revenue,
              GROUPING(yr, qtr, mon) AS g
       FROM (
         SELECT EXTRACT(YEAR FROM created_at)::int  AS yr,
                EXTRACT(QUARTER FROM created_at)::int AS qtr,
                EXTRACT(MONTH FROM created_at)::int   AS mon,
                total_amount AS amount
         FROM orders WHERE status = 'completed'
       ) s
       GROUP BY ROLLUP (yr, qtr, mon)
       ORDER BY yr, qtr, mon`
    );
    await client.query('COMMIT');
    return rows;
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();   // work_mem reverts automatically — SET LOCAL was scoped
  }
}
```

**Driver notes**: `GROUPING()` returns an `int4` → JS `number`. `SUM` over numeric/bigint returns a **string** in `pg` to avoid precision loss — parse with `Number()` or a decimal library. There is no special driver support needed; grouping sets are plain SQL and the result is a flat rowset. Your application code is responsible for splitting detail from subtotal rows using the `GROUPING()` columns you select.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do GROUPING SETS / ROLLUP / CUBE?** — No. Prisma's `groupBy` supports only a flat single grouping (one set of `by` columns) with aggregates. There is no concept of multiple grouping sets, subtotals, or `GROUPING()`.

```typescript
// This gives ONE grouping level only — no subtotals, no grand total:
const byRegion = await prisma.order.groupBy({
  by: ['region'],
  _sum: { amount: true },
  where: { status: 'completed' },
});

// For rollups/cubes you MUST drop to raw SQL:
const rollup = await prisma.$queryRaw`
  SELECT region, product, SUM(amount) AS revenue,
         GROUPING(region, product) AS g
  FROM orders
  GROUP BY ROLLUP (region, product)
  ORDER BY g, region, product
`;
```

**Where it breaks**: everything beyond one flat grouping. **Verdict**: use `$queryRaw` for any grouping-sets report.

### Drizzle ORM

**Can Drizzle do it?** — Only via the `sql` template escape hatch for the `GROUP BY` clause. Drizzle's `.groupBy()` accepts raw `sql`, so you can express `ROLLUP`/`CUBE`/`GROUPING SETS` and `GROUPING()` in `sql` expressions.

```typescript
import { sql } from 'drizzle-orm';

const rows = await db
  .select({
    region: orders.region,
    product: orders.product,
    revenue: sql<string>`SUM(${orders.amount})`,
    level: sql<number>`GROUPING(${orders.region}, ${orders.product})`,
  })
  .from(orders)
  .groupBy(sql`ROLLUP (${orders.region}, ${orders.product})`)
  .orderBy(sql`GROUPING(${orders.region}, ${orders.product})`);
```

**Where it breaks**: no typed builder for grouping sets — you hand-write the `sql` fragment; type inference on the grouping columns is manual. **Verdict**: workable and reasonably clean; the `sql` tag keeps parameters safe.

### Sequelize

**Can Sequelize do it?** — Not natively. `group` accepts an array of columns (one flat set) or `Sequelize.literal()` for raw fragments.

```javascript
const rows = await Order.findAll({
  attributes: [
    'region',
    [sequelize.fn('SUM', sequelize.col('amount')), 'revenue'],
    [sequelize.literal('GROUPING(region)'), 'is_total'],
  ],
  group: [sequelize.literal('ROLLUP (region)')],
  raw: true,
});
// Or just use sequelize.query() for the whole thing.
```

**Where it breaks**: `GROUPING()` and rollup syntax must be `literal()`, losing all safety and typing. **Verdict**: use `sequelize.query()` with bind parameters for real reports.

### TypeORM

**Can TypeORM do it?** — Only through raw `groupBy` strings in the QueryBuilder, or `.query()`.

```typescript
const rows = await dataSource
  .createQueryBuilder()
  .select('region').addSelect('product')
  .addSelect('SUM(amount)', 'revenue')
  .addSelect('GROUPING(region, product)', 'level')
  .from('orders', 'o')
  .groupBy('ROLLUP (region, product)')     // raw string — no builder support
  .orderBy('level')
  .getRawMany();
```

**Where it breaks**: `groupBy('ROLLUP (...)')` is an unchecked string; the entity mapper cannot map subtotal rows (mixed NULLs) to entities, so you must use `getRawMany()`. **Verdict**: raw group-by string works; treat results as raw rows.

### Knex.js

**Can Knex do it?** — Via `groupByRaw` and `knex.raw`, cleanly.

```javascript
const rows = await knex('orders')
  .select('region', 'product')
  .select(knex.raw('SUM(amount) AS revenue'))
  .select(knex.raw('GROUPING(region, product) AS level'))
  .where('status', 'completed')
  .groupByRaw('ROLLUP (region, product)')
  .orderByRaw('GROUPING(region, product), region, product');
```

**Where it breaks**: nothing structural — Knex is transparent enough that `groupByRaw` maps directly to SQL. Parameterize any values via `knex.raw('... ?', [val])`. **Verdict**: the most natural fit among query builders.

### ORM Summary Table

| ORM | Grouping sets support | How | Verdict |
|-----|----------------------|-----|---------|
| Prisma | None | `$queryRaw` only | Raw SQL |
| Drizzle | Escape hatch | `sql` tag in `.groupBy()` | Workable, safe params |
| Sequelize | None native | `literal()` / `.query()` | Raw query recommended |
| TypeORM | Raw string | `groupBy('ROLLUP(...)')` + `getRawMany()` | Raw rows only |
| Knex | Yes (raw) | `groupByRaw` | Cleanest builder fit |

The universal truth: **no mainstream ORM has first-class grouping-sets support.** Every one of them funnels you to raw SQL or a raw fragment. This is a feature you write by hand — which makes understanding the SQL non-optional.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given `orders(id, region, status, total_amount, created_at)`:

Write a query that returns total `revenue` (`SUM(total_amount)`) per region for `completed` orders, **plus** a grand-total row. Label the grand-total row's region as `'ALL REGIONS'` and every other row with its real region name (treat a genuine NULL region as `'(unassigned)'`). Sort so detail rows appear first and the grand total appears last.

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines Topic 20 GROUP BY + Topic 22 aggregates)

Given `orders(id, region, customer_id, total_amount, status, created_at)`:

For completed orders in the last 90 days, produce a report grouped by `GROUPING SETS` that shows, at (region) level and at grand-total level only (no leaf detail):
- `region` (labeled, using `GROUPING()`)
- `revenue` = `SUM(total_amount)`
- `unique_customers` = `COUNT(DISTINCT customer_id)`
- `avg_order_value` rounded to 2 dp

Then explain in a comment: why is the grand-total row's `unique_customers` *not* equal to the sum of the per-region `unique_customers`?

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

Given `orders(id, region, status, total_amount)` where **some orders have `region = NULL`** in the real data, a developer wrote:

```sql
SELECT region, SUM(total_amount) AS revenue
FROM orders
GROUP BY ROLLUP (region);
```

and then, in application code, filtered `rows.filter(r => r.region === null)` to find "unassigned-region revenue" — and got a number that was wildly too high.

1. Explain exactly why the number is too high.
2. Rewrite the SQL so the application can unambiguously identify the unassigned-region row and the grand-total row.
3. Add a per-region `pct_of_total` column (each region's revenue as a percentage of the grand total) computed in the same single-pass query. Hint: you cannot reference the grand-total subtotal row directly from a detail row inside one grouping-sets query — think about a window function over the result, or a CTE.

```sql
-- Write your query here
```

### Exercise 4 — Interview Simulation

You are asked, live: "Build one query that produces a revenue report by `region` and `payment_method` showing every combination, every single-dimension subtotal, and the grand total — and make it possible for the front-end to render each row at the correct indentation level without guessing from the NULLs."

Write the query, and be ready to explain (a) why you chose `CUBE` vs `GROUPING SETS`, (b) what the `GROUPING(region, payment_method)` bitmask values mean, and (c) how many grouping sets and hash tables this creates.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What does `GROUP BY ROLLUP (a, b)` produce that a plain `GROUP BY a, b` does not?**

Junior answer: "It adds subtotals."

Principal answer: `GROUP BY a, b` produces exactly one grouping set — one row per distinct `(a, b)`. `ROLLUP (a, b)` produces N+1 = 3 grouping sets in a single pass: `(a, b)` leaf rows, `(a)` subtotals (with `b` set to NULL), and `()` the grand total (both NULL). So you get the same detail rows *plus* a subtotal per `a` *plus* one grand-total row, all computed from one scan of the table rather than three separate queries. The rolled-up columns carry NULL as a marker.

Follow-up the interviewer asks: "How would you tell a subtotal NULL apart from a real NULL in column `b`?" → `GROUPING(b)` returns 1 for the subtotal marker, 0 for real data.

---

**Q: What is the difference between `ROLLUP` and `CUBE`?**

Junior answer: "CUBE does more combinations."

Principal answer: `ROLLUP(a, b, c)` produces only the prefix hierarchy — `(a,b,c), (a,b), (a), ()` — N+1 sets, dropping columns from the right. It models a strict hierarchy like year→quarter→month. `CUBE(a, b, c)` produces all 2^N subsets — every combination including `(a,c)`, `(b,c)`, `(b)` — 8 sets for 3 columns. Use `ROLLUP` for nested dimensions where only prefixes are meaningful; use `CUBE` for independent dimensions where any cross-tab is valid. The cost difference is N+1 vs 2^N grouping sets, which matters enormously for memory and output size as N grows.

Follow-up: "You have `CUBE(a,b,c,d,e)` on a 50M-row table and it's spilling to disk. What do you do?" → Reduce to the grouping sets actually needed via explicit `GROUPING SETS`, or `ROLLUP` if hierarchical; raise `work_mem` for the session; pre-aggregate first.

---

**Q: Why might a `ROLLUP` report show two rows with the same NULL in a column?**

Junior answer: "A bug in the query."

Principal answer: It is not a bug — it is the collision between the subtotal marker and real data. `ROLLUP` uses NULL to mark "this column was rolled up." If the underlying data also contains genuine NULLs in that column, you get one row for the real null-valued group and one row for the subtotal, both showing NULL. They are semantically different rows. The fix is `GROUPING(col)`: it returns 1 only for the rollup-generated marker, letting you label or filter them apart. This is why any rollup over a nullable column must use `GROUPING()`.

Follow-up: "Show me the `ORDER BY` that puts subtotals below their detail rows." → `ORDER BY <group cols>, GROUPING(rolled_col), rolled_col`.

---

### Principal Level

**Q: Explain, at the executor level, why `GROUPING SETS` is faster than the equivalent `UNION ALL` of separate `GROUP BY` queries.**

Principal answer: Each branch of a `UNION ALL` is planned as an independent subplan with its own scan and its own aggregate node — so three grouping levels means three full scans of the table (or three index scans), three times the buffer traffic and CPU. `GROUPING SETS` compiles to a single `MixedAggregate` or `HashAggregate` node fed by **one** scan. That node maintains multiple aggregate states concurrently — one hash table per grouping set for the hash strategy, or resets at key boundaries for the sorted strategy — and emits all levels from the single input stream. You trade extra memory (concurrent hash tables) for eliminating repeated scans. On large fact tables the scan is the dominant cost, so single-pass wins decisively. The trade-off flips only if the grouping sets are so numerous and high-cardinality that the concurrent hash tables spill to disk — then you may be I/O-bound again, but on temp files rather than the base table.

Follow-up: "How do you know from EXPLAIN whether it spilled?" → `Batches > 1` and a `Disk Usage:` line on the aggregate node.

---

**Q: A finance report runs `GROUP BY ROLLUP(year, quarter, month), category` on 100M rows and takes 40 seconds. Walk me through diagnosis and fixes.**

Principal answer: First, `EXPLAIN (ANALYZE, BUFFERS)`. Check: (1) Is it one scan feeding a `MixedAggregate`, or did the planner do something pathological? (2) `Batches`/`Disk Usage` — is it spilling? `ROLLUP(y,q,m) , category` is 4 × (categories set count). Actually crossing `ROLLUP(y,q,m)` (4 sets) with a bare `category` column means 4 grouping sets each also grouping by category — potentially large output. (3) Is the input filtered? If there's no `WHERE` narrowing the time range, all 100M rows enter the aggregate. Fixes in order of impact: add/verify a `WHERE created_at >= ...` to prune partitions and cut input rows (biggest lever); pre-aggregate to a daily or monthly summary in a CTE so the rollup processes millions not hundreds of millions of rows; raise `SET LOCAL work_mem` to keep hash tables resident and stop spilling; if the same report runs constantly, materialize it into a summary table refreshed on schedule. Finally, confirm the grouping sets are all needed — if only year/quarter subtotals matter, enumerate them explicitly rather than the full cross-product.

Follow-up: "Why pre-aggregate instead of just raising work_mem?" → work_mem stops spilling but you still scan and hash 100M rows every run; pre-aggregation permanently reduces the row count entering the expensive multi-hash-table operation, and the summary can be reused.

---

**Q: How does `GROUPING(a, b, c)` return its value, and how would you use it to render an indented report?**

Principal answer: `GROUPING(a, b, c)` returns an integer bitmask where each column is a bit, leftmost most-significant, set to 1 when that column is rolled up (not part of the current grouping set). So `000 = 0` is a full-detail leaf row, `111 = 7` is the grand total, and intermediate values identify each subtotal level exactly. This maps one-to-one to the executor's internal grouping-set identifier, so it costs nothing to compute. For an indented report, the front-end derives indentation depth from the number of *un-rolled* columns: `popcount(~bits)` among the grouped columns, or simply switch on the bitmask value. You can also drive `ORDER BY` with it to place each subtotal directly under its detail block, and drive `HAVING` with it to keep or drop whole levels. It turns the otherwise-ambiguous NULL layout into a precise, machine-readable hierarchy tag.

Follow-up: "What's the value for the per-`b` subtotal in `CUBE(a,b)`?" → `a` rolled up, `b` present → `10` binary → `GROUPING(a,b) = 2`.

---

## 15. Mental Model Checkpoint

1. You run `GROUP BY CUBE (a, b, c, d)`. How many grouping sets does this produce, and roughly how many concurrent hash tables might the executor hold? Why might this spill to disk where `ROLLUP(a,b,c,d)` would not?

2. A column `region` contains real NULLs in the data. You write `GROUP BY ROLLUP(region)`. Describe every distinct row that will appear with `region IS NULL`, and how you tell them apart.

3. Why can `GROUPING()` appear in `SELECT`, `HAVING`, and `ORDER BY`, but never in `WHERE`? Tie your answer to logical execution order.

4. You have `GROUP BY ROLLUP(a), ROLLUP(b)`. How many grouping sets is that, and what are they? How does it differ from `ROLLUP(a, b)`?

5. On an empty table (after WHERE filters everything out), which grouping sets in `GROUPING SETS ((region), ())` emit a row, and what values do the aggregates hold?

6. Two engineers get different totals from "the same" subtotal report — one used `GROUPING SETS ... UNION ALL` split across two queries with slightly different WHERE clauses, the other used one `GROUPING SETS` query. Why is the single-query form structurally safer, beyond performance?

7. You need leaf detail rows and the grand total, but *not* the intermediate per-region and per-product subtotals, from two dimensions. Do you use `CUBE`, `ROLLUP`, or explicit `GROUPING SETS` — and what exactly do you write?

---

## 16. Quick Reference Card

```sql
-- EXPLICIT grouping sets (you list each one)
GROUP BY GROUPING SETS ( (a, b), (a), (b), () )

-- ROLLUP: hierarchy, N+1 sets, drops columns from the RIGHT
GROUP BY ROLLUP (a, b, c)
--   ≡ GROUPING SETS ( (a,b,c), (a,b), (a), () )

-- CUBE: all combinations, 2^N sets
GROUP BY CUBE (a, b, c)
--   ≡ GROUPING SETS ( (a,b,c),(a,b),(a,c),(b,c),(a),(b),(c),() )

-- () = grand total (empty grouping set) — always emits 1 row, even on empty input

-- GROUPING(): 1 = column ROLLED UP (subtotal marker), 0 = real value
SELECT GROUPING(region)              -- single column → 0 or 1
SELECT GROUPING(region, product)     -- multi column → integer bitmask (leftmost = MSB)

-- Disambiguate the subtotal NULL from a real NULL:
CASE WHEN GROUPING(region) = 1 THEN 'ALL' ELSE COALESCE(region,'(none)') END

-- Keep only subtotals / drop leaf rows: use HAVING (post-grouping), NOT WHERE
HAVING GROUPING(product) = 1

-- Order so subtotals sit below their detail block:
ORDER BY region, GROUPING(product), product

-- Combine constructs (grouping sets MULTIPLY):
GROUP BY payment_method, ROLLUP (year, month)   -- pm × 3 time levels
GROUP BY ROLLUP(a), CUBE(b, c)                  -- 2 × 4 = 8 sets

-- PERF RULES OF THUMB:
--  * single scan, but 1 hash table PER grouping set held concurrently
--  * prefer ROLLUP (N+1 sets) over CUBE (2^N) for hierarchies
--  * enumerate only needed levels with explicit GROUPING SETS
--  * EXPLAIN: one Seq Scan → MixedAggregate/HashAggregate = correct (single pass)
--  * Batches > 1 + Disk Usage = spilled → raise work_mem / fewer sets / pre-aggregate
--  * SET LOCAL work_mem = '256MB' inside the report txn to avoid spill

-- INTERVIEW ONE-LINERS:
--  "ROLLUP is prefix subtotals; CUBE is all subsets; GROUPING SETS is BYO list."
--  "GROUPING() = 1 means that column's NULL is a subtotal marker, not real data."
--  "One scan feeds all grouping sets — that's why it beats UNION ALL of GROUP BYs."
--  "No mainstream ORM does grouping sets natively — it's a raw-SQL feature."
```

---

## Connected Topics

- **Topic 20 — GROUP BY Fundamentals**: grouping sets are the multi-key generalization of a single GROUP BY; the "every non-aggregate in SELECT must be grouped" rule still governs here.
- **Topic 22 — Aggregate Functions**: SUM/COUNT/AVG and `COUNT(DISTINCT ...)` run independently per grouping set; the grand-total row's distinct counts are computed over the whole set, not summed.
- **Topic 23 — FILTER Clause**: `FILTER (WHERE ...)` composes with grouping sets — each set evaluates its filtered aggregates separately.
- **Topic 25 — Aggregate Performance**: the next topic; work_mem, hash vs sorted aggregation, and spill behavior that grouping sets stress hardest.
- **Topic 07 — NULL in Depth**: the three-valued logic behind why the subtotal marker is NULL and why `GROUPING()` (not `IS NULL`) is required to disambiguate.
- **Topic 26 — Window Functions**: the alternative for running totals and percent-of-total that grouping sets cannot express within one grouping row (see Exercise 3).
- **Topic 02 — SELECT & Logical Execution Order**: WHERE-before-GROUP BY-before-HAVING is what determines where `GROUPING()` is legal.
- **Internals — HashAggregate / MixedAggregate**: the physical operators that hold one hash table per grouping set in a single pass.
</invoke>
