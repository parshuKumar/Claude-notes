# 03 — The Query Planner in Depth
## Phase: Foundations (How SQL Actually Executes)
## Interview Priority: MEDIUM

---

## ELI5 — The Simple Analogy

You're driving from your house to a friend's place across town. Before you leave, you open Google Maps and it gives you 3 route options:

- Route A: Highway, 22 minutes, no traffic
- Route B: Side streets, 28 minutes, no tolls
- Route C: Back roads, 45 minutes, scenic

Google Maps picks Route A. It made this decision using data: your current location, known road speeds, current traffic, distance. It did NOT ask you HOW to drive — it figured out the optimal path on its own.

The PostgreSQL query planner is Google Maps for your data. You tell it WHERE you want to go (the SQL query). It figures out the optimal path to get there (the execution plan). It uses data — statistics about your tables, cost estimates for each operation, available indexes — to pick the cheapest route.

The difference from Google Maps: if the planner uses outdated traffic data (stale table statistics), it might route you into a traffic jam (full table scan) when a clear highway (index) was available.

Understanding the planner means understanding the data it uses, the decisions it makes, and — critically — when that data is wrong and why.

---

## Internals Connection

The query planner sits at Stage 4 of the pipeline (from Topic 01). It receives a query tree from the Rewriter and produces an execution plan for the Executor.

The planner is a **cost-based optimizer (CBO)**. It works like this:

```
Query Tree
    ↓
[Generate candidate plans]
    ↓ For each candidate:
[Estimate cost using statistics]
    ↓
[Pick the lowest-cost plan]
    ↓
Execution Plan
```

The statistics it uses live in the system catalog — specifically in `pg_statistic` (raw) and `pg_stats` (human-readable view). These statistics are collected by `ANALYZE` (run manually or by autovacuum). They describe the *shape* of your data: how many distinct values, what the distribution looks like, how ordered the data is physically.

The planner connects to everything you know about database internals:

- **B-tree indexes**: The planner checks `pg_index` to know which indexes exist and what they cover. Every access path decision (use index vs seq scan) involves evaluating the B-tree's cost.
- **Buffer pool**: The planner's cost model uses `seq_page_cost` and `random_page_cost` — these model whether pages are likely already in the buffer pool or need to be read from disk.
- **MVCC**: Dead tuple bloat affects row count estimates. If a table has 1M live rows and 5M dead rows, the planner sees the physical size and adjusts — but only if statistics are fresh.
- **WAL**: No direct connection to planning, but large VACUUM operations (which compact dead tuples and update statistics) directly influence what the planner sees.

---

## The Execution Order Context

The planner is Stage 4 of the 5-stage pipeline (Topic 01). It runs:
- **After** the Rewriter (Stage 3) — which means views are already inlined. The planner never sees views, only base tables.
- **Before** the Executor (Stage 5) — the plan it produces is the blueprint the Executor follows exactly.

The planner does NOT execute anything. It purely evaluates strategies and selects the best one. The actual data access happens in Stage 5.

The logical execution order (Topic 02) describes what the plan must *produce* — the planner's job is to figure out the most efficient physical operations that produce that result.

---

## What Is This?

The **query planner** (also called the query optimizer) is the PostgreSQL component that transforms a query tree into a physical execution plan. It uses a **cost model** — a set of mathematical formulas that assign a numerical cost to each possible execution strategy — and **table statistics** — collected data about the size, distribution, and structure of your tables — to pick the lowest-cost plan. It is a cost-based optimizer, meaning it makes decisions based on estimates, not guarantees.

---

## Why Does It Matter in Production?

The planner is the single biggest factor in query performance after schema design. A query with perfect indexes can still run for minutes if the planner chooses the wrong plan. Understanding the planner tells you:

1. **Why a query is slow even though the right index exists** — the planner decided the index wasn't worth using (stale statistics, bad cost estimates).
2. **Why two syntactically similar queries have radically different performance** — the planner treats them differently based on statistics.
3. **Why a query is fast in development and slow in production** — different data volumes → different statistics → different plans.
4. **How to intervene** — when the planner is wrong, you need to know HOW it made its decision to fix it correctly (not just throw `SET enable_seqscan = off` blindly).
5. **Why ANALYZE matters** — stale statistics = wrong estimates = wrong plan = slow queries. This is one of the most common production performance issues.

---

## The Cost Model in Depth

### Cost Units

PostgreSQL cost is measured in **arbitrary units** — not milliseconds. The base unit is:
> One sequential page read from disk = `seq_page_cost` = **1.0**

All other costs are expressed relative to this:

```
COST PARAMETERS (default values, configurable in postgresql.conf):

seq_page_cost        = 1.0    Base unit: cost to read one page sequentially
random_page_cost     = 4.0    Random page read = 4x more expensive than sequential
                               (models disk seek + rotational latency for HDDs)
                               For SSDs, set this to 1.1–2.0

cpu_tuple_cost       = 0.01   Cost to process one row (visibility check, projection)
cpu_index_tuple_cost = 0.005  Cost to process one index entry (cheaper than heap tuple)
cpu_operator_cost    = 0.0025 Cost to evaluate one operator (=, <, etc.)

parallel_tuple_cost  = 0.1    Cost to transfer a tuple from parallel worker to leader
parallel_setup_cost  = 1000.0 Fixed overhead of setting up parallel workers
```

### How a Plan Node's Cost Is Calculated

```
For a SEQUENTIAL SCAN:

startup_cost = 0
              (no setup needed — start reading immediately)

total_cost   = (pages_in_table × seq_page_cost)
             + (rows_in_table  × cpu_tuple_cost)
             + (rows_in_table  × predicate_operators × cpu_operator_cost)

Example: users table, 1000 pages, 50000 rows, WHERE active = true
  = (1000 × 1.0) + (50000 × 0.01) + (50000 × 1 × 0.0025)
  = 1000 + 500 + 125
  = 1625 cost units
```

```
For an INDEX SCAN:

startup_cost = log2(index_rows) × cpu_index_tuple_cost
               (B-tree traversal to first matching leaf)

total_cost   = startup_cost
             + (matching_index_rows × cpu_index_tuple_cost)
             + (matching_heap_rows  × random_page_cost)   ← random heap access!
             + (matching_heap_rows  × cpu_tuple_cost)

Example: users table, 50000 rows, WHERE active = true (selectivity = 0.7 → 35000 rows)
  startup = log2(50000) × 0.005 = 15.6 × 0.005 = 0.08
  index tuples = 35000 × 0.005 = 175
  heap pages = 35000 × random_page_cost × (1/rows_per_page) ≈ 35000 × 4.0 × 0.02 = 2800
  cpu tuples = 35000 × 0.01 = 350
  total ≈ 0.08 + 175 + 2800 + 350 = 3325 cost units

Seq Scan cost: 1625  ← CHEAPER → planner picks Seq Scan
Index Scan cost: 3325

WHY? Fetching 70% of the table via random index lookups is MORE expensive
than reading every page sequentially. The planner knows this.
```

