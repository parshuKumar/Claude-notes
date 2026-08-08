# 15 — Selectivity, Cardinality and Statistics
## Phase: Indexes

---

## ELI5 — The Simple Analogy

You're a courier deciding how to deliver 500 parcels in a city.

If **three** parcels go to Koramangala, you drive there directly, drop them, come back. Targeted trips.

If **four hundred** go to Koramangala, you don't make 400 targeted trips — you load the van and do one systematic sweep street by street.

The decision hinges entirely on **how many**. And here's the thing: you decide *before you leave*, using an estimate from the manifest. If the manifest says "3 parcels" but there are actually 400, you make 400 separate trips and your day is destroyed. **The plan wasn't wrong. The estimate was.**

A database does exactly this. The planner reads its "manifest" — the statistics table — estimates how many rows will match, and picks a strategy. When the estimate is off by 100×, the plan is catastrophically wrong, and no index you add will fix it, because the index isn't the problem.

---

## Where this fits in the big picture

```
   10 what an index is (the break-even)
   11 B-tree · 12 lookup · 13 types · 14 column order
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 15 SELECTIVITY & STATISTICS              │ ← YOU ARE HERE
        │ the INPUT to every planner decision      │
        └────────────────────┬─────────────────────┘
                             │
              ┌──────────────┼───────────────┐
              ▼              ▼               ▼
        17 when indexes  18 the planner  19 join algorithms
        hurt             (uses these)    (row estimates decide
                                          the join order)
```

Everything in Phase 2 so far assumed the planner *knows* how many rows match. **This topic is where that number comes from, and what happens when it's wrong** — which is the single most common cause of a catastrophically bad plan.

---

## What is this?

**Cardinality** — how many *distinct* values a column has. `status` has 4; `user_id` has 200,000.

**Selectivity** — what *fraction* of rows a predicate matches. `WHERE status='paid'` → 0.25. `WHERE user_id=4471` → 0.000005.

**Statistics** — the sampled summary PostgreSQL keeps about each column (`pg_statistic`, readable via `pg_stats`), collected by `ANALYZE`. The planner uses these to *estimate* selectivity, and every plan choice follows from that estimate.

The relationship: `estimated_rows = total_rows × estimated_selectivity`, and estimated selectivity comes from the statistics.

---

## Why does it matter for a backend developer?

Because a wrong row estimate produces a wrong plan, and a wrong plan is 100–10,000× slower — while `EXPLAIN` looks perfectly reasonable.

```
 THE SIGNATURE OF EVERY STATISTICS PROBLEM
   Nested Loop  (cost=0.86..48.2 rows=1 width=64)
                (actual time=0.04..8412.9 rows=184220 loops=1)
                              ^^^^^^^^^^^^^^^^^^^^^^^^^
                              estimated 1, got 184,220

 The planner chose a Nested Loop because it expected ONE row on the
 inner side. It got 184,220, so it executed the inner scan 184,220
 times. A Hash Join would have taken 40 ms.

 ⇒ No index fixes this. No `random_page_cost` fixes this.
   The ESTIMATE was wrong, so every decision above it was wrong.
```

Three concrete situations you will hit:

1. **A query is fast for months, then suddenly slow after a bulk load** — `ANALYZE` hasn't run, so the planner thinks the table is still empty.
2. **A query is fast for most users and 4 seconds for one** — that user has 38% of the rows, and the planner used an *average* estimate.
3. **A two-column filter is 200× off** — `city='Mumbai' AND state='Maharashtra'` are correlated, and PostgreSQL multiplies their selectivities as if they were independent.

Each has a specific, different fix. Knowing which is which is the skill.

---

## The physical reality

### What `ANALYZE` actually collects

```
 ANALYZE orders;

 1. Sample the table: default_statistics_target × 300 rows
    = 100 × 300 = 30,000 rows, chosen by a two-stage block sampler.
    ⚠ It is a SAMPLE. On a 900M-row table it reads 30,000 rows —
      0.003%. Everything below is an extrapolation.

 2. For each column, compute and store in pg_statistic:
      • null_frac            fraction of NULLs
      • avg_width            average bytes
      • n_distinct           distinct values (or negative = ratio)
      • most_common_vals     the top N values (MCV list)
      • most_common_freqs    their frequencies
      • histogram_bounds     N+1 boundaries dividing the REST into
                             equal-frequency buckets
      • correlation          how well physical order matches value order

 3. Update pg_class.reltuples and relpages.
```

```sql
SELECT attname, null_frac, n_distinct, correlation,
       most_common_vals, most_common_freqs
FROM pg_stats WHERE tablename='orders' AND attname IN ('status','user_id','created_at');
```
```
  attname   | null_frac | n_distinct | correlation |   most_common_vals    |  most_common_freqs
------------+-----------+------------+-------------+-----------------------+---------------------
 status     |         0 |          4 |       0.412 | {delivered,cancelled, | {0.9412,0.0301,
                                                    |  shipped,awaiting}    |  0.0237,0.0050}
 user_id    |         0 |     198412 |       0.002 | {4471,8812,...}       | {0.0004,0.0003,...}
 created_at |         0 |         -1 |       0.998 |                       |
```

**Read the interesting fields:**

- `n_distinct = 4` for status — an absolute count.
- `n_distinct = -1` for `created_at` — **negative means a ratio**: −1 = every value unique; −0.5 = half as many distinct values as rows. PostgreSQL uses ratios when distinctness scales with table size.
- `correlation = 0.998` for `created_at` — physical row order almost perfectly matches value order (rows inserted in time order). **This is what makes BRIN work** (Topic 16) and what makes index scans cheap.
- `correlation = 0.002` for `user_id` — random. An index scan here does fully random heap I/O.

### The MCV list and the histogram — two different mechanisms

