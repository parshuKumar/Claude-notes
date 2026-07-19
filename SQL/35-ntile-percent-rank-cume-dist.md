# Topic 35 — NTILE, PERCENT_RANK, CUME_DIST
### SQL Mastery Curriculum — Phase 6: Window Functions — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine a school gym class of 100 students lined up shortest to tallest, facing the wall. The coach needs to do three different things with that single line.

**First job — split them into teams.** The coach says "count off into 4 teams." Walking down the line she taps 25 kids for Team 1, the next 25 for Team 2, and so on. That is `NTILE(4)`. She doesn't care about anyone's exact height — she cares only about their *position* in the line and wants roughly-equal-sized groups. If there were 102 students instead of 100, she couldn't make four groups of exactly 25.5, so she gives the first two teams 26 and the last two teams 25. The extra bodies always go to the **front** teams. That "extra goes to the earlier bucket" rule is the whole personality of NTILE.

**Second job — award a percentile ribbon based on rank.** Now she wants to tell each kid "you are taller than X% of the people *ranked below you*." The shortest kid is taller than nobody, so gets 0%. The tallest is taller than everyone-else-ranked-below, so gets 100%. That's `PERCENT_RANK`: it always pins the first row at exactly 0.0 and the last row at exactly 1.0, and spaces everyone else by how many people they outrank. The formula is literally "(your rank − 1) ÷ (everyone else)".

**Third job — award a ribbon based on how much of the line is at-or-below you.** This time she says "you and everyone shorter than you make up X% of the class." The shortest kid — if he's alone at his height — is 1% (1 person out of 100 is at-or-below him). The tallest is 100% because the entire class is at-or-below the tallest. That's `CUME_DIST` (cumulative distribution): "what fraction of the group is ≤ me." It never returns 0, and it always returns exactly 1.0 for the last row.

Three tools, one line of students. NTILE ignores exact values and chops into groups. PERCENT_RANK and CUME_DIST both produce a 0–1 score, but they disagree on the endpoints and on how they treat ties — and that disagreement is exactly what interviewers probe.

---

## 2. Connection to SQL Internals

All three of these are **window functions**, so they inherit the entire window-function execution machinery from Topic 33 and Topic 34. Underneath, PostgreSQL implements them with a `WindowAgg` node sitting on top of a sorted input.

The critical internal fact: **NTILE, PERCENT_RANK, and CUME_DIST all require the input to be fully materialized and ordered before they can emit a single row.** Unlike a simple `SUM() OVER ()` running total that can emit as it streams, these three need to know the **total row count of the partition** (and for CUME_DIST, the count of peers) before they can compute any value. NTILE needs N to divide into buckets; PERCENT_RANK needs (N−1) as its denominator; CUME_DIST needs N as its denominator. That means the engine buffers the entire partition.

Concretely, the planner produces:

```
WindowAgg
  └── Sort   (Sort Key: <ORDER BY of the window>)
        └── Seq Scan / Index Scan
```

The `Sort` node is where the money is spent. If the sort spills `work_mem`, you get an external merge sort on disk — the dominant cost of these functions at scale. If an index already provides the required order, the planner can skip the Sort node entirely and stream from an `Index Scan`, but the `WindowAgg` still buffers the partition to know its size.

Internally the `WindowAgg` node maintains **partition boundaries** and **peer groups**. A "peer group" is a maximal run of rows that are equal under the window's `ORDER BY`. This peer-group concept is what makes RANK, PERCENT_RANK, and CUME_DIST assign the same value to tied rows. NTILE is different — it does **not** respect peer groups; it will happily split tied rows across two different buckets because it counts positions, not values. That asymmetry is a frequent source of production bugs.