```
CROSSOVER POINT:
Index scan becomes cheaper than seq scan at roughly 5-15% selectivity
(depending on table size, random_page_cost, and correlation).

This is why:
  WHERE id = 42           → Index Scan (0.002% selectivity)
  WHERE active = true     → Seq Scan (70% selectivity)
  WHERE city = 'London'   → depends on % of Londoners in your data
```

---

## Statistics: The Planner's Data Source

### Where Statistics Live

```sql
-- The raw storage (internal format, hard to read):
SELECT * FROM pg_statistic WHERE starelid = 'users'::regclass LIMIT 1;

-- The human-readable view (use this):
SELECT * FROM pg_stats WHERE tablename = 'users';

-- Key columns in pg_stats:
-- tablename       → which table
-- attname         → which column
-- null_frac       → fraction of NULLs (0.0 to 1.0)
-- n_distinct      → distinct value count
--                   positive = exact count
--                   negative = fraction of total rows (e.g., -0.95 means 95% distinct)
-- most_common_vals → array of most frequent values
-- most_common_freqs → their frequencies (must sum to ≤ 1.0)
-- histogram_bounds → bucket boundaries for distribution
-- correlation      → physical vs logical order correlation (−1 to 1)
-- avg_width        → average byte width of the column value
```

### The Statistics in Action

```sql
-- Example: see the statistics for employees.department
SELECT
  attname,
  null_frac,
  n_distinct,
  most_common_vals,
  most_common_freqs,
  histogram_bounds,
  correlation
FROM pg_stats
WHERE tablename = 'employees'
  AND attname = 'department';

-- Example output:
-- attname:           department
-- null_frac:         0.0
-- n_distinct:        8                    ← 8 distinct departments
-- most_common_vals:  {Engineering,Sales,HR,Marketing,Finance,Legal,Ops,Support}
-- most_common_freqs: {0.32,0.22,0.15,0.12,0.09,0.05,0.03,0.02}
-- histogram_bounds:  NULL                 ← no histogram (few distinct values)
-- correlation:       0.12                 ← weakly correlated with heap order

-- HOW THE PLANNER USES THIS:
-- "WHERE department = 'Engineering'" → selectivity = most_common_freqs[1] = 0.32
-- Estimated rows = 0.32 × total_rows

-- "WHERE department = 'Legal'" → selectivity = 0.05
-- Estimated rows = 0.05 × total_rows (much more selective → index scan preferred)

-- "WHERE department NOT IN ('Engineering', 'Sales')" →
-- selectivity = 1 - 0.32 - 0.22 = 0.46 (46% of rows)
-- → likely Seq Scan
```

### The Histogram

```sql
-- For high-cardinality columns, pg_stats uses a histogram instead of most_common_vals:
SELECT histogram_bounds
FROM pg_stats
WHERE tablename = 'payments' AND attname = 'amount';

-- Example output:
-- {1.00, 12.50, 25.00, 47.80, 89.95, 150.00, 245.00, 410.00, 720.00, 1500.00, 9999.00}
--  ^                                                                              ^
--  min                                                                           max
-- 10 buckets (default statistics_target = 100 gives up to 100 histogram buckets)

-- Each bucket contains approximately equal row count.
-- "WHERE amount BETWEEN 50 AND 150" →
-- The planner counts how many histogram buckets fall in this range,
-- estimates selectivity proportionally.

-- MORE HISTOGRAM BUCKETS = MORE ACCURATE ESTIMATES
-- Default: statistics_target = 100 → up to 100 buckets
-- Can increase per-column: ALTER TABLE payments ALTER COLUMN amount SET STATISTICS 500;
-- After running ANALYZE, pg_stats gets up to 500 histogram buckets for this column.
-- Use for columns in frequent range queries on large tables.
```

### Correlation

```sql
-- Correlation measures how well physical order matches logical order.
-- Range: -1.0 to 1.0
--   1.0  = perfectly correlated (rows with sequential values are on sequential pages)
--  -1.0  = perfectly anti-correlated
--   0.0  = completely random

-- WHY IT MATTERS FOR INDEX SCANS:
-- High correlation (≈1.0): index scan reads pages in order → effectively sequential I/O
--   → planner uses random_page_cost ≈ seq_page_cost (index scan is cheap)
-- Low correlation (≈0.0): index scan reads pages randomly across the heap
--   → planner uses full random_page_cost = 4.0 per page (index scan is expensive)

-- Example:
-- users.id has correlation ≈ 1.0 (rows inserted in id order, stored in id order)
--   → Index scan on id is cheap even for moderate selectivity
-- orders.customer_id has correlation ≈ 0.1 (random customer IDs, no heap order)
--   → Index scan on customer_id needs high selectivity to beat seq scan

-- Check correlation:
SELECT attname, correlation
FROM pg_stats
WHERE tablename = 'orders'
ORDER BY abs(correlation) DESC;
```

---

## How the Planner Generates Candidate Plans

### Access Path Generation

```
For each table in the query, the planner generates candidate ACCESS PATHS:

1. Sequential Scan
   Always available. Cost: linear in table size.

2. Index Scan (for each available index)
   Available if WHERE/JOIN condition matches the index.
   Cost: depends on selectivity and correlation.

3. Bitmap Index Scan
   Available for any index. Good for moderate selectivity.
   Can COMBINE multiple indexes (BitmapAnd, BitmapOr).

4. Index Only Scan
   Available if all needed columns are covered by one index.
   Avoids heap access entirely (subject to visibility map).

The planner considers ALL available indexes for the WHERE conditions,
computes cost for each, and keeps the cheapest option.
```

### Join Order Search

```
For a query joining N tables, the planner must choose the ORDER to join them.
With N tables: N! possible orderings (for N=10: 3.6 million orderings).

PostgreSQL uses DYNAMIC PROGRAMMING for N ≤ geqo_threshold (default: 12):
  • Builds up from 1-table plans to N-table plans
  • For each subset of tables, keeps only the cheapest join order
  • Total plan space explored: O(2^N) which is manageable for N ≤ 12

For N > geqo_threshold: switches to GENETIC ALGORITHM (GEQO):
  • Generates random join orders, mutates, selects fittest
  • May not find optimal plan (trade: planning time vs plan quality)
  • Can be tuned via geqo_effort, geqo_generations

For each join order, the planner also chooses the JOIN ALGORITHM:
  • Nested Loop: good for small outer side + indexed inner side
  • Hash Join: good for large tables, equality conditions, no index
  • Merge Join: good for already-sorted data, equality conditions
```

### Join Algorithm Selection

