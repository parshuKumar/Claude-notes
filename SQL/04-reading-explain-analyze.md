# 04 — Reading EXPLAIN and EXPLAIN ANALYZE
## Phase: Foundations (How SQL Actually Executes)
## Interview Priority: HIGH

---

## ELI5 — The Simple Analogy

When a surgeon is about to operate, they don't just start cutting. They first look at the MRI scan — a detailed picture of what's inside, where every organ is, where the problem lies.

`EXPLAIN` is your MRI scan for a SQL query.

`EXPLAIN` alone shows you the **plan before surgery** — what the database *intends* to do (cost estimates, strategy choices).

`EXPLAIN ANALYZE` shows you the **post-surgery report** — what it *actually did* (real times, real row counts, whether the plan was good or bad).

The difference is critical: `EXPLAIN` can lie (it shows estimates). `EXPLAIN ANALYZE` cannot — it shows what actually happened because it *runs the query*.

A senior engineer who cannot read an EXPLAIN output is like a surgeon who cannot read an MRI. They might get lucky, but they cannot diagnose systematically. Reading EXPLAIN is not optional. It is the fundamental debugging tool for every performance problem you will ever encounter.

---

## Internals Connection

`EXPLAIN` shows you the **output of Stage 4** (the Planner) from Topic 01. Every node in the plan tree corresponds to a physical operation the Executor (Stage 5) will perform.

`EXPLAIN ANALYZE` runs through all 5 stages including the Executor, then overlays actual measurements on the planned structure.

The plan tree is a direct representation of the **Volcano iterator model** from Topic 01:
- The tree is read **bottom-to-top** (children execute before parents)
- Each node **pulls** rows from its child nodes via `next()` calls
- The indentation shows the parent-child relationship
- Every node you see corresponds to a physical data structure: B-tree pages, hash tables in memory, sort buffers, heap pages read from the buffer pool

From Topic 03, you know the planner assigns **cost estimates** based on statistics and the cost model. `EXPLAIN` lets you see those estimates, and `EXPLAIN ANALYZE` lets you compare them to reality — which is how you detect stale statistics and wrong plan choices.

---

## The Execution Order Context

`EXPLAIN` output is read in **reverse execution order**:
- You write `EXPLAIN SELECT ... FROM ... JOIN ... WHERE ...`
- The plan tree shows the physical execution — bottom nodes run first
- The root node (topmost) is what delivers the final result to you

The logical execution order (Topic 02) maps to the plan tree like this:

```
Logical Order          → Physical Plan Node
FROM + JOIN            → Seq Scan, Index Scan, Hash Join, Nested Loop, Merge Join
WHERE                  → Filter inside Scan nodes
GROUP BY               → HashAggregate or Sort + GroupAggregate
HAVING                 → Filter inside Aggregate nodes
SELECT expressions     → computed implicitly in most nodes
DISTINCT               → HashAggregate or Sort + Unique
Window Functions       → WindowAgg (with Sort below it)
ORDER BY               → Sort
LIMIT                  → Limit
```

---

## What Is This?

`EXPLAIN` is a PostgreSQL command that shows the **execution plan** the query planner chose for a query — without actually executing the query. `EXPLAIN ANALYZE` executes the query, measures real timing and row counts at every node, and overlays that data on the plan. Together they are the primary tool for diagnosing slow queries, understanding plan choices, and verifying that your indexes and statistics are working correctly.

---

## Why Does It Matter in Production?

Without being able to read EXPLAIN output:
- You cannot confirm an index is actually being used
- You cannot identify which part of a complex query is slow
- You cannot tell whether a plan is slow because the planner chose wrong (bad estimate) or because the data volume is genuinely large (correct plan, big data)
- You cannot diagnose memory spills, bad join algorithms, or row count explosions
- You cannot have a meaningful conversation with your team about performance

Every production performance investigation starts here. No exceptions.

---

## The EXPLAIN Syntax — All Options

```sql
EXPLAIN [ ( option [, ...] ) ] statement

Options:
  ANALYZE   [ boolean ]   -- Actually execute the query and measure real times/counts
  VERBOSE   [ boolean ]   -- Show output column list, schema-qualified names
  COSTS     [ boolean ]   -- Show estimated costs (default: on)
  SETTINGS  [ boolean ]   -- Show non-default planner settings that affected the plan
  BUFFERS   [ boolean ]   -- Show buffer usage (hits, reads, dirtied, written) — requires ANALYZE
  WAL       [ boolean ]   -- Show WAL usage statistics (inserts, updates, bytes) — requires ANALYZE
  TIMING    [ boolean ]   -- Show per-node timing (default: on when ANALYZE is on)
  SUMMARY   [ boolean ]   -- Show Planning Time and Execution Time at the bottom
  FORMAT    { TEXT | XML | JSON | YAML }  -- Output format (default: TEXT)

-- COMMON COMBINATIONS:

-- Development: see the plan quickly
EXPLAIN SELECT ...;

-- Debugging performance: run it and measure everything
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;

-- Machine-readable for tooling
EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT ...;

-- Show what settings affected the plan
EXPLAIN (ANALYZE, SETTINGS) SELECT ...;

-- IMPORTANT WARNINGS:
-- EXPLAIN ANALYZE ACTUALLY RUNS THE QUERY.
-- For SELECT: no problem (reads only)
-- For INSERT/UPDATE/DELETE: WRAP IN A TRANSACTION AND ROLLBACK!

BEGIN;
EXPLAIN ANALYZE DELETE FROM orders WHERE status = 'cancelled';
ROLLBACK;
-- The query runs, rows are deleted, then ROLLBACK undoes it.
-- Without the transaction: your data is gone.
```

---

## The Plan Tree — How to Read It

### Reading Direction

```
RULE 1: Read bottom-to-top, inside-out.
The innermost (most indented) nodes execute FIRST.
The root node (least indented) executes LAST and delivers the final result.

Sort  (root — executes last, delivers sorted result to client)
  └── Hash Join  (executes second — feeds sorted rows upward)
        ├── Seq Scan on orders  (executes first — reads orders)
        └── Hash               (executes first — builds hash table)
              └── Seq Scan on users  (executes first — reads users)

RULE 2: Indentation = parent-child relationship.
Each node's children are indented under it.
A node consumes rows from all its children.

RULE 3: Arrows (→) and └── show data flow direction.
Data flows UP from child to parent.
```

### Anatomy of a Single Plan Node

```
Hash Join  (cost=450.00..2100.50 rows=8500 width=64) (actual time=12.5..85.3 rows=8312 loops=1)
│           │     │       │        │       │           │      │     │     │      │      │
│           │     │       │        │       │           │      │     │     │      │      └── loops
│           │     │       │        │       │           │      │     │     │      └── actual rows returned
│           │     │       │        │       │           │      │     │     └── actual end time (ms)
│           │     │       │        │       │           │      │     └── actual start time (ms)
│           │     │       │        │       │           │      └── "actual" = EXPLAIN ANALYZE data
│           │     │       │        │       └── estimated avg row width (bytes)
│           │     │       │        └── estimated rows this node will return
│           │     │       └── estimated total cost (to return all rows)
│           │     └── estimated startup cost (before first row is returned)
│           └── cost section
└── node type

STARTUP COST vs TOTAL COST:
  startup = cost before the FIRST row is returned
            (Hash Join: must build the hash table before probing → high startup)
            (Seq Scan: 0.0 — can return first row immediately)
  total   = cost to return ALL rows

WHY THIS MATTERS:
  If your query has LIMIT 10, only the startup + a fraction of total cost matters.
  The planner knows this and may prefer a Nested Loop (low startup) over
  Hash Join (high startup) when LIMIT is present.

LOOPS:
  How many times this node was executed.
  "loops=1" = ran once (the common case for outer nodes)
  "loops=5000" = this node ran 5000 times (inner side of a Nested Loop!)
  
  ACTUAL TIME IS PER-LOOP — multiply by loops to get total time:
  (actual time=0.01..0.05 rows=1 loops=5000)
  → Total time = 0.05ms × 5000 = 250ms for this node alone
  → Total rows = 1 × 5000 = 5000 rows total
```