```
 STATUS column, 8M rows:

 MCV LIST (exact frequencies for the top values):
   'delivered'  0.9412   → 7,529,600 rows
   'cancelled'  0.0301   →   240,800 rows
   'shipped'    0.0237   →   189,600 rows
   'awaiting'   0.0050   →    40,000 rows
   ⇒ For WHERE status='awaiting', the estimate is EXACT. No guessing.

 USER_ID column, 198,412 distinct values, MCV holds only 100:
   MCV covers the 100 most frequent → say 3% of rows
   HISTOGRAM covers the remaining 97%, in 100 equal-frequency buckets:
     bucket 0: [1 .. 1988]        ← each bucket holds ~1% of the
     bucket 1: [1989 .. 3977]        non-MCV rows
     ...
     bucket 99: [196421 .. 200000]

   WHERE user_id = 4471:
     not in MCV → estimate = (1 − mcv_total_freq) / (n_distinct − n_mcv)
                = (1 − 0.03) / (198412 − 100)
                = 0.0000049
     × 8,000,000 = 39 rows.
   ⇒ An AVERAGE. If user 4471 actually has 4 orders, or 900,000,
     the planner does not know.
```

**This is the fundamental limitation.** For any value not in the MCV list, PostgreSQL assumes uniform distribution among the rest. Skew outside the MCV list is invisible.

### Correlation — the field nobody looks at

```
 correlation ∈ [−1, 1]: how well physical row order matches value order.

  correlation = 1.0        correlation = 0.0
  heap:                     heap:
   page 0: [t1][t2][t3]      page 0: [t900][t4][t551]
   page 1: [t4][t5][t6]      page 1: [t2][t822][t19]
   values ascending          values scattered

  Index scan for a RANGE of 1,000 values:
    corr 1.0 → those 1,000 rows are on ~10 adjacent pages   → 10 reads
    corr 0.0 → those 1,000 rows are on ~1,000 random pages  → 1,000 reads

 ⇒ THE PLANNER USES THIS. `cost_index()` interpolates between the
   best case (correlation ±1: sequential-like) and the worst case
   (correlation 0: fully random) when costing an index scan.

 ⇒ Practical consequence: an index on a well-correlated column is
   far cheaper to scan than the same index on an uncorrelated column,
   and the planner knows it. This is why `CLUSTER` helps, and why
   BRIN only works on correlated columns.
```

---

## How it works — step by step

### Estimating a single predicate

```
 WHERE status = 'awaiting'
  1. Look up 'awaiting' in most_common_vals for status.
  2. FOUND at index 3 → selectivity = most_common_freqs[3] = 0.0050
  3. rows = 8,000,000 × 0.0050 = 40,000
  ⇒ EXACT, because MCV stores measured frequencies.

 WHERE user_id = 4471
  1. Look up 4471 in MCV for user_id. NOT FOUND.
  2. selectivity = (1 − sum(mcv_freqs)) / (n_distinct − num_mcv)
                 = (1 − 0.03) / (198412 − 100) = 0.00000489
  3. rows = 39
  ⇒ AN AVERAGE. Correct only if non-MCV users are uniformly distributed.

 WHERE created_at >= '2026-03-01'
  1. Binary-search histogram_bounds for '2026-03-01'.
  2. It falls 40% through bucket 62 of 100.
  3. selectivity = (100 − 62 − 0.4) / 100 = 0.376
  4. rows = 3,008,000
  ⇒ Good, IF the histogram is current. A histogram built before March
    has no bucket covering March at all — see "the moving window" below.

 WHERE user_id = $1                                  ← A PARAMETER
  1. CUSTOM PLAN (first 5 executions): the actual value is known,
     so the MCV/histogram lookup happens normally.
  2. GENERIC PLAN (6th+, if PostgreSQL judges it cheaper): no value.
     selectivity = 1 / n_distinct  (the average)
  ⇒ For a skewed column this is how one plan ends up serving both
    a user with 4 rows and a user with 900,000. (Topics 09, 18.)
```

### Multi-column estimation — where it goes badly wrong

```
 WHERE city = 'Mumbai' AND state = 'Maharashtra'

 POSTGRESQL'S DEFAULT ASSUMPTION: independence.
   sel(city='Mumbai')          = 0.08
   sel(state='Maharashtra')    = 0.14
   sel(both) = 0.08 × 0.14     = 0.0112  →  89,600 rows of 8M

 REALITY: every Mumbai row IS a Maharashtra row.
   sel(both) = sel(city='Mumbai') = 0.08  →  640,000 rows

 ⇒ UNDERESTIMATE BY 7×. And it compounds: add
   country='India' (sel 0.95) and the estimate drops to 85,120 while
   reality stays 640,000 — now 7.5× off, and every join above it
   inherits the error.

 THE FIX — extended statistics (PG10+):
   CREATE STATISTICS stat_city_state (dependencies, ndistinct, mcv)
     ON city, state FROM addresses;
   ANALYZE addresses;
   ⇒ PostgreSQL now stores the functional dependency (city → state)
     and a multivariate MCV list, and estimates correctly.
```

### The moving window — the most common statistics failure

```
 TABLE: orders, insert-ordered by created_at. ANALYZE ran on 1 March.

 histogram_bounds for created_at:
   [2025-01-01, 2025-01-15, ..., 2026-02-25, 2026-03-01]
                                             ^^^^^^^^^^ the MAXIMUM

 It is now 20 March. The query:
   WHERE created_at >= '2026-03-15'

 The value is ABOVE the histogram's maximum. PostgreSQL's estimate
 for out-of-range values is very small — often 1 row.

 ACTUAL: 19 days of orders = 380,000 rows.

 ⇒ ESTIMATE 1, ACTUAL 380,000. The planner picks a Nested Loop and
   executes the inner side 380,000 times.

 ★ THIS IS WHY "queries on recent data get slow between ANALYZE runs"
   is such a common complaint on high-insert tables. Autovacuum's
   ANALYZE threshold is
       autovacuum_analyze_threshold + 0.1 × reltuples
   — on an 8M-row table that's 800,000 new rows before it re-analyses.
```

---

## Concept breakdown