```
NESTED LOOP JOIN:
  For each row in outer table:
    Find matching rows in inner table (ideally via index)
  
  Cost: outer_rows × inner_lookup_cost
  
  Best when:
  • Small outer result set (after filtering)
  • Inner table has an index on the join key
  • LIMIT is involved (can stop early)
  
  Planner chooses when: small outer set, indexed inner

HASH JOIN:
  Phase 1 (Build): Scan inner table, build hash table in memory
  Phase 2 (Probe): For each row in outer table, probe hash table
  
  Cost: inner_rows × cpu_operator_cost (build)
       + outer_rows × cpu_operator_cost (probe)
       + pages for both tables (I/O)
  
  Memory: hash table size = inner_table_size
  If hash table exceeds work_mem: multi-batch hash join (disk spill)
  
  Best when:
  • Large tables, equality join
  • No usable index on join key
  • Planner sees it's doing a lot of join work

MERGE JOIN:
  Sort both sides by join key (if not already sorted)
  Scan both sorted streams simultaneously (merge step)
  
  Cost: sort cost (O(n log n)) + merge cost (O(n+m))
  
  Best when:
  • Both inputs already sorted by join key (index scan provides order)
  • Equality or range join conditions
  • Both tables are large (avoid hash memory pressure)
  
  Planner chooses when: data already sorted OR sort is cheap + tables large
```

---

## Why Estimates Go Wrong

### Problem 1: Stale Statistics

```sql
-- Scenario: you just bulk-loaded 10 million rows into orders table.
-- autovacuum hasn't run yet. pg_stats still reflects the OLD data.

SELECT n_live_tup, n_dead_tup, last_analyze
FROM pg_stat_user_tables
WHERE relname = 'orders';

-- n_live_tup:  500000   ← OLD: 500K rows
-- n_dead_tup:  0
-- last_analyze: 2026-01-15   ← STALE: analyzed 4 months ago

-- Actual rows:  10,500,000   ← REAL: 10.5M rows

-- What happens:
-- Planner estimates 500K rows → uses Nested Loop (appropriate for 500K)
-- Executor processes 10.5M rows → Nested Loop on 10.5M rows = DISASTER

-- Fix:
ANALYZE orders;  -- updates statistics immediately
-- Now n_live_tup ≈ 10,500,000 → planner re-estimates → chooses Hash Join

-- For very large tables, ANALYZE scans a random sample (default 30,000 rows).
-- It does NOT scan the whole table — so it's fast even on 100M row tables.
```

### Problem 2: Column Correlation (Multi-Column Dependencies)

```sql
-- The planner assumes columns are STATISTICALLY INDEPENDENT.
-- This is often WRONG.

-- Example: employees table with city and country columns.
-- city = 'Paris' → 5% of rows
-- country = 'France' → 8% of rows
-- The planner estimates: WHERE city = 'Paris' AND country = 'France'
--   selectivity = 0.05 × 0.08 = 0.004 (0.4% of rows)

-- ACTUAL: if city = 'Paris', then country = 'France' is almost certain.
-- Real selectivity ≈ 0.05 (same as city = 'Paris' alone)
-- Planner underestimates: thinks 0.4% but really 5%

-- Impact: planner chooses an index scan (0.4% is very selective)
-- but at runtime processes 5% of rows via the index → unexpected performance

-- FIX: Extended statistics (PostgreSQL 10+)
CREATE STATISTICS employees_city_country (dependencies)
ON city, country FROM employees;

ANALYZE employees;  -- must re-analyze after creating statistics

-- Now the planner knows about the dependency between city and country.
-- Selectivity estimate for city='Paris' AND country='France' ≈ 0.05 (correct)

-- Verify:
SELECT stxname, stxkeys, stxkind
FROM pg_statistic_ext
WHERE stxrelid = 'employees'::regclass;
```

### Problem 3: Non-Uniform Data (Skew)

```sql
-- 99% of orders have status = 'completed', 1% have status = 'pending'
-- Histogram: mostly 'completed' with tiny tail for others

-- Planner has most_common_vals with accurate frequencies.
-- So planner KNOWS: status = 'pending' → 1% selectivity → use index.
-- This is CORRECT and handled well.

-- BUT: what about correlated columns?
-- status = 'pending' AND created_at > '2026-01-01' →
-- ALL pending orders are recent (business logic: old orders always complete)
-- Real selectivity: 1% × 100% = 1% (all pending are recent)
-- Planner estimate: 1% × 30% (portion of recent dates) = 0.3%
-- Planner underestimates → might choose an even more aggressive index strategy
-- that's actually good here. But this also means plan cost estimates are wrong,
-- which affects join decisions upstream.

-- FIX: Extended statistics for specific combinations:
CREATE STATISTICS orders_status_date (dependencies, ndistinct, mcv)
ON status, created_at FROM orders;
```

### Problem 4: Function Calls in WHERE

```sql
-- The planner has ZERO statistics for function output.
-- It uses a default selectivity of 0.5% (arbitrary estimate) for unknown functions.

WHERE EXTRACT(YEAR FROM created_at) = 2026
-- Planner: "EXTRACT() is a function. I'll estimate 0.5% selectivity."
-- Reality: Maybe 2026 data is 80% of the table (if table is new).
-- Planner way underestimates → chooses index scan over seq scan → bad plan.

-- FIX: Rewrite without the function to expose the column directly:
WHERE created_at >= '2026-01-01' AND created_at < '2027-01-01'
-- Now planner uses the histogram for created_at → accurate estimate.
-- Also: this is sargable → can use an index. The function version cannot.
-- (More on sargability in Topic 08.)
```

### Problem 5: Prepared Statement Plan Reuse

```sql
-- PREPARE + EXECUTE: the planner may generate a generic plan after 5 executions.
-- A generic plan uses $1 as a placeholder with estimated (average) selectivity.

PREPARE get_orders(text) AS
  SELECT * FROM orders WHERE status = $1;

-- First 5 executions: custom plans
--   EXECUTE get_orders('pending')   → planner uses selectivity(status='pending') = 0.01
--   EXECUTE get_orders('completed') → planner uses selectivity(status='completed') = 0.99

-- After 5 executions: PostgreSQL considers switching to GENERIC plan.
-- Generic plan: uses estimated average selectivity for $1 ≈ 0.20
-- Now BOTH 'pending' AND 'completed' queries use the SAME plan (optimised for 20%).
-- For 'pending' (1%): generic plan may use Seq Scan (too permissive) → bad
-- For 'completed' (99%): generic plan may use Index Scan (too aggressive) → bad

-- DETECT THIS:
-- Check pg_prepared_statements to see if generic plan is in use
SELECT name, statement, generic_plans, custom_plans
FROM pg_prepared_statements;

-- FORCE CUSTOM PLANS:
SET plan_cache_mode = 'force_custom_plan';  -- always re-plan (higher planning overhead)

-- OR: use NOT MATERIALIZED CTE to force re-evaluation
-- OR: use $queryRaw in ORMs that cache plans
```

---

## EXPLAIN Output for This Pattern

```sql
-- See the planner's estimates vs reality:
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT e.name, e.salary, d.name AS dept
FROM employees e
JOIN departments d ON e.department_id = d.id
WHERE e.salary BETWEEN 70000 AND 90000
ORDER BY e.salary DESC;
```

