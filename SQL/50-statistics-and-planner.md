# Topic 50 — Statistics and the Planner
### SQL Mastery Curriculum — Phase 7: Indexes and Query Optimisation

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a huge warehouse and you're about to send a picker to fetch items for an order. You could tell the picker one of two things:

- "Go find every single blue widget in the building." (Walk every aisle — a full scan.)
- "Go to aisle 12, shelf 4 — there's exactly one blue widget there." (A pinpoint pickup — an index lookup.)

Which instruction is right depends entirely on **how many blue widgets exist**. If there are 3 blue widgets in a warehouse of a million items, the pinpoint pickup wins by a landslide. If half the warehouse is blue widgets, sending the picker aisle by aisle to look up each shelf number is far slower than just walking every aisle once.

Now — how does the warehouse manager *know* how many blue widgets there are without counting them all every time? He keeps a **notebook**. Every so often (say, once a night) an assistant walks a sample of the shelves and jots down summary notes: "roughly 3% of items are blue", "the most common colour is grey — about 40% of stock", "colours are spread evenly across aisles" or "actually, blue items are all clustered in the north wing". The manager never recounts live; he trusts the notebook.

That notebook is **the statistics**. The assistant who refreshes it is **ANALYZE** (and its automatic cousin, **autovacuum**). The manager deciding pinpoint-vs-walk-everything based on the notebook is **the query planner**. And the single most common reason the manager makes a catastrophically bad call — sending a picker to do a million pinpoint lookups when he should have said "just walk the aisles" — is that **the notebook was wrong**: stale, too coarse, or blind to the fact that "blue" and "north wing" always go together.

This entire topic is about that notebook: what's written in it, who updates it, how the manager reads it, and — most importantly — the specific ways it lies and how you fix the lies.

---

## 2. Connection to SQL Internals

The planner is a **cost-based optimiser**. It never executes anything to decide a plan; it *estimates* the cost of each candidate plan and picks the cheapest. Every estimate bottoms out in one number: the **estimated row count** (the "cardinality") flowing out of each node. Get the cardinality right and the cost model usually picks a good plan. Get it wrong and everything downstream — join algorithm, join order, whether to sort, whether to materialise — collapses.

Cardinality estimation is driven entirely by **statistics** stored in the catalog:

- **`pg_statistic`** — the raw, internal, per-column statistics table. Cryptic (columns named `stakind1`, `stanumbers1`, `stavalues1`). One row per column (or expression) per table.
- **`pg_stats`** — a human-readable *view* over `pg_statistic` that decodes those columns into named fields: `null_frac`, `n_distinct`, `most_common_vals`, `most_common_freqs`, `histogram_bounds`, `correlation`.
- **`pg_class.reltuples` and `pg_class.relpages`** — the estimated total row count and page count of the whole relation. These scale the per-column fractions into absolute row estimates.
- **`pg_statistic_ext` / `pg_statistic_ext_data`** — extended (multivariate) statistics created with `CREATE STATISTICS`.