```
CARDINALITY
│  └── The number of DISTINCT values in a column.
│      HIGH (user_id, email, uuid)  → indexes work well
│      LOW  (status, boolean, enum) → indexes usually useless
│                                     (unless PARTIAL — Topic 13)

SELECTIVITY
│  └── matching_rows / total_rows.  LOWER = MORE selective = BETTER.
│      < 0.01  → index scan wins decisively
│      0.01–0.10 → bitmap scan territory
│      > 0.10  → sequential scan usually wins

n_distinct — the field with two meanings
│
├── POSITIVE  → an absolute count of distinct values
├── NEGATIVE  → a RATIO of distinct values to total rows
│                −1   = every value unique
│                −0.5 = half as many distinct values as rows
└── ⚠ Sampled from 30,000 rows. On a huge table with many rare values,
     n_distinct is notoriously underestimated. Override it:
       ALTER TABLE t ALTER COLUMN c SET (n_distinct = 500000);

MCV — most common values
│
├── The top `statistics_target` values with MEASURED frequencies
├── Estimates for MCV values are EXACT
└── ⇒ Raising statistics_target for a skewed column is often the
     single highest-value statistics fix available

HISTOGRAM
│
├── `statistics_target` equal-FREQUENCY buckets over the non-MCV values
├── Used for range predicates
└── ⚠ has a MAXIMUM. Values above it estimate to ~1 row.

CORRELATION
│
├── [−1, 1]: physical order vs value order
├── Feeds the index-scan cost model (random vs sequential heap access)
└── ⇒ near ±1 → BRIN viable, index scans cheap, CLUSTER pointless
     near 0    → index scans do random I/O; consider CLUSTER

EXTENDED STATISTICS (PG10+)   ★ the fix for correlated columns
│
├── dependencies  functional dependencies: city → state
├── ndistinct     distinct count of a COMBINATION of columns
│                 (fixes GROUP BY estimates)
└── mcv           multivariate MCV list (PG12+)
     CREATE STATISTICS s (dependencies, ndistinct, mcv) ON a, b FROM t;

WHO RUNS ANALYZE
│
├── Manual:     ANALYZE tbl;   /  ANALYZE tbl (col);
├── Autovacuum: when n_mod_since_analyze >
│               autovacuum_analyze_threshold (50)
│             + autovacuum_analyze_scale_factor (0.1) × reltuples
└── ⚠ NEVER automatically after: CREATE INDEX, bulk COPY into a new
     table, a restore, or a major data change. You must run it yourself.
```

---

## Diagrams

**Diagram 1 — big picture: how an estimate becomes a plan**

```
   pg_statistic                    pg_class
   ├── null_frac                   ├── reltuples  (8,000,000)
   ├── n_distinct                  └── relpages   (98,000)
   ├── most_common_vals/freqs
   ├── histogram_bounds                    │
   └── correlation                         │
            │                              │
            └──────────┬───────────────────┘
                       ▼
              ┌─────────────────┐
              │  SELECTIVITY    │  0.005
              │  ESTIMATION     │
              └────────┬────────┘
                       ▼
              estimated_rows = 8,000,000 × 0.005 = 40,000
                       │
         ┌─────────────┼──────────────┬────────────────┐
         ▼             ▼              ▼                ▼
   scan choice    join choice    join ORDER      work_mem
   (seq/index/    (nested loop/  (which table    allocation
    bitmap)        hash/merge)    drives)        (sort/hash)

  ★ ONE number drives FOUR independent decisions.
    Get it wrong and all four are wrong.
```

**Diagram 2 — data flow: MCV vs histogram**

```
  8,000,000 rows of `user_id`
  ┌─────────────────────────────────────────────────────────────────┐
  │ MCV (top 100 values, exact frequencies)                         │
  │  4471→0.0004  8812→0.0003  ...  covers 3% of rows               │
  ├─────────────────────────────────────────────────────────────────┤
  │ HISTOGRAM (100 equal-frequency buckets over the other 97%)      │
  │  [1..1988][1989..3977][3978..5960]...[196421..200000]           │
  │   ~77,600 rows per bucket                                        │
  └─────────────────────────────────────────────────────────────────┘

  WHERE user_id = 4471       → MCV hit    → EXACT estimate ✓
  WHERE user_id = 51229      → MCV miss   → AVERAGE estimate (39 rows)
                                            ⚠ actual could be 4 or 90,000
  WHERE user_id BETWEEN 1000 AND 5000  → histogram → good estimate ✓
  WHERE user_id = 999999999  → above max → ~1 row   ⚠
```

**Diagram 3 — before/after: what a bad estimate does to a plan**

```
 ESTIMATE: 1 row                          REALITY: 184,220 rows
 ┌──────────────────────────────┐         ┌──────────────────────────────┐
 │ Nested Loop                  │         │ Nested Loop                  │
 │   rows=1  cost=48.2          │         │   actual rows=184220         │
 │   ├─ Index Scan on orders    │         │   ├─ Index Scan on orders    │
 │   │    rows=1                │         │   │    actual rows=184220    │
 │   └─ Index Scan on users     │         │   └─ Index Scan on users     │
 │        rows=1  (×1 loop)     │         │        (×184,220 LOOPS!)     │
 │                              │         │                              │
 │ planner thinks: 48 cost units│         │ reality: 8,412 ms            │
 └──────────────────────────────┘         └──────────────────────────────┘

 WHAT THE PLANNER WOULD HAVE CHOSEN WITH A CORRECT ESTIMATE:
 ┌──────────────────────────────┐
 │ Hash Join                    │
 │   ├─ Seq Scan on orders      │  read once
 │   └─ Hash → Seq Scan users   │  built once, probed 184,220 times
 │ actual: 41 ms                │      in memory
 └──────────────────────────────┘
                                              205× faster

 ⇒ The fix is not an index. The fix is ANALYZE, or extended statistics,
   or a higher statistics_target. Adding indexes to a bad-estimate
   problem makes it WORSE — more wrong options for the planner.
```

---

## Example 1 — basic