```
-- Output with annotations:

Sort  (cost=1250.80..1258.30 rows=3000 width=52)
│     (actual time=45.2..46.1 rows=2847 loops=1)
│      ↑               ↑
│      estimated 3000   actual 2847  ← close estimate (only 5% off) — good statistics
│
│  Sort Key: e.salary DESC
│  Sort Method: quicksort  Memory: 312kB  ← fits in work_mem, no disk spill
│
└── Hash Join  (cost=85.00..1100.80 rows=3000 width=52)
              (actual time=2.1..38.5 rows=2847 loops=1)
    │
    │  Hash Cond: (e.department_id = d.id)
    │  ← PLANNER CHOSE: Hash Join
    │    WHY: employees side is 3000 rows (estimated), departments is small.
    │    Nested Loop would require 3000 index lookups on departments.
    │    Hash Join: build hash table on departments (50 rows, tiny), probe with employees.
    │
    ├── Seq Scan on employees e  (cost=0.00..1000.00 rows=3000 width=40)
    │                            (actual time=0.04..25.3 rows=2847 loops=1)
    │   Filter: ((salary >= 70000) AND (salary <= 90000))
    │   Rows Removed by Filter: 47153
    │   Buffers: shared hit=450
    │
    │   ← WHY Seq Scan and NOT Index Scan on salary?
    │     Look: 2847 rows from 50000 total = 5.7% selectivity.
    │     For 5.7% with correlation ≈ 0.3 (salary not well-correlated with heap order):
    │     Index Scan cost would be: 2847 × 4.0 (random_page_cost) × ~0.02 ≈ 228
    │     Seq Scan cost: 450 pages × 1.0 = 450 → similar!
    │     The planner chose Seq Scan here. If we reduce random_page_cost to 1.1 (SSD),
    │     the planner would switch to Index Scan.
    │
    └── Hash  (cost=50.00..50.00 rows=50 width=20)
             (actual time=2.05..2.05 rows=50 loops=1)
        │  Buckets: 1024  Memory Usage: 10kB  ← hash table for 50 departments
        └── Seq Scan on departments d  (cost=0.00..50.00 rows=50 width=20)
                                       (actual time=0.01..0.9 rows=50 loops=1)
            Buffers: shared hit=2

Planning Time: 0.385 ms  ← stage 4 took 0.385ms
Execution Time: 46.8 ms  ← stage 5 took 46.8ms

KEY TAKEAWAYS:
• Estimated 3000 rows, got 2847 → statistics are accurate (within 5%)
• Hash Join chosen because: small departments table → tiny hash table (10kB)
• Seq Scan on employees despite salary index: 5.7% selectivity + low correlation
• Sort fits in memory (312kB < work_mem) → no disk spill

IF STATISTICS WERE STALE (estimated 300 rows instead of 3000):
• Planner would likely choose Nested Loop instead of Hash Join
• Nested Loop + 300 estimated rows = fine
• Actual 2847 rows × Nested Loop = 2847 lookups on departments = still OK here
  (departments is tiny), but bad for larger tables
```

---

## Example 1 — Basic: Observing the Planner's Selectivity Estimates

```sql
-- Watch the planner's estimates change as selectivity changes:

-- Setup context:
-- employees: 100,000 rows
-- Departments: Engineering(30%), Sales(25%), HR(15%), rest spread

-- Query 1: Very selective (0.001% → 1 row)
EXPLAIN SELECT * FROM employees WHERE id = 42;
-- Index Scan using employees_pkey: estimated rows=1
-- Cost: ~8. Planner: "1 row, use primary key index"

-- Query 2: Moderately selective (1%)
EXPLAIN SELECT * FROM employees WHERE email LIKE 'smith%';
-- Bitmap Heap Scan: estimated rows≈1000
-- Cost: ~200. Planner: "1000 rows from LIKE, bitmap scan"

-- Query 3: Low selectivity (30%)
EXPLAIN SELECT * FROM employees WHERE department = 'Engineering';
-- Seq Scan: estimated rows≈30000
-- Cost: ~1800. Planner: "30% of table, sequential scan is cheaper"

-- Query 4: Very low selectivity (0.01%) with a range
EXPLAIN SELECT * FROM employees WHERE salary BETWEEN 95000 AND 96000;
-- Index Scan using idx_emp_salary: estimated rows≈100
-- Cost: ~50. Planner: "100 rows from narrow range, use salary index"

-- OBSERVE: The planner's selectivity estimate directly determines which access path wins.
-- You can check what selectivity the planner computed:
SELECT
  attname,
  n_distinct,
  most_common_vals::text::text[],
  most_common_freqs
FROM pg_stats
WHERE tablename = 'employees'
  AND attname = 'department';
```

---

## Example 2 — Intermediate: Forcing ANALYZE and Watching Plans Change

```sql
-- Demonstrate stale statistics causing a bad plan.
-- (Run this in a test environment)

-- Step 1: Create a test table with known data
CREATE TABLE test_orders (
  id SERIAL PRIMARY KEY,
  status TEXT,
  amount NUMERIC
);

-- Step 2: Insert mostly 'completed' orders
INSERT INTO test_orders (status, amount)
SELECT
  CASE WHEN random() < 0.99 THEN 'completed' ELSE 'pending' END,
  (random() * 1000)::NUMERIC(10,2)
FROM generate_series(1, 100000);

-- Step 3: Add an index
CREATE INDEX idx_test_status ON test_orders(status);

-- Step 4: ANALYZE to set initial statistics
ANALYZE test_orders;

-- Step 5: See the plan for pending orders (should use index - only 1% of rows)
EXPLAIN ANALYZE SELECT * FROM test_orders WHERE status = 'pending';
-- Index Scan using idx_test_status: estimated rows ≈ 1000
-- Correct: planner knows pending = 1% → uses index

-- Step 6: Now delete all pending rows and insert MANY more
DELETE FROM test_orders WHERE status = 'pending';
INSERT INTO test_orders (status, amount)
SELECT 'pending', (random() * 100)::NUMERIC(10,2)
FROM generate_series(1, 50000);  -- now pending = ~33% of rows!

-- Step 7: Plan WITHOUT running ANALYZE (stale statistics)
EXPLAIN ANALYZE SELECT * FROM test_orders WHERE status = 'pending';
-- STILL shows: Index Scan, estimated rows ≈ 1000
-- BUT actual rows = 50,000!
-- The planner chose Index Scan based on stale 1% estimate.
-- Actually 33% → Index Scan is now WRONG (should be Seq Scan or Bitmap Scan)
-- Execution time is much slower than optimal.

-- Step 8: Run ANALYZE to update statistics
ANALYZE test_orders;

-- Step 9: Plan WITH fresh statistics
EXPLAIN ANALYZE SELECT * FROM test_orders WHERE status = 'pending';
-- NOW: Seq Scan (or Bitmap Scan), estimated rows ≈ 50000
-- Planner correctly detects 33% selectivity and chooses appropriately.

-- LESSON: ANALYZE matters. In production, autovacuum_analyze_threshold and
-- autovacuum_analyze_scale_factor control when automatic ANALYZE runs.
-- After bulk loads: always run ANALYZE manually.
```