---

## Every Node Type — Complete Reference

### Scan Nodes (Access Methods)

#### Sequential Scan

```
Seq Scan on employees  (cost=0.00..1250.00 rows=50000 width=36)
                       (actual time=0.005..42.3 rows=49823 loops=1)
  Filter: (active = true)
  Rows Removed by Filter: 177

WHAT IT MEANS:
  Read every page of the table from start to finish.
  Apply filter predicate to each row (not from index).
  "Rows Removed by Filter" = rows that failed the WHERE condition.

WHEN THE PLANNER CHOOSES THIS:
  • No usable index for the WHERE condition
  • High selectivity (>10-15% of rows match) — sequential I/O is cheaper
  • Small table (index overhead not worth it)
  • After VACUUM, if statistics say most rows match

GOOD OR BAD?
  On a small table (< a few thousand rows): FINE
  On a large table with a very selective WHERE: INDEX MISSING or STALE STATISTICS
  On a large table with 70%+ match: CORRECT — seq scan is the right choice

PERFORMANCE NOTE:
  "Rows Removed by Filter: 177" is important — 177 rows were read from disk
  but discarded. If this number is very large (millions removed, few kept),
  you need a better index strategy.
```

#### Index Scan

```
Index Scan using idx_emp_salary on employees
  (cost=0.29..8.52 rows=1 width=36) (actual time=0.012..0.013 rows=1 loops=1)
  Index Cond: (id = 42)
  Buffers: shared hit=3

WHAT IT MEANS:
  Traverse the B-tree index to find matching rows.
  For each index entry found, fetch the corresponding heap page (random I/O).
  Apply any remaining conditions as a "Filter" (not "Index Cond").

TWO TYPES OF CONDITIONS:
  Index Cond: the condition was applied INSIDE the index traversal
              (only matching index entries were followed)
  Filter:     the condition was applied AFTER fetching the heap row
              (checked on the heap page, still causes random I/O)

EXAMPLE:
  Index on (status, created_at)
  WHERE status = 'active' AND amount > 100
  
  Index Cond: (status = 'active')    ← applied at index level
  Filter: (amount > 100)             ← applied after fetching heap row
  
  If many rows match status='active' but few match amount>100:
  → Many heap fetches (random I/O) with high filter rejections
  → Consider adding amount to the index: (status, created_at, amount)
    to make amount an Index Cond too, or use a covering index

WHEN CHOSEN: Low selectivity (< 5-15% depending on correlation)
COST: random_page_cost per heap page fetched (default 4.0)
```

#### Index Only Scan

```
Index Only Scan using idx_emp_dept_salary on employees
  (cost=0.29..45.30 rows=500 width=8) (actual time=0.018..1.25 rows=487 loops=1)
  Index Cond: (department_id = 5)
  Heap Fetches: 12
  Buffers: shared hit=8

WHAT IT MEANS:
  ALL required columns are in the index — no heap access needed.
  The executor answers the query entirely from the index pages.
  "Heap Fetches: 12" = 12 rows required heap access anyway.

WHY HEAP FETCHES > 0:
  PostgreSQL needs to check if a row is visible (MVCC).
  The visibility map tracks which pages are "all visible" (no dead tuples).
  If a page is not "all-visible", the executor must fetch the heap row to check.
  After VACUUM marks pages all-visible, Heap Fetches drops to 0.

THIS IS THE FASTEST SCAN TYPE:
  No random heap I/O = dramatically cheaper for moderately selective queries.
  To enable: create index covering ALL columns needed by the query.
  Use INCLUDE clause for non-searchable columns: 
    CREATE INDEX ON employees(department_id) INCLUDE (name, salary);

WHEN CHOSEN: All projected columns + filter columns are in the index
```

#### Bitmap Index Scan + Bitmap Heap Scan

```
Bitmap Heap Scan on employees  (cost=120.50..850.25 rows=3000 width=36)
                               (actual time=2.5..18.3 rows=2947 loops=1)
  Recheck Cond: (city = 'London')
  Heap Blocks: exact=1820
  Buffers: shared hit=1835
  ->  Bitmap Index Scan on idx_emp_city  (cost=0.00..119.75 rows=3000 width=0)
                                         (actual time=2.1..2.1 rows=2947 loops=1)
        Index Cond: (city = 'London')
        Buffers: shared hit=15

WHAT IT MEANS:
  Two-phase scan:
  Phase 1 (Bitmap Index Scan): 
    Scan the index and build an IN-MEMORY BITMAP.
    Each bit = one heap page.
    Bit is SET if any row on that page matches the condition.
    Does NOT access heap pages yet.
  
  Phase 2 (Bitmap Heap Scan):
    Read heap pages whose bit is set — IN PAGE ORDER (sequential-ish I/O).
    Recheck the condition for each row (because bitmap is page-level, not row-level).
    Return matching rows.

WHY TWO PHASES?
  Simple Index Scan: 3000 rows → 3000 random heap I/Os → expensive
  Bitmap approach: build bitmap → sort page list → read 1820 pages sequentially
  Converting random I/O to near-sequential I/O — much cheaper.

"Heap Blocks: exact=1820"
  exact = precise bitmap fit in memory (each bit = one row, not one page)
  lossy = bitmap too large for memory → degraded to page-level → more rechecks

"Recheck Cond" = always present in Bitmap Heap Scan
  (the bitmap may include false positives — the filter re-checks on heap)

WHEN CHOSEN: Moderate selectivity (roughly 1-15%)
SPECIAL POWER: Can combine multiple indexes with BitmapAnd / BitmapOr:
  ->  BitmapAnd
        ->  Bitmap Index Scan on idx_city (city = 'London')
        ->  Bitmap Index Scan on idx_active (active = true)
  Combines TWO single-column indexes to serve a multi-column WHERE condition.
```

---

### Join Nodes

#### Nested Loop Join

```
Nested Loop  (cost=0.58..250.45 rows=87 width=52) (actual time=0.025..8.5 rows=87 loops=1)
  ->  Index Scan using idx_orders_user on orders  (cost=0.29..120.50 rows=87 width=28)
        (actual time=0.012..4.2 rows=87 loops=1)
        Index Cond: (status = 'active')
  ->  Index Scan using users_pkey on users  (cost=0.29..1.50 rows=1 width=24)
        (actual time=0.003..0.003 rows=1 loops=87)
        Index Cond: (id = orders.user_id)
        Buffers: shared hit=261

READ THIS:
  Outer (top child): runs ONCE → 87 rows
  Inner (bottom child): runs 87 TIMES (loops=87), once per outer row
  Each inner execution: looks up users by primary key

COST MODEL:
  For each of 87 outer rows:
    Probe inner index → 1 row (fast — primary key lookup)
  Total: 87 × (B-tree traversal cost) ≈ 87 × 3 pages = 261 buffer hits ✓

WHEN THE PLANNER CHOOSES NESTED LOOP:
  • Small outer result (few rows to loop over)
  • Inner table has an index on the join key
  • LIMIT is present (can stop after LIMIT rows without full scan)
  • Correlated data (inner lookups are likely cache-hits)

WHEN NESTED LOOP IS DANGEROUS:
  loops=50000 with (actual time=0.02..0.04 rows=1 loops=50000)
  → 50,000 index lookups on inner table
  → If these cause cache misses: 50,000 × random_page_cost → very slow
  → Symptom: estimated loops=100, actual loops=50000 (bad estimate → bad plan)
```

#### Hash Join

