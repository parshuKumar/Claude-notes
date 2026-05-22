# 01 — The SQL Processing Pipeline
## Phase: Foundations (How SQL Actually Executes)
## Interview Priority: HIGH

---

## ELI5 — The Simple Analogy

Imagine you walk into a restaurant and say: "I want something spicy with chicken, not too expensive."

Here's what happens behind the scenes:
1. **The waiter hears your words** — makes sure you spoke a real language, not gibberish (PARSING)
2. **The waiter checks the menu** — "do we even have chicken? is 'spicy' a real category here?" (ANALYSIS)
3. **The kitchen manager rewrites your order** — "ah, 'not too expensive' means items under $15, and we have a daily special that matches — substitute that in" (REWRITING)
4. **The head chef plans the cooking order** — "should I grill the chicken first then add sauce, or marinate first? Which is faster given what's already prepped?" (PLANNING)
5. **The line cooks execute the plan** — actual cooking happens (EXECUTION)

Your SQL query goes through the exact same pipeline. You write English-like text. The database transforms it into a physical execution plan through 5 distinct stages. Understanding these stages is the foundation for understanding everything else in SQL.

---

## Internals Connection

When you type a SQL query and hit enter, you are NOT talking to the storage engine directly. You are handing a **string of text** to a sophisticated multi-stage compiler pipeline. This pipeline exists because:

- SQL is **declarative** — you say WHAT you want, not HOW to get it
- The database must figure out HOW — and it has thousands of possible strategies
- The difference between the best and worst strategy can be 10,000x in execution time

This connects directly to what you know about database internals:
- The **buffer pool** is what the executor reads pages from (not disk directly)
- The **B-tree indexes** are what the planner considers as access paths
- The **WAL** is what records changes during execution for durability
- The **MVCC visibility rules** are what the executor applies to filter dead tuples
- The **statistics catalog** is what the planner uses to estimate costs

Every decision the planner makes is based on the physical data structures underneath. The SQL pipeline is the bridge between your declarative text and those physical structures.

---

## The Execution Order Context

This is Topic 01 — it IS the context. Everything else builds on this.

The key insight: SQL is processed in **two very different orders**:

```
ORDER YOU WRITE SQL:          ORDER THE ENGINE PROCESSES IT:
SELECT ...                    1. Parser (syntax check)
FROM ...                      2. Analyzer (semantic check)
WHERE ...                     3. Rewriter (rule application)
GROUP BY ...                  4. Planner (strategy selection)
HAVING ...                    5. Executor (actual data retrieval)
ORDER BY ...
LIMIT ...
```

Within the executor itself, there is ANOTHER order — the logical execution order of clauses (Topic 02). But first, you must understand the 5-stage pipeline that transforms your text into an executable plan.

---

## What Is This?

The SQL processing pipeline is the complete sequence of transformations a SQL statement undergoes from the moment the database receives your query string to the moment it returns results. In PostgreSQL, this pipeline has **5 stages**: Parser, Analyzer (also called Semantic Analysis), Rewriter, Planner/Optimizer, and Executor. Each stage has a specific responsibility, produces a specific output, and can fail in specific ways.

This is not an implementation detail you can ignore. Understanding this pipeline tells you:
- Why certain syntax errors happen (parser stage)
- Why "column does not exist" errors happen (analyzer stage)
- Why views and rules work the way they do (rewriter stage)
- Why the same query can be fast or slow (planner stage)
- Why memory and I/O are consumed (executor stage)

---

## Why Does It Matter in Production?

Without understanding this pipeline:

1. **You cannot debug query performance** — you don't know WHERE the problem is. Is it a bad plan? Bad statistics? A rewrite rule interfering? You're guessing.

2. **You cannot understand EXPLAIN output** — EXPLAIN shows you the planner's output. If you don't know what the planner does, EXPLAIN is just noise.

3. **You misinterpret errors** — "prepared statement already exists" (parser/planner boundary), "relation does not exist" (analyzer), "permission denied" (analyzer) — knowing the stage tells you the fix.

4. **You cannot predict query behaviour** — will my view be inlined or materialised? Will my rule fire? Does my function get called once or per-row? These depend on which stage handles what.

5. **You cannot optimise systematically** — is the query slow because the planner chose wrong? Because statistics are stale? Because the executor is spilling to disk? Each requires a different fix.

---

## Syntax Breakdown — The 5 Stages

### Stage 1: PARSER (Lexer + Grammar)

```
INPUT:  Raw SQL string
        "SELECT name, salary FROM employees WHERE department = 'Engineering'"

OUTPUT: Parse tree (raw, unresolved AST)

WHAT HAPPENS:
┌─────────────────────────────────────────────────────────────────┐
│ LEXER (tokenisation)                                            │
│                                                                 │
│ "SELECT name, salary FROM employees WHERE department = 'Eng'"   │
│         ↓                                                       │
│ Tokens: [SELECT] [name] [,] [salary] [FROM] [employees]        │
│         [WHERE] [department] [=] ['Engineering']                │
│                                                                 │
│ The lexer breaks the string into tokens.                        │
│ It knows SQL keywords, identifiers, literals, operators.        │
│ It does NOT know if "employees" is a real table.                │
│ It does NOT know if "name" is a real column.                    │
│ It ONLY checks: "is this valid SQL grammar?"                    │
├─────────────────────────────────────────────────────────────────┤
│ GRAMMAR PARSER (syntax validation)                              │
│                                                                 │
│ Tokens → Parse Tree (Abstract Syntax Tree)                      │
│                                                                 │
│   SelectStmt                                                    │
│   ├── targetList: [ColumnRef("name"), ColumnRef("salary")]      │
│   ├── fromClause: [RangeVar("employees")]                       │
│   └── whereClause: A_Expr(=, ColumnRef("department"),           │
│                              Const("Engineering"))              │
│                                                                 │
│ This tree represents the STRUCTURE of your query.               │
│ "name", "salary", "employees" are just strings at this point.   │
│ The parser has no idea what they refer to.                       │
└─────────────────────────────────────────────────────────────────┘

ERRORS AT THIS STAGE:
- Syntax errors: "SELEC name FROM..." → syntax error at or near "SELEC"
- Unmatched parentheses: "SELECT (name FROM..." → syntax error
- Invalid token sequences: "SELECT FROM WHERE" → syntax error
- Missing required clauses: "SELECT" alone → syntax error

PERFORMANCE COST: Negligible. Parsing is O(n) where n = query length.
For typical queries this is microseconds. Only matters if you're
parsing millions of queries per second (connection pooler scenario).
```

### Stage 2: ANALYZER (Semantic Analysis)