---

## Example 3 — Production Grade: Diagnosing and Fixing a Bad Plan

```sql
-- Production scenario: An e-commerce query that's slow in production (50M orders)
-- but fast in the dev environment (100K orders).

-- The slow query:
SELECT
  u.id, u.name, u.email,
  COUNT(o.id) AS order_count,
  SUM(o.total) AS lifetime_value
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.status = 'completed'
  AND o.created_at > CURRENT_DATE - INTERVAL '90 days'
GROUP BY u.id, u.name, u.email
HAVING COUNT(o.id) > 5
ORDER BY lifetime_value DESC
LIMIT 100;

-- Step 1: Run EXPLAIN ANALYZE
EXPLAIN (ANALYZE, BUFFERS)
SELECT
  u.id, u.name, u.email,
  COUNT(o.id) AS order_count,
  SUM(o.total) AS lifetime_value
FROM users u
JOIN orders o ON u.id = o.user_id
WHERE o.status = 'completed'
  AND o.created_at > CURRENT_DATE - INTERVAL '90 days'
GROUP BY u.id, u.name, u.email
HAVING COUNT(o.id) > 5
ORDER BY lifetime_value DESC
LIMIT 100;

-- BAD PLAN OUTPUT:
-- Limit  (cost=950000..950010 rows=100) (actual time=185000..185002 rows=100 loops=1)
--   -> Sort (actual time=184995..185000 rows=100 loops=1)
--       Sort Method: external merge  Disk: 45000kB   ← DISK SORT! work_mem too small
--       -> HashAggregate (actual time=180000..184500 rows=2100000 loops=1)
--            -> Nested Loop  (actual time=0.05..150000 rows=20000000 loops=1)
--                 -> Seq Scan on orders o (actual time=0.04..80000 rows=20000000 loops=1)
--                      Filter: (status='completed' AND created_at > ...)
--                      Rows Removed by Filter: 30000000   ← 30M rows scanned then discarded!
--                 -> Index Scan on users pkey (actual time=0.002..0.003 rows=1 loops=20000000)

-- DIAGNOSIS:
-- Problem 1: Seq Scan on orders scanning 50M rows (30M discarded by filter)
--   Why: orders has no index on (status, created_at) combined
--   Fix: CREATE INDEX idx_orders_status_date ON orders(status, created_at)
--   Expected: eliminates 30M scanned rows

-- Problem 2: Nested Loop joining 20M order rows to users
--   Why: planner estimated 200K rows after filter (stale stats?) but got 20M
--        Nested Loop with 20M outer rows = 20M index lookups on users → catastrophic
--   Fix: after index added, filtered rows reduce → planner re-estimates → may switch to Hash Join
--   Also: ANALYZE orders to ensure fresh statistics

-- Problem 3: Sort spills to disk (45MB)
--   Why: HashAggregate produces 2.1M groups → sort of 2.1M rows > work_mem (default 4MB)
--   Fix: SET work_mem = '64MB' for this session, or increase in postgresql.conf

-- APPLY THE FIXES:
CREATE INDEX CONCURRENTLY idx_orders_status_date
ON orders(status, created_at DESC)
WHERE status = 'completed';   -- partial index! only index completed orders

ANALYZE orders;

-- GOOD PLAN AFTER FIXES:
-- Limit  (cost=4500..4510 rows=100) (actual time=120..122 rows=100 loops=1)
--   -> Sort (actual time=118..120 rows=100 loops=1)
--       Sort Method: top-N heapsort  Memory: 32kB   ← in memory, tiny
--       -> HashAggregate (actual time=85..115 rows=45000 loops=1)
--            -> Hash Join (actual time=15..70 rows=1500000 loops=1)
--                 -> Index Scan on idx_orders_status_date (rows=1500000 loops=1)
--                      Index Cond: (status='completed' AND created_at > ...)
--                      Buffers: shared hit=8500   ← only reads relevant pages
--                 -> Hash on users (rows=2000000) Memory: 120MB
-- Execution Time: 122 ms   ← from 185,000ms to 122ms: 1500x improvement

-- WHAT CHANGED IN THE PLAN:
-- Seq Scan → Index Scan on partial index (only reads 1.5M relevant rows, not 50M)
-- Nested Loop → Hash Join (planner now correctly estimates 1.5M rows → hash is better)
-- Disk sort → top-N heapsort in memory (fewer groups to aggregate + sort)
```

---

## Wrong Query → What Breaks → Right Query

### Mistake 1: Misreading the Planner — Panicking at Seq Scan

```sql
-- WRONG REACTION: "There's a Seq Scan! I need to add an index!"
EXPLAIN SELECT * FROM users WHERE active = true;
-- Seq Scan on users (rows=700000)

-- Developer adds:
CREATE INDEX idx_users_active ON users(active);

-- EXPLAIN again — still Seq Scan!
-- Developer: "The index isn't being used — something is broken!"

-- WHAT ACTUALLY HAPPENED:
-- active = true matches 70% of users (700,000 out of 1,000,000 rows).
-- An index on a boolean column with 70% matching is almost never useful.
-- The planner correctly ignores the index — reading 70% of rows via
-- random index lookups would be SLOWER than a sequential scan.
-- Adding this index wastes disk space and write overhead for zero benefit.

-- RIGHT APPROACH: Accept the Seq Scan. It is correct here.
-- If performance is a problem, the issue is:
--   1. Too many rows being scanned (add better filters)
--   2. Wrong query structure (join earlier to reduce rows)
--   3. Need a partial index for the INACTIVE users (minority):
CREATE INDEX idx_users_inactive ON users(id) WHERE active = false;
-- This helps queries on inactive users (30%) without touching active queries.
```

### Mistake 2: Wrong Statistics Target

```sql
-- Problem: A column has very uneven distribution but the planner always
-- guesses wrong because the histogram is too coarse.

-- orders.order_value: 80% of orders are between $10-$50, but some go to $50,000
-- With default statistics_target = 100, the histogram has 100 buckets.
-- Most buckets are in the $10-$50 range (where 80% of data lives).
-- The $50-$50,000 range has very few histogram buckets → poor estimates for
-- queries like WHERE order_value > 1000 (high-value orders).

-- DIAGNOSE: Check if estimated ≈ actual in EXPLAIN ANALYZE
EXPLAIN ANALYZE SELECT * FROM orders WHERE order_value > 1000;
-- Estimated: 500 rows (planner using coarse histogram)
-- Actual: 8500 rows (planner was 17x off!)

-- FIX: Increase statistics target for this column
ALTER TABLE orders ALTER COLUMN order_value SET STATISTICS 500;
ANALYZE orders;

-- VERIFY: pg_stats now has finer histogram for order_value
SELECT array_length(histogram_bounds, 1) AS bucket_count
FROM pg_stats
WHERE tablename = 'orders' AND attname = 'order_value';
-- Was: 101 buckets (100 bounds + endpoints)
-- Now: 501 buckets → much finer granularity in $50-$50000 range

-- Re-run EXPLAIN ANALYZE:
-- Estimated: 8200 rows (much closer!)
-- Actual: 8500 rows → planner makes better decisions now
```