```
Hash Join  (cost=1250.00..8500.50 rows=45000 width=52)
           (actual time=125.3..850.4 rows=44823 loops=1)
  Hash Cond: (orders.user_id = users.id)
  ->  Seq Scan on orders  (cost=0.00..6500.00 rows=300000 width=28)
        (actual time=0.01..320.5 rows=300000 loops=1)
  ->  Hash  (cost=750.00..750.00 rows=50000 width=24)
            (actual time=124.8..124.8 rows=50000 loops=1)
        Buckets: 65536  Memory Usage: 3200kB
        ->  Seq Scan on users  (cost=0.00..750.00 rows=50000 width=24)
              (actual time=0.01..55.3 rows=50000 loops=1)

READ THIS:
  Build phase: Scan users → build hash table (Memory: 3200kB)
  Probe phase: Scan orders → for each order row, probe hash table
  Hash Cond: the equality condition used to probe the hash

"Memory Usage: 3200kB" — THIS IS CRITICAL:
  If Memory Usage > work_mem (default 4MB): hash join splits into BATCHES
  Batches > 1 means hash join spilled to disk → dramatic slowdown
  
  Watch for:
  Hash  (... Batches: 4  Memory Usage: 4096kB  Disk Usage: 45000kB)
  → 4 batches because hash table didn't fit in work_mem
  → Fix: SET work_mem = '64MB' or increase globally

WHEN THE PLANNER CHOOSES HASH JOIN:
  • Large outer table (too many rows for nested loop)
  • No usable index on join key (or index not selective enough)
  • Equality join condition (hash requires equality)
  • Planner estimates hash table fits in work_mem

BUCKETS:
  Power of 2 sized hash table.
  If actual rows >> estimated rows → too few buckets → hash collisions → slower
  (Another reason why accurate row estimates matter)
```

#### Merge Join

```
Merge Join  (cost=1850.00..3200.50 rows=25000 width=52)
            (actual time=285.4..520.8 rows=24915 loops=1)
  Merge Cond: (orders.user_id = users.id)
  ->  Sort  (cost=900.00..962.50 rows=25000 width=28)
            (actual time=142.5..155.3 rows=25000 loops=1)
        Sort Key: orders.user_id
        Sort Method: external merge  Disk: 1800kB  ← DISK SORT!
        ->  Seq Scan on orders ...
  ->  Sort  (cost=550.00..562.50 rows=5000 width=24)
            (actual time=85.2..87.4 rows=5000 loops=1)
        Sort Key: users.id
        Sort Method: quicksort  Memory: 480kB
        ->  Seq Scan on users ...

READ THIS:
  Both inputs are sorted by the join key.
  Then a single pass through both sorted streams finds matches (like merging two sorted arrays).

"Sort Method: external merge  Disk: 1800kB"
  → Sort spilled to disk (input exceeded work_mem)
  → This is a significant performance problem
  → Fix: increase work_mem, or ensure the data comes pre-sorted from an index

"Sort Method: quicksort  Memory: 480kB"
  → Sort fit entirely in memory (good)

WHEN THE PLANNER CHOOSES MERGE JOIN:
  • One or both inputs are already sorted by join key (index provides order)
  • Both tables are large (hash join would need huge hash table)
  • Range join conditions (BETWEEN, <, >) — hash join can't do these, merge can

WHEN MERGE JOIN IS FREE:
  If an Index Scan already returns rows sorted by the join key:
  No Sort node needed → the index provided the sort → merge join for free
  EXPLAIN shows: "-> Index Scan (not "Sort -> Index Scan")"
```

---

### Aggregation Nodes

#### HashAggregate

```
HashAggregate  (cost=2800.00..2900.00 rows=500 width=44)
               (actual time=285.3..295.8 rows=487 loops=1)
  Group Key: department_id, name
  Batches: 1  Memory Usage: 280kB

  vs

HashAggregate  (...) (actual time=...)
  Group Key: department_id
  Batches: 4  Memory Usage: 4096kB  Disk Usage: 12000kB
  ← SPILLED TO DISK

WHAT IT MEANS:
  Build a hash table keyed on the GROUP BY columns.
  For each input row: find or create the group's entry, update aggregates.
  When all input consumed: emit one row per group.

"Batches: 1" = all groups fit in memory → good
"Batches: 4" = too many distinct groups → spilled to disk → bad

WHEN CHOSEN: GROUP BY on unsorted data, moderate cardinality groups
vs SORT AGGREGATE: when input is already sorted by GROUP BY key, or when
  sort-based grouping is cheaper (low cardinality groups on large data)
```

#### Sort + GroupAggregate

```
GroupAggregate  (cost=1850.00..2100.00 rows=500 width=44)
                (actual time=180.5..195.3 rows=487 loops=1)
  Group Key: department_id, name
  ->  Sort  (cost=1850.00..1875.00 rows=10000 width=36)
            (actual time=178.2..179.8 rows=10000 loops=1)
        Sort Key: department_id, name
        Sort Method: quicksort  Memory: 1200kB
        ->  Seq Scan on employees ...

WHAT IT MEANS:
  Sort the input by GROUP BY key first.
  Then scan the sorted stream: when the key changes, emit a group.
  Like reading a sorted list and collecting runs.

WHEN CHOSEN:
  • Input is already sorted (index scan order matches GROUP BY) → sort is free
  • Low cardinality (few distinct groups) → sort cheaper than hash table
  • Very large number of distinct groups → hash table would exceed work_mem

GROUPAGGREGATE vs HASHAGGGREGATE:
  HashAggregate: random group access, better for high cardinality, needs memory
  GroupAggregate: sequential access, better for low cardinality or pre-sorted input
```

---

### Other Important Nodes

#### Sort

```
Sort  (cost=1250.80..1275.80 rows=10000 width=28)
      (actual time=45.2..47.8 rows=10000 loops=1)
  Sort Key: salary DESC
  Sort Method: quicksort  Memory: 1100kB

  vs

Sort  (cost=...) (actual time=...)
  Sort Key: created_at DESC
  Sort Method: external merge  Disk: 45000kB  ← BIG PROBLEM

SORT METHODS:
  quicksort     → all data fits in work_mem → in-memory sort → fast
  top-N heapsort → LIMIT present → only keeps N rows in heap → very efficient
  external merge → data exceeds work_mem → spills to disk → dramatically slow

"top-N heapsort  Memory: 32kB"
  → The planner recognized LIMIT N and used a heap-based top-N selection
  → Only needs to track N rows at a time → tiny memory usage
  → This is the optimal sort strategy when LIMIT is present

FIX FOR external merge:
  SET work_mem = '64MB';  -- for the session
  -- or increase globally in postgresql.conf
  -- Rule of thumb: work_mem should be at least 2× the sort buffer size
  -- Check: "Disk: Xkb" → set work_mem to at least 2X
```

#### Limit

```
Limit  (cost=0.56..8.44 rows=10 width=36) (actual time=0.025..0.089 rows=10 loops=1)
  ->  Index Scan Backward using idx_emp_salary on employees ...
        (actual time=0.022..0.081 rows=10 loops=1)
        Index Cond: (department_id = 5)

WHAT IT MEANS:
  Stop pulling rows after N rows are produced.
  
CRITICAL PERFORMANCE INTERACTION:
  Limit "shortcuts" the executor — when Limit is satisfied, it stops pulling.
  This means inner nodes may not complete:
  
  Limit rows=10
    → Sort: sorts only enough to produce 10 rows (top-N heapsort)
    → Nested Loop inner: only runs for as many outer rows as needed
    → Seq Scan: may stop before reading all pages
  
  This is why EXPLAIN shows "Index Scan Backward" for ORDER BY salary DESC LIMIT 10:
  The planner uses the index in reverse order (DESC) and stops after 10 rows.
  No Sort node needed at all — the index provides the sort order for free.
```

#### WindowAgg