```sql
CREATE TABLE orders (
  id bigserial PRIMARY KEY,
  user_id bigint NOT NULL,
  merchant_id bigint NOT NULL,
  status text NOT NULL,
  country text NOT NULL,
  state text NOT NULL,
  city text NOT NULL,
  created_at timestamptz NOT NULL
);

INSERT INTO orders (user_id, merchant_id, status, country, state, city, created_at)
SELECT (random()*200000)::bigint,
       (random()*2000)::bigint,
       CASE WHEN random()<0.94 THEN 'delivered' WHEN random()<0.97 THEN 'cancelled'
            WHEN random()<0.995 THEN 'shipped' ELSE 'awaiting' END,
       'India',
       CASE WHEN random()<0.35 THEN 'Maharashtra' WHEN random()<0.6 THEN 'Karnataka'
            WHEN random()<0.8 THEN 'Delhi' ELSE 'Tamil Nadu' END,
       CASE WHEN random()<0.35 THEN 'Mumbai' WHEN random()<0.6 THEN 'Bengaluru'
            WHEN random()<0.8 THEN 'New Delhi' ELSE 'Chennai' END,
       now() - (random()*400)::int * interval '1 day'
FROM generate_series(1,8000000);
ANALYZE orders;
```

**Step 1 — read the statistics.**

```sql
SELECT attname, n_distinct, correlation, null_frac,
       array_length(most_common_vals::text[], 1) AS n_mcv
FROM pg_stats WHERE tablename='orders' AND attname IN
  ('status','user_id','merchant_id','created_at','city');
```
```
   attname   | n_distinct | correlation | null_frac | n_mcv
-------------+------------+-------------+-----------+-------
 status      |          4 |      0.4881 |         0 |     4
 user_id     |     196441 |      0.0011 |         0 |   100
 merchant_id |       2000 |      0.0004 |         0 |   100
 created_at  |        401 |      0.0021 |         0 |   100
 city        |          4 |      0.2214 |         0 |     4
```

**Step 2 — MCV gives exact estimates.**

```sql
EXPLAIN SELECT * FROM orders WHERE status='awaiting';
--  rows=39812
SELECT count(*) FROM orders WHERE status='awaiting';
--  39894           ← 0.2% error. The MCV list is exact.
```

**Step 3 — non-MCV values get the average.**

```sql
-- create deliberate skew: one user with a huge order count
INSERT INTO orders (user_id, merchant_id, status, country, state, city, created_at)
SELECT 999999, 1, 'delivered', 'India','Karnataka','Bengaluru', now()
FROM generate_series(1, 400000);
ANALYZE orders;

EXPLAIN SELECT * FROM orders WHERE user_id = 999999;
--  rows=397102     ← now IN the MCV list, so it's exact ✓

EXPLAIN SELECT * FROM orders WHERE user_id = 51229;
--  rows=39         ← the average. Check it:
SELECT count(*) FROM orders WHERE user_id = 51229;
--  47              ← close enough, because non-MCV users ARE uniform here
```

**Step 4 — correlated columns break independence.**

```sql
EXPLAIN SELECT * FROM orders WHERE city='Mumbai' AND state='Maharashtra';
```
```
Seq Scan on orders  (cost=0.00..218471.00 rows=344117 width=88)
```
```sql
SELECT count(*) FROM orders WHERE city='Mumbai' AND state='Maharashtra';
```
```
  count
---------
 2801442             ← estimate 344,117 vs actual 2,801,442 = 8.1× off
```

Fix it:

```sql
CREATE STATISTICS stat_city_state (dependencies, ndistinct, mcv) ON city, state FROM orders;
ANALYZE orders;
EXPLAIN SELECT * FROM orders WHERE city='Mumbai' AND state='Maharashtra';
```
```
Seq Scan on orders  (cost=0.00..218471.00 rows=2798104 width=88)
                                          ^^^^^^^^^^^^ 0.1% error ✓
```

Inspect what it learned:

```sql
SELECT stxname, stxddependencies FROM pg_statistic_ext e
JOIN pg_statistic_ext_data d ON d.stxoid = e.oid WHERE stxname='stat_city_state';
```
```
     stxname     |         stxddependencies
-----------------+-----------------------------------
 stat_city_state | {"5 => 6": 0.998400, "6 => 5": 0.412000}
```
`5 => 6` with strength 0.998 — column 5 (city) determines column 6 (state) 99.84% of the time.

**Step 5 — the moving window.**

```sql
-- simulate: insert a month of new data WITHOUT analysing
INSERT INTO orders (user_id, merchant_id, status, country, state, city, created_at)
SELECT (random()*200000)::bigint, 1, 'delivered','India','Karnataka','Bengaluru',
       now() + (random()*30)::int * interval '1 day'
FROM generate_series(1, 300000);
-- deliberately NO ANALYZE

EXPLAIN (ANALYZE) SELECT count(*) FROM orders WHERE created_at > now() + interval '1 day';
```
```
Aggregate  (cost=... rows=1)
  ->  Index Scan using idx_created on orders
        (cost=0.43..8.45 rows=1 width=0) (actual rows=289104 loops=1)
                              ^^^^^^         ^^^^^^^^^^^^^^^^
```
**Estimated 1, actual 289,104.** The histogram's maximum predates the new data.

```sql
ANALYZE orders;
EXPLAIN (ANALYZE) SELECT count(*) FROM orders WHERE created_at > now() + interval '1 day';
--  rows=287441  (actual 289104)  ← 0.6% error
```

**Step 6 — raising `statistics_target` for a skewed column.**

```sql
ALTER TABLE orders ALTER COLUMN merchant_id SET STATISTICS 1000;
ANALYZE orders;
SELECT attname, array_length(most_common_vals::text[],1) AS n_mcv,
       array_length(histogram_bounds::text[],1) AS n_hist
FROM pg_stats WHERE tablename='orders' AND attname='merchant_id';
```
```
   attname   | n_mcv | n_hist
-------------+-------+--------
 merchant_id |  1000 |   1001      ← was 100/101
```
**All 2,000 merchants: 1,000 now have exact frequencies instead of 100.** Cost: `ANALYZE` samples 300,000 rows instead of 30,000, and planning time rises slightly.