---

## Performance Profile

```
PLANNER PERFORMANCE CHARACTERISTICS:

Planning Time:
  • Simple queries (1-3 tables): < 0.5 ms
  • Moderate queries (4-8 tables): 1-5 ms
  • Complex queries (10+ tables): 5-50 ms
  • Joins exceeding geqo_threshold (12 tables): GEQO, variable
  • Very complex queries (20+ tables, many predicates): 50-200 ms

When Planning Time is a Problem:
  • Connection overhead: 0.1ms planning amortised over 1000ms query = negligible
  • Connection overhead: 5ms planning for a 2ms query = significant (>50% overhead)
  • Fix: connection pooling (PgBouncer) or reduce query complexity
  • Fix: prepared statements (plan once, execute many — but watch for generic plan traps)

Planner Memory:
  • The planner builds plan trees in memory during planning
  • Very complex queries (50+ tables, subqueries, CTEs) can use significant memory
  • geqo_threshold controls when genetic algorithm kicks in (reduces memory but may miss optimal plan)

The Work Done By Statistics:
  • ANALYZE scans a random sample (default 30,000 rows)
  • ANALYZE does NOT scan the whole table — fast even on 100M row tables
  • autovacuum_analyze_threshold = 50 (run ANALYZE after 50 row changes)
  • autovacuum_analyze_scale_factor = 0.2 (run ANALYZE after 20% of table changes)
  • After bulk loads (INSERT millions of rows): run ANALYZE manually
  • After DELETE large portions of table: run ANALYZE manually
  • Per-column statistics: ALTER TABLE t ALTER COLUMN c SET STATISTICS 500
    (use for high-cardinality, frequently queried columns with skewed distributions)

THE MOST IMPORTANT PERFORMANCE RULE FOR THE PLANNER:
Fresh statistics → accurate estimates → correct plan choices → fast queries.
Stale statistics → wrong estimates → wrong plan → slow queries.
When a query suddenly gets slow after a data change: run ANALYZE first.
```

---

## Node.js Integration

```javascript
// === HOW THE QUERY PLANNER AFFECTS NODE.JS APPLICATIONS ===

const { Pool } = require('pg');
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// 1. PREPARED STATEMENTS AND PLAN CACHING
// The pg library uses named prepared statements automatically for parameterized queries.
// After ~5 executions, PostgreSQL may switch to a generic plan.

// DETECTING GENERIC PLAN ISSUES:
async function detectPlanCacheIssue(userId) {
  // Run EXPLAIN EXECUTE on your prepared statement to see which plan is in use:
  const client = await pool.connect();
  try {
    await client.query('PREPARE check_user(int) AS SELECT * FROM users WHERE id = $1');
    // First few executions: custom plan
    const explained = await client.query('EXPLAIN (FORMAT JSON) EXECUTE check_user($1)', [userId]);
    const plan = explained.rows[0]['QUERY PLAN'][0];
    console.log('Plan type:', plan.Plan['Node Type']);
    console.log('Estimated rows:', plan.Plan['Plan Rows']);
  } finally {
    await client.query('DEALLOCATE check_user');
    client.release();
  }
}

// 2. FORCING FRESH PLANS FOR SKEWED DATA
// If your data is highly skewed and prepared statement reuse causes bad plans:
async function queryWithFreshPlan(status) {
  // Option A: SET plan_cache_mode for the session
  const client = await pool.connect();
  try {
    await client.query("SET plan_cache_mode = 'force_custom_plan'");
    const result = await client.query(
      'SELECT * FROM orders WHERE status = $1 ORDER BY created_at DESC LIMIT 100',
      [status]
    );
    return result.rows;
  } finally {
    client.release();
  }
}

// 3. TRIGGERING ANALYZE AFTER BULK OPERATIONS
// After a large INSERT/DELETE/UPDATE in your Node.js app:
async function bulkLoadAndAnalyze(records) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    // Bulk insert (simplified — see Topic 60 for COPY-based bulk insert)
    for (const record of records) {
      await client.query(
        'INSERT INTO orders (user_id, total, status) VALUES ($1, $2, $3)',
        [record.userId, record.total, record.status]
      );
    }

    await client.query('COMMIT');

    // After inserting a large batch: update statistics so planner knows the new shape
    if (records.length > 10000) {
      // ANALYZE is safe to run outside a transaction
      await client.query('ANALYZE orders');
      console.log('Statistics updated after bulk load');
    }
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

// 4. MONITORING PLANNER HEALTH IN PRODUCTION
// Check for tables with stale statistics (common source of bad plans):
async function findStaleStatistics() {
  const { rows } = await pool.query(`
    SELECT
      schemaname,
      relname AS tablename,
      n_live_tup AS live_rows,
      n_dead_tup AS dead_rows,
      last_analyze,
      last_autoanalyze,
      -- Staleness: rows changed since last analyze / total rows
      CASE
        WHEN n_live_tup = 0 THEN 0
        ELSE ROUND(
          100.0 * (n_ins_since_vacuum + n_upd_since_vacuum + n_del_since_vacuum)
          / GREATEST(n_live_tup, 1), 2
        )
      END AS pct_changed_since_analyze
    FROM pg_stat_user_tables
    WHERE n_live_tup > 10000   -- only care about non-trivial tables
    ORDER BY pct_changed_since_analyze DESC
    LIMIT 20
  `);

  const stale = rows.filter(r => r.pct_changed_since_analyze > 10);
  if (stale.length > 0) {
    console.warn('Tables with stale statistics:', stale.map(r => r.tablename));
  }
  return rows;
}
```

### ORM Comparison — How ORMs Interact with the Planner

```javascript
// === THE ORMS DON'T CONTROL THE PLANNER — BUT THEY AFFECT IT ===

// The planner makes decisions based on the SQL it receives.
// ORM-generated SQL quality directly impacts plan quality.

// === PRISMA ===
// Prisma generates clean, predictable SQL.
// BUT: Prisma always uses prepared statements internally.
// For queries with highly skewed parameters, the generic plan trap applies.

// Prisma has NO way to run ANALYZE or control plan_cache_mode natively.
// You must drop to $queryRaw for planner interventions:
const result = await prisma.$queryRaw`ANALYZE orders`;
const result2 = await prisma.$executeRaw`SET plan_cache_mode = 'force_custom_plan'`;

// === DRIZZLE ===
// Similar to Prisma — uses prepared statements.
// Can run arbitrary SQL via sql template:
import { sql } from 'drizzle-orm';
await db.execute(sql`ANALYZE orders`);
await db.execute(sql`SET plan_cache_mode = 'force_custom_plan'`);

// === SEQUELIZE ===
// Sequelize uses parameterized queries but handles plan caching at a lower level.
// For planner control:
await sequelize.query('ANALYZE orders', { type: QueryTypes.RAW });
await sequelize.query("SET plan_cache_mode = 'force_custom_plan'");

// === KNEX ===
// Most flexible for planner control:
await knex.raw('ANALYZE orders');
await knex.raw("SET plan_cache_mode = 'force_custom_plan'");

// Can also access query plans easily:
const plan = await knex.raw('EXPLAIN (FORMAT JSON) ??', [knex('orders').select('*')]);

// === THE ORM-GENERATED SQL QUALITY ISSUE ===
// Some ORMs generate SQL that is harder for the planner to optimize:

// Bad ORM pattern (Sequelize eager loading):
// SELECT users.*, orders.* FROM users LEFT JOIN orders ON users.id = orders.user_id
// WHERE users.id IN (1, 2, 3, ..., 1000)
// This IN list with 1000 elements is hard to estimate accurately.
// The planner treats IN(1000 values) as a "row comparison" with rough selectivity.
// Better: use a JOIN with a subquery or explicit parameterized query.

// === VERDICT ===
// None of the ORMs give you direct planner control.
// For planner-affecting operations (ANALYZE, SET work_mem, extended statistics,
// statistics targets), you always need raw SQL regardless of ORM.
// The ORM is transparent to the planner — what matters is the SQL that reaches it.
```