```
INPUT:  Parse tree (raw AST with unresolved names)
OUTPUT: Query tree (resolved, typed, validated)

WHAT HAPPENS:
┌─────────────────────────────────────────────────────────────────┐
│ CATALOG LOOKUP                                                  │
│                                                                 │
│ The analyzer takes every name in the parse tree and resolves it │
│ against the system catalog (pg_class, pg_attribute, pg_type):   │
│                                                                 │
│ "employees" → pg_class lookup → OID 16384 (relation exists ✓)  │
│ "name"      → pg_attribute lookup → column #2, type text ✓     │
│ "salary"    → pg_attribute lookup → column #4, type numeric ✓  │
│ "department"→ pg_attribute lookup → column #3, type text ✓     │
│                                                                 │
│ Now the database KNOWS what physical objects you're referring to.│
├─────────────────────────────────────────────────────────────────┤
│ TYPE CHECKING & COERCION                                        │
│                                                                 │
│ WHERE department = 'Engineering'                                 │
│       ^^^^^^^^      ^^^^^^^^^^^^^                               │
│       type: text    type: unknown literal                       │
│                                                                 │
│ The analyzer resolves 'Engineering' as type text (compatible).   │
│ If you wrote WHERE salary = 'abc', type checking would FAIL.    │
│                                                                 │
│ Implicit coercions happen here:                                  │
│ WHERE id = '5'  → '5' is coerced from text to integer           │
│ WHERE price > 10 → 10 is already numeric, no coercion needed   │
├─────────────────────────────────────────────────────────────────┤
│ PERMISSION CHECKING                                             │
│                                                                 │
│ Does the current user have SELECT permission on employees?      │
│ Does the current user have access to columns name, salary, dept?│
│ (Column-level permissions exist in PostgreSQL)                   │
├─────────────────────────────────────────────────────────────────┤
│ FUNCTION RESOLUTION                                             │
│                                                                 │
│ If your query has functions like lower(), count(), etc:          │
│ - Find the function in pg_proc                                  │
│ - Resolve overloads based on argument types                     │
│ - Determine return type                                         │
│                                                                 │
│ SELECT lower(name) → lower(text) → returns text ✓              │
│ SELECT lower(salary) → lower(numeric) → no such function ✗     │
└─────────────────────────────────────────────────────────────────┘

ERRORS AT THIS STAGE:
- "relation 'employeez' does not exist" → typo in table name
- "column 'nam' does not exist" → typo in column name  
- "function lower(integer) does not exist" → wrong argument type
- "permission denied for relation employees" → access control
- "column reference 'name' is ambiguous" → multiple tables have 'name'

OUTPUT — THE QUERY TREE:
The query tree is the same structure as the parse tree, but now every
node is resolved: tables have OIDs, columns have attribute numbers,
types are verified, functions are resolved to specific implementations.
```

### Stage 3: REWRITER (Rule System)

```
INPUT:  Query tree (resolved)
OUTPUT: Rewritten query tree (possibly multiple queries)

WHAT HAPPENS:
┌─────────────────────────────────────────────────────────────────┐
│ RULE APPLICATION                                                │
│                                                                 │
│ PostgreSQL has a RULE system — rewrite rules that transform      │
│ queries before planning. The most important use: VIEWS.         │
│                                                                 │
│ If "employees" is actually a VIEW:                              │
│                                                                 │
│ CREATE VIEW employees AS                                        │
│   SELECT e.name, e.salary, d.name as department                 │
│   FROM emp e JOIN dept d ON e.dept_id = d.id;                   │
│                                                                 │
│ Then the rewriter REPLACES your query with:                     │
│                                                                 │
│ SELECT e.name, e.salary                                         │
│ FROM emp e JOIN dept d ON e.dept_id = d.id                      │
│ WHERE d.name = 'Engineering';                                   │
│                                                                 │
│ Your simple query became a JOIN — and you never knew.            │
│ This is why views are "transparent" — the rewriter inlines them.│
├─────────────────────────────────────────────────────────────────┤
│ OTHER RULES                                                     │
│                                                                 │
│ - INSTEAD rules (used by updatable views before triggers)       │
│ - INSERT/UPDATE/DELETE rules on views                           │
│ - Row Level Security policies (added as extra WHERE conditions) │
│                                                                 │
│ RLS example: if policy says "users can only see own rows":      │
│ The rewriter adds: AND user_id = current_user_id()              │
│ to your WHERE clause. You never see this happening.             │
└─────────────────────────────────────────────────────────────────┘

WHY THIS MATTERS:
1. Views have ZERO runtime cost — they are inlined at rewrite time.
   The planner never sees "a view" — it sees the underlying tables.
   (Exception: MATERIALIZED views are NOT rewritten — they ARE tables)

2. RLS is invisible to your application — the rewriter adds conditions
   your query never explicitly had.

3. Debug tip: if a query behaves unexpectedly on a view, check what
   the rewriter produced. Use: EXPLAIN to see the rewritten plan.

PERFORMANCE COST: Negligible — simple tree transformation.
```

### Stage 4: PLANNER / OPTIMIZER