These are populated by **ANALYZE**, which reads a *random sample* of pages (not the whole table — that's the key scalability trick) and computes the distribution summaries. **Autovacuum** runs ANALYZE automatically when enough rows have changed.

At plan time the planner consults these numbers through **selectivity functions** (`eqsel` for `=`, `scalarltsel` for `<`, and so on). A selectivity is a fraction between 0 and 1: "what fraction of rows satisfy this predicate?" Multiply by `reltuples` and you have an estimated row count. That estimate then feeds the **cost model** (`seq_page_cost`, `random_page_cost`, `cpu_tuple_cost`, `cpu_index_tuple_cost`, `cpu_operator_cost`) to produce the cost numbers you see as `cost=x..y` in EXPLAIN.

So statistics sit at the very root of the optimiser. Indexes (Topics 40–48), join algorithms (Topic 17), and scan choices are all *downstream consequences* of what the statistics say. This is why a Principal engineer treats `pg_stats` as the first place to look when a plan goes wrong — not the query text, not the indexes, but the numbers the planner believed.

---

## 3. Logical Execution Order Context

Statistics don't appear in the logical clause order (`FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → window → ORDER BY → LIMIT`) because they aren't a query operation — they're **metadata consulted before any of it runs**. But they govern how *every* stage of that order is physically implemented:

```
              ┌─────────────────────────────────────────────┐
              │  PLANNING (uses statistics — happens ONCE)  │
              └─────────────────────────────────────────────┘
                                  │
   FROM / JOIN   → stats decide join order + algorithm (NL/Hash/Merge)
   WHERE         → stats give per-predicate selectivity → scan choice
   GROUP BY      → n_distinct on the grouping columns → HashAgg vs GroupAgg
                   and the hash table size estimate
   DISTINCT      → same as GROUP BY: n_distinct drives the estimate
   ORDER BY      → correlation decides index-scan-in-order vs Sort;
                   row estimate decides in-memory Sort vs external merge
   LIMIT         → row estimate + ordering decide whether a cheap
                   "abort early" plan is viable
                                  │
              ┌─────────────────────────────────────────────┐
              │  EXECUTION (plan is now fixed — runs)       │
              └─────────────────────────────────────────────┘
```

Two consequences worth internalising:

1. **Statistics are consulted at plan time, once.** For a prepared statement or a cached generic plan, the plan can be built on statistics that were fresh then and stale by execution. This is why "the same query is sometimes fast, sometimes slow" is so often a statistics-and-plan-caching story.

2. **The estimate for an early stage feeds the estimate for a later stage.** If WHERE's selectivity estimate is 100× too high, the GROUP BY node above it inherits a 100×-too-high input estimate and may pick the wrong aggregation strategy. Errors *compound upward* through the plan tree. This is the single most important mental model in this topic: **a bad estimate at a leaf poisons every node above it.**

---

## 4. What Is Statistics and the Planner?

**Statistics** are per-column (and multi-column) summaries of a table's data distribution, stored in the catalog and refreshed by ANALYZE. The **planner** is the cost-based optimiser that reads those summaries to estimate how many rows each part of a query will produce, and uses those estimates to choose scan methods, join algorithms, and join order. The quality of every plan is therefore bounded by the quality of the statistics.

### 4.1 The `pg_stats` view — annotated

```sql
SELECT
  schemaname,             -- │ schema of the table
  tablename,              -- │ table the column belongs to
  attname,                -- │ the column name (or stat-target expression)
  inherited,              -- │ t if stats include child partitions/inheritance
  null_fraction,          -- │ fraction of values that are NULL (0.0–1.0)
  avg_width,              -- │ average stored width of the column in bytes
  n_distinct,             -- │ number of distinct values:
                          -- │   > 0  = absolute count of distinct values
                          -- │   < 0  = NEGATIVE ratio to row count (-1 = all unique)
                          -- │   = 0  = unknown / not enough data
  most_common_vals   AS mcv,   -- │ the MCV list: the most frequent values (an array)
  most_common_freqs  AS mcf,   -- │ parallel array: each MCV's frequency (0.0–1.0)
  histogram_bounds,       -- │ bucket boundaries for the NON-MCV values;
                          -- │   each adjacent pair bounds ~equal-count buckets
  correlation,            -- │ -1..+1: how physically ordered on disk the values are
                          -- │   +1 = stored in ascending key order (index-friendly)
                          -- │    0 = random physical order
                          -- │   -1 = stored in descending order
  most_common_elems,      -- │ MCVs for ARRAY element membership (for @>, && etc.)
  most_common_elem_freqs, -- │ parallel freqs for most_common_elems
  elem_count_histogram    -- │ histogram of array lengths
FROM pg_stats
WHERE tablename = 'orders'
  AND attname   = 'status';
```

### 4.2 The ANALYZE command — annotated

```sql
ANALYZE                       -- │ collect statistics
  (VERBOSE)                   -- │ print per-table progress + sample size
  orders                      -- │ target table (omit to analyze the whole DB)
  (status, created_at);       -- │ optional column list (omit = all columns)

--  Key behaviours:
--   • Reads a RANDOM SAMPLE of pages, not the whole table
--   • Sample size = 300 × default_statistics_target rows (default 300 × 100 = 30,000)
--   • Updates pg_statistic, pg_stats, and pg_class.reltuples/relpages
--   • Takes only a SHARE UPDATE EXCLUSIVE lock — does NOT block reads or writes
--   • VACUUM ANALYZE does both vacuum and analyze in one pass
```

### 4.3 CREATE STATISTICS — annotated

```sql
CREATE STATISTICS              -- │ create an EXTENDED (multivariate) statistics object
  orders_cust_status_stat      -- │ a name for the object (lives in a schema)
  (dependencies,               -- │ functional-dependency stats (A ⇒ B)
   ndistinct,                  -- │ multi-column distinct-count stats
   mcv)                        -- │ multi-column most-common-values list
  ON customer_id, status       -- │ the correlated columns to track together
  FROM orders;                 -- │ the table

ANALYZE orders;                -- │ REQUIRED: the object is empty until ANALYZE fills it
```

---

## 5. Why Statistics Mastery Matters in Production

1. **Wrong estimates cause wrong plans, and wrong plans are the #1 cause of sudden query slowdowns.** A query that ran in 20ms for a year suddenly takes 40s. Nothing in the code changed. What changed is the data distribution drifted and the statistics went stale, so the planner flipped from a Hash Join to a Nested Loop (or vice versa) on a now-wrong row estimate. Without understanding statistics, this looks like black magic.

2. **The Nested-Loop blowup.** The single most destructive planner mistake: it estimates a join will produce 5 rows, chooses a Nested Loop, but the join actually produces 5 million rows. Now it does 5 million index probes instead of one Hash Join. Execution goes from milliseconds to minutes. The root cause is almost always an underestimate driven by correlated columns the planner assumed were independent.

3. **Correlated columns are everywhere in real schemas** and the planner assumes independence by default. `city` and `postal_code`, `country` and `currency`, `product_id` and `category_id`, `status` and `shipped_at` — these are functionally dependent, but the planner multiplies their selectivities as if they were independent, producing estimates that can be off by orders of magnitude. Knowing to reach for `CREATE STATISTICS` is a senior-level reflex.

4. **After a bulk load, statistics are absent or stale.** You COPY 50M rows into a fresh partition, run a query, and get a terrible plan — because ANALYZE hasn't run yet and the planner thinks the table has the default ~0 or old row count. Every data pipeline needs an explicit `ANALYZE` step, and knowing *why* is the difference between a pipeline that's reliably fast and one that randomly isn't.

5. **`default_statistics_target` is a real, tunable knob** with a real trade-off (planning time and ANALYZE cost vs. estimate accuracy). Skewed, high-cardinality columns on huge tables often need per-column targets raised to 1000. Getting this right on the three columns that matter — rather than globally — is a production optimisation that pays for itself.

6. **You cannot debug a plan you can't read**, and you cannot read a plan without knowing where the row estimates came from. `pg_stats` is the ground truth behind every `rows=` in EXPLAIN. Reasoning from the statistics is what separates "let me add a random index and hope" from "the estimate is wrong *here*, for *this* reason, and here's the fix."

---

## 6. Deep Technical Content

### 6.1 What ANALYZE actually collects, column by column

For each analysed column, ANALYZE computes and stores:

- **`null_fraction`** — the fraction of sampled rows where the column is NULL. Used directly: a predicate `col IS NOT NULL` has selectivity `1 - null_fraction`; any `col = x` predicate is scaled by the non-null fraction.
- **`avg_width`** — average bytes per value. Feeds width estimates (the `width=` in EXPLAIN), which drive memory and I/O cost, and decisions like whether a sort fits in `work_mem`.
- **`n_distinct`** — the number of distinct non-null values. Central to GROUP BY, DISTINCT, and join estimates. Stored as a negative *ratio* when it scales with table size (see 6.4).
- **`most_common_vals` (MCV) + `most_common_freqs`** — the top-K most frequent values and their exact-ish frequencies. Handles **skew** (see 6.2).
- **`histogram_bounds`** — equi-depth bucket boundaries over the values *not* in the MCV list. Handles **range predicates** on the "body" of the distribution (see 6.3).
- **`correlation`** — how well the physical row order matches the sorted value order. Drives the cost of index scans (see 6.5).

The number of MCVs kept and the number of histogram buckets are both capped by the column's **statistics target** (default 100, from `default_statistics_target`).

### 6.2 Most Common Values (MCV) — handling skew

Real columns are rarely uniform. Consider `orders.status`:

```
status      | fraction
------------+---------
'completed' | 0.82
'shipped'   | 0.09
'pending'   | 0.05
'cancelled' | 0.03
'refunded'  | 0.01
```

The MCV list stores these values with their frequencies. When you write `WHERE status = 'refunded'`, the planner looks up `'refunded'` in `most_common_vals`, finds frequency `0.01`, and estimates `0.01 × reltuples` rows. When you write `WHERE status = 'completed'`, it finds `0.82` and estimates a huge scan — correctly choosing a Seq Scan even if an index exists.

This is why an index on a skewed column is useful for the *rare* values and useless for the *common* ones, and why the planner correctly uses the index for `status = 'refunded'` but ignores it for `status = 'completed'`. The MCV list is what makes that value-dependent decision possible.

For a value **not** in the MCV list, the planner estimates using the *residual* selectivity: it takes the fraction of rows not covered by any MCV, divides among the remaining distinct values, and (for equality) assumes uniformity across them:

```
selectivity(non-MCV value) = (1 - sum(most_common_freqs) - null_fraction)
                             / (n_distinct - length(most_common_vals))
```

If your skewed column has 10,000 distinct values but the statistics target only captures the top 100, the other 9,900 values all share a single averaged estimate — which is fine if they're uniform and terrible if some of them are secretly hot. **This is the classic "raise the statistics target" scenario.**

### 6.3 The Histogram — handling range predicates

The MCV list handles equality on frequent values. The **histogram** handles ranges (`<`, `>`, `BETWEEN`) over everything else. ANALYZE removes the MCVs, then divides the remaining values into buckets each holding *approximately the same number of rows* (equi-depth, not equi-width):

```
histogram_bounds for orders.total_amount (target 10 → 11 bounds, 10 buckets):
{0.00, 12.50, 24.99, 41.00, 63.50, 89.99, 128.00, 199.00, 340.00, 780.00, 25000.00}
 └──────┬──────┘
   each adjacent pair bounds a bucket holding ~10% of the (non-MCV) rows
```

Equi-depth means each bucket holds ~10% of rows, so buckets are *narrow* where data is dense and *wide* where data is sparse. For `WHERE total_amount < 50`:

```
< 50 covers: bucket[0]=0–12.50, bucket[1]=12.50–24.99, bucket[2]=24.99–41.00 fully
             plus part of bucket[3]=41.00–63.50, linearly interpolated:
             (50 - 41.00) / (63.50 - 41.00) ≈ 0.40 of that bucket
estimate ≈ (3 + 0.40) × 10% = 34% of rows
```

Two critical properties:

- **Interpolation within a bucket assumes uniform distribution inside the bucket.** If a bucket spans 340–780 but every value is actually clustered at 341, a predicate `< 400` will be badly estimated. More buckets (higher target) shrink this error.
- **The last bucket often spans a huge range** (here 780–25000) because outliers are rare. Range predicates that land in the long tail are the least accurately estimated. This is where high statistics targets help most on monetary/measurement columns with fat tails.

### 6.4 n_distinct — the sign convention and why it matters

`n_distinct` uses a sign convention that trips up everyone the first time:

- **Positive** (e.g. `n_distinct = 42`): the column has an estimated *fixed* 42 distinct values, independent of table size. Used for low-cardinality columns like `status`.
- **Negative** (e.g. `n_distinct = -1`): the distinct count is a *ratio* of the row count. `-1` means every row is unique (a primary key). `-0.5` means distinct values ≈ half the rows. Used when cardinality scales with the table.

Why the distinction exists: **ANALYZE only samples.** From a 30,000-row sample of a 100M-row table, estimating the true number of distinct values is statistically hard — it's the notoriously difficult "number of species" problem. If the sample suggests distinct-count grows with table size, PG stores a negative ratio so the estimate scales correctly as the table grows *without re-analysing*. If it looks bounded, it stores an absolute count.

**The failure mode:** on a large table with many distinct values that happen to be *unevenly distributed across sampled pages*, ANALYZE can badly under- or over-estimate `n_distinct`. A too-low `n_distinct` makes the planner think GROUP BY produces few groups → it picks a HashAggregate sized too small → the hash spills or a Nested Loop is chosen on a join. You can override it manually:

```sql
-- Tell the planner this column really has ~5,000,000 distinct values
ALTER TABLE sessions ALTER COLUMN user_id SET (n_distinct = 5000000);
-- Or as a ratio (negative): distinct ≈ 30% of rows
ALTER TABLE sessions ALTER COLUMN user_id SET (n_distinct = -0.3);
ANALYZE sessions;  -- required to take effect
```

This manual override survives ANALYZE (it's a column option, not a computed stat) and is one of the few "hints" PostgreSQL actually offers.

### 6.5 Correlation — the hidden driver of index-scan cost

`correlation` measures how closely the physical order of rows on disk matches the sorted order of the column's values, from `-1` to `+1`:

- **`+1`**: rows are stored on disk in ascending order of this column. An index range scan reads index entries in order and the corresponding heap rows are *also* in order → nearly sequential heap I/O → cheap.
- **`0`**: no relationship. An index range scan jumps randomly around the heap → one random page fetch per row → expensive. The planner may reject the index in favour of a Seq Scan.
- **`-1`**: descending physical order — also highly correlated (just reversed), still cheap.

This is why a `BRIN` index (Topic 47) works brilliantly on a column with correlation near ±1 (like an append-only `created_at`) and uselessly on an uncorrelated column. And it's why the *same index* on the *same column* is chosen for a range scan on one table and ignored on another: the difference is the physical clustering, captured entirely by `correlation`.

```sql
-- See correlation for a table's columns:
SELECT attname, correlation
FROM pg_stats
WHERE tablename = 'orders'
ORDER BY abs(correlation) DESC;

-- created_at is append-only → correlation ≈ 0.99 → index range scans are cheap
-- customer_id is scattered  → correlation ≈ 0.02 → index range scans cost more
```

`CLUSTER orders USING orders_created_at_idx` physically reorders the heap to match an index, driving that column's correlation to +1 — but it's a one-time, blocking operation and new inserts erode it.

### 6.6 The independence assumption — the root of most bad estimates

When a query has multiple predicates, the planner (by default) assumes they are **statistically independent** and *multiplies* their selectivities:

```sql
WHERE city = 'San Francisco' AND state = 'CA'
-- sel(city='SF')   ≈ 0.001   (1 in 1000 rows)
-- sel(state='CA')  ≈ 0.12    (12% of rows)
-- planner estimate = 0.001 × 0.12 = 0.00012  → 0.012% of rows
```

But **every** San Francisco row is already a California row. The true selectivity is just `0.001`. The planner underestimated by ~120×. On a 100M-row table that's the difference between an estimate of 12,000 rows and a reality of 100,000 — and if this feeds a join, the Nested-Loop-vs-Hash decision flips.

The same happens on the *high* side with `OR`, and with functionally dependent columns like:
- `product_id` and `category_id` (a product belongs to exactly one category)
- `order_id` and `customer_id`
- `zip` and `city`
- `status = 'shipped'` and `shipped_at IS NOT NULL`

**This independence assumption is the single most important thing to understand about why the planner is wrong.** The fix is extended statistics (6.8).

### 6.7 default_statistics_target — the accuracy/cost knob

`default_statistics_target` (default **100**) controls two things simultaneously:

- The **number of MCVs** stored (up to the target).
- The **number of histogram buckets** (up to the target).
- Indirectly, the **sample size**: `300 × target` rows. Default = 30,000 rows sampled.

Raising it makes estimates more accurate (more MCVs capture more of the skew, more buckets shrink interpolation error, a bigger sample estimates `n_distinct` better) at the cost of:

- **Longer ANALYZE** (reads more pages).
- **Larger `pg_statistic`** rows.
- **Slightly longer planning time** (the planner scans longer MCV/histogram arrays for every query).

The right approach is **surgical, not global**. Leave the global default at 100 and raise it only on the specific columns that need it:

```sql
-- A high-cardinality, skewed column on a huge table used in range/equality predicates
ALTER TABLE orders ALTER COLUMN total_amount SET STATISTICS 1000;
ALTER TABLE events ALTER COLUMN event_type   SET STATISTICS 500;
ANALYZE orders;
ANALYZE events;

-- Inspect the per-column target (attstattarget; -1 = use the global default):
SELECT attname, attstattarget
FROM pg_attribute
WHERE attrelid = 'orders'::regclass AND attnum > 0 AND NOT attisdropped;
```

Maximum is 10000. Values above ~1000 rarely pay off and inflate planning time noticeably.

### 6.8 CREATE STATISTICS — extended/multivariate statistics

Extended statistics teach the planner about relationships *between* columns, defeating the independence assumption. There are three kinds, requested in the `CREATE STATISTICS` column list:

**1. `dependencies` — functional dependencies.** Records "the value of column A determines the value of column B" as a degree (0–1). For `city ⇒ state`, once the planner knows `city = 'SF'`, adding `AND state = 'CA'` barely changes the estimate (because the dependency is ~1.0), so it *doesn't* multiply the second selectivity in.

```sql
CREATE STATISTICS orders_geo_deps (dependencies)
  ON city, state FROM customer_addresses;
ANALYZE customer_addresses;
```

**2. `ndistinct` — multi-column distinct counts.** Fixes GROUP BY / DISTINCT estimates over multiple columns. Without it, the planner estimates distinct `(a, b)` as `n_distinct(a) × n_distinct(b)`, which massively overestimates when the columns are correlated.

```sql
CREATE STATISTICS orders_grp_ndistinct (ndistinct)
  ON customer_id, status FROM orders;
ANALYZE orders;
-- Now GROUP BY customer_id, status is estimated correctly
```

**3. `mcv` — multi-column most-common values.** Stores the most frequent *combinations* of values, with their real joint frequencies. The most powerful and most expensive kind — it captures skew in specific value pairs (e.g. `(country='US', currency='USD')` is common; `(country='US', currency='JPY')` is rare).

```sql
CREATE STATISTICS orders_cs_mcv (mcv)
  ON country, currency FROM orders;
ANALYZE orders;
```

You can request several kinds in one object. Extended statistics currently apply to **base-table columns referenced together in the same table's predicates or grouping** — they help WHERE/GROUP BY on one table; they do not directly fix cross-table join-key correlation (that remains a harder problem).

```sql
-- Everything at once for a heavily-correlated pair:
CREATE STATISTICS orders_full_stat (dependencies, ndistinct, mcv)
  ON customer_id, status FROM orders;
ANALYZE orders;

-- Inspect what was built:
SELECT * FROM pg_stats_ext WHERE statistics_name = 'orders_full_stat';
```

You can also build statistics **on expressions** (PG 14+), fixing estimates for predicates on function results:

```sql
CREATE STATISTICS orders_lower_email (mcv)
  ON lower(email) FROM users;  -- helps WHERE lower(email) = '...'
```

### 6.9 Autovacuum-driven statistics

You rarely run ANALYZE by hand in steady state — **autovacuum** does it. The autovacuum daemon launches an `ANALYZE` on a table when the number of inserted/updated/deleted tuples since the last analyze crosses a threshold:

```
analyze threshold = autovacuum_analyze_threshold             (default 50)
                  + autovacuum_analyze_scale_factor           (default 0.1 = 10%)
                    × reltuples
```

So a 1,000-row table is re-analysed after ~150 changes; a 100M-row table only after ~10M changes. **That scale factor is the problem on large tables**: a 100M-row table can drift for 10 million row changes before autovacuum refreshes stats — plenty of time for a distribution shift to produce bad plans. The fix is a per-table override:

```sql
-- Analyze this big table after only 1% of rows change, not 10%
ALTER TABLE orders SET (autovacuum_analyze_scale_factor = 0.01);

-- Or a flat threshold for a large, steadily-growing table:
ALTER TABLE events SET (autovacuum_analyze_scale_factor = 0.0,
                        autovacuum_analyze_threshold = 500000);
```

Check when autovacuum last touched a table:

```sql
SELECT relname, last_analyze, last_autoanalyze,
       n_mod_since_analyze, n_live_tup
FROM pg_stat_user_tables
WHERE relname IN ('orders', 'order_items', 'events')
ORDER BY n_mod_since_analyze DESC;
```

Two notorious gaps autovacuum does **not** cover:
- **Bulk loads**: `COPY` / large `INSERT ... SELECT` don't trigger an immediate analyze; the table can serve queries with empty stats for minutes. Always `ANALYZE` explicitly after a load.
- **New partitions**: a freshly attached partition has no stats until analysed.

### 6.10 Statistics on partitioned tables and inheritance

For a partitioned parent, `pg_stats` can hold both:
- **Per-partition** statistics (`inherited = false` on each child) — used when the planner prunes to a single partition.
- **Inherited/whole-tree** statistics on the parent (`inherited = true`) — used for queries spanning partitions.

Autovacuum analyses *individual partitions* but historically **not** the parent's inherited stats automatically, so cross-partition planning could degrade. Run `ANALYZE parent_table` periodically (or rely on the improvements in recent PG versions) to keep the tree-level stats fresh. This is a frequent source of "the query is fast when it hits one partition and slow when it spans several" puzzles.

---

## 7. EXPLAIN — Statistics Errors in the Plan

The single most valuable diagnostic skill in this topic: **compare `rows=` (estimated) against `actual rows=` in `EXPLAIN ANALYZE`.** A large gap is the fingerprint of a statistics problem.

### 7.1 A healthy estimate

```sql
EXPLAIN ANALYZE
SELECT * FROM orders WHERE status = 'refunded';
```

```
Index Scan using orders_status_idx on orders
     (cost=0.43..812.00 rows=9800 width=84)
     (actual time=0.05..3.20 rows=9950 loops=1)
  Index Cond: (status = 'refunded')
Planning Time: 0.15 ms
Execution Time: 3.80 ms
```

`rows=9800` estimated vs `rows=9950` actual — within 2%. The MCV list has an accurate frequency for `'refunded'`. The planner correctly chose an Index Scan because the estimate is a small fraction of the table. **This is what "good stats" looks like.**

### 7.2 The correlated-column underestimate (the dangerous one)

```sql
EXPLAIN ANALYZE
SELECT o.id, o.total_amount
FROM orders o
WHERE o.city = 'San Francisco'
  AND o.state = 'CA';
```

```
Bitmap Heap Scan on orders o
     (cost=45.10..1230.00 rows=120 width=20)
     (actual time=8.4..410.0 rows=98750 loops=1)
  Recheck Cond: (city = 'San Francisco')
  Filter: (state = 'CA')
  Rows Removed by Filter: 0
  Heap Blocks: exact=8912
  ->  Bitmap Index Scan on orders_city_idx
        (cost=0.00..45.07 rows=99000 width=0)
        (actual time=6.1..6.1 rows=98750 loops=1)
Planning Time: 0.3 ms
Execution Time: 445 ms
```

Read the gap: **`rows=120` estimated, `rows=98750` actual — an 820× underestimate.** The independence assumption multiplied `sel(city)` by `sel(state)` even though every SF row is a CA row (note `Rows Removed by Filter: 0` — the state predicate removed *nothing*). This 120-row estimate is a landmine: if this node feeds a join, the planner will pick a Nested Loop expecting 120 outer rows and get 98,750 — a ~800× blowup in inner-side work.

The fix:

```sql
CREATE STATISTICS orders_geo (dependencies, mcv) ON city, state FROM orders;
ANALYZE orders;
```

After which the estimate lands near `rows=98000` and the planner picks a plan appropriate to 100K rows.

### 7.3 The stale-stats blowup driving a Nested Loop

```sql
EXPLAIN ANALYZE
SELECT c.name, o.id, o.total_amount
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.created_at >= '2026-07-01';   -- stats last refreshed when table was small
```

```
Nested Loop  (cost=0.43..2100.00 rows=45 width=52)
             (actual time=0.5..38200.0 rows=1240000 loops=1)
  ->  Seq Scan on orders o
        (cost=0.00..1800.00 rows=45 width=20)
        (actual time=0.2..920.0 rows=1240000 loops=1)
        Filter: (created_at >= '2026-07-01')
        Rows Removed by Filter: 210
  ->  Index Scan using customers_pkey on customers c
        (cost=0.43..6.60 rows=1 width=40)
        (actual rows=1 loops=1240000)   ← 1.24M index probes!
Planning Time: 0.2 ms
Execution Time: 41300 ms
```

The `created_at >= '2026-07-01'` estimate is `rows=45` because the histogram's last bucket predates July — the stats think almost nothing is that recent. Reality: 1.24M rows. The planner chose a Nested Loop (great for 45 outer rows) and executed **1.24 million** index probes (`loops=1240000`). 41 seconds. A single `ANALYZE orders` refreshes the histogram's upper bound, the estimate corrects to ~1.24M, and the planner switches to a Hash Join that finishes in under a second.

### 7.4 Forcing the planner to show its raw estimate

To see the estimate *without* running the query (fast, safe on production):

```sql
EXPLAIN                       -- no ANALYZE: estimate only, nothing executed
SELECT * FROM orders WHERE status = 'pending' AND customer_id = 42;
```

Compare the top-node `rows=` to a real `SELECT count(*)` with the same WHERE. If they diverge by more than ~10×, you have a statistics problem worth fixing before touching anything else.

### 7.5 The EXPLAIN estimate-error cheat table

| Symptom in EXPLAIN ANALYZE | Statistics cause | Fix |
|---|---|---|
| est `rows` ≪ actual, on multi-column AND | Independence assumption on correlated columns | `CREATE STATISTICS (dependencies, mcv)` |
| est `rows` ≪ actual, on a range predicate at the "recent" end | Stale histogram upper bound | `ANALYZE` (and lower `analyze_scale_factor`) |
| est `rows` way off on a rare value | Value not in MCV list; target too low | Raise `SET STATISTICS`, re-ANALYZE |
| GroupAgg/HashAgg with wrong group estimate | Bad `n_distinct` (single or multi-col) | `SET (n_distinct=...)` or `CREATE STATISTICS (ndistinct)` |
| Index ignored though few rows match | `correlation ≈ 0` inflating index cost, or bad estimate | Check `correlation`; consider CLUSTER; re-ANALYZE |
| Nested Loop with huge `loops=` count | Underestimated outer-side rows | Fix the underlying estimate (above); it flips to Hash |

---

## 8. Query Examples

### Example 1 — Basic: read a column's statistics

```sql
-- What does the planner "know" about orders.status?
SELECT
  attname,
  null_fraction,
  n_distinct,
  most_common_vals,
  most_common_freqs
FROM pg_stats
WHERE tablename = 'orders'
  AND attname = 'status';

-- Typical result:
--  attname |  status
--  null_fraction | 0
--  n_distinct    | 5              -- five statuses, fixed (positive = absolute)
--  most_common_vals  | {completed,shipped,pending,cancelled,refunded}
--  most_common_freqs | {0.82,0.09,0.05,0.03,0.01}
-- Reading it: WHERE status='completed' → ~82% of rows → Seq Scan expected.
--             WHERE status='refunded'  → ~1%  of rows → Index Scan expected.
```

### Example 2 — Intermediate: prove and fix a correlated-column misestimate

```sql
-- Step 1: expose the misestimate (est vs actual on two dependent columns)
EXPLAIN ANALYZE
SELECT id FROM orders
WHERE customer_id = 5551
  AND status = 'completed';
-- Suppose: rows=3 (est) vs rows=380 (actual)  → ~125× under.
-- Cause: planner multiplied sel(customer_id) × sel(status), but customer 5551
-- is a heavy buyer whose orders are overwhelmingly 'completed' — correlated.

-- Step 2: teach the planner the relationship
CREATE STATISTICS orders_cust_status (dependencies, ndistinct, mcv)
  ON customer_id, status
  FROM orders;

-- Step 3: statistics objects are EMPTY until analyzed
ANALYZE orders;

-- Step 4: re-check — estimate now lands near the true 380
EXPLAIN ANALYZE
SELECT id FROM orders
WHERE customer_id = 5551
  AND status = 'completed';
-- rows=372 (est) vs rows=380 (actual) → within 2%.
```

### Example 3 — Production Grade: post-load statistics hardening for a fact table

```sql
-- Context:
--   • order_items: 120M rows, partitioned monthly, freshly bulk-loaded via COPY
--   • Hot query filters on (product_id, warehouse_id) — strongly correlated
--     (each product ships from a small, fixed set of warehouses)
--   • total_amount is high-cardinality with a fat right tail (whale orders)
--   • Query SLA: p95 < 200ms; without stats hardening it intermittently hits 30s
-- Index availability: btree on (product_id), (warehouse_id), (created_at);
--   BRIN on (created_at) for the partition-local range scans.
-- Perf expectation: after this block, estimates are within ~2× of actual and the
--   planner reliably chooses a Bitmap Heap Scan feeding a Hash Join (not a Nested Loop).

-- 1. Bulk loads do NOT auto-analyze — do it explicitly, immediately after COPY.
ANALYZE order_items;

-- 2. Raise the statistics target on the skewed money column (fat tail needs
--    more histogram buckets to estimate range predicates in the long tail).
ALTER TABLE order_items ALTER COLUMN total_amount SET STATISTICS 1000;

-- 3. Defeat the independence assumption on the correlated filter pair.
CREATE STATISTICS oi_prod_wh (dependencies, ndistinct, mcv)
  ON product_id, warehouse_id
  FROM order_items;

-- 4. This table drifts fast; make autovacuum re-analyze after 1% churn, not 10%.
ALTER TABLE order_items SET (autovacuum_analyze_scale_factor = 0.01);

-- 5. Re-run ANALYZE so the raised target + new extended stats take effect.
ANALYZE order_items;

-- 6. Verify the hot query's estimate is now sane.
EXPLAIN ANALYZE
SELECT oi.product_id, sum(oi.quantity) AS units, sum(oi.total_amount) AS revenue
FROM order_items oi
WHERE oi.product_id  = 88123
  AND oi.warehouse_id = 4
  AND oi.total_amount >= 500.00
GROUP BY oi.product_id;
```

```
-- EXPLAIN ANALYZE (after hardening):
HashAggregate  (cost=41200.0..41201.0 rows=1 width=44)
               (actual time=132.0..132.0 rows=1 loops=1)
  Group Key: product_id
  ->  Bitmap Heap Scan on order_items oi
        (cost=980.0..41000.0 rows=41200 width=20)
        (actual time=12.0..118.0 rows=40310 loops=1)   ← est 41200 vs actual 40310: good
        Recheck Cond: (product_id = 88123)
        Filter: ((warehouse_id = 4) AND (total_amount >= 500.00))
        ->  Bitmap Index Scan on order_items_product_id_idx
              (cost=0.0..970.0 rows=52000 width=0)
              (actual time=9.0..9.0 rows=51900 loops=1)
Planning Time: 0.8 ms
Execution Time: 138 ms   ← within the 200ms SLA
```

---

## 9. Wrong → Right Patterns

### Wrong 1: Querying a freshly bulk-loaded table without analyzing

```sql
-- WRONG: COPY a new partition then immediately run analytics
COPY order_items_2026_07 FROM '/data/july.csv' CSV;
SELECT product_id, sum(total_amount)
FROM order_items_2026_07
WHERE warehouse_id = 4
GROUP BY product_id;
-- SYMPTOM: EXPLAIN shows rows=1 (or a tiny default) on a 12M-row partition.
-- The planner has NO statistics (COPY doesn't trigger autoanalyze), so it
-- assumes almost-empty and may pick a Nested Loop that runs for minutes.
-- At the execution level: pg_class.reltuples is still ~0 / stale, every
-- selectivity collapses to a default, join order is chosen blind.
```

```sql
-- RIGHT: always analyze immediately after a bulk load, before serving queries
COPY order_items_2026_07 FROM '/data/july.csv' CSV;
ANALYZE order_items_2026_07;          -- populate stats now
-- (and for a partitioned parent, refresh the tree-level stats too)
ANALYZE order_items;
-- Now reltuples, histograms, and MCVs reflect reality → correct plan.
```

### Wrong 2: Trusting multiplied selectivities on correlated columns

```sql
-- WRONG: assuming the planner "knows" city implies state
SELECT * FROM customer_addresses
WHERE city = 'San Francisco' AND state = 'CA';
-- The planner estimates sel(city) × sel(state), underestimating ~800×.
-- WHY at the execution level: with a tiny estimate it picks a Nested Loop
-- for any downstream join; the join then fans out to the real 100K rows and
-- performs 100K inner probes instead of building one hash table.
```

```sql
-- RIGHT: declare the dependency so the planner stops multiplying
CREATE STATISTICS ca_geo (dependencies, mcv) ON city, state
  FROM customer_addresses;
ANALYZE customer_addresses;
-- Now sel(city AND state) ≈ sel(city); estimate ~ actual; Hash Join chosen.
```

### Wrong 3: Cranking default_statistics_target globally to "fix" one query

```sql
-- WRONG: a global sledgehammer
SET default_statistics_target = 5000;   -- session-wide, or worse, in postgresql.conf
ANALYZE;   -- now re-analyzes EVERY column of EVERY table with a huge sample
-- COST: ANALYZE across the DB becomes very slow, pg_statistic bloats, and
-- EVERY query's planning time rises because the planner scans 5000-entry
-- MCV/histogram arrays even for columns that never needed it.
```

```sql
-- RIGHT: surgical, per-column targeting on the few columns that need it
ALTER TABLE orders  ALTER COLUMN total_amount SET STATISTICS 1000;
ALTER TABLE events  ALTER COLUMN event_type   SET STATISTICS 500;
ANALYZE orders;
ANALYZE events;
-- Global default stays at 100; only the skewed/high-card columns pay the cost.
```

### Wrong 4: Overriding n_distinct without understanding the sign convention

```sql
-- WRONG: meaning "30% of rows are distinct" but writing a positive number
ALTER TABLE sessions ALTER COLUMN user_id SET (n_distinct = 0.3);
-- BUG: a POSITIVE value means "exactly 0.3 distinct values" (rounds toward 0/1) —
-- nonsensical. The planner now thinks the column has ~1 distinct value and
-- estimates GROUP BY user_id yields a single group → wrong aggregation plan.
```

```sql
-- RIGHT: negative = ratio of row count; positive = absolute count
ALTER TABLE sessions ALTER COLUMN user_id SET (n_distinct = -0.3); -- 30% of rows
-- or, if you truly know the absolute count:
ALTER TABLE sessions ALTER COLUMN user_id SET (n_distinct = 5000000);
ANALYZE sessions;   -- required for the override to take effect
```

### Wrong 5: Blaming the index when the estimate is the problem

```sql
-- WRONG mental model: "the index isn't being used, so I'll force it"
SET enable_seqscan = off;    -- brute-force override in production
SELECT * FROM orders WHERE created_at >= '2026-07-01';
-- This masks the real issue: the histogram's upper bound is stale, so the
-- estimate (rows=45) is wrong. Forcing the index "fixes" this ONE query while
-- leaving every other plan built on the same stale stats broken — and
-- enable_seqscan=off is a global footgun that corrupts unrelated plans.
```

```sql
-- RIGHT: fix the statistics, let the planner choose correctly on its own
ANALYZE orders;             -- refresh the histogram upper bound
-- Optionally keep it fresh automatically:
ALTER TABLE orders SET (autovacuum_analyze_scale_factor = 0.02);
-- Now the estimate reflects the 1.24M recent rows and the planner picks the
-- right scan and join for EVERY query, not just this one.
```

---

## 10. Performance Profile

### 10.1 The cost of ANALYZE itself

ANALYZE reads a **bounded random sample** — `300 × statistics_target` rows (default 30,000) — regardless of table size. This is the key scalability property: analyzing a 100M-row table costs roughly the same I/O as a 1M-row table *at the same target*, because both sample ~30,000 rows. The cost scales with the **target**, not the table:

| statistics_target | Rows sampled | Relative ANALYZE cost | Planning-time impact |
|---|---|---|---|
| 100 (default) | 30,000 | 1× | negligible |
| 500 | 150,000 | ~5× | small |
| 1000 | 300,000 | ~10× | noticeable on wide MCV lists |
| 10000 (max) | 3,000,000 | ~100× | significant — avoid globally |

The sample is of *pages then rows within pages*, so very wide rows or bloated heaps make each sampled page more expensive to read, but the row count sampled is fixed.

### 10.2 Estimate accuracy vs table size

| Table size | Default-target risk | Recommendation |
|---|---|---|
| 1M rows | Low. 30K sample ≈ 3% — usually fine. | Defaults + extended stats on correlated pairs. |
| 10M rows | Moderate. Rare values and fat tails start slipping through the 100-entry MCV/histogram. | Raise target to 500–1000 on skewed columns; lower analyze_scale_factor. |
| 100M rows | High. 30K sample ≈ 0.03%; `n_distinct` estimation gets shaky; the 10%-churn autovacuum threshold means ~10M rows of drift between analyzes. | Per-column target 1000; `autovacuum_analyze_scale_factor = 0.01`; consider manual `n_distinct` overrides; extended stats mandatory on correlated filters. |

### 10.3 Where bad stats hurt most, by operation

- **Joins (worst):** an underestimate flips Hash Join → Nested Loop; the blowup is *multiplicative* (loops × inner cost). A 1000× row underestimate on the outer side ≈ 1000× slower join. This is the highest-stakes consequence of bad stats.
- **Aggregation:** a bad `n_distinct` undersizes a HashAggregate → hash spills to disk (`Batches > 1`), or picks GroupAggregate with an unnecessary Sort. Typically 2–10× slowdown.
- **Sorts:** a row underestimate makes the planner expect an in-memory sort that then spills to an external merge sort on disk. 5–20× slower for the sort node.
- **Scans:** an overestimate on a selective predicate can push the planner to a Seq Scan when an Index Scan was right (and vice versa). Usually the least severe, bounded by table size.

### 10.4 Planning-time overhead

Every query pays a planning-time tax proportional to the size of the MCV/histogram arrays it must scan. At target 100 this is microseconds. At target 10000 across many predicated columns it can add milliseconds *per plan*, which matters for high-QPS OLTP where the same cheap query runs 50,000×/second. This is the concrete reason to keep the global target at 100 and raise it surgically.

### 10.5 Keeping stats fresh at scale — the maintenance recipe

```sql
-- 1. Identify tables drifting furthest since last analyze
SELECT relname, n_live_tup, n_mod_since_analyze,
       round(100.0 * n_mod_since_analyze / NULLIF(n_live_tup,0), 1) AS pct_drift,
       last_autoanalyze
FROM pg_stat_user_tables
ORDER BY pct_drift DESC NULLS LAST
LIMIT 20;

-- 2. Tighten autovacuum analyze on the big, hot tables
ALTER TABLE orders      SET (autovacuum_analyze_scale_factor = 0.02);
ALTER TABLE order_items SET (autovacuum_analyze_scale_factor = 0.01);

-- 3. Pin high targets only where skew/cardinality demands it
ALTER TABLE order_items ALTER COLUMN total_amount SET STATISTICS 1000;

-- 4. After ETL / bulk load: explicit analyze is non-negotiable
ANALYZE order_items;
```

---

## 11. Node.js Integration

### 11.1 Reading statistics from application code (a stats-health endpoint)

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Surface tables whose statistics are drifting, for an ops dashboard.
async function statsHealth(driftThresholdPct = 20) {
  const { rows } = await pool.query(
    `SELECT relname,
            n_live_tup,
            n_mod_since_analyze,
            round(100.0 * n_mod_since_analyze / NULLIF(n_live_tup, 0), 1)
              AS pct_drift,
            last_autoanalyze
     FROM pg_stat_user_tables
     WHERE n_live_tup > 0
       AND (100.0 * n_mod_since_analyze / NULLIF(n_live_tup, 0)) >= $1
     ORDER BY pct_drift DESC`,
    [driftThresholdPct]
  );
  return rows;   // [] means all tables within threshold
}
```

### 11.2 Analyzing after a bulk load inside a pipeline

```javascript
// A COPY-based ingest must ANALYZE before the data is served to query traffic.
async function ingestAndPrepare(partition, csvPath) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    // (COPY via pg-copy-streams elided for brevity)
    await client.query(`COPY ${partition} FROM $1 WITH (FORMAT csv)`, [csvPath]);
    await client.query('COMMIT');

    // ANALYZE cannot run inside the same transaction block as the load if you
    // want its stats visible immediately to other backends — run it after COMMIT.
    // It takes only a SHARE UPDATE EXCLUSIVE lock, so it won't block readers.
    await client.query(`ANALYZE ${partition}`);
    await client.query(`ANALYZE order_items`);   // refresh partitioned-parent stats
  } catch (err) {
    await client.query('ROLLBACK').catch(() => {});
    throw err;
  } finally {
    client.release();
  }
}
```

Note the identifier interpolation above is only safe because `partition` is an internally-controlled constant, never user input. For any user-supplied identifier you must validate against an allow-list — you cannot parameterize table names with `$1` (only values bind that way).

### 11.3 Checking an estimate from code before shipping a query

```javascript
// Guardrail: run EXPLAIN (no ANALYZE — nothing executes) and compare the top
// node's estimate against a count, alerting when they diverge badly.
async function estimateVsActual(sql, params = []) {
  const plan = await pool.query(`EXPLAIN (FORMAT JSON) ${sql}`, params);
  const estRows = plan.rows[0]['QUERY PLAN'][0].Plan['Plan Rows'];

  const countSql = `SELECT count(*) AS c FROM (${sql}) sub`;
  const { rows } = await pool.query(countSql, params);
  const actual = Number(rows[0].c);

  const ratio = actual === 0 ? 1 : estRows / actual;
  return {
    estRows,
    actual,
    ratio,
    suspect: ratio < 0.1 || ratio > 10,   // >10× off → likely a stats problem
  };
}
```

### 11.4 Applying targeted stats tuning as a migration

```javascript
// Run once, as a migration — not per-request. These are DDL and persist.
async function hardenStatistics() {
  await pool.query(`
    ALTER TABLE order_items ALTER COLUMN total_amount SET STATISTICS 1000;
  `);
  await pool.query(`
    CREATE STATISTICS IF NOT EXISTS oi_prod_wh (dependencies, ndistinct, mcv)
      ON product_id, warehouse_id FROM order_items;
  `);
  await pool.query(`
    ALTER TABLE order_items SET (autovacuum_analyze_scale_factor = 0.01);
  `);
  await pool.query(`ANALYZE order_items;`);  // make the new objects/targets take effect
}
```

**Do ORMs handle this?** No. Statistics management (`ANALYZE`, `CREATE STATISTICS`, `SET STATISTICS`, autovacuum tuning) is DBA/DDL territory that no ORM models. Every ORM issues queries and trusts the planner; none of them create extended statistics, raise targets, or analyze after bulk loads. This is always raw SQL in a migration.

---

## 12. ORM Comparison

The headline for this topic is unusual: **no ORM manages statistics at all.** ANALYZE, `CREATE STATISTICS`, `SET STATISTICS`, `n_distinct` overrides, and autovacuum tuning are DDL and catalog operations that live entirely outside the ORM's query-building world. Every ORM emits SQL and then *trusts the planner* — and the planner is only as good as the statistics underneath it. So the real question for each ORM is not "can it build statistics?" (none can, natively) but **"where do you put the raw SQL that hardens the stats, and does the ORM's migration system run it reliably?"**

The second, subtler point: ORMs *shape the queries* that the statistics have to serve. An ORM that fires N+1 point-lookups leans on single-column MCV/`n_distinct` accuracy; an ORM that builds big multi-condition `WHERE` clauses leans on extended statistics to beat the independence assumption. So the ORM choice indirectly changes *which* statistics matter.

---

### Prisma

**Can Prisma manage statistics?** — No. There is no `prisma.$analyze()` or schema-level statistics target. You drop raw SQL into a migration file.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// One-off / pipeline ANALYZE after a bulk import via $executeRawUnsafe
// (ANALYZE takes no bind parameters, and the table name is a trusted constant).
await prisma.$executeRawUnsafe('ANALYZE orders');

// Read the planner's ground truth from application code:
const stats = await prisma.$queryRaw`
  SELECT attname, n_distinct, most_common_vals, correlation
  FROM pg_stats
  WHERE tablename = 'orders' AND attname IN ('status', 'customer_id', 'created_at')