---

## Practice Exercises

### Exercise 1 — Easy (Reading Statistics)

You run this query against your users table (500,000 rows):
```sql
SELECT attname, null_frac, n_distinct, most_common_vals, most_common_freqs
FROM pg_stats
WHERE tablename = 'users' AND attname = 'subscription_tier';
```

It returns:
```
attname:           subscription_tier
null_frac:         0.05
n_distinct:        4
most_common_vals:  {free, basic, pro, enterprise}
most_common_freqs: {0.72, 0.18, 0.08, 0.02}
```

Answer these questions:
1. How many rows have `subscription_tier = 'enterprise'`?
2. How many rows have `subscription_tier IS NULL`?
3. For `WHERE subscription_tier = 'enterprise'` — will the planner choose an Index Scan or Seq Scan? Why?
4. For `WHERE subscription_tier = 'free'` — will the planner choose Index Scan or Seq Scan? Why?
5. If you need to frequently query only enterprise users, what index structure would you recommend?

```
-- Write your answers here
```

### Exercise 2 — Medium (Diagnosing Wrong Estimates)

You have a query that EXPLAIN ANALYZE reveals:
```
Nested Loop (cost=0.43..85000.00 rows=50) (actual time=0.05..45000.00 rows=48500 loops=1)
```

1. What does "rows=50 estimated, rows=48500 actual" tell you about the planner's knowledge?
2. WHY did the planner choose Nested Loop — what did it think would happen?
3. What would the planner likely have chosen if it had correct statistics (48500 rows)?
4. Write the SQL to identify whether this is a stale statistics problem.
5. Write the fix — both the immediate action and the long-term prevention.

```sql
-- Write your analysis and SQL here
```

### Exercise 3 — Hard (End-to-End Plan Investigation)

You have a production query that runs in 8 seconds. Your task:

```sql
-- The slow query:
SELECT
  p.id, p.name, p.price, p.category_id,
  COUNT(oi.id) AS times_ordered,
  SUM(oi.quantity) AS total_units_sold
FROM products p
LEFT JOIN order_items oi ON p.id = oi.product_id
LEFT JOIN orders o ON oi.order_id = o.id
WHERE o.created_at > CURRENT_DATE - INTERVAL '30 days'
   OR o.id IS NULL   -- include products never ordered
GROUP BY p.id, p.name, p.price, p.category_id
HAVING SUM(oi.quantity) > 10 OR SUM(oi.quantity) IS NULL
ORDER BY times_ordered DESC NULLS LAST
LIMIT 50;
```

EXPLAIN ANALYZE shows:
```
Limit (actual time=8234..8235 rows=50 loops=1)
  -> Sort (actual time=8232..8233 rows=50 loops=1)
       Sort Method: external merge  Disk: 12000kB
       -> HashAggregate (actual time=7800..8100 rows=180000 loops=1)
            -> Hash Right Join (actual time=1200..5800 rows=8500000 loops=1)
                 -> Hash Left Join (actual time=800..4200 rows=9000000 loops=1)
                      -> Seq Scan on orders o (rows=9000000 loops=1)
                      -> Hash on order_items oi (Memory: 1800MB)  ← !
                 -> Hash on products p (rows=180000 loops=1)
```

Write your complete diagnosis and step-by-step fix plan. For each fix, explain which planner decision it changes and why.

```sql
-- Write your diagnosis and fixes here
```

### Exercise 4 — Interview Simulation

**Interviewer:** "In production we have a query that runs in 50ms every day for a week, then suddenly starts taking 45 seconds. No code changed. No schema changed. How do you diagnose this?"

Write your complete interview answer. Include the specific commands you'd run, what you'd look for in each, and how you'd fix the most likely cause.

```
-- Write your interview answer here
```

---

## Interview Questions This Topic Covers

### Q: "How does PostgreSQL decide whether to use an index or do a full table scan?"

**Junior answer:** "It uses the index if there's a WHERE clause on the indexed column."

**Principal answer:** "PostgreSQL uses a cost-based model. For each access path — sequential scan, index scan, bitmap scan — it estimates a cost using the formula: I/O cost (pages × page cost) plus CPU cost (rows × tuple processing cost). The critical factors are: selectivity (what fraction of rows match the WHERE condition, estimated from pg_stats histogram and most_common_vals), the random_page_cost parameter (4.0 for HDD, ~1.1 for SSD), and column correlation (how well physical row order matches index order). For a highly selective query like WHERE id = 42, the index wins easily — 1 row vs full table scan. For WHERE active = true with 70% matching, sequential scan wins — reading 70% of the table via random index lookups costs 4x per page, making it slower than a sequential pass. The crossover is typically around 5-15% selectivity, depending on correlation and random_page_cost."

**Follow-up:** "What if the index exists but PostgreSQL is ignoring it on a column you know is selective?"

**Principal answer:** "First suspect: stale statistics. Run `EXPLAIN ANALYZE` — if estimated rows are wildly off from actual rows, the planner made its decision on wrong data. Fix: `ANALYZE tablename`. Second suspect: function on the column — `WHERE lower(email) = $1` cannot use an index on `email` because the index stores original case values, not lower-cased values. Fix: expression index on `lower(email)`. Third suspect: type mismatch — if the column is `bigint` and you're passing an `int`, implicit casting may prevent index use. Fourth: correlation is near 0 and random_page_cost is high — the planner correctly determines the index costs more than a scan at that selectivity."

---

### Q: "What is ANALYZE and when should you run it manually?"

**Junior answer:** "It updates table statistics. You run it if queries are slow."