There is no B-tree, MVCC, or WAL subtlety unique to these functions beyond what any read incurs — the internal story is entirely about **sorting, partition buffering, and peer-group counting** inside the `WindowAgg` executor node (`nodeWindowAgg.c` in the PostgreSQL source, if you're curious).

---

## 3. Logical Execution Order Context

```
FROM
JOIN
WHERE
GROUP BY
HAVING
SELECT
  ├── window functions (NTILE / PERCENT_RANK / CUME_DIST) run HERE
DISTINCT
ORDER BY
LIMIT
```

Window functions execute **after** `GROUP BY`, `HAVING`, and the non-window parts of `SELECT`, but **before** `DISTINCT`, the final `ORDER BY`, and `LIMIT`. This ordering has three concrete consequences that trip people up constantly:

1. **You cannot reference a window function in `WHERE` or `HAVING`.** By the time NTILE runs, WHERE and HAVING have already been evaluated. To filter on the output of NTILE (e.g. "give me only bucket 4"), you must wrap the query in a subquery or CTE and filter in the outer query. This is the single most common beginner error with these functions.

2. **`WHERE` filtering changes the input population, which changes every bucket and percentile.** Because the window sees only the rows that survive WHERE, adding `WHERE status = 'active'` silently redefines what "the top quartile" means. The quartile is relative to the filtered set, not the whole table.

3. **The window's own `ORDER BY` is independent of the query's final `ORDER BY`.** `NTILE(4) OVER (ORDER BY score DESC)` computes buckets by descending score, but you can still `ORDER BY name` at the end. The two orderings serve different purposes and don't have to match.

Remember from Topic 34 that the window's `PARTITION BY` and `ORDER BY` live inside the `OVER()` clause and are evaluated as part of this window phase — not the query-level clauses of the same name.

---

## 4. What Is NTILE / PERCENT_RANK / CUME_DIST?

These are three **ranking-family window functions** that map ordered rows to bucket numbers or to a [0,1] distribution position. NTILE divides rows into a specified number of roughly-equal groups. PERCENT_RANK gives the relative rank as a fraction from 0 to 1. CUME_DIST gives the cumulative distribution — the fraction of rows at or before the current row.

```sql
NTILE(num_buckets) OVER (
  [PARTITION BY partition_expr]     -- optional: reset buckets per group
  ORDER BY sort_expr                -- REQUIRED: defines row sequence
)
--    │           │           │
--    │           │           └── ORDER BY is MANDATORY (unlike SUM/COUNT which allow no ORDER BY)
--    │           └────────────── optional partition; each partition numbered 1..n independently
--    └────────────────────────── integer > 0; number of buckets to divide rows into
--                                 returns integer bucket number 1..num_buckets
```

```sql
PERCENT_RANK() OVER (
  [PARTITION BY partition_expr]
  ORDER BY sort_expr                -- REQUIRED
)
--    │
--    └── takes NO arguments (empty parens)
--        returns double precision in [0, 1]
--        formula: (rank - 1) / (total_rows - 1)
--        first row is ALWAYS 0.0; last distinct value is ALWAYS 1.0
--        single-row partition returns 0 (division by zero avoided: defined as 0)
```

```sql
CUME_DIST() OVER (
  [PARTITION BY partition_expr]
  ORDER BY sort_expr                -- REQUIRED
)
--    │
--    └── takes NO arguments (empty parens)
--        returns double precision in (0, 1]
--        formula: (rows with value <= current) / (total_rows)
--        NEVER returns 0; last row is ALWAYS 1.0
--        tied rows all get the SAME (highest) value in their peer group
```

The three formulas, side by side, for a partition of `N` rows where the current row has `RANK() = r` (1-based, ties share the lowest rank) and where `k` = number of rows whose ORDER BY value is ≤ the current row's value:

| Function | Formula | Range | First row | Last row | Respects ties? |
|----------|---------|-------|-----------|----------|----------------|
| `NTILE(b)` | position-based split into `b` groups | `1..b` (int) | `1` | `b` | No (splits peers) |
| `PERCENT_RANK()` | `(r - 1) / (N - 1)` | `[0, 1]` | `0.0` | `1.0` | Yes (via `r`) |
| `CUME_DIST()` | `k / N` | `(0, 1]` | `1/N` or more | `1.0` | Yes (via `k`) |

---

## 5. Why NTILE / PERCENT_RANK / CUME_DIST Mastery Matters in Production

1. **Every analytics dashboard uses buckets.** Quartiles, deciles, percentile bands, cohort tiers, "top 10% of customers by spend" — these are the bread and butter of reporting. NTILE is how you assign a customer to a decile in one pass. Doing it wrong (e.g. by hand with `COUNT` subqueries) is both slower and buggier.

2. **Uneven bucket handling causes silent off-by-one reporting errors.** When 103 customers get split into 4 quartiles, the buckets are 26/26/26/25 — not 25.75 each. If your business logic assumes equal buckets, your "top 25%" actually contains a different count than your "bottom 25%," and finance notices when the numbers don't reconcile.

3. **NTILE splitting tied values across buckets is a correctness landmine.** If ten customers all spent exactly $500 and the bucket boundary falls in the middle of them, five land in "high tier" and five in "medium tier" — identical customers, different treatment. This is indefensible in credit scoring, pricing, or any regulated context. Knowing this forces you to either use CUME_DIST/PERCENT_RANK (which keep ties together) or add a deterministic tiebreaker.

4. **PERCENT_RANK vs CUME_DIST confusion produces wrong percentiles.** They differ at the endpoints and in tie handling. Reporting "this transaction is at the 100th percentile" (CUME_DIST for the max) vs "at the 99.7th percentile" (PERCENT_RANK) is a materially different statement in risk and SLA reporting. Picking the wrong one misrepresents the data.

5. **The ordered-set aggregates `percentile_cont` / `percentile_disc` are the inverse operation and are frequently the *right* tool instead.** People reach for NTILE/CUME_DIST when they actually want "what is the median" or "what is the 95th-percentile latency" — which is `percentile_cont`. Confusing the two leads to convoluted, slow queries. A principal engineer knows which direction each tool runs.

6. **Cost at scale.** These functions force a full partition sort. On a 100M-row table with no supporting index, that's a multi-gigabyte external sort. Understanding that lets you pre-index, pre-aggregate, or push the computation to a materialized view rather than recomputing deciles on every dashboard load.

---

## 6. Deep Technical Content

### 6.1 NTILE — The Uneven Bucket Algorithm (Exact Rules)

`NTILE(b)` divides `N` rows into `b` buckets. When `N` is not evenly divisible by `b`, the remainder rows are distributed **one each to the earliest buckets**. The exact algorithm PostgreSQL uses:

- Let `N` = total rows in the partition, `b` = number of buckets.
- Base bucket size = `floor(N / b)`.
- Remainder = `N mod b`.
- The **first `remainder` buckets** each get `floor(N/b) + 1` rows.
- The **remaining `b − remainder` buckets** each get `floor(N/b)` rows.

Worked example — 11 rows, 4 buckets:

```
N = 11, b = 4
floor(11/4) = 2, remainder = 11 mod 4 = 3
First 3 buckets get 2+1 = 3 rows each  → buckets 1,2,3 have 3 rows
Last 1 bucket gets 2 rows              → bucket 4 has 2 rows
Distribution: 3, 3, 3, 2  (sums to 11 ✓)
```

```sql
SELECT
  val,
  NTILE(4) OVER (ORDER BY val) AS bucket
FROM (VALUES (10),(20),(30),(40),(50),(60),(70),(80),(90),(100),(110)) AS t(val);
```

```
 val | bucket
-----+--------
  10 |   1
  20 |   1
  30 |   1      ← bucket 1: 3 rows
  40 |   2
  50 |   2
  60 |   2      ← bucket 2: 3 rows
  70 |   3
  80 |   3
  90 |   3      ← bucket 3: 3 rows
 100 |   4
 110 |   4      ← bucket 4: 2 rows (the "short" bucket)
```

**The extra rows always go to the front.** There is no way to change this with a flag; if you need the extra in the back, reverse the `ORDER BY` and remap bucket numbers (`b + 1 - bucket`).

**Edge case — more buckets than rows.** If `b > N`, you get `N` buckets each with one row, and buckets `N+1 .. b` simply don't exist (no rows are assigned to them). `NTILE(10)` over 3 rows produces buckets 1, 2, 3 — not 1..10.

```sql
SELECT val, NTILE(10) OVER (ORDER BY val) AS bucket
FROM (VALUES (5),(15),(25)) AS t(val);
-- 5→1, 15→2, 25→3.  Buckets 4..10 are empty.
```

**Edge case — NTILE(1).** Everything goes in bucket 1. Useless but legal.

**Edge case — NTILE(0) or NTILE(negative).** Errors: `ERROR: argument of ntile must be greater than zero`.

**Edge case — NTILE(NULL).** Returns NULL for every row (NULL argument propagates).

### 6.2 NTILE Ignores Ties — The Split-Peer Problem

This is the defining gotcha. NTILE counts **positions**, not distinct values. If tied rows straddle a bucket boundary, they land in different buckets.

```sql
SELECT
  spend,
  NTILE(2) OVER (ORDER BY spend) AS half
FROM (VALUES (100),(100),(100),(100)) AS t(spend);
```

```
 spend | half
-------+------
   100 |  1
   100 |  1
   100 |  2     ← identical value, different bucket!
   100 |  2
```

Four identical customers, split 2/2 across "bottom half" and "top half." NTILE gives you no way to keep them together. If keeping peers together matters, you must use `CUME_DIST` or `PERCENT_RANK` (which are value-based and assign identical values to peers), or accept an arbitrary-but-deterministic tiebreaker in the `ORDER BY` (e.g. `ORDER BY spend, id`).

### 6.3 NTILE Requires ORDER BY

Unlike `SUM() OVER ()` or `COUNT() OVER ()` which are legal without an `ORDER BY` (they then treat the whole partition as one frame), NTILE, PERCENT_RANK, and CUME_DIST are **rank-family** functions and **require** an `ORDER BY` inside `OVER()`. Omitting it is an error:

```sql
SELECT NTILE(4) OVER () FROM users;
-- ERROR: window function ntile requires an ORDER BY clause
```

Actually, PostgreSQL is lenient here for NTILE in some versions and will assign buckets in an arbitrary physical order if ORDER BY is omitted — **never rely on this**. Semantically, buckets without an ORDER BY are meaningless. Always specify the order.

### 6.4 PERCENT_RANK — Formula and Endpoints

```
PERCENT_RANK() = (RANK() - 1) / (total_rows - 1)
```

Because it's built on `RANK()`, it inherits RANK's tie behavior: tied rows share the same (lowest) rank, so they share the same PERCENT_RANK. The first row (RANK 1) always yields `(1-1)/(N-1) = 0`. The last distinct value always yields `1.0`.

```sql
SELECT
  score,
  RANK()         OVER (ORDER BY score) AS rnk,
  PERCENT_RANK() OVER (ORDER BY score) AS pct_rank
FROM (VALUES (50),(60),(60),(70),(90)) AS t(score);
```

```
 score | rnk | pct_rank
-------+-----+----------
    50 |  1  | 0.0        ← (1-1)/(5-1) = 0/4
    60 |  2  | 0.25       ← (2-1)/(5-1) = 1/4
    60 |  2  | 0.25       ← tied → same rank → same pct_rank
    70 |  4  | 0.75       ← (4-1)/(5-1) = 3/4  (rank jumps 2→4, skipping 3)
    90 |  5  | 1.0        ← (5-1)/(5-1) = 4/4
```

Note rank skipped 3 because of the tie at rank 2. PERCENT_RANK reflects that gap.

**Single-row partition:** the denominator `(N−1)` would be zero. PostgreSQL defines PERCENT_RANK of a lone row as `0` (not an error, not NULL).

```sql
SELECT PERCENT_RANK() OVER (ORDER BY x) FROM (VALUES (42)) t(x);
-- returns 0
```

### 6.5 CUME_DIST — Formula and Endpoints

```
CUME_DIST() = (count of rows with value <= current value) / (total_rows)
```

CUME_DIST is peer-aware in a subtly different way: for a group of tied rows, the "count of rows ≤ current" includes **all** the peers, so every tied row gets the value computed from the *last* member of the peer group.

```sql
SELECT
  score,
  CUME_DIST() OVER (ORDER BY score) AS cume
FROM (VALUES (50),(60),(60),(70),(90)) AS t(score);
```

```
 score | cume
-------+------
    50 |  0.2    ← 1 row ≤ 50, out of 5 → 1/5
    60 |  0.6    ← 3 rows ≤ 60 (the 50 and both 60s) → 3/5
    60 |  0.6    ← same peer group → same value
    70 |  0.8    ← 4 rows ≤ 70 → 4/5
    90 |  1.0    ← all 5 rows ≤ 90 → 5/5
```

**CUME_DIST never returns 0** — even the smallest value has at least itself at-or-below, so the minimum possible is `1/N`. **CUME_DIST of the last row is always exactly 1.0.**

**Single-row partition:** `1/1 = 1.0`.

### 6.6 PERCENT_RANK vs CUME_DIST — The Full Comparison

They both produce [0,1]-ish numbers over the same ordered input but differ in three ways:

| Property | PERCENT_RANK | CUME_DIST |
|----------|--------------|-----------|
| Formula | `(rank − 1)/(N − 1)` | `(rows ≤ current)/N` |
| First row value | always `0.0` | `1/N` (or higher if ties) |
| Last row value | always `1.0` | always `1.0` |
| Can be 0? | Yes (first row) | Never |
| Interpretation | "% of rows strictly ranked below me" | "% of rows at-or-below me" |
| Denominator | `N − 1` | `N` |

Side-by-side on the same data:

```sql
SELECT
  score,
  ROUND(PERCENT_RANK() OVER (ORDER BY score)::numeric, 3) AS pct_rank,
  ROUND(CUME_DIST()    OVER (ORDER BY score)::numeric, 3) AS cume_dist
FROM (VALUES (50),(60),(60),(70),(90)) AS t(score);
```

```
 score | pct_rank | cume_dist
-------+----------+-----------
    50 |  0.000   |  0.200
    60 |  0.250   |  0.600
    60 |  0.250   |  0.600
    70 |  0.750   |  0.800
    90 |  1.000   |  1.000
```

**When to use which:**
- Use **PERCENT_RANK** when you want a symmetric 0-to-1 scale where the extremes are pinned (e.g. normalizing scores to a 0–1 range, or "this student is in the top 5% = pct_rank ≥ 0.95").
- Use **CUME_DIST** when you want a true empirical cumulative distribution — "what fraction of transactions were ≤ this amount" — which is the statistically standard ECDF and is what most "percentile" business definitions actually mean.

### 6.7 The Ordered-Set Aggregates: percentile_cont and percentile_disc

These are the **inverse** operation and are constantly confused with the above. NTILE/CUME_DIST take a value and tell you its percentile position. `percentile_cont(p)` and `percentile_disc(p)` take a percentile `p` and tell you the **value** at that position. They are aggregates (they collapse rows), used with `WITHIN GROUP (ORDER BY ...)`.

```sql
-- "What is the median and p95 order amount?"
SELECT
  percentile_cont(0.5)  WITHIN GROUP (ORDER BY total_amount) AS median,
  percentile_cont(0.95) WITHIN GROUP (ORDER BY total_amount) AS p95_interpolated,
  percentile_disc(0.95) WITHIN GROUP (ORDER BY total_amount) AS p95_actual_value
FROM orders
WHERE status = 'completed';
```

- `percentile_cont(p)` — **continuous**: interpolates between the two nearest rows. The result may be a value that does not exist in the data (e.g. median of {10, 20} is 15). Returns `double precision` / `interval`.
- `percentile_disc(p)` — **discrete**: returns an actual value from the data set — the first value whose cumulative distribution ≥ p. No interpolation. Median of {10, 20} is 10.

**They can also be used as window functions** (ordered-set aggregates gained `OVER()` support), but the far more common form is the aggregate `WITHIN GROUP` form.

The relationship: `percentile_disc(p)` returns the value where `CUME_DIST() >= p` first becomes true. They are two sides of the same coin.

| You want... | Use |
|-------------|-----|
| Assign each row to decile 1–10 | `NTILE(10)` |
| Each row's percentile position (0–1) | `CUME_DIST()` or `PERCENT_RANK()` |
| The value at the 95th percentile | `percentile_cont(0.95) WITHIN GROUP (...)` |
| The median value | `percentile_cont(0.5) WITHIN GROUP (...)` |
| A real data value at p95 (no interpolation) | `percentile_disc(0.95) WITHIN GROUP (...)` |

### 6.8 PARTITION BY — Per-Group Buckets

All three reset per partition. `NTILE(4) OVER (PARTITION BY region ORDER BY sales)` computes quartiles **within each region independently**.

```sql
SELECT
  region,
  sales_rep,
  sales,
  NTILE(4) OVER (PARTITION BY region ORDER BY sales DESC) AS region_quartile
FROM sales_summary;
-- Each region has its own quartiles 1..4, independent of other regions.
```

This is how you build "top quartile within each cohort" reports. Without `PARTITION BY`, the quartile is global across the whole table.

### 6.9 Frame Clauses Are Ignored

You cannot put a `ROWS BETWEEN` / `RANGE BETWEEN` frame on NTILE, PERCENT_RANK, or CUME_DIST. They operate on the **whole partition** by definition — a frame makes no sense for them. PostgreSQL raises an error:

```sql
SELECT NTILE(4) OVER (ORDER BY x ROWS BETWEEN 1 PRECEDING AND CURRENT ROW) FROM t;
-- ERROR: window function ntile cannot have a frame specification
```

This is different from aggregate window functions (SUM, AVG) where frames are central. Rank-family functions are frame-immune.

### 6.10 NULL Ordering

`NULL` values in the `ORDER BY` column are ordered per the `NULLS FIRST` / `NULLS LAST` rule (default `NULLS LAST` for `ASC`, `NULLS FIRST` for `DESC`). They participate in the ordering and get bucket/percentile assignments like any other value — NULLs are **not** excluded (unlike, say, an aggregate `AVG` which skips NULLs). All NULLs are considered equal to each other for peer-grouping purposes.

```sql
SELECT
  x,
  NTILE(2)       OVER (ORDER BY x NULLS LAST) AS bucket,
  CUME_DIST()    OVER (ORDER BY x NULLS LAST) AS cume
FROM (VALUES (10),(20),(NULL),(NULL)) AS t(x);
```

```
  x   | bucket | cume
------+--------+------
  10  |   1    | 0.25
  20  |   1    | 0.50
 NULL |   2    | 1.00   ← both NULLs are peers → both get 1.0 under CUME_DIST
 NULL |   2    | 1.00
```

### 6.11 Building "Top N%" Filters Correctly

The idiomatic way to select the top 10% by some metric:

```sql
-- Top 10% of customers by lifetime spend
SELECT customer_id, lifetime_spend
FROM (
  SELECT
    customer_id,
    lifetime_spend,
    CUME_DIST() OVER (ORDER BY lifetime_spend) AS cd
  FROM customer_ltv
) ranked
WHERE cd >= 0.90;   -- top 10% = cumulative distribution in the top decile
```

Note you must use a subquery — you cannot filter `CUME_DIST()` in `WHERE` directly. Also note the semantic choice: `CUME_DIST() >= 0.90` gives you the customers whose value is in the top 10% *by distribution*, which naturally keeps tied customers together. Using `NTILE(10) ... WHERE decile = 10` would give you approximately the top 10% but could split ties.

---

## 7. EXPLAIN — WindowAgg + Sort in the Plan

### Basic NTILE plan (no supporting index)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT
  customer_id,
  lifetime_spend,
  NTILE(4) OVER (ORDER BY lifetime_spend DESC) AS quartile
FROM customer_ltv;
```

```
WindowAgg  (cost=11240.00..13115.00 rows=100000 width=20)
           (actual time=142.1..201.3 rows=100000 loops=1)
  ->  Sort  (cost=11240.00..11490.00 rows=100000 width=12)
            (actual time=142.0..158.7 rows=100000 loops=1)
        Sort Key: lifetime_spend DESC
        Sort Method: quicksort  Memory: 8544kB
        ->  Seq Scan on customer_ltv
              (cost=0.00..1735.00 rows=100000 width=12)
              (actual time=0.01..18.4 rows=100000 loops=1)
Buffers: shared hit=735
Planning Time: 0.2 ms
Execution Time: 208.9 ms
```

**Reading it:**
- `Seq Scan` reads all 100K rows (no WHERE, so nothing to filter).
- `Sort` orders by `lifetime_spend DESC`. `Sort Method: quicksort  Memory: 8544kB` means the sort fit in `work_mem` — no disk spill. This is the dominant cost (142ms just to reach the sort's first output).
- `WindowAgg` sits on top, buffers the partition (here the whole table is one partition), computes bucket numbers, and emits rows. Its incremental cost over the sort is small.
- No index was usable, so the full sort was mandatory.

### Same query with a supporting index

```sql
CREATE INDEX idx_ltv_spend_desc ON customer_ltv (lifetime_spend DESC);

EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id, lifetime_spend,
       NTILE(4) OVER (ORDER BY lifetime_spend DESC) AS quartile
FROM customer_ltv;
```

```
WindowAgg  (cost=0.42..4863.42 rows=100000 width=20)
           (actual time=0.05..78.2 rows=100000 loops=1)
  ->  Index Scan using idx_ltv_spend_desc on customer_ltv
        (cost=0.42..3113.42 rows=100000 width=12)
        (actual time=0.03..41.1 rows=100000 loops=1)
Buffers: shared hit=100342
Planning Time: 0.3 ms
Execution Time: 84.6 ms
```

**Reading it:**
- **The Sort node is gone.** The index already provides `lifetime_spend DESC` order, so `WindowAgg` streams directly from the `Index Scan`. Execution dropped from 209ms to 85ms.
- Buffers went *up* (`shared hit=100342`) because the Index Scan does a heap fetch per row (this table isn't index-only-able since we select `customer_id`). Trade-off: more buffer touches, but no expensive sort. For a covering index (`INCLUDE (customer_id)`) you'd get an Index Only Scan and far fewer buffers.
- `WindowAgg` still buffers the partition to know N for the bucket math — but buffering an already-ordered stream is cheap compared to sorting.

### Partitioned window with spill to disk

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT region, sales_rep,
       CUME_DIST() OVER (PARTITION BY region ORDER BY sales DESC) AS cd
FROM sales_summary;   -- 10M rows, no index
```

```
WindowAgg  (cost=1508234..1633234 rows=10000000 width=28)
           (actual time=8421..14300 rows=10000000 loops=1)
  ->  Sort  (cost=1508234..1533234 rows=10000000 width=20)
            (actual time=8420..11200 rows=10000000 loops=1)
        Sort Key: region, sales DESC
        Sort Method: external merge  Disk: 254880kB
        ->  Seq Scan on sales_summary
              (actual time=0.02..1980 rows=10000000 loops=1)
Buffers: shared hit=3120 read=88214, temp read=31860 written=31860
Planning Time: 0.4 ms
Execution Time: 15102 ms
```

**Reading it — the danger signs:**
- `Sort Method: external merge  Disk: 254880kB` — the sort **spilled 250MB to disk** because it exceeded `work_mem`. This is the killer: `temp read=31860 written=31860` shows temp-file I/O.
- `Sort Key: region, sales DESC` — note the partition key `region` leads the sort key, then the ORDER BY. The window sorts by partition-then-order.
- 15 seconds. Fixes: raise `work_mem`, add an index on `(region, sales DESC)` to eliminate the sort, or pre-aggregate.

### EXPLAIN signals cheat-sheet

| Signal in plan | Meaning | Fix |
|----------------|---------|-----|
| `Sort` node under `WindowAgg` | No index provides window order | Add index on `(partition_cols, order_cols)` |
| `Sort Method: external merge  Disk: N` | Sort spilled to disk | Raise `work_mem` or add index |
| No `Sort` node, `Index Scan` feeds `WindowAgg` | Order came free from index | Ideal — nothing to do |
| `WindowAgg` with high actual time vs Sort | Rare; usually sort dominates | Check for expensive per-row exprs |
| `temp read/written` in Buffers | Disk spill confirmed | Same as external merge fix |

---

## 8. Query Examples

### Example 1 — Basic: Quartiles of Order Value

```sql
-- Assign every completed order to a value quartile (1 = cheapest 25%, 4 = priciest 25%)
SELECT
  o.id                                              AS order_id,
  o.total_amount,
  NTILE(4) OVER (ORDER BY o.total_amount) AS value_quartile
    -- │        │
    -- │        └── order all orders ascending by amount
    -- └── split into 4 equal-sized buckets; extras go to buckets 1,2,3
FROM orders o
WHERE o.status = 'completed'
ORDER BY o.total_amount;
```

### Example 2 — Intermediate: Per-Category Percentile of Product Price

```sql
-- For each product, where does its price sit within its category's distribution?
-- Uses CUME_DIST so ties (same price) share a percentile and the max is exactly 1.0.
SELECT
  p.id                                                          AS product_id,
  p.name                                                        AS product_name,
  c.name                                                        AS category,
  p.price,
  ROUND(
    CUME_DIST() OVER (PARTITION BY p.category_id ORDER BY p.price)::numeric
  , 4)                                                          AS price_percentile,
  NTILE(10)  OVER (PARTITION BY p.category_id ORDER BY p.price) AS price_decile
    -- both windows PARTITION BY category → each category computed independently
FROM products p
INNER JOIN categories c ON c.id = p.category_id
ORDER BY c.name, p.price DESC;
```

### Example 3 — Production Grade: Customer RFM-Style Decile Cohorts

```sql
-- Segment customers into spend deciles and recency quartiles for a marketing cohort export.
-- Table sizes: customers ~2M rows, orders ~40M rows.
-- Indexes assumed: orders(customer_id, created_at), orders(status).
-- Perf expectation: ~2-4s on a warm cache; dominated by the pre-aggregation over orders,
--   not by the window functions themselves (which operate on the 2M-row aggregate).

WITH customer_stats AS (
  -- Pre-aggregate first so the window functions run over 2M rows, not 40M.
  SELECT
    o.customer_id,
    SUM(o.total_amount)                              AS lifetime_spend,
    COUNT(*)                                         AS order_count,
    MAX(o.created_at)                                AS last_order_at
  FROM orders o
  WHERE o.status = 'completed'
  GROUP BY o.customer_id
),
scored AS (
  SELECT
    cs.customer_id,
    cs.lifetime_spend,
    cs.order_count,
    cs.last_order_at,
    NTILE(10) OVER (ORDER BY cs.lifetime_spend DESC)  AS spend_decile,   -- 1 = top spenders
    NTILE(4)  OVER (ORDER BY cs.last_order_at   DESC) AS recency_quartile,-- 1 = most recent
    ROUND(
      CUME_DIST() OVER (ORDER BY cs.lifetime_spend)::numeric
    , 4)                                              AS spend_cume_dist
  FROM customer_stats cs
)
SELECT
  s.customer_id,
  u.email,
  s.lifetime_spend,
  s.order_count,
  s.last_order_at,
  s.spend_decile,
  s.recency_quartile,
  s.spend_cume_dist,
  CASE
    WHEN s.spend_decile = 1 AND s.recency_quartile = 1 THEN 'champion'
    WHEN s.spend_decile <= 3 AND s.recency_quartile <= 2 THEN 'loyal'
    WHEN s.recency_quartile >= 4                        THEN 'at_risk'
    ELSE 'standard'
  END                                                 AS segment
FROM scored s
INNER JOIN users u ON u.id = s.customer_id
ORDER BY s.spend_decile, s.recency_quartile;
```

EXPLAIN sketch for Example 3:

```
Sort (final ORDER BY spend_decile, recency_quartile)
  └── Hash Join  (scored ⋈ users on u.id = s.customer_id)
        ├── Subquery Scan scored
        │     └── WindowAgg (CUME_DIST over spend)
        │           └── WindowAgg (NTILE recency_quartile)
        │                 └── Sort (last_order_at DESC)
        │                       └── WindowAgg (NTILE spend_decile)
        │                             └── Sort (lifetime_spend DESC)
        │                                   └── Subquery Scan customer_stats
        │                                         └── HashAggregate (GROUP BY customer_id)
        │                                               └── Index Scan orders (status='completed')
        └── Seq Scan / Index Scan users
```

Key insight: **three window functions with three different ORDER BYs mean up to three Sort nodes** stacked in the plan. Each distinct window ordering that no index satisfies adds a sort. If two windows share the same ORDER BY, the planner reuses one sort.

---

## 9. Wrong → Right Patterns

### Wrong 1: Filtering a window function in WHERE

```sql
-- WRONG: window functions cannot be referenced in WHERE
SELECT customer_id, lifetime_spend,
       NTILE(10) OVER (ORDER BY lifetime_spend DESC) AS decile
FROM customer_ltv
WHERE NTILE(10) OVER (ORDER BY lifetime_spend DESC) = 1;
-- ERROR: window functions are not allowed in WHERE
--   LINE ...: WHERE NTILE(10) OVER (ORDER BY lifetime_spend DESC) = 1
```

**Why it's wrong at the execution level:** WHERE runs in the logical order *before* the window phase (Section 3). At the moment WHERE is evaluated, the NTILE value does not yet exist. There is no way for the engine to filter on a value it hasn't computed.

```sql
-- RIGHT: compute the window in a subquery/CTE, filter in the outer query
SELECT customer_id, lifetime_spend, decile
FROM (
  SELECT customer_id, lifetime_spend,
         NTILE(10) OVER (ORDER BY lifetime_spend DESC) AS decile
  FROM customer_ltv
) t
WHERE decile = 1;   -- top decile
```

### Wrong 2: Assuming NTILE keeps tied values together

```sql
-- WRONG assumption: "customers with equal spend get the same tier"
SELECT customer_id, spend,
       NTILE(4) OVER (ORDER BY spend) AS tier
FROM customer_ltv;
-- If many customers share spend = 500 and the boundary falls among them,
-- identical customers land in tier 2 vs tier 3. Indefensible for pricing/credit.
```

**Why it's wrong:** NTILE counts positions, not values (Section 6.2). It has no peer-group awareness.

```sql
-- RIGHT option A: use CUME_DIST (value-based, keeps peers together), then band it
SELECT customer_id, spend,
       CASE
         WHEN cd <= 0.25 THEN 1 WHEN cd <= 0.50 THEN 2
         WHEN cd <= 0.75 THEN 3 ELSE 4
       END AS tier
FROM (
  SELECT customer_id, spend,
         CUME_DIST() OVER (ORDER BY spend) AS cd
  FROM customer_ltv
) t;
-- All customers with spend=500 share the same cd → same tier. No split.

-- RIGHT option B: if you must use NTILE, add a deterministic tiebreaker so the
-- split is at least reproducible (does NOT keep peers together, but is stable):
SELECT customer_id, spend,
       NTILE(4) OVER (ORDER BY spend, customer_id) AS tier
FROM customer_ltv;
```

### Wrong 3: Confusing PERCENT_RANK with CUME_DIST for "percentile"

```sql
-- WRONG: reporting "this order is at the Nth percentile" using PERCENT_RANK
-- when the business definition is "% of orders at or below this amount" (ECDF).
SELECT id, total_amount,
       PERCENT_RANK() OVER (ORDER BY total_amount) AS "percentile"
FROM orders;
-- The cheapest order reports percentile 0.0 — but it is NOT true that 0% of
-- orders are ≤ the cheapest order (the cheapest order itself is ≤ itself).
```

**Why it's wrong:** PERCENT_RANK is `(rank−1)/(N−1)` — it measures rows *strictly ranked below*. The standard statistical "percentile / cumulative distribution" is `CUME_DIST` = rows *at or below*. Using PERCENT_RANK understates every value's percentile, most visibly at the low end.

```sql
-- RIGHT: use CUME_DIST for a true empirical cumulative distribution
SELECT id, total_amount,
       ROUND(CUME_DIST() OVER (ORDER BY total_amount)::numeric, 4) AS percentile
FROM orders;
-- cheapest order → 1/N (correct: it and nothing-cheaper is that fraction of the data)
```

### Wrong 4: Using NTILE/CUME_DIST when you want a percentile *value*

```sql
-- WRONG: convoluted attempt to find the p95 order amount using window functions
SELECT total_amount
FROM (
  SELECT total_amount, CUME_DIST() OVER (ORDER BY total_amount) AS cd
  FROM orders WHERE status='completed'
) t
WHERE cd >= 0.95
ORDER BY total_amount
LIMIT 1;
-- Works, but sorts the whole table, materializes a subquery, and is fragile.
```

**Why it's suboptimal:** you're using the inverse tool. You want a *value at a percentile*, which is exactly what the ordered-set aggregate computes in one shot.

```sql
-- RIGHT: the ordered-set aggregate — one row, no subquery, clear intent
SELECT
  percentile_cont(0.95) WITHIN GROUP (ORDER BY total_amount) AS p95_interpolated,
  percentile_disc(0.95) WITHIN GROUP (ORDER BY total_amount) AS p95_actual
FROM orders
WHERE status = 'completed';
```

### Wrong 5: NTILE over an unfiltered/unindexed huge table on every request

```sql
-- WRONG: recomputing deciles across 100M rows on every dashboard hit
SELECT customer_id, NTILE(10) OVER (ORDER BY lifetime_spend DESC) AS decile
FROM customer_ltv;   -- full sort of 100M rows, external merge, 30s+ each call
```

**Why it's wrong:** every call pays for a full external sort (Section 7). Deciles change slowly; recomputing them per request is wasteful.

```sql
-- RIGHT: materialize the deciles, refresh on a schedule
CREATE MATERIALIZED VIEW customer_deciles AS
SELECT customer_id, lifetime_spend,
       NTILE(10) OVER (ORDER BY lifetime_spend DESC) AS decile
FROM customer_ltv;
CREATE UNIQUE INDEX ON customer_deciles (customer_id);
-- REFRESH MATERIALIZED VIEW CONCURRENTLY customer_deciles;  (nightly cron)
-- Dashboard reads are now an index lookup, not a 100M-row sort.
```

---

## 10. Performance Profile

### The dominant cost is always the sort

For all three functions, the CPU and memory profile is: **one sort of the partition, then a cheap linear pass** by the `WindowAgg`. There is no hash table (unlike Hash Join), no random I/O beyond the scan. So the performance question reduces entirely to "how expensive is the sort?"

| Rows | Sort in `work_mem`? | Typical NTILE/CUME_DIST time | Note |
|------|--------------------|------------------------------|------|
| 10K | Yes (KBs) | < 5ms | Trivial |
| 100K | Yes (~8MB) | 80–210ms | Fits default work_mem (4MB may spill) |
| 1M | Borderline | 0.8–2s | Likely spills at default work_mem=4MB |
| 10M | No (250MB+ temp) | 12–20s | External merge sort dominates |
| 100M | No (GBs temp) | minutes | Must pre-aggregate or materialize |

### Memory

- `WindowAgg` buffers the current partition to know N. For a single global partition, that's the whole result set held in a tuplestore — which itself can spill to `temp` if large. This is *separate* from the sort spill and also governed by `work_mem`.
- Practical implication: a global (no PARTITION BY) window over 50M rows holds 50M tuples in a tuplestore **and** sorts them. Partitioning into many small groups keeps each buffered partition small.

### CPU

- Sort is `O(N log N)` comparisons. NTILE/PERCENT_RANK/CUME_DIST add `O(N)` on top (one or two linear passes to count peer groups and assign values). The window overhead is negligible next to the sort.

### Index interactions — the one real optimization

An index on `(partition_cols..., order_cols...)` in matching direction lets the planner **skip the Sort node entirely** (Section 7, second plan). This is the single highest-leverage optimization:

```sql
-- For: NTILE(4) OVER (PARTITION BY region ORDER BY sales DESC)
CREATE INDEX idx_ss_region_sales ON sales_summary (region, sales DESC);
-- Planner can now feed WindowAgg from an ordered Index Scan → no sort.
```

Caveats:
- The index order must match the window order *exactly*, including ASC/DESC and NULLS FIRST/LAST, and the partition columns must lead.
- If you `SELECT` columns not in the index, you get heap fetches (more buffers). Use `INCLUDE (extra_cols)` for a covering index → Index Only Scan.
- Multiple windows with different orderings can't all be satisfied by one index; the planner sorts for the others.

### Other optimization techniques specific to these functions

1. **Pre-aggregate before windowing.** As in Example 3, running deciles over a 2M-row aggregate instead of a 40M-row raw table cuts the sort input 20×.
2. **Materialized views** for slowly-changing deciles/percentiles (Wrong 5 → Right).
3. **Raise `work_mem`** for the session/query running the window if the sort spills — set it high enough to keep the sort in memory (`SET work_mem = '256MB'` for the transaction).
4. **Push `WHERE` before the window** — filtering the input shrinks the sort. But remember filtering changes what the buckets *mean*.
5. **Avoid redundant windows** — reuse one `OVER (...)` via a named `WINDOW` clause if several columns share a definition:

```sql
SELECT
  NTILE(4)       OVER w AS quartile,
  CUME_DIST()    OVER w AS cd,
  PERCENT_RANK() OVER w AS pr
FROM t
WINDOW w AS (PARTITION BY region ORDER BY sales);
-- All three share ONE sort instead of three.
```

---

## 11. Node.js Integration

### 11.1 Basic NTILE query with pg

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Return orders bucketed into value quartiles
async function getOrderQuartiles() {
  const { rows } = await pool.query(
    `SELECT
       id           AS order_id,
       total_amount,
       NTILE(4) OVER (ORDER BY total_amount) AS value_quartile
     FROM orders
     WHERE status = 'completed'
     ORDER BY total_amount`
  );
  return rows;
}
```

### 11.2 Parameterized "top decile" with filtering in the outer query

```javascript
// $1 = decile cutoff (e.g. 1 for the top decile). Window filter must be in a subquery.
async function getTopDecileCustomers(topDecile = 1) {
  const { rows } = await pool.query(
    `SELECT customer_id, email, lifetime_spend, decile
     FROM (
       SELECT
         cs.customer_id,
         u.email,
         cs.lifetime_spend,
         NTILE(10) OVER (ORDER BY cs.lifetime_spend DESC) AS decile
       FROM customer_ltv cs
       JOIN users u ON u.id = cs.customer_id
     ) ranked
     WHERE decile = $1
     ORDER BY lifetime_spend DESC`,
    [topDecile]
  );
  return rows;
}
```

Note: `NTILE(10)` is hardcoded, **not** a `$1` parameter. You *can* parameterize the bucket count, but be aware that `NTILE($1)` requires the driver to bind an integer, and some pooling/prepared-statement setups infer the type — cast explicitly if needed: `NTILE($1::int)`.

### 11.3 percentile_cont for a latency SLA report

```javascript
// p50/p95/p99 of request latency — the ordered-set aggregate, one round trip
async function getLatencyPercentiles(sinceHours = 24) {
  const { rows } = await pool.query(
    `SELECT
       percentile_cont(0.50) WITHIN GROUP (ORDER BY duration_ms) AS p50,
       percentile_cont(0.95) WITHIN GROUP (ORDER BY duration_ms) AS p95,
       percentile_cont(0.99) WITHIN GROUP (ORDER BY duration_ms) AS p99,
       COUNT(*)                                                  AS sample_size
     FROM request_logs
     WHERE created_at >= NOW() - ($1 || ' hours')::interval`,
    [sinceHours]
  );
  return rows[0];
}
```

### 11.4 Handling numeric types in results

```javascript
// CUME_DIST / PERCENT_RANK return double precision → JS number (safe).
// percentile_cont over a numeric column returns numeric → pg returns a STRING
// by default to preserve precision. Parse if you need arithmetic.
async function getSpendPercentiles() {
  const { rows } = await pool.query(
    `SELECT
       customer_id,
       lifetime_spend,                                   -- numeric → string in JS
       CUME_DIST() OVER (ORDER BY lifetime_spend) AS cd  -- float8 → number in JS
     FROM customer_ltv`
  );
  return rows.map(r => ({
    customerId: r.customer_id,
    lifetimeSpend: Number(r.lifetime_spend),  // string → number
    cumeDist: r.cd,                           // already a number
  }));
}
```

**ORM note:** No mainstream Node ORM has first-class NTILE/CUME_DIST/PERCENT_RANK helpers — window functions in general are weakly supported. In practice you drop to raw SQL (`pool.query`, `$queryRaw`, `knex.raw`) for all three. See Section 12.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do NTILE/PERCENT_RANK/CUME_DIST?** — No, not through the query API. Prisma has no window-function support at all. You must use `$queryRaw`.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

const topDecile = await prisma.$queryRaw`
  SELECT customer_id, lifetime_spend, decile
  FROM (
    SELECT customer_id, lifetime_spend,
           NTILE(10) OVER (ORDER BY lifetime_spend DESC) AS decile
    FROM customer_ltv
  ) t
  WHERE decile = 1
`;
```

**Where it breaks:** everything window-related. `groupBy` and `aggregate` cover simple stats, but there is no equivalent for per-row percentile assignment.

**Verdict:** Raw SQL only. Use `Prisma.sql` for safe interpolation of the bucket count.

---

### Drizzle ORM

**Can Drizzle do it?** — Partially. Drizzle has no dedicated NTILE/CUME_DIST helpers, but its `sql` template makes raw window expressions clean and typed.

```typescript
import { db } from './db';
import { customerLtv } from './schema';
import { sql } from 'drizzle-orm';

const ranked = await db
  .select({
    customerId: customerLtv.customerId,
    lifetimeSpend: customerLtv.lifetimeSpend,
    decile: sql<number>`NTILE(10) OVER (ORDER BY ${customerLtv.lifetimeSpend} DESC)`,
    cumeDist: sql<number>`CUME_DIST() OVER (ORDER BY ${customerLtv.lifetimeSpend})`,
  })
  .from(customerLtv);

// To filter on decile you still need a subquery — wrap with .as() and select again,
// or use a raw sql`` fragment for the whole thing.
```

**Where it breaks:** filtering on the window output requires manually building a subquery (Drizzle's `.$dynamic()`/subquery API) since you can't put the window in `.where()`.

**Verdict:** Best-in-class among ORMs for this — the `sql` tag keeps it readable and typed. Still fundamentally raw SQL under the hood.

---

### Sequelize

**Can Sequelize do it?** — Only via `sequelize.literal()` or raw queries. No native support.

```javascript
const { QueryTypes } = require('sequelize');

const rows = await sequelize.query(
  `SELECT customer_id, lifetime_spend, decile
   FROM (
     SELECT customer_id, lifetime_spend,
            NTILE(10) OVER (ORDER BY lifetime_spend DESC) AS decile
     FROM customer_ltv
   ) t
   WHERE decile = :topDecile`,
  { replacements: { topDecile: 1 }, type: QueryTypes.SELECT }
);

// Or inline via literal (fragile, no subquery filtering):
const { fn, col, literal } = require('sequelize');
const model = await CustomerLtv.findAll({
  attributes: [
    'customer_id',
    'lifetime_spend',
    [literal('NTILE(10) OVER (ORDER BY lifetime_spend DESC)'), 'decile'],
  ],
});
```

**Where it breaks:** `literal()` is an unescaped string — no type safety, and you cannot filter on the alias without a raw subquery.

**Verdict:** Use `sequelize.query()` raw. `literal()` works for read-only projection but not for filtering.

---

### TypeORM

**Can TypeORM do it?** — Via QueryBuilder `addSelect` with a raw expression, or `query()` for full raw.

```typescript
const rows = await dataSource
  .getRepository(CustomerLtv)
  .createQueryBuilder('cs')
  .select('cs.customer_id', 'customerId')
  .addSelect('cs.lifetime_spend', 'lifetimeSpend')
  .addSelect('NTILE(10) OVER (ORDER BY cs.lifetime_spend DESC)', 'decile')
  .getRawMany();

// Filtering on decile → wrap as a subquery source:
const top = await dataSource
  .createQueryBuilder()
  .select('*')
  .from(qb => qb
    .select('cs.customer_id', 'customer_id')
    .addSelect('NTILE(10) OVER (ORDER BY cs.lifetime_spend DESC)', 'decile')
    .from(CustomerLtv, 'cs'), 't')
  .where('t.decile = :d', { d: 1 })
  .getRawMany();
```

**Where it breaks:** the raw expression string in `addSelect` has no type safety, and you must use `getRawMany()` (entity hydration won't map the computed column).

**Verdict:** QueryBuilder subquery-from pattern works and is reasonably clean. Raw string for the window expression.

---

### Knex.js

**Can Knex do it?** — Yes, via `knex.raw()` in `select`, and its subquery builder for filtering.

```javascript
// Projection
const rows = await knex('customer_ltv')
  .select('customer_id', 'lifetime_spend')
  .select(knex.raw('NTILE(10) OVER (ORDER BY lifetime_spend DESC) AS decile'));

// Filter on the window via a subquery
const top = await knex
  .select('*')
  .from(
    knex('customer_ltv')
      .select('customer_id', 'lifetime_spend')
      .select(knex.raw('NTILE(10) OVER (ORDER BY lifetime_spend DESC) AS decile'))
      .as('t')
  )
  .where('t.decile', 1);
```

**Where it breaks:** the window expression is `knex.raw` (string), no structured builder. But the subquery composition is idiomatic and readable.

**Verdict:** Most transparent. `knex.raw` for the window + subquery `.from()` for filtering is the cleanest ORM-side pattern.

---

### ORM Summary Table

| ORM | Native window support | NTILE/CUME_DIST approach | Filter on window output | Verdict |
|-----|----------------------|--------------------------|-------------------------|---------|
| Prisma | None | `$queryRaw` only | Raw subquery | Raw SQL only |
| Drizzle | Via `sql` tag (typed) | `sql<number>\`NTILE(...) OVER (...)\`` | Manual subquery | Best typed option |
| Sequelize | None | `literal()` or `query()` | Raw subquery | Use raw `query()` |
| TypeORM | Raw `addSelect` | `addSelect('NTILE(...) OVER (...)')` | Subquery `.from(qb=>...)` | QueryBuilder subquery |
| Knex | Raw | `knex.raw('NTILE(...) OVER (...)')` | Subquery `.from()` | Most transparent |

**Bottom line:** none of these ORMs treat window functions as first-class. For NTILE/PERCENT_RANK/CUME_DIST, plan on raw SQL. Drizzle and Knex make the raw fragment cleanest.

---

## 13. Practice Exercises

### Exercise 1 — Easy

Given `products(id, name, category_id, price)`:

Write a query that assigns every product to a price **quartile** (1–4) across the entire table, returning `id`, `name`, `price`, and the quartile. Order the output by price ascending.

Then answer in a comment: if the table has 22 products, how many products land in each of the 4 quartiles, and which buckets get the "extra" rows?

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines Topic 20 GROUP BY + Topic 34 ranking)

Given `orders(id, customer_id, total_amount, status, created_at)`:

For each customer's **completed** lifetime spend, produce:
- `customer_id`
- `lifetime_spend` (sum of completed order amounts)
- `spend_decile` — NTILE(10) over lifetime spend, decile 1 = highest spenders
- `spend_percentile` — CUME_DIST over lifetime spend, rounded to 4 decimals
- `rank_position` — dense rank of the customer by lifetime spend (highest = 1)

Only include customers with at least 3 completed orders. Order by lifetime_spend descending.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; naive answer is wrong)

Given `customer_ltv(customer_id, lifetime_spend)` with 5 million rows. Marketing wants the **top 5% of customers by spend** for a VIP campaign. A colleague proposes:

```sql
SELECT customer_id FROM customer_ltv
WHERE NTILE(20) OVER (ORDER BY lifetime_spend DESC) = 1;
```

1. State every reason this query is wrong or dangerous (there are at least two — one is a hard error, one is a correctness/semantic issue around ties).
2. Write a correct query that returns exactly the customers whose spend places them in the top 5% of the distribution, keeping tied customers together (no customer with the same spend as an included customer may be excluded).
3. Explain what index you'd add and whether it eliminates the sort.
4. This runs on every campaign build. Propose a caching strategy.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

You're in an interview. The interviewer says:

> "I have a `request_logs(id, endpoint, duration_ms, created_at)` table. I want, per endpoint, the p95 latency AND I want to flag every individual request that is in the slowest 5% for its endpoint. Write the query. Then tell me the difference between how you'd compute the p95 *value* versus how you'd *flag* a row as being in the slowest 5%."

```sql
-- Write your query here (both parts)
```

---

## 14. Interview Questions

### Q1 — Junior/Mid

**Question:** "What does `NTILE(4)` do, and what happens when the number of rows isn't divisible by 4?"

**Junior answer:** "It splits the rows into 4 groups. If it doesn't divide evenly, the groups are slightly different sizes."

**Principal-engineer answer:** "NTILE(4) divides the ordered partition into 4 buckets numbered 1–4, assigning each row a bucket based on its *position*, not its value. When N isn't divisible by 4, the remainder rows are distributed one each to the **earliest** buckets: for N=11, buckets are sized 3,3,3,2 — the extra rows go to buckets 1, 2, 3. The critical caveat is that NTILE is position-based, so it will **split tied values across buckets** — two rows with identical ORDER BY values can land in different buckets if the boundary falls between them. That makes NTILE unsafe for any scenario where equal inputs must get equal treatment; there I'd use CUME_DIST-based banding instead, or add a deterministic tiebreaker to at least make the split reproducible."

**Interviewer follow-up:** "How would you get the extra rows into the *last* buckets instead of the first?"

*(Expected: reverse the ORDER BY and remap `bucket = b + 1 - NTILE(b) OVER (ORDER BY x DESC)`, or accept it's not directly configurable.)*

---

### Q2 — Mid/Senior

**Question:** "What's the difference between `PERCENT_RANK` and `CUME_DIST`? When would you pick one over the other?"

**Junior answer:** "They both give a number between 0 and 1 showing where a row ranks. They're basically the same thing."

**Principal-engineer answer:** "They differ in formula, endpoints, and tie handling. PERCENT_RANK is `(rank − 1)/(N − 1)` — it measures the fraction of rows *strictly ranked below* the current row, so the first row is always exactly 0.0 and the last is 1.0. CUME_DIST is `(count of rows ≤ current)/N` — the fraction *at or below* — so it never returns 0 (the minimum is 1/N) and the last row is 1.0. For a true empirical cumulative distribution — 'what fraction of orders are ≤ this amount' — CUME_DIST is statistically correct; that's what most business 'percentile' definitions actually mean. PERCENT_RANK is better when you want a normalized 0-to-1 scale with both endpoints pinned, e.g. min-max normalizing a score. The endpoint at the low end is where they visibly diverge: the cheapest row is 0.0 under PERCENT_RANK but 1/N under CUME_DIST."

**Interviewer follow-up:** "Both are window functions that return a row's position. What if I want the *value* at the 95th percentile instead?"

*(Expected: `percentile_cont(0.95) WITHIN GROUP (ORDER BY col)` — the inverse operation, an ordered-set aggregate; distinguish cont/interpolated vs disc/actual-value.)*

---

### Q3 — Senior/Principal

**Question:** "I have a query `SELECT *, NTILE(10) OVER (ORDER BY spend DESC) FROM customer_ltv` on a 50M-row table and it takes 40 seconds. Walk me through why, and how you'd make it fast."

**Junior answer:** "Add an index on spend and it'll be faster."

**Principal-engineer answer:** "The cost is almost entirely a sort. NTILE, like all rank-family window functions, requires the entire partition ordered and buffered before it can assign buckets — it needs N to do the bucket math. With no index providing `spend DESC` order, the planner inserts a Sort node over 50M rows. At default `work_mem` that spills to disk as an external merge sort — you'd see `Sort Method: external merge  Disk: <hundreds of MB>` and `temp read/written` in the buffers. That disk I/O is the 40 seconds.

Fixes, in order of leverage: (1) Add an index on `(spend DESC)` — ideally covering with `INCLUDE (customer_id, ...)` — so the planner feeds the WindowAgg from an ordered Index Only Scan and drops the Sort node entirely. (2) If the projection is wide and a covering index isn't practical, raise `work_mem` for the query so the sort stays in memory — turns external merge into an in-memory quicksort. (3) Deciles change slowly, so the real fix is a materialized view refreshed nightly with a unique index on customer_id; dashboard reads become index lookups instead of a 50M-row sort. (4) If this feeds a larger pipeline, pre-aggregate upstream so the window runs over fewer rows. Note the WindowAgg *also* buffers the partition in a tuplestore independent of the sort — for a single global partition that's 50M tuples held in memory/temp, so partitioning or pre-aggregating helps there too."

**Interviewer follow-up:** "You add the covering index and the Sort disappears, but buffer hits go *up*. Why, and is that bad?"

*(Expected: heap fetches per row if not truly index-only; with INCLUDE it's an Index Only Scan and buffers drop; the trade — eliminating an O(N log N) external sort — is worth more buffer touches; also visibility-map/vacuum affects index-only-scan effectiveness.)*

---

### Q4 — Principal

**Question:** "A regulated pricing system tiers customers into 5 bands by spend using `NTILE(5)`. Legal flagged that two customers with identical spend received different prices. Explain the bug and design a fix that will survive an audit."

**Junior answer:** "Use ROUND or add a tiebreaker to the order by."

**Principal-engineer answer:** "The bug is fundamental to NTILE: it partitions by row *position*, not by value, and has no peer-group awareness. When identical-spend customers straddle a bucket boundary, they split across bands — indefensible when the input (spend) is identical. A tiebreaker like `ORDER BY spend, customer_id` makes the split *deterministic and reproducible*, but it does NOT fix the injustice — identical customers still land in different bands, just predictably.

The correct fix is to band on a **value-based** function that assigns identical values to peers: CUME_DIST or PERCENT_RANK. I'd compute `CUME_DIST() OVER (ORDER BY spend)` and then map ranges (`<=0.2 → band 1`, etc.) with a CASE. Because CUME_DIST gives every tied customer the same value, they always land in the same band — the property legal needs. For an audit trail I'd (1) persist the computed cume_dist and the band together, (2) document the band boundaries as fixed thresholds rather than 'whatever NTILE produced,' making the mapping a pure function of spend, and (3) snapshot the distribution used, since CUME_DIST is relative to the population and a different WHERE filter changes the bands. That gives a reproducible, value-driven, explainable tiering."

**Interviewer follow-up:** "CUME_DIST bands can now be uneven in count — band 1 might have 30% of customers if many share low spend. Is that acceptable?"

*(Expected: yes, because fairness-by-value outranks equal-counts here; if equal counts are also required, the requirements conflict and you must escalate the trade-off rather than silently splitting ties.)*

---

## 15. Mental Model Checkpoint

1. You run `NTILE(3)` over 8 rows. How many rows land in each bucket, and which buckets get the extra rows? Now do the same for `NTILE(3)` over 100 rows.

2. Five rows all have `score = 90`. Under `NTILE(2) OVER (ORDER BY score)`, can they end up in different buckets? Under `CUME_DIST() OVER (ORDER BY score)`, will they all get the same value? Explain the difference in one sentence.

3. For a partition of exactly one row, what does `PERCENT_RANK()` return? What does `CUME_DIST()` return? Why do they differ?

4. You wrote `... WHERE NTILE(4) OVER (ORDER BY x) = 1` and got an error. Explain — in terms of logical execution order — why the engine cannot evaluate this, and what the minimal rewrite is.

5. The cheapest order in a table gets `PERCENT_RANK = 0.0` but `CUME_DIST = 1/N`. Which of these correctly answers "what fraction of orders cost this much or less"? Why is the other one wrong for that question?

6. You need the p99 latency *value*, not each row's percentile. Which function family do you reach for, and how does `percentile_cont` differ from `percentile_disc` when the exact p99 falls between two stored values?

7. An EXPLAIN shows `WindowAgg → Sort → Seq Scan` with `Sort Method: external merge Disk: 300MB`. Describe two independent changes that would each remove the disk spill, and one that would remove the Sort node entirely.

---

## 16. Quick Reference Card

```sql
-- ── NTILE: split into N buckets by POSITION ──────────────────────────
NTILE(4) OVER (ORDER BY x)                 -- buckets 1..4
NTILE(4) OVER (PARTITION BY g ORDER BY x)  -- per-group quartiles
-- Uneven: extra rows go to the FRONT buckets. N=11,b=4 → 3,3,3,2
-- Splits TIES across buckets (position-based, no peer awareness)
-- ORDER BY is REQUIRED. No frame clause allowed. NTILE(0) errors.

-- ── PERCENT_RANK: (rank-1)/(N-1), pinned 0..1 ────────────────────────
PERCENT_RANK() OVER (ORDER BY x)
-- first row = 0.0 ALWAYS; last = 1.0 ALWAYS
-- ties share value (built on RANK); single row = 0

-- ── CUME_DIST: (rows <= current)/N, the ECDF ─────────────────────────
CUME_DIST() OVER (ORDER BY x)
-- never 0 (min = 1/N); last = 1.0 ALWAYS
-- ties share value; single row = 1.0

-- ── INVERSE: value AT a percentile (ordered-set aggregate) ───────────
percentile_cont(0.95) WITHIN GROUP (ORDER BY x)  -- interpolated, may not exist in data
percentile_disc(0.95) WITHIN GROUP (ORDER BY x)  -- actual data value, no interpolation
percentile_cont(0.5)  WITHIN GROUP (ORDER BY x)  -- median (interpolated)

-- ── Filter on a window output: SUBQUERY required ─────────────────────
SELECT * FROM (
  SELECT *, NTILE(10) OVER (ORDER BY x DESC) AS d FROM t
) s WHERE d = 1;                            -- can't put NTILE in WHERE

-- ── Share one sort across multiple windows ───────────────────────────
SELECT NTILE(4) OVER w, CUME_DIST() OVER w, PERCENT_RANK() OVER w
FROM t WINDOW w AS (PARTITION BY g ORDER BY x);
```

**Perf rules of thumb**
- Cost = one **sort** of the partition, then a cheap linear pass. Optimize the sort.
- Index on `(partition_cols, order_cols)` (matching direction) → planner drops the Sort node.
- Watch for `Sort Method: external merge  Disk: N` → spill; raise `work_mem` or add index.
- Pre-aggregate before windowing; materialize slowly-changing deciles.
- WindowAgg buffers the whole partition (tuplestore) → global (no PARTITION BY) windows over huge tables also spill there.

**Interview one-liners**
- "NTILE splits by position; CUME_DIST/PERCENT_RANK split by value — that's why NTILE breaks ties and they don't."
- "Extra rows in an uneven NTILE always go to the earliest buckets."
- "PERCENT_RANK pins the first row at 0; CUME_DIST never returns 0 — CUME_DIST is the true ECDF."
- "NTILE/CUME_DIST give a row its percentile; percentile_cont/disc give the value at a percentile — inverse operations."
- "You can't filter a window function in WHERE — it hasn't been computed yet; wrap it in a subquery."
- "All three force a full partition sort; an index on the order columns is the one real optimization."

---

## Connected Topics

- **Topic 33 — Window Functions Fundamentals**: The `OVER()`, `PARTITION BY`, and `WindowAgg` machinery all three functions inherit; frames (which these functions forbid).
- **Topic 34 — Ranking Functions (ROW_NUMBER, RANK, DENSE_RANK)**: PERCENT_RANK is built on RANK; the peer-group/tie behavior comes straight from there. NTILE is the fourth ranking-family function.
- **Topic 36 — LAG and LEAD**: The other offset-based window functions; combine with deciles to compute period-over-period movement between cohorts.
- **Topic 20 — GROUP BY Fundamentals**: Pre-aggregation before windowing (Example 3) collapses fan-out so the window sorts fewer rows.
- **Topic 30 — Ordered-Set & Hypothetical-Set Aggregates**: `percentile_cont` / `percentile_disc` / `mode` — the inverse of CUME_DIST, covered in depth.
- **Internals — Sort & work_mem**: The external merge sort that dominates these functions' cost; `work_mem` tuning and spill diagnosis.
- **Internals — Index-Ordered Scans**: How a matching B-tree index lets the planner skip the Sort node feeding `WindowAgg`.
- **Topic 03 — NULL in Depth**: How NULLs order (NULLS FIRST/LAST) and peer-group together in the window ORDER BY.