```
WindowAgg  (cost=1850.00..2100.00 rows=10000 width=52)
           (actual time=180.5..215.3 rows=10000 loops=1)
  ->  Sort  (cost=1850.00..1875.00 rows=10000 width=44)
            (actual time=178.5..182.3 rows=10000 loops=1)
        Sort Key: department_id, salary DESC
        Sort Method: quicksort  Memory: 1100kB
        ->  Seq Scan on employees ...

WHAT IT MEANS:
  Computes window functions (OVER clause).
  The Sort below it provides the ordering needed by PARTITION BY + ORDER BY.
  WindowAgg then processes the sorted stream, computing values per window frame.

ONE SORT PER WINDOW DEFINITION:
  If you have multiple window functions with different ORDER BY:
    RANK() OVER (PARTITION BY dept ORDER BY salary DESC)
    LAG(salary) OVER (PARTITION BY dept ORDER BY hire_date)
  These require TWO Sort nodes (different sort keys).
  
  If both use the SAME ORDER BY:
    RANK() OVER (ORDER BY salary DESC)
    SUM(salary) OVER (ORDER BY salary DESC)
  The planner can share ONE Sort node for both.
  
  Using the WINDOW clause makes this explicit:
    WINDOW w AS (PARTITION BY dept ORDER BY salary DESC)
    RANK() OVER w, SUM(salary) OVER w
  → Guaranteed single sort.
```

---

## The Numbers That Matter

### Estimated vs Actual Rows — The Most Important Signal

```
Node: Hash Join  (cost=..  rows=100 width=..)  (actual time=.. rows=48500 loops=1)
                              ↑                                  ↑
                         estimated 100                     actual 48,500
                              ← 485x off! →

This gap is the MOST IMPORTANT thing in any EXPLAIN ANALYZE output.

INTERPRETATION:
  estimated ≈ actual    → statistics are good, plan decisions are correct
  estimated << actual   → planner underestimated → chose wrong strategy
                          (Nested Loop instead of Hash Join, etc.)
  estimated >> actual   → planner overestimated → chose overly aggressive strategy
                          (might have built a huge hash table for 10 rows)

THE IMPACT OF A BAD ESTIMATE CASCADES:
  If a Seq Scan returns 100x more rows than expected:
  → The join node above it receives 100x more rows than expected
  → The aggregate above that processes 100x more groups
  → Every node above the bad estimate is potentially using the wrong algorithm

FIX: ANALYZE the table with bad estimates.
VERIFY: Re-run EXPLAIN ANALYZE and check if estimated ≈ actual now.
```

### Buffer Statistics (EXPLAIN ANALYZE BUFFERS)

```
Seq Scan on orders  (...)
  Buffers: shared hit=12500 read=8500 dirtied=0 written=0

WHAT EACH MEANS:
  shared hit   = pages served from the buffer pool (in-memory) → FAST
  shared read  = pages read from disk into buffer pool → SLOW
  shared dirtied = pages modified during this query (for DML queries)
  shared written = dirty pages written to disk during this query (rare for SELECT)
  local hit/read = temporary table accesses

KEY METRIC: HIT RATIO
  Total pages = hit + read = 12500 + 8500 = 21000
  Hit ratio = 12500 / 21000 = 59.5%
  
  Good: > 95% hit ratio (most data in buffer pool)
  Concerning: < 80% hit ratio (many disk reads)
  Bad: very high read counts on critical tables

I/O BOUND vs CPU BOUND:
  High "read" with fast execution time → likely cached in OS page cache
  High "read" with slow execution time → real disk I/O (spinning disk or cold SSD)
  High "hit" with slow execution time → CPU bound (no more I/O to save, need algorithm change)

WHAT TO DO WITH THIS:
  Large "read" on a table → consider more shared_buffers, or schedule work during off-peak
  Large "read" on a specific index scan → consider if data fits in cache vs needs partitioning
  All "hit", still slow → the bottleneck is CPU (sorts, hash joins, aggregations)
```

### Timing

```
Planning Time: 0.385 ms   ← time in Stage 4 (planner)
Execution Time: 8502.3 ms ← time in Stage 5 (executor)

INTERPRETING TIMING:
  Planning time > 10ms: complex query with many tables — consider simplifying
  Planning time > execution time: query is trivially fast but over-engineered

  For execution time, look at the actual time of individual nodes:
  
  Sort  (actual time=4500..4502 rows=50000 loops=1)
  → Sort alone took 4.5 seconds!
  → This is your bottleneck. Everything else is fast.
  
  Hash  (actual time=2100..2100 rows=0 loops=1)
  Memory Usage: 4096kB  Disk Usage: 45000kB
  → Hash build spilled to disk (45MB!) — major bottleneck.

FINDING THE BOTTLENECK SYSTEMATICALLY:
  1. Find the node with the highest actual time (start..end)
  2. Check if it's a Seq Scan on a large table → needs index
  3. Check if it's a Sort with "Disk" → needs more work_mem
  4. Check if it's a Hash with "Batches > 1" → hash spill → more work_mem
  5. Check if it's a Nested Loop with many loops → bad estimate caused wrong join type
  6. Check estimated vs actual rows at that node → if wildly off → ANALYZE
```

---

## EXPLAIN Output for This Pattern

```sql
-- A representative complex query to demonstrate ALL the node types at once:
EXPLAIN (ANALYZE, BUFFERS)
SELECT
  d.name                                            AS department,
  e.name                                            AS employee,
  e.salary,
  RANK() OVER (PARTITION BY d.id ORDER BY e.salary DESC) AS dept_rank,
  AVG(e.salary) OVER (PARTITION BY d.id)            AS dept_avg_salary
FROM departments d
JOIN employees e ON d.id = e.department_id
WHERE d.active = true
  AND e.salary > 50000
ORDER BY d.name, dept_rank
LIMIT 50;
```

```
Limit  (cost=1850.22..1850.35 rows=50 width=68) (actual time=185.3..185.4 rows=50 loops=1)
│      Buffers: shared hit=892
│
│  ← LIMIT: stops after 50 rows. top-N heap would be used inside Sort if applicable.
│    Note: Limit feeds from Sort — all rows must be sorted before Limit can cut.
│    Except if the Sort uses top-N heapsort — then only 50 rows are kept in heap.
│
└── Sort  (cost=1850.22..1860.22 rows=4000 width=68) (actual time=185.2..185.3 rows=50 loops=1)
    │  Sort Key: d.name, (rank() OVER (...))
    │  Sort Method: top-N heapsort  Memory: 26kB   ← Optimal! LIMIT allowed top-N.
    │
    │  ← Sort for final ORDER BY d.name, dept_rank.
    │    "top-N heapsort" = keeps only 50 rows at a time = tiny memory.
    │    This is optimal because LIMIT 50 is present.
    │
    └── WindowAgg  (cost=1650.00..1820.22 rows=4000 width=68)
        │          (actual time=155.2..183.5 rows=4000 loops=1)
        │  Buffers: shared hit=892
        │
        │  ← Window functions computed here (RANK, AVG OVER).
        │    Receives rows already sorted by (department_id, salary DESC)
        │    from the Sort node below.
        │
        └── Sort  (cost=1650.00..1660.00 rows=4000 width=52)
            │     (actual time=153.8..155.1 rows=4000 loops=1)
            │  Sort Key: d.id, e.salary DESC
            │  Sort Method: quicksort  Memory: 580kB  ← fits in work_mem
            │
            │  ← Sort for PARTITION BY d.id ORDER BY e.salary DESC.
            │    WindowAgg needs rows sorted this way to compute RANK and AVG correctly.
            │    "quicksort  Memory: 580kB" → all fits in work_mem. No disk spill.
            │
            └── Hash Join  (cost=85.00..1420.50 rows=4000 width=48)
                │          (actual time=5.2..138.5 rows=4000 loops=1)
                │  Hash Cond: (e.department_id = d.id)
                │  Buffers: shared hit=892
                │
                │  ← Hash Join between employees and departments.
                │    Hash table built on departments (small table).
                │    Probed by employees rows.
                │
                ├── Seq Scan on employees e  (cost=0.00..1300.00 rows=4000 width=36)
                │                            (actual time=0.008..55.2 rows=4000 loops=1)
                │   Filter: (salary > 50000)
                │   Rows Removed by Filter: 46000
                │   Buffers: shared hit=850
                │
                │   ← Seq Scan because salary > 50000 matches 8% of rows.
                │     "Rows Removed by Filter: 46000" = 46K rows read, discarded.
                │     If this table grows 10x, consider index on salary.
                │
                └── Hash  (cost=60.00..60.00 rows=200 width=20)
                          (actual time=5.1..5.1 rows=200 loops=1)
                     Buckets: 1024  Memory Usage: 18kB
                     →  Seq Scan on departments d  (cost=0.00..60.00 rows=200 width=20)
                                                   (actual time=0.005..2.3 rows=200 loops=1)
                           Filter: (active = true)
                           Rows Removed by Filter: 50
                           Buffers: shared hit=42

Planning Time: 0.485 ms
Execution Time: 185.8 ms

FULL BOTTOM-TO-TOP TRACE:
1. Seq Scan departments: read 250 rows, filter to 200 active ones
2. Hash (departments): build 18kB hash table (tiny — 200 rows)
3. Seq Scan employees: read 50000 rows, filter to 4000 with salary > 50k
4. Hash Join: probe hash table for each of 4000 employee rows → 4000 joined rows
5. Sort (dept/salary): sort 4000 rows by (department_id, salary DESC) for windows
6. WindowAgg: compute RANK and AVG over sorted rows → 4000 rows with window values
7. Sort (final): top-N heapsort → keep only 50 rows by (d.name, dept_rank)
8. Limit: return exactly 50 rows
```