**Principal answer:** "ANALYZE collects statistical samples from the table and stores them in pg_statistic: the null fraction, number of distinct values, most common values with their frequencies, histogram bucket boundaries for value distribution, and the physical-to-logical correlation. The planner uses these statistics to estimate selectivity — how many rows will match a condition. Wrong statistics → wrong selectivity estimate → wrong access method or join algorithm → bad performance. autovacuum runs ANALYZE automatically when a table changes by more than 20% (configurable via autovacuum_analyze_scale_factor). You should run ANALYZE manually in three situations: after a bulk INSERT or DELETE that changes table data significantly before autovacuum fires, after VACUUM on a table with heavy dead tuple bloat (which may have skewed the old statistics), and on newly created tables before any significant queries run against them. For frequently queried columns with skewed distributions, increase the statistics target: `ALTER TABLE t ALTER COLUMN c SET STATISTICS 500` then `ANALYZE t`."

---

### Q: "What is a generic plan in PostgreSQL and when is it a problem?"

**Junior answer:** "I'm not sure — something to do with prepared statements?"

**Principal answer:** "When you use a prepared statement, PostgreSQL generates a custom plan for the first 5 executions, where it sees the actual parameter values and can estimate selectivity accurately. After 5 executions, if the estimated cost of a generic plan (one that uses average estimated selectivity for the parameter, not the actual value) is similar to the average custom plan cost, PostgreSQL switches to the generic plan for efficiency. The problem: if your data is highly skewed, the generic plan is optimized for the average case, which may be terrible for specific values. For example, a query on `WHERE status = $1` with status distribution of 99% 'completed' / 1% 'pending': the generic plan might use a sequential scan (good for 'completed', bad for 'pending'), while the custom plan would use an index scan for 'pending'. You diagnose this by checking if plan changes happen between the first few executions and later ones. The fix is `SET plan_cache_mode = 'force_custom_plan'` for sessions where this matters, or by restructuring the query."

---

## Mental Model Checkpoint

1. **The planner estimates a query will return 100 rows. EXPLAIN ANALYZE shows it returned 50,000 rows. The query was slow. What is the single most likely root cause, and what is the single command you'd run to fix it?**

2. **You set `random_page_cost = 1.1` on your server (you're on SSDs). What specific types of queries will become faster — and why does this one setting change?**

3. **A developer says "I added a composite index on (user_id, created_at) but the planner is still doing a Seq Scan on a query that filters WHERE user_id = 5 AND created_at > '2026-01-01'." What are the 3 possible reasons the index isn't being used?**

4. **You need to join 15 tables in a single query. The planning time is 2 seconds. What is happening inside the planner, what setting controls this, and what are the trade-offs of changing it?**

5. **Two columns are correlated: country and city (if city = 'Tokyo', country = 'Japan' always). The planner estimates 0.01% selectivity for `WHERE city = 'Tokyo' AND country = 'Japan'` but the real selectivity is 0.5%. What PostgreSQL feature fixes this, and what SQL creates it?**

6. **After running `VACUUM FULL` on a large table, query performance suddenly gets worse. Why might VACUUM FULL cause bad plans, and what do you run to fix it?**

7. **You have a query that runs in 5ms for most parameter values but 30 seconds for one specific value. The same SQL, the same indexes, the same table. What is the likely cause and what are two different ways to fix it?**

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ QUERY PLANNER QUICK REFERENCE                                               │
├─────────────────────────────────────────────────────────────────────────────┤
│ KEY COST PARAMETERS (postgresql.conf)                                       │
│   seq_page_cost        = 1.0  (base unit — sequential page read)            │
│   random_page_cost     = 4.0  (set to 1.1-2.0 for SSDs)                   │
│   cpu_tuple_cost       = 0.01                                               │
│   cpu_operator_cost    = 0.0025                                             │
│                                                                             │
│ STATISTICS COMMANDS                                                         │
│   ANALYZE tablename;                    update statistics for one table     │
│   ANALYZE;                              update all tables                   │
│   ALTER TABLE t ALTER COLUMN c          increase histogram resolution       │
│     SET STATISTICS 500;                 (default: 100, max: 10000)          │
│   CREATE STATISTICS name (dependencies) create multi-column statistics      │
│     ON col1, col2 FROM table;                                               │
│                                                                             │
│ STATISTICS VIEWS                                                            │
│   pg_stats              human-readable per-column statistics                │
│   pg_statistic          raw statistics storage                              │
│   pg_stat_user_tables   last_analyze, n_live_tup, n_dead_tup               │
│   pg_statistic_ext      extended (multi-column) statistics                 │
│                                                                             │
│ JOIN ALGORITHMS                                                             │
│   Nested Loop   → small outer + indexed inner, LIMIT present               │
│   Hash Join     → large tables, equality, no usable index                  │
│   Merge Join    → already sorted inputs, equality/range                    │
│                                                                             │
│ PLANNER CONFIGURATION                                                       │
│   geqo_threshold = 12          switch to genetic algo above N tables       │
│   plan_cache_mode = 'auto'     auto/force_custom_plan/force_generic_plan   │
│   enable_seqscan = on/off      emergency override (use sparingly)          │
│   enable_hashjoin = on/off     emergency override (use sparingly)          │
│   enable_nestloop = on/off     emergency override (use sparingly)          │
│                                                                             │
│ BAD PLAN DIAGNOSIS CHECKLIST                                                │
│   1. EXPLAIN ANALYZE → compare estimated vs actual rows                     │
│   2. Huge gap? → Run ANALYZE tablename                                      │
│   3. Function in WHERE? → Rewrite to expose column (sargability)           │
│   4. Multi-column conditions? → CREATE STATISTICS (dependencies)           │
│   5. Correlated skewed data? → Increase STATISTICS target                  │
│   6. Prepared statement? → Check plan_cache_mode                           │
│   7. Still wrong? → See Topic 49 (Optimisation Methodology)               │
│                                                                             │
│ WHEN TO RUN ANALYZE MANUALLY                                                │
│   • After bulk INSERT/DELETE (> 10% of table changed)                      │
│   • After table is newly populated                                          │
│   • When EXPLAIN estimated ≠ actual rows by > 2x                          │
│   • After changing statistics target (ALTER COLUMN SET STATISTICS)         │
│   • After VACUUM FULL (resets physical layout → correlation changes)       │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Connected Topics

**Internals this builds on:**
- Topic 01: SQL Processing Pipeline (planner is Stage 4)
- Topic 02: Logical Execution Order (planner must produce a plan that honours this order)
- B-tree index structure (planner enumerates B-tree access paths for each WHERE condition)
- Buffer pool (random_page_cost / seq_page_cost model buffer cache hit likelihood)
- MVCC (dead tuple bloat affects physical page count → affects seq scan cost estimates)

**Next topic:** Topic 04 — Reading EXPLAIN and EXPLAIN ANALYZE (the output of the planner, interpreted in detail)

**Topics this is prerequisite for:**
- Topic 04: EXPLAIN output is the planner's decision made visible
- Topic 08: Sargability — which WHERE conditions the planner can use indexes for
- Topic 42–51: All index and optimisation topics assume planner knowledge
- Topic 49: Query optimisation methodology is built on planner understanding
- Topic 50: Statistics and the planner (deep dive into pg_stats, ANALYZE internals)