```
INPUT:  Rewritten query tree
OUTPUT: Execution plan (the optimal strategy to retrieve data)

WHAT HAPPENS:
┌─────────────────────────────────────────────────────────────────┐
│ THIS IS THE MOST IMPORTANT STAGE                                │
│                                                                 │
│ The planner must decide HOW to execute your query.              │
│ For any non-trivial query, there are hundreds or thousands      │
│ of possible execution strategies. The planner picks the cheapest.│
│                                                                 │
│ DECISIONS THE PLANNER MAKES:                                    │
│                                                                 │
│ 1. ACCESS METHOD — for each table:                              │
│    • Sequential scan (read every page)?                          │
│    • Index scan (use B-tree to find specific rows)?             │
│    • Index only scan (answer from index without heap)?          │
│    • Bitmap scan (combine multiple indexes)?                    │
│                                                                 │
│ 2. JOIN STRATEGY — for each join:                               │
│    • Nested loop (for each row in A, scan B)?                   │
│    • Hash join (build hash table on A, probe with B)?           │
│    • Merge join (sort both sides, merge)?                       │
│                                                                 │
│ 3. JOIN ORDER — which table to access first:                    │
│    • A JOIN B JOIN C → (A⋈B)⋈C or A⋈(B⋈C) or (A⋈C)⋈B?       │
│    • With 5 tables: 5! = 120 possible orders                   │
│    • With 10 tables: 10! = 3,628,800 possible orders           │
│    • The planner uses dynamic programming or genetic algorithm   │
│                                                                 │
│ 4. AGGREGATION STRATEGY:                                        │
│    • Hash aggregate (hash table of groups)?                     │
│    • Sort + group (sort by group key, scan for boundaries)?     │
│                                                                 │
│ 5. SORT STRATEGY:                                               │
│    • In-memory quicksort?                                        │
│    • External merge sort (when data > work_mem)?                │
│    • Use an existing index to avoid sorting?                    │
│                                                                 │
│ 6. PARALLELISM:                                                 │
│    • Can this be parallelised?                                  │
│    • How many workers?                                          │
│    • Gather node placement?                                     │
├─────────────────────────────────────────────────────────────────┤
│ HOW IT DECIDES — COST-BASED OPTIMIZATION                        │
│                                                                 │
│ The planner assigns a COST to each possible plan.               │
│ Cost is measured in abstract units (not milliseconds).           │
│                                                                 │
│ Cost inputs:                                                    │
│ • seq_page_cost = 1.0 (reading a sequential page from disk)     │
│ • random_page_cost = 4.0 (reading a random page — seek)        │
│ • cpu_tuple_cost = 0.01 (processing one row)                   │
│ • cpu_index_tuple_cost = 0.005 (processing one index entry)    │
│ • cpu_operator_cost = 0.0025 (evaluating one operator)         │
│                                                                 │
│ Statistics inputs (from pg_stats — updated by ANALYZE):         │
│ • n_distinct: how many distinct values in a column              │
│ • most_common_vals: the most frequent values                   │
│ • histogram_bounds: value distribution                          │
│ • correlation: physical order vs logical order                  │
│ • null_frac: fraction of NULL values                           │
│                                                                 │
│ With these inputs, the planner ESTIMATES:                       │
│ • How many rows each operation will produce (selectivity)       │
│ • How many pages it must read (I/O cost)                        │
│ • How much CPU work is needed (CPU cost)                        │
│                                                                 │
│ It generates candidate plans and picks the lowest total cost.   │
├─────────────────────────────────────────────────────────────────┤
│ WHEN THE PLANNER GETS IT WRONG                                  │
│                                                                 │
│ The planner can choose a BAD plan when:                         │
│ • Statistics are stale (table changed but ANALYZE hasn't run)   │
│ • Correlation between columns (planner assumes independence)    │
│ • Parameter sniffing with prepared statements                   │
│ • Very skewed data distributions                               │
│ • Custom functions the planner cannot estimate                  │
│                                                                 │
│ This is why EXPLAIN ANALYZE exists — to show you when           │
│ estimated rows ≠ actual rows (the planner was wrong).           │
└─────────────────────────────────────────────────────────────────┘

OUTPUT: An execution plan tree — a tree of plan nodes, each 
representing one physical operation (SeqScan, IndexScan, 
HashJoin, Sort, Aggregate, etc.) with estimated costs.
```

### Stage 5: EXECUTOR

```
INPUT:  Execution plan tree
OUTPUT: Result rows (the actual data you asked for)

WHAT HAPPENS:
┌─────────────────────────────────────────────────────────────────┐
│ VOLCANO/ITERATOR MODEL (Pull-based execution)                   │
│                                                                 │
│ PostgreSQL uses the Volcano model (also called iterator model): │
│ Each plan node is an ITERATOR with three methods:               │
│                                                                 │
│   init()  → set up the node (allocate memory, open files)       │
│   next()  → produce the NEXT row (pull one tuple up)            │
│   close() → cleanup (release memory, close files)               │
│                                                                 │
│ Execution flows TOP-DOWN (demand-driven):                       │
│                                                                 │
│   [Client requests rows]                                        │
│        ↓ next()                                                 │
│   [Sort node] "I need all rows to sort — let me pull them all"  │
│        ↓ next() next() next() ...                               │
│   [Hash Join node] "I need a row from each side"                │
│        ↓ next()              ↓ next()                           │
│   [Seq Scan: users]    [Index Scan: orders]                     │
│        ↓                      ↓                                 │
│   [Buffer Pool]          [Buffer Pool]                          │
│        ↓                      ↓                                 │
│   [Disk if not cached]   [Disk if not cached]                   │
│                                                                 │
│ KEY INSIGHT: Not all rows are materialised at once.             │
│ A Seq Scan produces rows one-at-a-time.                          │
│ A Hash Join must build the hash table (blocking) but then       │
│ probes one row at a time.                                       │
│ A Sort must consume ALL input before producing ANY output       │
│ (it is a BLOCKING node — this is why sorts are expensive).      │
├─────────────────────────────────────────────────────────────────┤
│ INTERACTION WITH BUFFER POOL                                    │
│                                                                 │
│ The executor NEVER reads from disk directly.                    │
│ It requests pages from the buffer pool:                         │
│                                                                 │
│ executor.next() → "I need page 42 of table employees"           │
│ buffer_pool.get(42) →                                           │
│   • Page in cache? Return pointer immediately (buffer hit)      │
│   • Page not in cache? Read from disk, put in cache, return     │
│                                                                 │
│ This is why shared_buffers matters — more buffer pool = more     │
│ pages cached = fewer disk reads = faster execution.             │
├─────────────────────────────────────────────────────────────────┤
│ INTERACTION WITH MVCC                                           │
│                                                                 │
│ When the executor reads a tuple from a heap page, it must       │
│ check VISIBILITY:                                               │
│                                                                 │
│ • Is this tuple's xmin committed? (was the inserter committed?) │
│ • Is this tuple's xmax set? (has it been deleted/updated?)      │
│ • Is this tuple visible to MY snapshot?                         │
│                                                                 │
│ Dead tuples are skipped silently — you never see them.          │
│ This is MVCC in action during query execution.                  │
├─────────────────────────────────────────────────────────────────┤
│ INTERACTION WITH WAL                                            │
│                                                                 │
│ For SELECT queries: no WAL interaction (reads don't log).       │
│ For INSERT/UPDATE/DELETE: every modification is written to WAL   │
│ BEFORE the page is modified in the buffer pool.                 │
│ This ensures durability — the executor writes WAL first.        │
├─────────────────────────────────────────────────────────────────┤
│ MEMORY MANAGEMENT                                               │
│                                                                 │
│ The executor allocates memory for:                              │
│ • Hash tables (hash joins, hash aggregates) — bounded by work_mem│
│ • Sort buffers — bounded by work_mem                            │
│ • Tuple buffers — per-node temporary storage                    │
│                                                                 │
│ If work_mem is exceeded, the executor SPILLS TO DISK:           │
│ • Sorts become external merge sorts (temp files)                │
│ • Hash joins become multi-batch hash joins                      │
│ • This is dramatically slower — watch for it in EXPLAIN ANALYZE │
│   (look for "Disk:" in Sort nodes or "Batches: N" in Hash nodes)│
└─────────────────────────────────────────────────────────────────┘
```

---

## EXPLAIN Output for This Pattern

Let's trace a real query through all 5 stages and see the plan:

```sql
-- The query we'll trace through the pipeline:
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT e.name, e.salary, d.name as department_name
FROM employees e
JOIN departments d ON e.department_id = d.id
WHERE e.salary > 80000
ORDER BY e.salary DESC
LIMIT 10;
```

```
-- What EXPLAIN ANALYZE shows (the planner's output + executor's actual numbers):

Limit  (cost=0.58..34.25 rows=10 width=52) (actual time=0.045..0.089 rows=10 loops=1)
  Buffers: shared hit=15
  ->  Nested Loop  (cost=0.58..1205.40 rows=358 width=52) (actual time=0.043..0.085 rows=10 loops=1)
        Buffers: shared hit=15
        ->  Index Scan Backward using idx_emp_salary on employees e  (cost=0.29..95.20 rows=358 width=44) (actual time=0.025..0.038 rows=10 loops=1)
              Index Cond: (salary > 80000)
              Buffers: shared hit=5
        ->  Index Scan using departments_pkey on departments d  (cost=0.29..3.10 rows=1 width=20) (actual time=0.003..0.003 rows=1 loops=10)
              Index Cond: (id = e.department_id)
              Buffers: shared hit=10

Planning Time: 0.215 ms
Execution Time: 0.112 ms
```

```
READING THIS — TRACE THE PIPELINE:

STAGE 1 (Parser): Verified syntax is valid SQL.

STAGE 2 (Analyzer): Resolved employees → OID, departments → OID,
  verified columns exist, checked types (salary > 80000 → numeric > int, OK),
  verified user has SELECT permission on both tables.

STAGE 3 (Rewriter): No views, no rules → query tree unchanged.

STAGE 4 (Planner decided):
  • Access employees via Index Scan Backward on idx_emp_salary
    (WHY backward? Because ORDER BY salary DESC — the index is ASC,
    so reading backwards gives DESC order without a separate Sort node!)
  • Filter salary > 80000 using the same index (Index Cond)
  • Access departments via Index Scan on primary key (for each employee row)
  • Join algorithm: Nested Loop
    (WHY? Because LIMIT 10 means we only need 10 rows from the outer side.
    Nested loop is optimal when we process few outer rows.)
  • No Sort node needed (index provides the order!)
  • LIMIT stops pulling after 10 rows

STAGE 5 (Executor actually did):
  • Read 5 buffer pages from employees index (shared hit=5, all in cache)
  • For each of 10 employees found, lookup department (10 index probes)
  • Total: 15 buffer hits, 0 disk reads
  • Actual time: 0.112ms total — extremely fast
  
KEY INSIGHTS FROM THIS PLAN:
  • "shared hit=15" → all pages were in buffer pool (no disk I/O)
  • "loops=10" on departments scan → this ran 10 times (once per employee row)
  • Estimated rows=358, actual rows=10 → LIMIT stopped early
  • No "Sort" node → the index provided the ORDER BY for free
  • Planning Time (0.215ms) > Execution Time (0.112ms) → plan was costlier to make than execute!
```

---

## Example 1 — Basic: Tracing a Simple Query

```sql
-- A simple query — trace it mentally through the 5 stages:
SELECT name, email 
FROM users 
WHERE active = true;

-- STAGE 1 — PARSER:
-- Tokenize: [SELECT][name][,][email][FROM][users][WHERE][active][=][true]
-- Build parse tree: SelectStmt with targetList, fromClause, whereClause
-- Result: Raw AST (names are just strings)

-- STAGE 2 — ANALYZER:
-- Look up "users" in pg_class → found, OID 16385 ✓
-- Look up "name" in pg_attribute for users → column #1, type text ✓
-- Look up "email" in pg_attribute for users → column #2, type text ✓
-- Look up "active" in pg_attribute for users → column #5, type boolean ✓
-- Type check: active (boolean) = true (boolean) → compatible ✓
-- Permission check: current user has SELECT on users(name, email, active) ✓
-- Result: Resolved query tree

-- STAGE 3 — REWRITER:
-- "users" is a base table (not a view) → no rewriting needed
-- No RLS policies → no additional WHERE conditions
-- Result: Query tree unchanged

-- STAGE 4 — PLANNER:
-- Check statistics for users table:
--   total rows: 500,000
--   active = true: ~70% of rows (from most_common_vals)
--   estimated result: 350,000 rows
-- Decision: 70% of table → Sequential Scan is cheaper than index scan
--   (reading 70% of pages sequentially is faster than 350,000 random index lookups)
-- Result: Plan = Seq Scan with Filter: active = true

-- STAGE 5 — EXECUTOR:
-- Iterate through all heap pages of users table
-- For each tuple: check MVCC visibility, then check active = true
-- Return matching (name, email) pairs to client
-- Uses buffer pool for page access

EXPLAIN ANALYZE SELECT name, email FROM users WHERE active = true;

-- Seq Scan on users  (cost=0.00..12500.00 rows=350000 width=36) 
--                    (actual time=0.015..95.2 rows=347823 loops=1)
--   Filter: (active = true)
--   Rows Removed by Filter: 152177
--   Buffers: shared hit=6250
-- Planning Time: 0.08 ms
-- Execution Time: 120.5 ms

-- WHY Seq Scan? Because 70% of the table matches.
-- An index scan would be SLOWER here (random I/O for 350K rows > sequential read of all pages).
-- This is the planner being SMART, not dumb.
```

---

## Example 2 — Intermediate: When the Planner Chooses Different Strategies

```sql
-- Same table, different WHERE clause — watch the planner change strategy:

-- Query A: highly selective (few rows match)
EXPLAIN ANALYZE 
SELECT name, email FROM users WHERE id = 42;

-- Index Scan using users_pkey on users  (cost=0.42..8.44 rows=1 width=36) 
--                                       (actual time=0.015..0.016 rows=1 loops=1)
--   Index Cond: (id = 42)
--   Buffers: shared hit=3
-- Execution Time: 0.035 ms

-- WHY Index Scan? Only 1 row expected. B-tree traversal (3 pages: root → branch → leaf)
-- is far cheaper than reading the entire table.


-- Query B: moderately selective
EXPLAIN ANALYZE 
SELECT name, email FROM users WHERE city = 'London';

-- Bitmap Heap Scan on users  (cost=125.0..5500.0 rows=5000 width=36)
--                            (actual time=1.2..15.8 rows=4823 loops=1)
--   Recheck Cond: (city = 'London')
--   Heap Blocks: exact=3200
--   ->  Bitmap Index Scan on idx_users_city  (cost=0.0..123.0 rows=5000 width=0)
--         Index Cond: (city = 'London')
--         Buffers: shared hit=15
--   Buffers: shared hit=3215
-- Execution Time: 22.3 ms

-- WHY Bitmap Scan? ~1% of rows (5000 out of 500K). Too many for simple index scan
-- (5000 random page reads would be expensive), too few for seq scan.
-- Bitmap scan: build a bitmap of which pages contain matching rows,
-- then read those pages in sequential order (converts random I/O to sequential I/O).


-- INSIGHT: The planner chose 3 DIFFERENT strategies for 3 different selectivities:
-- • 1 row     → Index Scan (direct B-tree lookup)
-- • 5000 rows → Bitmap Scan (index to find pages, then sequential read)
-- • 350K rows → Seq Scan (just read everything)
-- The crossover points depend on table size, correlation, and cost parameters.
```