`;
```

For the *persistent* hardening (targets and extended stats), you hand-edit a Prisma migration — Prisma cannot generate this from `schema.prisma`:

```sql
-- prisma/migrations/20260719_stats_hardening/migration.sql  (hand-written)
ALTER TABLE orders ALTER COLUMN total_amount SET STATISTICS 1000;
CREATE STATISTICS IF NOT EXISTS orders_cust_status (dependencies, ndistinct, mcv)
  ON customer_id, status FROM orders;
ALTER TABLE orders SET (autovacuum_analyze_scale_factor = 0.02);
ANALYZE orders;
```

**Where it breaks**: `prisma migrate` will *not* detect drift in these objects — extended statistics and per-column targets are invisible to Prisma's schema diff, so they never appear in an auto-generated migration and are silently dropped if you ever reset from `schema.prisma` alone. You must treat the hardening migration as append-only, hand-maintained SQL.

**Verdict**: Prisma is a pure consumer of statistics. Put ANALYZE in your ingest code via `$executeRawUnsafe`, put targets/extended-stats in hand-written migrations, and never expect the schema diff to preserve them.

---

### Drizzle ORM

**Can Drizzle manage statistics?** — No native API, but Drizzle's `sql` tag and its plain-SQL migration files make it the least awkward place to keep the hardening.