---

## Example 1 — Basic: Your First EXPLAIN Output

```sql
-- Run this and read the output:
EXPLAIN SELECT name, email FROM users WHERE id = 1;

-- Output:
Index Scan using users_pkey on users  (cost=0.42..8.44 rows=1 width=36)
  Index Cond: (id = 1)

-- READ IT:
-- Node type: Index Scan (using the primary key B-tree)
-- Table: users
-- Cost: startup=0.42, total=8.44
-- Estimated rows: 1 (primary key lookup → always 1)
-- Width: 36 bytes per output row (name + email column widths combined)
-- Index Cond: the condition was evaluated inside the B-tree traversal

-- NO ANALYZE here → no actual timing. Just the plan.
-- Cost of 8.44 = B-tree traversal (3 pages) × random_page_cost(4.0) / page +
--               1 heap fetch × random_page_cost + cpu costs
-- This is very cheap. A simple lookup.

-- NOW ADD ANALYZE:
EXPLAIN ANALYZE SELECT name, email FROM users WHERE id = 1;

-- Output:
Index Scan using users_pkey on users  (cost=0.42..8.44 rows=1 width=36)
                                      (actual time=0.015..0.016 rows=1 loops=1)
  Index Cond: (id = 1)
Planning Time: 0.085 ms
Execution Time: 0.035 ms

-- actual time=0.015..0.016 → started at 0.015ms, finished at 0.016ms
-- rows=1 → found exactly 1 row (matches the estimate of 1 — perfect)
-- loops=1 → ran once
-- Total time: 0.035ms — essentially instant
```

---

## Example 2 — Intermediate: Spotting the Problem

```sql
-- A query that looks innocent but has a hidden problem:
EXPLAIN ANALYZE
SELECT u.name, COUNT(o.id) AS orders
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2025-01-01'
GROUP BY u.id, u.name;
```

```
HashAggregate  (cost=15000.00..15100.00 rows=500)  (actual time=8520.3..8545.2 rows=48500 loops=1)
│              ← estimated 500 groups, actual 48,500 groups!
│                This is a 97x underestimate. HashAggregate allocated wrong bucket count.
│                Statistics for created_at are stale — ANALYZE users needed.
│
  Group Key: u.id, u.name
  Batches: 8  Memory Usage: 4096kB  Disk Usage: 82000kB
  ← 8 batches! 82MB disk spill! The hash table overflowed because
    there were 48,500 groups, not 500. Hash table was 97x too small.
    If the planner had known about 48,500 groups, it might have chosen
    Sort+GroupAggregate (which doesn't need to fit everything in memory).
    
  ->  Hash Right Join  (cost=5000.00..12000.00 rows=250000)
                       (actual time=120.5..6500.3 rows=22000000 loops=1)
                       ← estimated 250K rows, actual 22 MILLION rows
                         This is the root cause. Exploded join product.
      Hash Cond: (o.user_id = u.id)
      ->  Seq Scan on orders  (actual rows=22000000 loops=1)
      ->  Hash  (actual rows=48500 loops=1)
            ->  Seq Scan on users  (actual rows=48500 loops=1)
                  Filter: (created_at > '2025-01-01')
                  Rows Removed by Filter: 1551500
                  ← 1.5M users filtered out → 48,500 passed
                     Planner estimated only 500 users. 97x off.
                     The created_at statistics are stale.

DIAGNOSIS:
1. Root cause: Seq Scan on users estimated 500 but returned 48,500.
   → stale statistics for created_at column
   → Fix: ANALYZE users

2. Consequence: HashAggregate received 22M rows, built for 250K
   → 8 disk batches (82MB disk spill)
   → Fix comes automatically after ANALYZE (fewer users → fewer rows → correct plan)

3. The RIGHT JOIN: 48,500 users × ~450 orders per user average = 22M rows
   That's actually correct after fixing the user count.
   But with correct statistics (48,500 users, not 500), the planner would
   choose different join strategy.

IMMEDIATE FIX:
ANALYZE users;
-- Re-run EXPLAIN ANALYZE to verify estimated ≈ actual
```

---

## Example 3 — Production Grade: Complete Investigation Workflow

```sql
-- Slow production query: 45 seconds
-- Task: diagnose using EXPLAIN (ANALYZE, BUFFERS)

EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT
  p.id, p.name, p.category_id,
  SUM(oi.quantity * oi.unit_price) AS revenue,
  COUNT(DISTINCT o.user_id)         AS unique_buyers
FROM products p
JOIN order_items oi ON p.id = oi.product_id
JOIN orders o       ON oi.order_id = o.id
WHERE o.created_at BETWEEN '2026-01-01' AND '2026-03-31'
  AND p.category_id = 15
GROUP BY p.id, p.name, p.category_id
ORDER BY revenue DESC
LIMIT 20;
```

```
-- ACTUAL OUTPUT:
Limit  (...) (actual time=45123.5..45123.8 rows=20 loops=1)
  Buffers: shared hit=82500 read=245000   ← 245K disk reads = I/O bound
  ->  Sort  (...) (actual time=45120.4..45122.1 rows=20 loops=1)
        Sort Method: top-N heapsort  Memory: 28kB   ← good, LIMIT optimised
        ->  HashAggregate  (...) (actual time=44850.3..45100.2 rows=1850 loops=1)
              Batches: 1  Memory Usage: 320kB   ← good, fits in memory
              ->  Hash Join  (...) (actual time=8500.4..42000.3 rows=4500000 loops=1)
                    Hash Cond: (oi.order_id = o.id)
                    Buffers: shared hit=80000 read=245000
                    ->  Hash Join  (...) (actual time=150.5..9500.2 rows=5200000 loops=1)
                          Hash Cond: (oi.product_id = p.id)
                          ->  Seq Scan on order_items oi  (...) (rows=25000000 loops=1)
                                Buffers: shared hit=45000 read=195000
                                ← 195K disk reads on order_items! 195K × 8KB pages = 1.5GB read
                          ->  Hash  (... rows=1200 loops=1)
                                ->  Index Scan on idx_products_category on products
                                      Index Cond: (category_id = 15)
                                      ← Good: only 1200 products in category 15
                    ->  Hash  (...) (actual time=8498.3..8498.3 rows=1850000 loops=1)
                          Buckets: 2097152  Memory Usage: 115MB
                          ->  Seq Scan on orders  (...) (rows=25000000 loops=1)
                                Filter: (created_at BETWEEN ...)
                                Rows Removed by Filter: 23150000   ← scanning 25M, keeping 1.85M
                                Buffers: shared hit=35000 read=50000

Planning Time: 2.3 ms
Execution Time: 45128.5 ms
```