---

## Example 3 — Production Grade: A Complex Query Through the Pipeline

```sql
-- Production scenario: Monthly revenue report with top customers per region.
-- Table sizes: orders (50M rows), users (2M rows), regions (50 rows)

EXPLAIN (ANALYZE, BUFFERS)
SELECT 
  r.name as region,
  u.name as customer,
  SUM(o.total_amount) as revenue,
  RANK() OVER (PARTITION BY r.id ORDER BY SUM(o.total_amount) DESC) as rank
FROM orders o
JOIN users u ON o.user_id = u.id
JOIN regions r ON u.region_id = r.id
WHERE o.created_at >= '2025-01-01' AND o.created_at < '2025-02-01'
GROUP BY r.id, r.name, u.id, u.name
HAVING SUM(o.total_amount) > 1000
ORDER BY r.name, rank
LIMIT 100;
```

```
-- PLANNER'S DECISIONS (visible in EXPLAIN):

Limit  (cost=285420.50..285421.00 rows=100 width=72) (actual time=1823.4..1823.9 rows=100 loops=1)
  Buffers: shared hit=125000 read=43000
  ->  Sort  (cost=285420.50..285470.50 rows=20000 width=72) (actual time=1823.3..1823.7 rows=100 loops=1)
        Sort Key: r.name, (rank() OVER (...))
        Sort Method: top-N heapsort  Memory: 45kB
        ->  WindowAgg  (cost=284100.00..285100.00 rows=20000 width=72) (actual time=1780.2..1815.5 rows=18500 loops=1)
              ->  Sort  (cost=284100.00..284150.00 rows=20000 width=64) (actual time=1779.8..1785.2 rows=18500 loops=1)
                    Sort Key: r.id, (sum(o.total_amount)) DESC
                    Sort Method: external merge  Disk: 1200kB
                    ->  HashAggregate  (cost=280000.00..283000.00 rows=20000 width=64) (actual time=1650.5..1770.4 rows=18500 loops=1)
                          Group Key: r.id, r.name, u.id, u.name
                          Filter: (sum(o.total_amount) > 1000)
                          Rows Removed by Filter: 45000
                          Batches: 1  Memory Usage: 8500kB
                          ->  Hash Join  (cost=75000.00..250000.00 rows=800000 width=48) (actual time=320.5..1200.8 rows=780000 loops=1)
                                Hash Cond: (u.region_id = r.id)
                                ->  Hash Join  (cost=74500.00..240000.00 rows=800000 width=36) (actual time=318.2..1050.4 rows=780000 loops=1)
                                      Hash Cond: (o.user_id = u.id)
                                      ->  Index Scan using idx_orders_created_at on orders o  (cost=0.56..150000.00 rows=800000 width=20) (actual time=0.08..450.2 rows=780000 loops=1)
                                            Index Cond: (created_at >= '2025-01-01' AND created_at < '2025-02-01')
                                            Buffers: shared hit=95000 read=43000
                                      ->  Hash  (cost=50000.00..50000.00 rows=2000000 width=24) (actual time=315.5..315.5 rows=2000000 loops=1)
                                            Buckets: 2097152  Memory Usage: 112MB
                                            ->  Seq Scan on users u  (cost=0.00..50000.00 rows=2000000 width=24) (actual time=0.01..150.2 rows=2000000 loops=1)
                                ->  Hash  (cost=1.50..1.50 rows=50 width=20) (actual time=0.02..0.02 rows=50 loops=1)
                                      ->  Seq Scan on regions r  (cost=0.00..1.50 rows=50 width=20) (actual time=0.005..0.01 rows=50 loops=1)

Planning Time: 2.5 ms
Execution Time: 1825.2 ms
```

```
PIPELINE TRACE FOR THIS QUERY:

STAGE 4 (PLANNER) CHOSE:
1. Access orders via Index Scan on created_at (good — date range is selective: 800K out of 50M)
2. Access users via Seq Scan + build hash table (all 2M users needed for join — 112MB hash table!)
3. Access regions via Seq Scan (50 rows — trivial)
4. Join strategy: Hash Join for both joins (large datasets, equality conditions)
5. Aggregation: HashAggregate (group by hash, not sort)
6. Window function: Sort by partition key, then WindowAgg
7. Final sort: top-N heapsort (smart! only keep top 100, not sort all 18500)

STAGE 5 (EXECUTOR) REVEALED:
• "Disk: 1200kB" in the window sort → work_mem exceeded, spilled to disk!
  FIX: SET work_mem = '4MB' for this query, or increase globally.
• "shared read=43000" → 43,000 pages read from DISK (not in buffer pool)
  This is the slow part — 780K order rows across many pages.
• "Memory Usage: 112MB" for users hash table → this is large.
  If users table grows, this becomes a problem.
• "Rows Removed by Filter: 45000" in HAVING → 45K groups computed then discarded.
  Not much we can do — we need to aggregate first to know the sum.

OPTIMIZATION OPPORTUNITIES:
1. The disk sort: increase work_mem
2. The 43K disk reads: more shared_buffers, or partition orders by date
3. The 112MB hash: if we could filter users first (only users WITH orders in January),
   we'd build a much smaller hash table → add a semi-join hint via EXISTS
```

---

## Wrong Query → What Breaks → Right Query