---

## Example 2 — production scenario

**The situation.** A multi-tenant analytics API. `events` table, 1.4 billion rows. p99 is 80 ms for 39,998 tenants and **41 seconds** for two of them. Same query, same code, same index.

```sql
SELECT e.id, e.kind, e.occurred_at, u.name
FROM events e JOIN users u ON u.id = e.actor_id
WHERE e.tenant_id = $1 AND e.occurred_at >= $2
ORDER BY e.occurred_at DESC LIMIT 100;
```

**Step 1 — compare the plans for a small and a large tenant.**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT e.id, e.kind, e.occurred_at, u.name FROM events e JOIN users u ON u.id=e.actor_id
WHERE e.tenant_id = 88214 AND e.occurred_at >= '2026-03-01'
ORDER BY e.occurred_at DESC LIMIT 100;
```
```
Limit  (actual time=0.084..0.412 rows=100 loops=1)
  ->  Nested Loop  (cost=1.28..842.10 rows=104) (actual rows=100 loops=1)
        ->  Index Scan using idx_events_tenant_time on events e
              (cost=0.70..410.2 rows=104) (actual rows=100 loops=1)
        ->  Index Scan using users_pkey on users u  (loops=100)
  Buffers: shared hit=412
Execution Time: 0.44 ms                                    ← healthy tenant
```

```sql
-- the whale
EXPLAIN (ANALYZE, BUFFERS) ... WHERE e.tenant_id = 12 ...
```
```
Limit  (actual time=41208.4..41208.6 rows=100 loops=1)
  ->  Nested Loop  (cost=1.28..842.10 rows=104)
                   (actual time=0.1..41208.2 rows=100 loops=1)
        ->  Index Scan using idx_events_tenant_time on events e
              (cost=0.70..410.2 rows=104)
              (actual time=0.06..8104.2 rows=41200884 loops=1)   ← ⚠⚠
        ->  Index Scan using users_pkey on users u
              (actual time=0.0007..0.0008 rows=1 loops=41200884) ← ⚠⚠⚠
  Buffers: shared hit=98204112 read=8912004
Execution Time: 41209.1 ms
```

**`rows=104` estimated. `rows=41,200,884` actual. 396,000× off.** And `loops=41200884` — the inner index scan ran 41 million times.

**Step 2 — why.**

```sql
SELECT n_distinct, array_length(most_common_vals::text[],1) AS n_mcv
FROM pg_stats WHERE tablename='events' AND attname='tenant_id';
```
```
 n_distinct | n_mcv
------------+-------
      39412 |   100
```
```sql
SELECT tenant_id, count(*), round(100.0*count(*)/sum(count(*)) OVER (),2) AS pct
FROM events GROUP BY tenant_id ORDER BY 2 DESC LIMIT 5;
```
```
 tenant_id |   count   |  pct
-----------+-----------+-------
        12 | 532004112 | 38.00     ← ⚠ 38% of the entire table
         7 | 210004881 | 15.00
      4471 |   1400412 |  0.10
      8812 |   1204008 |  0.09
```

**Tenant 12 is 38% of a 1.4-billion-row table.** Is it in the MCV list?

```sql
SELECT unnest(most_common_vals::text[]) AS v, unnest(most_common_freqs) AS f
FROM pg_stats WHERE tablename='events' AND attname='tenant_id' LIMIT 5;
```
```
   v   |    f
-------+---------
 12    | 0.38004
 7     | 0.15001
 ...
```

**It IS in the MCV list with the correct frequency.** So the statistics are fine — which means the problem is elsewhere. Two possibilities:

**Step 3 — check for a generic plan.**

```sql
-- the app uses parameterised queries via node-postgres with named statements
PREPARE p(bigint, timestamptz) AS
  SELECT e.id FROM events e WHERE e.tenant_id=$1 AND e.occurred_at >= $2
  ORDER BY e.occurred_at DESC LIMIT 100;
EXPLAIN EXECUTE p(12, '2026-03-01');    -- ×6
```
```
Limit
  ->  Index Scan using idx_events_tenant_time on events e
        (cost=0.70..410.2 rows=104)               ← ⚠ 104 again, for tenant 12
        Index Cond: ((tenant_id = $1) AND (occurred_at >= $2))
```

**There it is.** With a **generic plan**, PostgreSQL has no value for `$1`, so it cannot consult the MCV list. It falls back to `1 / n_distinct = 1/39412`, giving 104 rows — the average tenant. **One plan, built for the average, serving a tenant with 38% of the table.**

**Step 4 — the fixes, in order of leverage.**

```sql
-- (1) FORCE A CUSTOM PLAN per execution. Costs ~0.2 ms of planning,
--     saves 41 seconds. Trivially the right trade here.
SET plan_cache_mode = 'force_custom_plan';
```
```js
// or per-connection in Node.js
await pool.query("SET plan_cache_mode = 'force_custom_plan'");
// or avoid named/prepared statements for this query specifically —
// node-postgres only prepares when you pass a `name`
```

```sql
-- (2) RAISE statistics_target so more tenants get exact MCV entries
ALTER TABLE events ALTER COLUMN tenant_id SET STATISTICS 2000;
ANALYZE events;
-- now the top 2,000 tenants have measured frequencies instead of 100
```

```sql
-- (3) EXTENDED STATISTICS for the correlated pair
CREATE STATISTICS stat_events_tenant_kind (dependencies, ndistinct, mcv)
  ON tenant_id, kind FROM events;