```typescript
import { db } from './db';
import { sql } from 'drizzle-orm';

// After a bulk load in a job:
await db.execute(sql`ANALYZE order_items`);

// Inspect drift for an ops check:
const drift = await db.execute(sql`
  SELECT relname, n_mod_since_analyze, last_autoanalyze
  FROM pg_stat_user_tables
  WHERE relname IN ('orders', 'order_items', 'payments')
  ORDER BY n_mod_since_analyze DESC
`);
```

Because `drizzle-kit` migrations are just `.sql` files, the extended-stats DDL lives right beside the schema migrations:

```sql
-- drizzle/0007_stats_hardening.sql
ALTER TABLE order_items ALTER COLUMN total_amount SET STATISTICS 1000;
CREATE STATISTICS IF NOT EXISTS oi_prod_cat (dependencies, ndistinct, mcv)
  ON product_id, category_id FROM order_items;
ALTER TABLE order_items SET (autovacuum_analyze_scale_factor = 0.01);
ANALYZE order_items;
```

**Where it breaks**: `drizzle-kit` diffs tables/columns/indexes, not statistics objects or per-column targets — so like Prisma, these must be hand-authored migrations it will never regenerate. The upside is Drizzle never *fights* you: raw SQL migrations are first-class, not an escape hatch.