```
STEP-BY-STEP DIAGNOSIS:

PROBLEM 1 (CRITICAL): Seq Scan on orders, 23.15M rows removed by filter
  → orders has NO index on created_at
  → Every query with date range scans all 25M orders
  → 50K pages read from disk (50K × 8KB = 400MB of reads)
  → Fix: CREATE INDEX ON orders(created_at)
  → Better: CREATE INDEX ON orders(created_at, order_id)  ← covers the join too

PROBLEM 2 (CRITICAL): Seq Scan on order_items, 195K disk reads
  → order_items has 25M rows, no useful index for this join path
  → order_items.order_id join needs an index
  → Fix: CREATE INDEX ON order_items(order_id)
  → (This likely already exists for FK — check: \d order_items in psql)

PROBLEM 3 (NOTABLE): Hash on orders, Memory Usage: 115MB
  → 1.85M orders loaded into 115MB hash table
  → With fix #1 (index on created_at), fewer orders match → smaller hash table
  → After the index, maybe 200K orders match → 12MB hash table → much better

APPLY FIXES:
CREATE INDEX CONCURRENTLY idx_orders_created_at ON orders(created_at);
CREATE INDEX CONCURRENTLY idx_order_items_order_id ON order_items(order_id);

ANALYZE orders;
ANALYZE order_items;

RE-RUN EXPLAIN ANALYZE (expected after fixes):
  -- Bitmap Index Scan on idx_orders_created_at
  -- Only ~1.85M orders read (not 25M), minimal disk reads
  -- Hash Join on order_items now uses index → nested loop or better
  -- Expected time: ~200ms (from 45 seconds)
```

---

## Wrong Query → What Breaks → Right Query

### Reading a Plan Upside Down

```sql
-- WRONG MENTAL MODEL: "The first thing I see at the top runs first."

-- EXPLAIN output:
Sort  (cost=1250.80..1275.80 rows=10000 width=28)  ← SHOWS AT TOP
  ->  Hash Join  (cost=85.00..1100.80 rows=10000 width=28)
        ->  Seq Scan on orders  (cost=0.00..850.00 rows=25000 width=20)  ← ACTUALLY RUNS FIRST
        ->  Hash
              ->  Seq Scan on users  (cost=0.00..300.00 rows=5000 width=16)  ← ACTUALLY RUNS FIRST

-- WRONG: "Sort is the most expensive part — I should try to eliminate the ORDER BY."
-- REALITY: Sort cost includes all child costs. Reading top-to-bottom gives wrong intuition.

-- RIGHT: Read BOTTOM-TO-TOP.
-- Step 1: Seq Scan users (runs first, builds hash)
-- Step 2: Seq Scan orders (runs first, provides probe rows)
-- Step 3: Hash Join (joins the two)
-- Step 4: Sort (sorts the joined result)

-- The BOTTLENECK is wherever actual time is highest.
-- Use EXPLAIN ANALYZE and look for the node with the largest actual time span.
```

### Misinterpreting Loops

```sql
-- EASY MISTAKE: misreading node timing for nodes with loops > 1

-- EXPLAIN ANALYZE output:
Nested Loop  (actual time=0.02..350.5 rows=50000 loops=1)
  ->  Index Scan on orders  (actual time=0.01..4.2 rows=50000 loops=1)
  ->  Index Scan on users   (actual time=0.003..0.004 rows=1 loops=50000)
                             ←              ↑              ↑     ↑
                             total node time (per loop)    rows per loop
                             
-- WRONG READING: "users index scan took 0.004ms — very fast"
-- RIGHT READING: 0.004ms × 50000 loops = 200ms for this node total

-- Total time for users scan = (actual_end_time - actual_start_time) × loops
--                           = (0.004 - 0.003) × 50000
--                           = 0.001ms × 50000 = 50ms total

-- The actual_time shown is PER LOOP. Always multiply by loops for total.
-- This is one of the most common misreadings of EXPLAIN output.
```

---

## Performance Profile

```
EXPLAIN ANALYZE OVERHEAD:
  EXPLAIN ANALYZE runs the query and measures timing at every node.
  The instrumentation overhead is typically 5-15% on fast queries.
  On very fast queries (< 1ms), instrumentation overhead may dominate.
  For production: use pg_stat_statements to find slow queries without running them.
  Use EXPLAIN ANALYZE in development/staging for detailed diagnosis.

FORMAT JSON FOR TOOLING:
  EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) is machine-readable.
  Tools like:
    https://explain.depesz.com  — paste text output, visual analysis
    pganalyze                   — continuous production monitoring
    pev (plan explorer)         — tree visualization
    auto_explain extension      — logs slow query plans automatically

auto_explain SETUP (in postgresql.conf):
  shared_preload_libraries = 'auto_explain'
  auto_explain.log_min_duration = 1000  -- log plans for queries > 1 second
  auto_explain.log_analyze = true
  auto_explain.log_buffers = true
  -- Plans appear in PostgreSQL logs for queries exceeding threshold
  -- Critical for production where you can't run EXPLAIN manually
```

---

## Node.js Integration

```javascript
// === USING EXPLAIN FROM NODE.JS ===

const { Pool } = require('pg');
const pool = new Pool();

// 1. DEVELOPMENT: Explain a query before running it
async function explainQuery(sql, params = []) {
  const { rows } = await pool.query(
    `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sql}`,
    params
  );
  return rows[0]['QUERY PLAN'][0];
}

// Usage:
const plan = await explainQuery(
  'SELECT * FROM orders WHERE user_id = $1 AND status = $2',
  [userId, 'pending']
);

console.log('Planning Time:', plan['Planning Time'], 'ms');
console.log('Execution Time:', plan['Execution Time'], 'ms');

// Walk the plan tree recursively:
function findBottleneck(node, depth = 0) {
  const actualTime = node['Actual Total Time'] || 0;
  const estimatedRows = node['Plan Rows'];
  const actualRows = node['Actual Rows'];
  const loops = node['Actual Loops'] || 1;

  const rowRatio = actualRows > 0 ? actualRows / estimatedRows : 1;

  if (actualTime > 100) {  // node took > 100ms
    console.warn(`SLOW NODE at depth ${depth}: ${node['Node Type']}`);
    console.warn(`  Time: ${actualTime}ms`);
    console.warn(`  Est rows: ${estimatedRows}, Actual rows: ${actualRows} (${rowRatio.toFixed(1)}x)`);
    if (node['Sort Space Used']) console.warn(`  Sort disk: ${node['Sort Space Used']}kB`);
  }

  if (rowRatio > 10 || rowRatio < 0.1) {
    console.warn(`BAD ESTIMATE at depth ${depth}: ${node['Node Type']}`);
    console.warn(`  Est: ${estimatedRows}, Actual: ${actualRows} (${rowRatio.toFixed(1)}x off)`);
  }

  // Recurse into children
  if (node['Plans']) {
    node['Plans'].forEach(child => findBottleneck(child, depth + 1));
  }
}

findBottleneck(plan.Plan);

// 2. PRODUCTION: Log slow queries automatically
// Better: use auto_explain in PostgreSQL config.
// From Node.js: log query time and flag slow ones for manual EXPLAIN later.

async function timedQuery(sql, params = []) {
  const start = process.hrtime.bigint();
  const result = await pool.query(sql, params);
  const elapsed = Number(process.hrtime.bigint() - start) / 1_000_000; // ms

  if (elapsed > 500) {  // flag queries > 500ms
    console.warn(`SLOW QUERY (${elapsed.toFixed(1)}ms): ${sql.slice(0, 200)}`);
    // In production: log to monitoring system for investigation
    // Never run EXPLAIN ANALYZE in production hot path — it double-executes the query
  }

  return result;
}

// 3. EXPLAIN FOR WRITE QUERIES — ALWAYS WRAP IN TRANSACTION
async function explainWrite(sql, params = []) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const result = await client.query(
      `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sql}`,
      params
    );
    await client.query('ROLLBACK');  // Undo the write!
    return result.rows[0]['QUERY PLAN'][0];
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}

// Usage: safely explain a DELETE without actually deleting
const plan = await explainWrite(
  'DELETE FROM sessions WHERE expires_at < $1',
  [new Date()]
);
console.log('Plan:', plan.Plan['Node Type']);
console.log('Rows to delete:', plan.Plan['Actual Rows']);
```