ANALYZE events;
```

```sql
-- (4) THE STRUCTURAL FIX: partition by tenant so the whale's data is
--     in its own partition with its own statistics (Topic 59)
--     — and per-partition stats are automatically accurate for that
--     partition, because the partition IS one tenant.
```

**Step 5 — results.**

| | Before | After (1) | After (1)+(2)+(4) |
|---|---|---|---|
| p99, small tenant | 80 ms | 82 ms | 45 ms |
| p99, tenant 12 | **41,209 ms** | **310 ms** | **48 ms** |
| Planning time | 0.02 ms | 0.31 ms | 0.28 ms |
| Estimate error, tenant 12 | 396,000× | 1.02× | 1.01× |

**Fix (1) is one line of configuration and gives 133× on the worst case.** The extra 0.29 ms of planning per query is nothing against 41 seconds.

**Step 6 — the monitoring that catches this class of bug.**

```sql
-- Tables where autovacuum hasn't ANALYZEd recently relative to churn
SELECT relname, n_live_tup, n_mod_since_analyze,
       round(100.0*n_mod_since_analyze/nullif(n_live_tup,0),1) AS pct_changed,
       last_autoanalyze
FROM pg_stat_user_tables
WHERE n_mod_since_analyze > 100000
ORDER BY pct_changed DESC LIMIT 20;
```

```sql
-- Columns whose distribution is extremely skewed → candidates for a
-- higher statistics_target, extended stats, or partitioning
SELECT tablename, attname, n_distinct,
       (most_common_freqs)[1] AS top_value_freq
FROM pg_stats
WHERE schemaname='public' AND (most_common_freqs)[1] > 0.1
ORDER BY 4 DESC;
```

---

## Common mistakes

**1. Not running `ANALYZE` after a bulk load, restore, or major migration.**
- *Symptom:* a freshly restored database is catastrophically slow; a new table performs terribly for hours.
- *Engine-level why:* `pg_class.reltuples` is 0 and there are no statistics. The planner assumes tiny tables and picks nested loops everywhere.
- *Diagnose:* `SELECT relname, last_analyze, last_autoanalyze FROM pg_stat_user_tables;`
- *Fix:* `ANALYZE;` after every restore, `COPY`, or large migration. **Add it to the migration script**, not to a runbook.

**2. Assuming autovacuum's ANALYZE is frequent enough.**
- *Symptom:* queries on recent data are slow and get better after someone runs `ANALYZE`.
- *Engine-level why:* the threshold is `50 + 0.1 × reltuples`. On a 100M-row table that's **10 million modifications** before re-analysing — days on a moderately busy table.
- *Fix:* per-table `ALTER TABLE t SET (autovacuum_analyze_scale_factor = 0.01);` on high-insert tables, especially those queried by recent timestamps.

**3. Ignoring column correlation.**
- *Symptom:* `city AND state`, `country AND currency`, `product AND category` estimates off by 5–100×.
- *Engine-level why:* PostgreSQL multiplies selectivities assuming independence.
- *Diagnose:* compare `EXPLAIN` estimated rows against `SELECT count(*)` with the same predicate.
- *Fix:* `CREATE STATISTICS ... (dependencies, ndistinct, mcv) ON a, b FROM t;` then `ANALYZE`.

**4. Adding indexes to fix a bad-estimate problem.**
- *Symptom:* five new indexes, no improvement, slower writes.
- *Engine-level why:* the planner picks the wrong *strategy* because the row count is wrong. More index options give it more ways to be wrong.
- *Diagnose:* the estimated-vs-actual ratio on the deepest plan node. If it's > 10×, fix statistics first — always.

**5. Generic plans on skewed columns.**
- *Symptom:* one tenant, one merchant, or one customer is 500× slower than everyone else.
- *Engine-level why:* after 5 executions PostgreSQL may build a generic plan using `1/n_distinct` — the *average* — which is wrong for both the whale and the minnow.
- *Diagnose:* `PREPARE`, `EXECUTE` six times, `EXPLAIN EXECUTE` and look for `$1` in the `Index Cond` with an average-looking row estimate.
- *Fix:* `SET plan_cache_mode = 'force_custom_plan'` for that workload, or don't use named prepared statements there.

**6. Trusting `n_distinct` on very large tables.**
- *Symptom:* the planner thinks a column has 50,000 distinct values when it has 40 million.
- *Engine-level why:* `n_distinct` is extrapolated from a 30,000-row sample. Estimating distinctness from a sample is a known-hard statistical problem and PostgreSQL's estimator is conservative.
- *Diagnose:* compare `pg_stats.n_distinct` with `SELECT count(DISTINCT col)`.
- *Fix:* `ALTER TABLE t ALTER COLUMN c SET (n_distinct = -0.9);` (negative = ratio). This is a manual override the planner trusts absolutely — so verify it.

**7. Raising `default_statistics_target` globally to 1000.**
- *Symptom:* `ANALYZE` takes 20 minutes; planning time triples on complex queries.
- *Engine-level why:* the sample is `target × 300` rows *per table*, and MCV/histogram lookups are linear in target during planning.
- *Fix:* raise it **per column**, on the columns that are actually skewed. `ALTER TABLE t ALTER COLUMN c SET STATISTICS 1000;`

---

## Hands-on proof

**PROVE IT #1 — read your own statistics.**
```sql
SELECT attname, n_distinct, correlation, null_frac, avg_width,
       array_length(most_common_vals::text[],1) AS n_mcv
FROM pg_stats WHERE tablename='orders' ORDER BY attname;
```

**PROVE IT #2 — estimate vs actual, systematically.**
```sql
CREATE OR REPLACE FUNCTION est_vs_actual(q text)
RETURNS TABLE(estimated bigint, actual bigint, ratio numeric) AS $$
DECLARE p json; e bigint; a bigint;
BEGIN
  EXECUTE 'EXPLAIN (FORMAT JSON) ' || q INTO p;
  e := (p->0->'Plan'->>'Plan Rows')::bigint;
  EXECUTE 'SELECT count(*) FROM (' || q || ') s' INTO a;
  RETURN QUERY SELECT e, a, round(GREATEST(e,a)::numeric/GREATEST(LEAST(e,a),1),1);
END $$ LANGUAGE plpgsql;

SELECT * FROM est_vs_actual($$SELECT * FROM orders WHERE status='awaiting'$$);
SELECT * FROM est_vs_actual($$SELECT * FROM orders WHERE city='Mumbai' AND state='Maharashtra'$$);
```
```
 estimated | actual  | ratio