**Verdict**: Best-fit of the bunch for keeping stats DDL version-controlled next to the schema, purely because raw SQL migrations are the native format.

---

### Sequelize

**Can Sequelize manage statistics?** — No. Use `sequelize.query()` inside a migration's `up()`.

```javascript
// migrations/20260719-stats-hardening.js
module.exports = {
  async up(queryInterface) {
    const sql = queryInterface.sequelize;
    await sql.query(`ALTER TABLE orders ALTER COLUMN total_amount SET STATISTICS 1000`);
    await sql.query(`
      CREATE STATISTICS IF NOT EXISTS orders_cust_status (dependencies, ndistinct, mcv)
        ON customer_id, status FROM orders`);
    await sql.query(`ALTER TABLE orders SET (autovacuum_analyze_scale_factor = 0.02)`);
    await sql.query(`ANALYZE orders`);
  },
  async down(queryInterface) {
    await queryInterface.sequelize.query(`DROP STATISTICS IF EXISTS orders_cust_status`);
    await queryInterface.sequelize.query(`ALTER TABLE orders ALTER COLUMN total_amount SET STATISTICS -1`);
  },
};
```

**Where it breaks**: Sequelize's LEFT-JOIN-by-default `include` (see Topic 11) *changes which statistics the planner leans on* — a forgotten `required: true` turns an INNER into an OUTER join, and the outer-join row estimate depends on different selectivity paths, so a plan that was fine can regress purely from the join-type default. The statistics aren't wrong; the ORM changed the query shape underneath them.