```sql
-- WRONG: Trying to use EXPLAIN to see parse/analyze stages
-- Common misconception: EXPLAIN shows ALL pipeline stages

EXPLAIN SELECT name FROM nonexistent_table;
-- ERROR: relation "nonexistent_table" does not exist

-- WHAT HAPPENED:
-- The query never reached the PLANNER. It failed at STAGE 2 (Analyzer).
-- EXPLAIN can only show the planner's output — if earlier stages fail,
-- EXPLAIN fails too. There is no "EXPLAIN PARSE" or "EXPLAIN ANALYZE SEMANTICS".


-- WRONG: Assuming views add execution cost
-- "I'll avoid views because they add overhead"

CREATE VIEW active_users AS SELECT * FROM users WHERE active = true;

-- Misconception: this query hits "the view" then filters:
SELECT name FROM active_users WHERE city = 'London';

-- REALITY (what the rewriter produces — Stage 3):
SELECT name FROM users WHERE active = true AND city = 'London';

-- The view is GONE by the time the planner sees it.
-- There is NO "view overhead". The planner sees one table scan with two filters.
-- PROVE IT:
EXPLAIN SELECT name FROM active_users WHERE city = 'London';
-- Shows: Seq Scan on users  (Filter: active AND city = 'London')
-- Notice: it says "users" not "active_users" — the view was inlined.


-- WRONG: Thinking prepared statements skip all 5 stages every time
PREPARE my_query(int) AS SELECT name FROM users WHERE id = $1;
EXECUTE my_query(42);

-- MISCONCEPTION: "Prepared statements are parsed once, executed many times — 
-- so stages 1-4 happen once and only stage 5 repeats"

-- REALITY:
-- Stages 1-3: happen at PREPARE time (once) ✓
-- Stage 4: it's COMPLICATED:
--   • First 5 executions: PostgreSQL generates a CUSTOM plan each time
--     (re-plans with actual parameter values for accurate estimates)
--   • After 5 executions: if generic plan cost ≤ average custom plan cost,
--     switches to GENERIC plan (uses $1 as placeholder, estimated selectivity)
--   • This can cause plan REGRESSION — a generic plan may be worse for skewed data
-- Stage 5: happens every time ✓

-- PROVE IT:
EXPLAIN EXECUTE my_query(42);
-- Run this multiple times with different values — watch if the plan changes.
```

---

## Performance Profile

```
THE SQL PIPELINE — PERFORMANCE BREAKDOWN:

┌──────────────┬────────────────┬─────────────────────────────────────────┐
│ Stage        │ Typical Time   │ When It Becomes a Problem               │
├──────────────┼────────────────┼─────────────────────────────────────────┤
│ Parser       │ < 0.1 ms       │ Only with extremely long queries (>1MB) │
│ Analyzer     │ < 0.5 ms       │ Many tables with complex schemas        │
│ Rewriter     │ < 0.1 ms       │ Complex view hierarchies (views on views)│
│ Planner      │ 0.1 - 50 ms   │ Many-table joins (>10 tables), complex  │
│              │                │ queries with many possible plans         │
│ Executor     │ 0.01ms - hours │ THIS is where performance problems live │
└──────────────┴────────────────┴─────────────────────────────────────────┘

KEY INSIGHT: For 99.9% of performance problems, the issue is either:
1. The PLANNER chose a bad plan (wrong estimates → wrong strategy)
2. The EXECUTOR is doing too much work (right plan, big data)

You will almost never have a performance problem in stages 1-3.

PLANNER TIME PROBLEMS (rare but real):
- Queries joining 12+ tables → planner exhausts time exploring join orders
- Fix: geqo_threshold (default 12) — switches to genetic algorithm
- Fix: reduce join count via CTEs with MATERIALIZED

EXECUTOR TIME PROBLEMS (99% of all slow queries):
- Sequential scans on large tables → add appropriate index
- Hash join spilling to disk → increase work_mem
- Sort spilling to disk → increase work_mem
- Nested loop with no inner index → add index or force hash join
- Bad row estimates → ANALYZE the table, check statistics

THE NUMBERS THAT MATTER IN EXPLAIN:
- Planning Time: how long stage 4 took
- Execution Time: how long stage 5 took
- Buffers shared hit: pages read from buffer pool (fast)
- Buffers shared read: pages read from disk (slow)
- Disk: sort/hash spilled to disk (bad)
- Rows (estimated vs actual): if wildly different, statistics are wrong
```

---

## Node.js Integration

```javascript
// === Understanding the pipeline matters in Node.js for these reasons: ===

// 1. PREPARED STATEMENTS AND THE PLANNER
// The 'pg' library uses prepared statements by default for pool.query()
const { Pool } = require('pg');
const pool = new Pool();

// This creates a prepared statement internally:
const result = await pool.query(
  'SELECT name, email FROM users WHERE id = $1',
  [userId]
);
// First 5 calls: custom plan (planner runs with actual values)
// After 5 calls: may switch to generic plan (planner uses estimated selectivity)

// If you have skewed data and the generic plan is bad, you can force custom plans:
// Option A: Use a unique query name each time (prevents plan caching)
// Option B: Disable generic plans for specific queries via pg settings

// 2. UNDERSTANDING ERRORS FROM EACH STAGE
try {
  await pool.query('SELEC name FROM users'); // Parser error
} catch (err) {
  // err.code = '42601' (syntax_error) — failed at Stage 1
  // err.position = 1 — character position of error
}

try {
  await pool.query('SELECT name FROM nonexistent'); // Analyzer error
} catch (err) {
  // err.code = '42P01' (undefined_table) — failed at Stage 2
}

try {
  await pool.query('SELECT name FROM users WHERE id = $1', ['not_a_number']); 
  // This actually succeeds — pg coerces 'not_a_number' ... wait, no:
} catch (err) {
  // err.code = '22P02' (invalid_text_representation) — failed at Stage 5 (executor)
  // The ANALYZER said $1 is type int (inferred from column).
  // The EXECUTOR tried to cast 'not_a_number' to int and failed.
}

// 3. EXPLAIN FROM NODE.JS — DEBUGGING PRODUCTION QUERIES
async function explainQuery(queryText, params) {
  const explainResult = await pool.query(
    `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${queryText}`,
    params
  );
  const plan = explainResult.rows[0]['QUERY PLAN'][0];
  
  console.log('Planning Time:', plan['Planning Time'], 'ms');
  console.log('Execution Time:', plan['Execution Time'], 'ms');
  console.log('Plan:', JSON.stringify(plan.Plan, null, 2));
  
  // Detect problems:
  if (plan['Execution Time'] > 1000) {
    console.warn('SLOW QUERY DETECTED');
  }
  
  return plan;
}

// 4. MONITORING PIPELINE STAGES IN PRODUCTION
// Track planning time vs execution time to identify planner issues:
async function instrumentedQuery(text, params) {
  const start = process.hrtime.bigint();
  
  // Use EXPLAIN ANALYZE in development/staging to get both times:
  if (process.env.NODE_ENV !== 'production') {
    const explained = await pool.query(
      `EXPLAIN (ANALYZE, FORMAT JSON) ${text}`,
      params
    );
    const plan = explained.rows[0]['QUERY PLAN'][0];
    
    if (plan['Planning Time'] > 50) {
      console.warn(`Slow planning (${plan['Planning Time']}ms): ${text.slice(0, 100)}`);
      // Consider: too many joins? Reduce with CTEs or restructure.
    }
  }
  
  return pool.query(text, params);
}
```