-----------+---------+-------
     39812 |   39894 |   1.0     ← MCV: perfect
    344117 | 2801442 |   8.1     ← independence assumption: broken
```

**PROVE IT #3 — extended statistics fix it.**
```sql
CREATE STATISTICS s_cs (dependencies, ndistinct, mcv) ON city, state FROM orders;
ANALYZE orders;
SELECT * FROM est_vs_actual($$SELECT * FROM orders WHERE city='Mumbai' AND state='Maharashtra'$$);
-- ratio 1.0
```

**PROVE IT #4 — stale statistics.**
```sql
CREATE TABLE fresh AS SELECT i, now() - (i||' seconds')::interval AS t
                      FROM generate_series(1,1000000) i;
CREATE INDEX ON fresh(t);
ANALYZE fresh;
INSERT INTO fresh SELECT i, now() + (i||' seconds')::interval FROM generate_series(1,500000) i;
-- no ANALYZE
EXPLAIN (ANALYZE) SELECT count(*) FROM fresh WHERE t > now();
--  rows=1  ... actual rows=499998        ← the moving window
ANALYZE fresh;
EXPLAIN (ANALYZE) SELECT count(*) FROM fresh WHERE t > now();
--  rows=497102 ... actual rows=499998
```

**PROVE IT #5 — the generic-plan trap.**
```sql
PREPARE g(bigint) AS SELECT count(*) FROM orders WHERE user_id=$1;
EXPLAIN EXECUTE g(999999);   -- ×1: custom plan, rows≈397000
EXPLAIN EXECUTE g(999999);   -- ×6 → generic plan, rows≈41 (the average)
SET plan_cache_mode='force_custom_plan';
EXPLAIN EXECUTE g(999999);   -- rows≈397000 again
RESET plan_cache_mode;
```

**PROVE IT #6 — correlation and index-scan cost.**
```sql
CREATE TABLE corr_hi AS SELECT i AS k, repeat('x',50) AS pad FROM generate_series(1,3000000) i;
CREATE TABLE corr_lo AS SELECT i AS k, repeat('x',50) AS pad FROM generate_series(1,3000000) i
                        ORDER BY random();
CREATE INDEX ON corr_hi(k); CREATE INDEX ON corr_lo(k);
ANALYZE corr_hi; ANALYZE corr_lo;
SELECT tablename, correlation FROM pg_stats WHERE attname='k';
--  corr_hi | 1.0     corr_lo | 0.0002

EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM corr_hi WHERE k BETWEEN 100000 AND 200000;
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM corr_lo WHERE k BETWEEN 100000 AND 200000;
```
Same row count; `corr_lo` reads roughly one heap page per row while `corr_hi` reads adjacent pages.

**PROVE IT #7 — `n_distinct` underestimation.**
```sql
SELECT n_distinct FROM pg_stats WHERE tablename='orders' AND attname='user_id';
SELECT count(DISTINCT user_id) FROM orders;
-- compare. On big tables with a long tail, the sampled value is often low.
```

---

## The design decision framework

```
THE DIAGNOSTIC ORDER — run EXPLAIN (ANALYZE) and go top-down:

  ① Find the DEEPEST node where estimated and actual rows diverge > 10×.
     Everything above it inherits the error. Fix that node first.

  ② WHY is it wrong? Four causes, four different fixes:

     (a) STALE STATISTICS
         Check: SELECT last_analyze, last_autoanalyze, n_mod_since_analyze
                FROM pg_stat_user_tables WHERE relname='...';
         Fix:   ANALYZE tbl;
         Prevent: ALTER TABLE tbl SET (autovacuum_analyze_scale_factor=0.01);

     (b) SKEW OUTSIDE THE MCV LIST
         Check: is the value in most_common_vals? Is the distribution
                heavy-tailed?
         Fix:   ALTER TABLE tbl ALTER COLUMN c SET STATISTICS 1000;
                ANALYZE tbl;

     (c) CORRELATED COLUMNS (independence assumption)
         Check: est(A AND B) ≈ est(A) × est(B) but reality is much higher
         Fix:   CREATE STATISTICS s (dependencies, ndistinct, mcv)
                  ON a, b FROM tbl;  ANALYZE tbl;

     (d) GENERIC PLAN ON A SKEWED PARAMETER
         Check: EXPLAIN EXECUTE after 6 runs shows $1 and an average estimate
         Fix:   SET plan_cache_mode='force_custom_plan';
                or stop using named prepared statements for that query

  ③ ONLY IF ESTIMATES ARE NOW GOOD and the plan is still bad:
     look at cost constants (random_page_cost — Topic 07) and index
     design (Topic 14).

WHEN TO RAISE statistics_target:
  ✓ A column with a heavy-tailed distribution (a few values dominate)
  ✓ A column where est-vs-actual is consistently off for specific values
  ✓ A join key whose estimate drives join order
  ✗ NEVER globally to 1000 — do it per column
  Cost: ANALYZE samples target×300 rows; planning time rises slightly

WHEN TO CREATE EXTENDED STATISTICS:
  ✓ Two+ columns frequently filtered TOGETHER
  ✓ Where one functionally determines the other (city→state, sku→category)
  ✓ GROUP BY on multiple columns with bad estimates (use `ndistinct`)
  Cost: extra ANALYZE work; negligible planning cost

WHEN TO OVERRIDE n_distinct:
  ✓ You have measured count(DISTINCT c) and it differs from pg_stats by >5×
  ✓ Typically on very large tables with high-cardinality columns
  ⚠ It's a manual override the planner trusts completely. Re-verify it
    after major data changes.

THE SIGNAL TO LOOK FOR:
      The estimated:actual ratio on the DEEPEST plan node.
  • within 2×    → statistics are fine; look elsewhere
  • 10–100×      → one of the four causes above
  • > 1000×      → almost always (a) stale or (d) generic plan
  ★ Fix statistics BEFORE touching indexes. Always. An index added to
    compensate for a bad estimate makes the problem harder to find.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