**Verdict**: Works via `sequelize.query()` in migrations; just remember the ORM's join defaults quietly reshape what the planner has to estimate.

---

### TypeORM

**Can TypeORM manage statistics?** — No entity-level support; use `queryRunner.query()` in a migration class.

```typescript
import { MigrationInterface, QueryRunner } from 'typeorm';

export class StatsHardening20260719 implements MigrationInterface {
  async up(q: QueryRunner): Promise<void> {
    await q.query(`ALTER TABLE order_items ALTER COLUMN total_amount SET STATISTICS 1000`);
    await q.query(`
      CREATE STATISTICS IF NOT EXISTS oi_prod_cat (dependencies, ndistinct, mcv)
        ON product_id, category_id FROM order_items`);
    await q.query(`ALTER TABLE order_items SET (autovacuum_analyze_scale_factor = 0.01)`);
    await q.query(`ANALYZE order_items`);
  }
  async down(q: QueryRunner): Promise<void> {
    await q.query(`DROP STATISTICS IF EXISTS oi_prod_cat`);
  }
}
```

**Where it breaks**: `synchronize: true` (TypeORM's auto-schema mode) has *no concept* of statistics objects — it will happily rebuild tables while silently dropping every `CREATE STATISTICS` and per-column target. Never use `synchronize` in production, and if you do, expect all your stats hardening to vanish on the next sync.

**Verdict**: Fine through explicit migration classes; actively dangerous under `synchronize: true`.

---

### Knex.js

**Can Knex manage statistics?** — No builder method, but `knex.raw()` in a migration is clean and the closest to hand-written SQL.

```javascript
// migrations/20260719_stats_hardening.js
exports.up = async (knex) => {
  await knex.raw(`ALTER TABLE orders ALTER COLUMN total_amount SET STATISTICS 1000`);
  await knex.raw(`
    CREATE STATISTICS IF NOT EXISTS orders_cust_status (dependencies, ndistinct, mcv)
      ON customer_id, status FROM orders`);
  await knex.raw(`ALTER TABLE orders SET (autovacuum_analyze_scale_factor = 0.02)`);
  await knex.raw(`ANALYZE orders`);
};
exports.down = async (knex) => {
  await knex.raw(`DROP STATISTICS IF EXISTS orders_cust_status`);
  await knex.raw(`ALTER TABLE orders ALTER COLUMN total_amount SET STATISTICS -1`);
};

// Post-load ANALYZE in a pipeline (outside migrations):
await knex.raw('ANALYZE order_items');
```

**Where it breaks**: Nothing statistics-specific — Knex is transparent enough that `knex.raw()` is the intended tool, not a workaround. The only caveat is the usual one: `ANALYZE` cannot be parameterized (identifiers never bind with `?`), so any dynamic table name must be validated against an allow-list before interpolation.

**Verdict**: Most transparent. `knex.raw()` in migrations is exactly how you'd want to ship statistics DDL.

---

### ORM Summary Table

| ORM | Run ANALYZE from code | Persist targets / extended stats | Schema-diff aware of stats? | Landmine |
|-----|----------------------|----------------------------------|----------------------------|----------|
| Prisma | `$executeRawUnsafe('ANALYZE …')` | Hand-written migration `.sql` | No — invisible to `migrate diff` | Reset-from-schema drops stats |
| Drizzle | `db.execute(sql\`ANALYZE …\`)` | Native raw-SQL migration file | No — diffs tables/indexes only | None major; best fit |
| Sequelize | `sequelize.query('ANALYZE …')` | `query()` in migration `up()` | No | LEFT-JOIN default reshapes estimates |
| TypeORM | `queryRunner.query('ANALYZE …')` | Migration class `up()` | No | `synchronize:true` silently drops stats |
| Knex | `knex.raw('ANALYZE …')` | `knex.raw()` in migration | No | Cannot parameterize table names |

**The one-line takeaway**: statistics are a database-administration concern that *every* ORM delegates to the planner and *no* ORM tracks in its schema model. Own it explicitly — an `ANALYZE` step in every bulk-load path and a hand-maintained, append-only hardening migration for targets and extended statistics — because the ORM will neither generate nor preserve it for you.

---

## 13. Practice Exercises

### Exercise 1 — Easy

Given the `orders` table with a `status` column, you want to see exactly what the planner "knows" about that column before you trust any plan.

Write a query against `pg_stats` that returns, for `orders.status`: the null fraction, `n_distinct`, the most-common-values list, and their frequencies. Then, in one sentence, state which value the planner would use an index for and which it would Seq-Scan for — based only on the numbers you see.

```sql
-- Write your query here
```

---

### Exercise 2 — Medium

A query filtering `orders` on two correlated columns is producing a Nested Loop that blows up in production:

```sql
SELECT o.id, o.total_amount
FROM orders o
WHERE o.status = 'shipped'
  AND o.customer_id = 90210;
```

`EXPLAIN ANALYZE` shows `rows=2` estimated vs `rows=640` actual — a ~300x underestimate, because customer 90210 is a wholesale account whose orders are almost all `'shipped'` (the two columns are correlated, and the planner multiplied their selectivities).

Write the sequence of statements that (a) creates the extended statistics object that teaches the planner about the `customer_id`/`status` relationship, (b) makes it take effect, and (c) re-runs the estimate check. Name the object `orders_cust_status`.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard

You run a nightly pipeline that `COPY`s ~40M new rows into `order_items` (partitioned monthly). Hot analytics queries filter on `(product_id, category_id)` — strongly correlated, since each product belongs to one category — and do range scans on `total_amount`, which has a fat right tail (a few whale line-items). Autovacuum's default 10%-churn threshold is too slow for this table.

Write a single hardening block (to run once as a migration, plus the per-load ANALYZE) that:
1. Explicitly analyzes `order_items` after the load (bulk loads don't auto-analyze).
2. Raises the statistics target on `total_amount` to capture the long tail.
3. Creates extended statistics on the correlated `(product_id, category_id)` pair.
4. Makes autovacuum re-analyze after 1% churn instead of 10%.
5. Re-runs ANALYZE so the raised target and new object take effect.

Then write one `EXPLAIN ANALYZE` that would let you confirm the estimate is now within ~2x of actual for a query filtering all three columns.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview

In a live incident review, you are handed this plan for a query that "was fast for a year and is now taking 40 seconds," with nothing in the application code changed:

```
Nested Loop  (cost=0.43..2100.00 rows=38 width=52)
             (actual time=0.5..39800.0 rows=1310000 loops=1)
  ->  Seq Scan on orders o
        (cost=0.00..1800.00 rows=38 width=20)
        (actual time=0.2..900.0 rows=1310000 loops=1)
        Filter: (created_at >= '2026-07-01')
  ->  Index Scan using users_pkey on users u
        (actual rows=1 loops=1310000)
```

Write, in SQL, the *diagnostic* query you would run first (against a system catalog) to confirm your hypothesis about the root cause, and then the *single* statement that fixes it. In a comment, name the root cause and explain why the planner chose a Nested Loop.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What is the difference between `ANALYZE` and `VACUUM`?**

A: They do different jobs and are often confused because `VACUUM ANALYZE` runs them together. `VACUUM` reclaims storage occupied by dead tuples (rows deleted or updated by MVCC) and updates the visibility map. `ANALYZE` collects statistics about the *contents* of tables — the histogram, `n_distinct`, `most_common_vals`, `correlation` — and stores them in `pg_statistic` (readable via the `pg_stats` view). The planner reads those statistics to estimate row counts and choose plans. `VACUUM` keeps the table physically tidy; `ANALYZE` keeps the planner's mental model of the data accurate. Autovacuum runs both automatically, but on different triggers.

---

**Q: You loaded 10 million rows into an empty `orders` table with `COPY` and the first query is doing a Seq Scan when you expected an index. What is the most likely cause?**

A: Statistics are stale — specifically, they don't exist yet. A bulk load via `COPY` does not trigger autovacuum/autoanalyze synchronously; the statistics still reflect an empty (or tiny) table, so the planner thinks the table has almost no rows and picks a Seq Scan because for a "tiny" table that is cheapest. The fix is to run `ANALYZE orders;` immediately after a large load. This is one of the most common real-world causes of a bad first-query plan, and it is why data-loading scripts should always `ANALYZE` (or `VACUUM ANALYZE`) after bulk inserts rather than waiting for autovacuum to notice.

---

**Q: Where does PostgreSQL store the "most common values" for a column, and how do you see them?**

A: They live in the system catalog `pg_statistic`, but that catalog stores them in an internal, hard-to-read form. You read them through the `pg_stats` view, which unpacks the arrays into human-readable columns:

```sql
SELECT most_common_vals, most_common_freqs, n_distinct, correlation
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';
```

`most_common_vals` (MCV) is the list of the most frequent values; `most_common_freqs` is their fractional frequency. For a `status` column with a few dominant values, the MCV list lets the planner estimate `WHERE status = 'shipped'` very accurately.

---

### Principal Level

**Q: A query filters `WHERE city = 'SF' AND state = 'CA'`. The planner estimates 20 rows but the query actually returns 500,000. Both single-column predicates are individually well-estimated. Explain the root cause and the fix.**

A: This is the classic *correlated-column mis-estimate*. The planner assumes independence between columns: it computes `selectivity(city='SF') × selectivity(state='CA') × total_rows`. But `city` and `state` are functionally correlated — every 'SF' row is also 'CA', so the true selectivity of the pair equals the selectivity of `city='SF'` alone, not the product. Multiplying two already-small fractions produces an estimate that is orders of magnitude too low. That drives bad choices: a Nested Loop where a Hash Join was correct, or an index scan where a Seq Scan was correct.

The fix is extended statistics:

```sql
CREATE STATISTICS stat_city_state (dependencies, ndistinct)
  ON city, state FROM users;
ANALYZE users;
```

The `dependencies` type teaches the planner the functional dependency (city → state) so it stops multiplying the selectivities; `ndistinct` teaches it the true number of distinct *combinations* (which is far smaller than `n_distinct(city) × n_distinct(state)`). After `ANALYZE`, re-check with `EXPLAIN ANALYZE` that estimate and actual are within ~2–3x.

---

**Q: `default_statistics_target` is 100 by default. When would you raise it, when would you raise it only for one column, and what is the cost?**

A: `default_statistics_target` controls two things: the number of buckets in the histogram and the maximum size of the `most_common_vals` list. Raising it makes estimates more accurate on columns with skewed or high-cardinality distributions, at the cost of a larger `pg_statistic`, slower `ANALYZE`, and slightly more planning time (the planner scans longer MCV/histogram arrays).

- Raise it **globally** only if many columns across the database are mis-estimated — this is rare and expensive; usually a mistake.
- Raise it **per-column** when one specific column has a long tail the default 100-bucket histogram can't capture — e.g., a `products.price` column where a handful of prices dominate but the tail matters, or a `sessions.user_id` with severe skew:

```sql
ALTER TABLE sessions ALTER COLUMN user_id SET STATISTICS 1000;
ANALYZE sessions;
```

The per-column knob is almost always the right tool: you pay the `ANALYZE`/planning cost only where it buys accuracy. Values above ~1000 rarely help and start to hurt planning time.

---

**Q: `n_distinct` for `orders.customer_id` is stored as `-0.8`. What does the negative value mean, and why does it matter for a growing table?**

A: A negative `n_distinct` is a *ratio*, not an absolute count. `-0.8` means "the number of distinct values equals 80% of the number of rows" — the planner recomputes the absolute distinct count as `0.8 × reltuples` every time it plans. A positive value (e.g. `5000`) is an absolute count fixed until the next `ANALYZE`.

This matters for growing tables. On a table that constantly grows, an absolute `n_distinct` goes stale between analyzes and under-estimates cardinality; a negative ratio scales automatically with the table's row count and stays roughly correct as the table grows. ANALYZE stores a ratio when it detects that distinctness scales with table size (common for foreign keys and near-unique columns). You can also pin a known-correct value manually when sampling gets it wrong — ANALYZE only samples, so on a huge table with many rare values it can badly under-estimate `n_distinct`:

```sql
ALTER TABLE orders ALTER COLUMN customer_id SET (n_distinct = -0.8);
ANALYZE orders;
```

---

**Q: What is `correlation` in `pg_stats`, and how does a value near +1 vs near 0 change the planner's cost for an index scan?**

A: `correlation` measures how well the physical on-disk row order matches the logical order of the column's values, on a scale of -1 to +1. A value near ±1 (e.g. `created_at` on an append-only `orders` table, where new rows land at the end of the heap in timestamp order) means rows with nearby values are physically adjacent on disk. A value near 0 (e.g. a random UUID or a `status` column) means matching rows are scattered across the whole heap.

It changes the cost of an *index scan that fetches heap rows*. With correlation near +1, an index range scan touches a small contiguous block of heap pages — nearly sequential I/O, cheap. With correlation near 0, the same number of matching rows is scattered across many pages, so each heap fetch is a random I/O; the planner costs this much higher and may switch to a Seq Scan or a Bitmap Heap Scan (which sorts the page fetches back into physical order to recover sequential I/O). This is why the *same* index on a well-clustered column is far more valuable than on a scattered one, and why `CLUSTER` (physically reordering the heap by an index) can dramatically speed range queries.

---

## 15. Mental Model Checkpoint

Answer these from your own understanding before checking back through the topic. If any feel shaky, re-read the referenced section.

1. **The planner estimates 12 rows; the query actually returns 4 million. The plan uses a Nested Loop. Walk through the causal chain from "bad estimate" to "slow query." Which system view would you query first to test whether stale statistics are the cause?**

2. **You run `ANALYZE orders` on a 200M-row table and it finishes in under two seconds. It did not read all 200M rows — so how can the resulting `n_distinct` and histogram be trusted? What is the sampling doing, and for which column property is that sample *least* trustworthy?**

3. **Two columns, `payments.method` and `payments.currency`, are strongly correlated (most USD payments use one method). A query filters on both. Explain why the default estimate is too low, and write the `CREATE STATISTICS` statement that fixes it. Which of the statistic *kinds* (`dependencies`, `ndistinct`, `mcv`) does the work here, and why?**

4. **`pg_stats.correlation` for `audit_logs.created_at` is `0.98`, but for `audit_logs.actor_id` it is `0.01`. The same-size range query is cheap on the first column and expensive on the second even with an index on each. Explain why in terms of physical heap layout and I/O.**

5. **When does raising `default_statistics_target` *hurt* rather than help? Name two distinct costs, and describe the shape of a column where the extra buckets buy you nothing.**

6. **A column's `n_distinct` is stored as `-1`. What does that tell you about the column, and what would `-0.5` tell you instead? Why does the ratio form protect you on a rapidly growing table where an absolute count would not?**

7. **Autovacuum's autoanalyze fired an hour ago, yet your query still mis-estimates a value you inserted 10,000 copies of five minutes ago. Give two independent reasons the fresh value might still be invisible to the planner, and how you would confirm each.**

---

## 16. Quick Reference Card

```sql
-- ============ INSPECT STATISTICS ============
-- What the planner knows about a column:
SELECT attname, n_distinct, correlation, null_frac, avg_width,
       most_common_vals, most_common_freqs, histogram_bounds
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'status';

-- n_distinct:  positive = absolute distinct count
--              negative = ratio of distinct-to-rows (-1 = unique, -0.5 = half distinct)
-- correlation: +1/-1 = physical order matches value order (index range scan cheap)
--                   0 = scattered on disk (random I/O; may pick Seq/Bitmap scan)
-- null_frac:   fraction of NULLs; feeds IS NULL / IS NOT NULL estimates
-- most_common_vals + most_common_freqs: the MCV list for skewed columns
-- histogram_bounds: equi-depth buckets covering the non-MCV values

-- Table-level row count and last-analyze time:
SELECT relname, reltuples::bigint AS est_rows FROM pg_class WHERE relname='orders';
SELECT relname, last_analyze, last_autoanalyze, n_mod_since_analyze
FROM pg_stat_user_tables WHERE relname = 'orders';

-- ============ REFRESH STATISTICS ============
ANALYZE orders;                          -- stats only, this table
ANALYZE orders (status, created_at);     -- specific columns only
VACUUM ANALYZE orders;                    -- reclaim dead tuples + refresh stats
-- ALWAYS run after a bulk COPY/INSERT — autovacuum is not synchronous.

-- ============ TUNE SAMPLE DEPTH ============
SHOW default_statistics_target;              -- global default = 100
ALTER TABLE sessions ALTER COLUMN user_id SET STATISTICS 1000;  -- per-column
ANALYZE sessions;                            -- required to take effect
-- Higher = bigger MCV list + more histogram buckets = better estimates on
-- skewed/high-cardinality columns; costs slower ANALYZE + slower planning.
-- Prefer per-column over global. >1000 rarely helps.

-- ============ EXTENDED STATISTICS (correlated columns) ============
CREATE STATISTICS stat_city_state (dependencies, ndistinct, mcv)
  ON city, state FROM users;
ANALYZE users;                               -- required
-- dependencies -> stops multiplying selectivities for functionally dependent cols
-- ndistinct    -> true count of distinct COMBINATIONS (not the product)
-- mcv          -> most-common *combinations* of values
DROP STATISTICS stat_city_state;

-- ============ MANUAL OVERRIDES (when sampling is wrong) ============
ALTER TABLE orders ALTER COLUMN customer_id SET (n_distinct = -0.8);
ANALYZE orders;                              -- pins a ratio ANALYZE can't sample well

-- ============ AUTOVACUUM / AUTOANALYZE TRIGGER ============
-- autoanalyze fires when:
--   n_mod_since_analyze > autovacuum_analyze_threshold
--                         + autovacuum_analyze_scale_factor * reltuples
-- defaults: threshold=50, scale_factor=0.1 (10% of the table changed)
-- Lower scale_factor on huge hot tables so stats refresh sooner:
ALTER TABLE audit_logs SET (autovacuum_analyze_scale_factor = 0.02);

-- ============ DIAGNOSE A BAD ESTIMATE ============
EXPLAIN ANALYZE SELECT ... ;   -- compare rows=<est> vs actual rows=<real>
-- est << actual  -> stats stale, or correlated columns (independence assumption)
-- est >> actual  -> stale stats after mass DELETE, or over-counted MCV
```

**Decision flow when a plan is wrong:**

| Symptom | Likely cause | First action |
|---|---|---|
| Est. rows tiny, actual huge; Nested Loop | Stale stats OR correlated predicates | `ANALYZE`; if still off, `CREATE STATISTICS` |
| Bad plan right after bulk load | No stats yet (`COPY` doesn't analyze) | `ANALYZE table` |
| Long-tail column mis-estimated | Histogram too coarse | Raise per-column `SET STATISTICS` |
| Index scan slow despite matching index | Low `correlation` (scattered heap) | Consider Bitmap scan / `CLUSTER` |
| Multi-column filter wildly off | Independence assumption on correlated cols | `CREATE STATISTICS (dependencies, ndistinct)` |
| Growing table drifts off over time | Absolute `n_distinct` gone stale | Pin ratio `n_distinct = -X`, lower scale_factor |

---

## Connected Topics

- **Topic 17 — JOIN Performance Deep Dive**: Row-count estimates drive the Nested Loop vs Hash Join vs Merge Join choice. A stale or correlated-column mis-estimate is the single most common reason the planner picks the wrong join algorithm.
- **Topic 45 — EXPLAIN and EXPLAIN ANALYZE**: The tool that exposes the planner's estimates. Reading `rows=<estimate>` against `actual rows=<real>` is how you catch bad statistics in the first place.
- **Topic 46 — Index Fundamentals**: `correlation` decides whether an index range scan does cheap sequential I/O or expensive random I/O — the statistics are what make an index worth building or not.
- **Topic 47 — Bitmap Scans**: The planner falls back to a Bitmap Heap Scan precisely when `correlation` is low, sorting page fetches back into physical order to recover sequential I/O.
- **Topic 48 — Query Planner Cost Model**: Statistics are the *inputs*; the cost model is the *function* applied to them. Bad plans come from bad inputs (this topic) or a mis-tuned cost model (that topic) — you must distinguish the two.
- **Topic 49 — Selectivity and Cardinality Estimation**: The theory of how MCV lists, histograms, and `n_distinct` are combined into a selectivity fraction — the math behind the numbers stored in `pg_stats`.
- **Topic 51 — Autovacuum Tuning**: The autoanalyze half of autovacuum is what keeps these statistics fresh; its scale-factor thresholds determine how stale your planner's mental model is allowed to get.
- **Topic 07 — NULL in Depth**: `null_frac` in `pg_stats` is how the planner estimates `IS NULL` / `IS NOT NULL` predicates — the statistics layer's view of three-valued logic.