### ORM Comparison — How ORMs Interact with the Pipeline

```javascript
// === PRISMA ===
// Prisma generates SQL and sends it through the pipeline like any query.
// You CANNOT see what stage failed directly — Prisma wraps errors.
const users = await prisma.user.findMany({
  where: { active: true },
  select: { name: true, email: true }
});
// Generated SQL: SELECT "name", "email" FROM "users" WHERE "active" = true
// Goes through all 5 stages identically to raw SQL.
// Prisma CANNOT show you EXPLAIN output natively.
// Limitation: No built-in EXPLAIN. You must log the SQL and run EXPLAIN manually.

// === DRIZZLE ===
// Drizzle is closer to SQL — you can reason about the pipeline more directly:
const users = await db.select({ name: usersTable.name, email: usersTable.email })
  .from(usersTable)
  .where(eq(usersTable.active, true));
// Same pipeline. Drizzle also has no native EXPLAIN support.

// === KNEX (Query Builder) ===
// Knex lets you see the generated SQL:
const query = knex('users').select('name', 'email').where('active', true);
console.log(query.toString()); // See what goes to stage 1
// Knex also supports .explain() in some adapters

// === VERDICT ===
// All ORMs send SQL through the same 5-stage pipeline.
// The pipeline doesn't care if SQL was hand-written or ORM-generated.
// What differs: 
//   • Quality of generated SQL (affects planner decisions in stage 4)
//   • Visibility into the plan (can you EXPLAIN easily?)
//   • Prepared statement handling (affects plan caching behaviour)
// For pipeline debugging: always drop to raw SQL + EXPLAIN ANALYZE.
```

---

## Practice Exercises

### Exercise 1 — Easy (Concept Identification)

You run this query and get this error:
```
ERROR: column "status" does not exist
LINE 1: SELECT name, status FROM orders WHERE total > 100;
                     ^
HINT: Perhaps you meant to reference the column "orders.order_status".
```

**Question:** At which stage of the pipeline did this error occur? How do you know? What had already succeeded before this error was thrown?

```
-- Write your answer here
```

### Exercise 2 — Medium (Pipeline Reasoning)

Consider this scenario:
```sql
CREATE VIEW expensive_orders AS
  SELECT o.*, u.name as customer_name
  FROM orders o
  JOIN users u ON o.user_id = u.id
  WHERE o.total > 1000;

-- Now someone runs:
SELECT customer_name, total 
FROM expensive_orders 
WHERE total > 5000;
```

**Question:** Write out exactly what query the PLANNER receives after stages 1-3 complete. Will the planner see `total > 1000 AND total > 5000` or just `total > 5000`? Explain why, referencing the specific pipeline stage responsible.

```
-- Write your answer here
```

### Exercise 3 — Hard (Production Debugging)

You have a Node.js service where this query runs fine in development (1000 rows in users table) but takes 45 seconds in production (5 million rows):

```sql
SELECT u.name, u.email, 
       COUNT(o.id) as order_count,
       SUM(o.total) as total_spent
FROM users u
LEFT JOIN orders o ON u.id = o.user_id
WHERE u.created_at > '2024-01-01'
GROUP BY u.id, u.name, u.email
ORDER BY total_spent DESC NULLS LAST
LIMIT 50;
```

EXPLAIN ANALYZE in production shows:
```
Limit  (cost=892456.12..892456.25 rows=50 width=68) (actual time=45230.5..45230.8 rows=50 loops=1)
  ->  Sort  (cost=892456.12..893456.12 rows=400000 width=68) (actual time=45230.4..45230.7 rows=50 loops=1)
        Sort Key: (sum(o.total)) DESC NULLS LAST
        Sort Method: top-N heapsort  Memory: 32kB
        ->  HashAggregate  (cost=875000.00..880000.00 rows=400000 width=68) (actual time=44800.2..45100.5 rows=385000 loops=1)
              Group Key: u.id
              ->  Hash Right Join  (cost=125000.00..750000.00 rows=12000000 width=44) (actual time=2500.8..35000.2 rows=11500000 loops=1)
                    Hash Cond: (o.user_id = u.id)
                    ->  Seq Scan on orders o  (cost=0.00..450000.00 rows=25000000 width=20) (actual time=0.05..12000.5 rows=25000000 loops=1)
                    ->  Hash  (cost=100000.00..100000.00 rows=400000 width=36) (actual time=2500.2..2500.2 rows=385000 loops=1)
                          ->  Seq Scan on users u  (cost=0.00..100000.00 rows=400000 width=36) (actual time=0.03..1800.5 rows=385000 loops=1)
                                Filter: (created_at > '2024-01-01')
                                Rows Removed by Filter: 4615000
Planning Time: 3.2 ms
Execution Time: 45235.1 ms
```

**Question:** Identify the exact bottleneck. Which pipeline stage is responsible for the problem — is it a bad PLAN (stage 4 made a wrong decision) or too much WORK (stage 5 doing what it was told, but the data is huge)? What specific change would you make, and why would it help? Reference the EXPLAIN output to justify your answer.

```
-- Write your answer here
```

### Exercise 4 — Interview Simulation

**Interviewer asks:** "Walk me through what happens when PostgreSQL receives a SQL query. What are the stages, and at which stage would you focus if a query is performing poorly?"

**Write your answer as you would say it in an interview — clear, structured, demonstrates depth:**

```
-- Write your answer here
```

---

## Interview Questions This Topic Covers

### Q: "What happens when you execute a SQL query in PostgreSQL?"

**Junior answer:** "The database parses the SQL, then executes it and returns results."

**Principal answer:** "A SQL query goes through a 5-stage pipeline. First, the parser does lexical analysis and syntax validation — turning the string into an abstract syntax tree. Second, the analyzer does semantic validation — resolving table and column names against the system catalog, checking types, verifying permissions. Third, the rewriter applies rule transformations — most importantly inlining views and adding RLS policies. Fourth, the planner generates candidate execution plans and picks the lowest-cost one using statistics from pg_stats and the cost model. Finally, the executor runs the chosen plan using the Volcano iterator model, pulling tuples through the plan tree. When I'm debugging a slow query, I focus on stage 4 and 5 — either the planner chose a bad strategy because of stale statistics, or the plan is optimal but the data volume is simply large. EXPLAIN ANALYZE distinguishes these cases by showing estimated vs actual row counts."

**Follow-up:** "How does the planner decide between a sequential scan and an index scan?"

**Principal answer:** "It's purely cost-based. The planner estimates how many rows match the filter (using histogram and most-common-values from pg_stats), multiplies by the I/O cost per page (random_page_cost for index = 4.0 vs seq_page_cost = 1.0), and picks the cheaper option. For a highly selective query returning 0.1% of rows, the index wins despite random I/O. For a query returning 30%+ of rows, sequential scan wins because sequential I/O is 4x cheaper per page in the cost model. The crossover point depends on correlation — if the physical order matches the index order, index scans are cheaper because they're effectively sequential."