On the 8M-row `orders` table, use `pg_stats` to report `n_distinct`, `correlation`, and the MCV frequencies for `status`, `user_id`, and `created_at`. For each, predict the estimated row count for a specific equality predicate *before* running `EXPLAIN`, then check. Explain any column where your prediction was wrong.

### Exercise 2 — medium (apply it)
Create a table with two strongly correlated columns (e.g. `product_id` and `category_id`, where each product belongs to exactly one category). Then:
(a) measure the estimate vs actual for a two-column equality predicate,
(b) compute what the independence assumption would predict, by hand, from `pg_stats`,
(c) add extended statistics and re-measure,
(d) inspect `pg_statistic_ext_data` and interpret the dependency strength,
(e) explain what would happen to a *join* whose row estimate depended on this predicate.

### Exercise 3 — hard (production simulation)
A multi-tenant `events` table has 2 billion rows across 60,000 tenants. Tenant distribution: one tenant has 41% of rows, four have 8% each, the rest share the remainder. The hot query joins `events` to `users` and `sessions`, filtered by `tenant_id = $1` and a time range, sorted by time with `LIMIT 100`. p99 is 60 ms for most tenants and 38 s for the top five.

(a) Give the exact `EXPLAIN` signature you'd expect for the slow case, including which node shows the divergence and what `loops=` would be.
(b) Name all four possible causes and the specific query that rules each in or out.
(c) The whale tenant *is* in the MCV list with the correct frequency. Explain why the query can still be catastrophically slow, and prove it with a `PREPARE`/`EXECUTE` sequence.
(d) Give four fixes in increasing order of structural change, with the expected improvement and the cost of each.
(e) `SET plan_cache_mode='force_custom_plan'` adds ~0.3 ms of planning to every query at 20,000 queries/sec. Compute the aggregate CPU cost and argue whether it's worth it.
(f) Design the partitioning scheme that makes this problem disappear structurally, and explain precisely *why* per-partition statistics solve it.
(g) Write the monitoring query that would have flagged this before a customer did.

---

## Mental model checkpoint

1. Define cardinality and selectivity. Which one does the planner actually compute, and from what?
2. What does `n_distinct = -0.5` mean, and why does PostgreSQL use negative values at all?
3. What's the difference between the MCV list and the histogram? Which values get exact estimates?
4. Why is `WHERE city='Mumbai' AND state='Maharashtra'` underestimated? What's the mechanism and the fix?
5. What is `correlation`, and name two planner decisions it affects.
6. Explain the "moving window" problem on a high-insert table. Why doesn't autovacuum's ANALYZE prevent it?
7. A value is correctly represented in the MCV list, yet the query is still 400,000× off. What's happening, and how do you prove it in three commands?

---

## Quick reference card

| Statistic | Meaning |
|---|---|
| `null_frac` | fraction of NULLs |
| `n_distinct` | positive = count; **negative = ratio to row count** |
| `most_common_vals/freqs` | top-N values with **exact** frequencies |
| `histogram_bounds` | equal-frequency buckets over the rest |
| `correlation` | [−1,1] physical order vs value order |
| `reltuples`/`relpages` | table size estimates (in `pg_class`) |

**Selectivity thresholds**

| Selectivity | Plan |
|---|---|
| < 0.01 | Index Scan |
| 0.01–0.20 | Bitmap Heap Scan |
| > 0.10–0.20 | Seq Scan |

**The four causes of a bad estimate**

| Cause | Check | Fix |
|---|---|---|
| Stale stats | `n_mod_since_analyze` | `ANALYZE`; lower `analyze_scale_factor` |
| Skew outside MCV | distribution query | `SET STATISTICS 1000` per column |
| Correlated columns | est(A)×est(B) vs actual | `CREATE STATISTICS` |
| Generic plan | `EXPLAIN EXECUTE` ×6 | `plan_cache_mode='force_custom_plan'` |

**Commands**

```sql
ANALYZE tbl;                                          -- refresh
ALTER TABLE t ALTER COLUMN c SET STATISTICS 1000;      -- more MCV/histogram
ALTER TABLE t ALTER COLUMN c SET (n_distinct = -0.9);  -- manual override
CREATE STATISTICS s (dependencies, ndistinct, mcv) ON a,b FROM t;
ALTER TABLE t SET (autovacuum_analyze_scale_factor = 0.01);
SET plan_cache_mode = 'force_custom_plan';
```

**Defaults**

| Setting | Default |
|---|---|
| `default_statistics_target` | 100 |
| ANALYZE sample | target × 300 rows |
| `autovacuum_analyze_threshold` | 50 |
| `autovacuum_analyze_scale_factor` | 0.1 |

**The rule:** fix statistics before indexes. Always.

---

## When would I use this at work?

1. **"The query got slow after the migration."** Your first move is `SELECT last_analyze FROM pg_stat_user_tables` — not `EXPLAIN`. A restore or bulk load with no `ANALYZE` produces exactly this, and it's a 10-second fix that people spend days hunting for.

2. **"It's slow for one customer."** The est-vs-actual ratio plus a `PREPARE`/`EXECUTE ×6` test identifies a generic-plan problem in two minutes. `plan_cache_mode` is a one-line fix that turns a 41-second query into 310 ms.

3. **Reviewing a slow join.** Before anyone proposes new indexes or a rewrite, you check the deepest node's estimate. If it's off by 100×, you say so and fix the statistics — and the join order corrects itself, often making the proposed index unnecessary.

---

## Connected topics

**Understand before this:** 10 (the selectivity break-even), 12 (scan types the estimate chooses between), 14 (composite indexes, whose usefulness depends on estimates).

**This unlocks:**
- **16** — BRIN, which depends entirely on `correlation`
- **17** — when indexes hurt: low-cardinality columns identified here
- **18** — the planner's full cost model, of which these are the inputs
- **19** — join algorithms and join *order*, both driven by row estimates
- **59** — partitioning, which gives each partition its own accurate statistics
- **67** — the performance investigation methodology