### ORM Comparison — Getting EXPLAIN Output

```javascript
// === PRISMA ===
// No native EXPLAIN support. Must use $queryRaw.
const planResult = await prisma.$queryRaw`
  EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
  SELECT * FROM users WHERE email = ${email}
`;
// Returns the raw JSON plan. No ORM abstraction available.
// Limitation: you cannot $queryRaw a Prisma query builder query — you must
// write the SQL manually. This means the SQL you EXPLAIN may differ from what
// Prisma actually generates.
// Better approach: enable Prisma's query logging to see generated SQL, then explain that.

// Enable Prisma query logging:
const prisma = new PrismaClient({
  log: [{ level: 'query', emit: 'event' }]
});
prisma.$on('query', (e) => {
  console.log('SQL:', e.query);
  console.log('Params:', e.params);
  console.log('Duration:', e.duration, 'ms');
});

// === DRIZZLE ===
// Similar — use raw SQL for EXPLAIN:
import { sql } from 'drizzle-orm';
const plan = await db.execute(
  sql`EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT * FROM users WHERE email = ${email}`
);

// Drizzle has a .toSQL() method to get the generated SQL first:
const query = db.select().from(users).where(eq(users.email, email));
const { sql: sqlString, params } = query.toSQL();
console.log('Generated SQL:', sqlString); // Then manually run EXPLAIN on this

// === SEQUELIZE ===
const [results] = await sequelize.query(
  `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) SELECT * FROM "users" WHERE email = :email`,
  { replacements: { email }, type: QueryTypes.SELECT }
);

// To see Sequelize-generated SQL, enable logging:
const sequelize = new Sequelize(url, {
  logging: (sql, timing) => console.log(`SQL (${timing}ms): ${sql}`)
});

// === KNEX ===
// Knex has the most natural explain access:
const query = knex('users').where('email', email);
console.log('SQL:', query.toSQL().toNative()); // See the SQL

// Explain:
const plan = await knex.raw(
  `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${query.toSQL().toNative().sql}`,
  query.toSQL().toNative().bindings
);

// === VERDICT ===
// None of the ORMs make EXPLAIN easy.
// Best workflow regardless of ORM:
// 1. Enable query logging to capture generated SQL
// 2. Copy the SQL
// 3. Run EXPLAIN (ANALYZE, BUFFERS) manually in psql or pg admin
// 4. Or use auto_explain in postgresql.conf for production slow query analysis
```

---

## Practice Exercises

### Exercise 1 — Easy (Node Identification)

Match each EXPLAIN output fragment to the correct description:

**Fragment A:**
```
Seq Scan on orders  (cost=0.00..4500.00 rows=150000 width=28)
  Filter: (status = 'pending')
  Rows Removed by Filter: 1350000
```

**Fragment B:**
```
Index Only Scan using idx_users_email on users  (cost=0.29..8.31 rows=1 width=28)
  Index Cond: (email = 'test@example.com')
  Heap Fetches: 0
```

**Fragment C:**
```
Hash Join  (cost=1500.00..8900.00 rows=75000 width=52)
  Hash Cond: (o.user_id = u.id)
  ->  Seq Scan on orders o  (rows=500000)
  ->  Hash  (Memory Usage: 4500kB)
        ->  Seq Scan on users u  (rows=75000)
```

**Fragment D:**
```
Sort  (cost=25000.00..27500.00 rows=1000000 width=36)
  Sort Key: created_at DESC
  Sort Method: external merge  Disk: 125000kB
```

For each fragment:
1. What operation is happening?
2. Is there a problem? If yes, what is it?
3. What would you do to fix it?

```
-- Write your analysis here
```

### Exercise 2 — Medium (Estimate Gap Analysis)

You receive this EXPLAIN ANALYZE output:

```
HashAggregate  (cost=5000.00..5050.00 rows=100) (actual time=45000.3..45200.5 rows=85000 loops=1)
  Group Key: user_id
  Batches: 16  Memory Usage: 4096kB  Disk Usage: 280000kB
  ->  Nested Loop  (cost=0.58..4500.00 rows=50000) (actual time=0.04..35000.2 rows=25000000 loops=1)
        ->  Seq Scan on events  (cost=0.00..2000.00 rows=50000) (actual time=0.02..8500.3 rows=25000000 loops=1)
              Filter: (event_type = 'purchase')
              Rows Removed by Filter: 75000000
        ->  Index Scan using users_pkey on users  (cost=0.29..0.50 rows=1 width=24)
                                                  (actual time=0.0005..0.0006 rows=1 loops=25000000)
```

Answer these questions:
1. How many times was the users index scan executed? What is its total time?
2. The events scan estimated 50,000 rows but returned 25,000,000. What is the root cause?
3. The Nested Loop has 25,000,000 × users index lookup. What join type should the planner have chosen?
4. The HashAggregate has Batches=16 and 280MB disk spill. What caused this?
5. Write the complete fix plan in order of operations.

```sql
-- Write your analysis and fixes here
```

### Exercise 3 — Hard (Full Investigation)

You have a Node.js background job that runs every hour to generate a report. It used to take 2 seconds. After a traffic spike last week, it now takes 8 minutes. No code changed.

```sql
-- The query:
SELECT
  DATE_TRUNC('day', o.created_at) AS order_date,
  p.category_id,
  SUM(oi.quantity) AS units_sold,
  SUM(oi.quantity * oi.unit_price) AS revenue,
  COUNT(DISTINCT o.user_id) AS unique_buyers
FROM orders o
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.created_at >= NOW() - INTERVAL '30 days'
GROUP BY DATE_TRUNC('day', o.created_at), p.category_id
ORDER BY order_date DESC, revenue DESC;
```

EXPLAIN ANALYZE shows (key nodes only):
```
Sort (Sort Method: external merge  Disk: 85000kB)
  -> HashAggregate (Batches: 8  Disk: 320000kB)
       -> Hash Join (Hash on order_items, Memory: 3800MB)  ← !
            -> Seq Scan on orders (Rows Removed by Filter: 45000000)
            -> Hash Join
                 -> Seq Scan on order_items (rows=80000000)
                 -> Seq Scan on products (rows=500000)
```

Write your complete diagnosis and fix plan. Include:
1. The root cause of the regression (what changed after the traffic spike?)
2. Every problem in the plan with specific fix for each
3. The SQL for each fix
4. How you'd verify each fix worked
5. How you'd prevent this in the future (monitoring)

```sql
-- Write your investigation here
```

### Exercise 4 — Interview Simulation

**Interviewer shows you this EXPLAIN ANALYZE snippet:**

```
Nested Loop  (cost=0.29..45000.00 rows=1000) (actual time=0.05..92000.5 rows=1000 loops=1)
  ->  Seq Scan on audit_logs  (cost=0.00..25000.00 rows=1000) (actual time=0.02..15000.5 rows=1000 loops=1)
        Filter: (action = 'login' AND created_at > '2026-01-01')
        Rows Removed by Filter: 49000000
  ->  Index Scan using users_pkey on users  (cost=0.29..20.00 rows=1) (actual time=0.005..0.077 rows=1 loops=1000)
        Index Cond: (id = audit_logs.user_id)
```

**Interviewer asks:** "The query estimated 1000 rows and returned 1000 rows — the estimate was correct. So why is the query slow? What would you do?"

Write your complete answer. Note: this is a trick — the estimate was right but the query is still wrong. Explain why.

```
-- Write your interview answer here
```

---

## Interview Questions This Topic Covers

### Q: "How do you diagnose a slow query in PostgreSQL?"