---

### Q: "What is a query plan and how do you read one?"

**Junior answer:** "It shows how the database will run the query. You look for Seq Scans and try to avoid them."

**Principal answer:** "A query plan is the output of stage 4 — it's a tree of physical operators the executor will run. You read it bottom-to-top and inside-out. Each node shows its strategy (Seq Scan, Index Scan, Hash Join, etc.), its estimated cost (startup..total), estimated rows, and width. With ANALYZE, you also get actual time, actual rows, and loops. The most important thing I look for is the gap between estimated and actual rows — if the planner estimated 100 rows but got 1 million, it made a bad decision based on that estimate, and everything above that node in the tree may be suboptimal. Second, I look for which node consumed the most actual time. Third, I check BUFFERS to understand whether the bottleneck is I/O (high shared read) or CPU (high time with all shared hits)."

---

### Q: "Why might a prepared statement be slower than a one-off query?"

**Junior answer:** "I'm not sure — prepared statements should be faster since they're parsed once."

**Principal answer:** "Prepared statements save parsing and analysis time, but can hurt at the planning stage. In PostgreSQL, the first 5 executions use custom plans where the planner sees the actual parameter values and can make specific estimates. After 5 executions, PostgreSQL may switch to a generic plan where it uses estimated selectivity for $1 without knowing the actual value. If your data is skewed — say 99% of rows have status='completed' and 1% have status='pending' — a generic plan might choose Seq Scan (good for 'completed', bad for 'pending') when it should use an Index Scan for 'pending'. You can detect this by comparing EXPLAIN EXECUTE with different parameter values and checking if the plan changes. The fix is to either use dynamic SQL (force custom plans) or use plan_cache_mode = force_custom_plan for that session."

---

## Mental Model Checkpoint

Answer these without looking back. If you can't, re-read the relevant section:

1. **A query fails with "permission denied for relation users." At which pipeline stage did this fail? Could the query have had a syntax error that you'll never see because it failed earlier at permissions?**

2. **You create a view `CREATE VIEW v AS SELECT * FROM users WHERE active = true`. You then run `EXPLAIN SELECT * FROM v`. The EXPLAIN output shows "Seq Scan on users" — not "Seq Scan on v". Why? Which pipeline stage is responsible for this transformation?**

3. **A query joins 15 tables. Planning time is 800ms, execution time is 50ms. Where is the bottleneck and what PostgreSQL setting controls this behaviour?**

4. **You run the same query with EXPLAIN and get "Seq Scan" with estimated rows=1000. You run it with EXPLAIN ANALYZE and still get "Seq Scan" but actual rows=500,000. The planner's estimate was wildly wrong. What do you do? (Hint: which input to stage 4 is broken?)**

5. **A junior developer says "I added an index on users(email) but my query `SELECT * FROM users WHERE lower(email) = $1` still does a Seq Scan." Explain which pipeline stage this involves and why the index isn't used.**

6. **In Node.js with the pg library, pool.query() uses prepared statements. After running the same query 6+ times, performance suddenly degrades. Which pipeline stage changed its behaviour, and what specifically changed?**

7. **You have a view built on top of another view, built on top of another view (3 levels deep). A query against the top view is slow. Is the slowness caused by "view overhead" (multiple stages 3), or something else? How would you verify?**

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────┐
│ THE 5 STAGES OF SQL PROCESSING IN POSTGRESQL                           │
├──────────────┬──────────────────────────────────────────────────────────┤
│ 1. PARSER    │ String → Parse Tree (AST)                               │
│              │ Checks: syntax only                                      │
│              │ Errors: "syntax error at or near..."                     │
│              │ Cost: negligible                                         │
├──────────────┼──────────────────────────────────────────────────────────┤
│ 2. ANALYZER  │ Parse Tree → Query Tree (resolved, typed)               │
│              │ Checks: tables exist, columns exist, types match,       │
│              │         permissions, function resolution                  │
│              │ Errors: "relation does not exist", "column not found",   │
│              │         "permission denied", "function does not exist"   │
│              │ Cost: negligible (catalog lookups)                       │
├──────────────┼──────────────────────────────────────────────────────────┤
│ 3. REWRITER  │ Query Tree → Rewritten Query Tree                       │
│              │ Does: inlines views, applies rules, adds RLS conditions │
│              │ Key insight: views are GONE after this stage             │
│              │ Cost: negligible                                         │
├──────────────┼──────────────────────────────────────────────────────────┤
│ 4. PLANNER   │ Query Tree → Execution Plan                             │
│              │ Decides: access methods, join algorithms, join order,    │
│              │          sort strategy, parallelism                      │
│              │ Uses: pg_stats, cost model, available indexes            │
│              │ Errors: none (always produces SOME plan)                 │
│              │ Cost: 0.1ms–50ms (more tables = more time)              │
│              │ View with: EXPLAIN                                       │
├──────────────┼──────────────────────────────────────────────────────────┤
│ 5. EXECUTOR  │ Execution Plan → Result Rows                            │
│              │ Uses: buffer pool, MVCC, WAL (for writes)               │
│              │ Model: Volcano/Iterator (pull-based, one row at a time) │
│              │ Cost: 0.01ms – hours (where performance problems live)  │
│              │ View with: EXPLAIN ANALYZE                               │
└──────────────┴──────────────────────────────────────────────────────────┘

DEBUGGING CHECKLIST:
• Error message → identify which stage failed (see error patterns above)
• Slow query → EXPLAIN ANALYZE → is Planning Time or Execution Time high?
• Bad plan → estimated rows vs actual rows → stale statistics? → ANALYZE
• Good plan, slow → too much data → add index / restructure query / partition

PREPARED STATEMENT LIFECYCLE:
PREPARE   → stages 1, 2, 3 (once)
EXECUTE   → stage 4 (custom plan for first 5, then maybe generic), stage 5 (always)
```

---

## Connected Topics

**Internals this builds on:**
- Buffer pool (how executor reads pages)
- B-tree indexes (access paths the planner considers)
- MVCC (visibility checking during execution)
- WAL (write logging during execution)
- System catalog (what the analyzer queries)

**Next topic:** Topic 02 — The Logical Execution Order (the order WITHIN the executor — FROM, WHERE, GROUP BY, etc.)

**This topic is prerequisite for:**
- Topic 02 (execution order is about what the executor does)
- Topic 03 (query planner in depth — expanding stage 4)
- Topic 04 (reading EXPLAIN — understanding stage 4's output)
- Every performance topic (you must know which stage is the bottleneck)