**Junior answer:** "I run EXPLAIN and look for Seq Scans, then add indexes."

**Principal answer:** "I follow a systematic process. First, I run `EXPLAIN (ANALYZE, BUFFERS)` on the query to get the actual execution plan with real timing. I read the plan bottom-to-top, looking for the node with the highest actual time. Then I check three things: first, the estimated vs actual rows gap at each node — if they're wildly off, the root cause is stale statistics, and I run ANALYZE on the table. Second, disk spills: Sort nodes with 'external merge' or Hash nodes with 'Batches > 1' mean memory is exhausted, and I increase work_mem for that session. Third, I look at buffer stats — high 'shared read' means disk I/O bound, all 'shared hit' means CPU bound. A Seq Scan is only a problem if it's on a large table with high selectivity — if 90% of rows match, a Seq Scan is actually correct. An index is the fix for a Seq Scan only when the WHERE is selective and there's no appropriate index. After identifying the bottleneck, I form a hypothesis, implement the fix, re-run EXPLAIN ANALYZE, and verify improvement."

---

### Q: "What does 'loops' mean in EXPLAIN ANALYZE and why does it matter?"

**Junior answer:** "I think it shows how many times the query ran?"

**Principal answer:** "Loops shows how many times that specific plan node was executed. For a Nested Loop join, the inner side's loop count equals the number of rows in the outer side. For example, if the outer table returns 50,000 rows, the inner node executes 50,000 times. The critical thing about loops: the `actual time` shown per node is the per-loop time, not the total. So a node showing `actual time=0.004ms, rows=1, loops=50000` actually cost 200ms total — 50,000 × 0.004ms. This is one of the most common misreadings. When a Nested Loop has a high loop count, it either indicates a correct plan for a small join, or a catastrophically bad plan when the planner estimated 10 rows for the outer side but got 50,000 due to bad statistics."

---

### Q: "What's the difference between 'Index Cond' and 'Filter' in an Index Scan node?"

**Junior answer:** "I'm not sure — they both filter rows?"

**Principal answer:** "They filter at different levels with very different performance implications. 'Index Cond' is applied inside the B-tree index traversal itself — only index entries that match the condition are followed, so no heap page is fetched for non-matching rows. 'Filter' is applied after fetching the heap row — the executor already did the random I/O to get the row, then discards it if it doesn't match. So a large 'Rows Removed by Filter' count inside an Index Scan means many random heap fetches that produced nothing. The fix is to include the filter column in the index as an additional column, turning it into an Index Cond. For example, if the index is on `(status)` and the query has `WHERE status = 'active' AND amount > 1000`, the amount check is a Filter. Adding amount to the index: `CREATE INDEX ON orders(status, amount)` makes it an Index Cond — no wasted heap fetches."

---

## Mental Model Checkpoint

1. **You see `Sort Method: external merge  Disk: 45000kB` in an EXPLAIN ANALYZE. What does this mean, what is the most likely immediate fix, and what is the permanent fix?**

2. **A Nested Loop node shows `(actual time=0.01..0.02 rows=1 loops=100000)`. What is the total time this node consumed? Under what condition is this plan correct vs when is it a sign of a bad plan?**

3. **EXPLAIN shows `Hash  (Memory Usage: 4096kB  Batches: 8)`. What does Batches=8 tell you? What happened physically when the hash join executed?**

4. **You run EXPLAIN (without ANALYZE) on a query. It shows Seq Scan with estimated rows=50. You run EXPLAIN ANALYZE and get actual rows=5,000,000. The query takes 45 seconds. What exactly do you do, in order, to fix this?**

5. **You add an index on `orders(created_at)`. You run EXPLAIN ANALYZE and still see a Seq Scan on orders. The query has `WHERE created_at > '2026-01-01'`. What are the 3 possible reasons the index isn't used?**

6. **EXPLAIN ANALYZE shows Planning Time: 850ms, Execution Time: 2ms. The query runs 10,000 times per second. What is the problem and what are two solutions?**

7. **A developer says "EXPLAIN shows my query will return 1000 rows with low cost — so it must be fast." They run it and it takes 30 seconds. What could cause this discrepancy, and what additional information do you ask them to collect?**

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ EXPLAIN CHEAT SHEET                                                         │
├─────────────────────────────────────────────────────────────────────────────┤
│ COMMANDS                                                                    │
│  EXPLAIN sql                          plan only, no execution               │
│  EXPLAIN ANALYZE sql                  run + measure                         │
│  EXPLAIN (ANALYZE, BUFFERS) sql       run + measure + I/O stats             │
│  EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) sql   machine-readable             │
│  BEGIN; EXPLAIN ANALYZE delete...; ROLLBACK;   safe for write queries       │
├─────────────────────────────────────────────────────────────────────────────┤
│ READING THE PLAN                                                            │
│  • Bottom-to-top = execution order (innermost runs first)                   │
│  • Indentation = parent-child (child feeds parent)                          │
│  • (cost=X..Y rows=N) = estimated values                                    │
│  • (actual time=X..Y rows=N loops=M) = measured values                     │
│  • actual time is PER LOOP — multiply by loops for total                   │
├─────────────────────────────────────────────────────────────────────────────┤
│ SCAN NODES                                                                  │
│  Seq Scan        all pages, in order — good for >10-15% selectivity        │
│  Index Scan      B-tree + heap fetch — good for <5-10% selectivity         │
│  Index Only Scan all columns in index, no heap — fastest for read           │
│  Bitmap Heap Scan page-bitmap + sorted heap read — moderate selectivity     │
├─────────────────────────────────────────────────────────────────────────────┤
│ JOIN NODES                                                                  │
│  Nested Loop     outer × inner index lookup — small outer, indexed inner   │
│  Hash Join       build hash table + probe — large tables, no index         │
│  Merge Join      sort both + merge — already sorted inputs                 │
├─────────────────────────────────────────────────────────────────────────────┤
│ AGGREGATION NODES                                                           │
│  HashAggregate   hash table per group — moderate distinct groups            │
│  GroupAggregate  sequential scan of sorted groups — sorted input            │
├─────────────────────────────────────────────────────────────────────────────┤
│ SORT METHODS                                                                │
│  quicksort           fits in work_mem → fast                                │
│  top-N heapsort      LIMIT present → only keeps N rows → tiny memory       │
│  external merge      exceeds work_mem → DISK SPILL → fix: increase work_mem│
├─────────────────────────────────────────────────────────────────────────────┤
│ RED FLAGS                                                                   │
│  estimated ≠ actual rows (>5x off)   → ANALYZE the table                   │
│  Sort Method: external merge          → increase work_mem                   │
│  Hash Batches > 1                     → increase work_mem                   │
│  Rows Removed by Filter: huge number  → add/improve index                  │
│  Nested Loop loops > 10,000           → check if estimate drove wrong join  │
│  shared read >> shared hit            → I/O bound, check indexes/partitions │
│  Index Scan with large Filter count   → add filter col to index            │
├─────────────────────────────────────────────────────────────────────────────┤
│ PRODUCTION TOOLING                                                          │
│  auto_explain extension               automatic slow query plan logging     │
│  pg_stat_statements                   find slow queries without running them│
│  explain.depesz.com                   visual plan analysis (paste output)   │
│  pganalyze                            continuous production monitoring       │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Connected Topics

**Internals this builds on:**
- Topic 01: EXPLAIN shows Stage 4 output (planner); EXPLAIN ANALYZE shows Stage 5 execution
- Topic 02: Every logical step maps to a physical plan node
- Topic 03: The planner generates these plans — understanding cost model explains why nodes are chosen

**Topics this unlocks:**
- Topic 42–48: Every index topic uses EXPLAIN to verify index usage
- Topic 49: Query optimisation methodology is built around the EXPLAIN workflow
- Topic 68: EXPLAIN options deep dive (BUFFERS FORMAT JSON SETTINGS TIMING)
- Every performance topic in this curriculum — EXPLAIN is the universal debugging tool
