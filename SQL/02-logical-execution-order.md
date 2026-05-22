# 02 — The Logical Execution Order
## Phase: Foundations (How SQL Actually Executes)
## Interview Priority: HIGH

---

## ELI5 — The Simple Analogy

Imagine you're a chef given this instruction card:

> "From the pantry and fridge, get all ingredients, throw out anything expired, group similar items together, keep only groups with more than 3 items, list their names, remove duplicates, sort by quantity, and only put out the first 5."

Notice something: the instruction was written in a specific order for *human readability*, but the chef executes it in a completely different order. You can't sort before you've grouped. You can't keep only groups with 3+ items before you've formed the groups. The *writing order* is designed to be readable. The *execution order* is dictated by logic.

SQL is exactly this. You write:

```sql
SELECT → FROM → WHERE → GROUP BY → HAVING → ORDER BY → LIMIT
```

The engine executes:

```sql
FROM → WHERE → GROUP BY → HAVING → SELECT → ORDER BY → LIMIT
```

This single insight — that SQL is **not executed in the order you write it** — explains virtually every "why doesn't this work?" moment a developer ever has with SQL. Every single one.

---

## Internals Connection

From Topic 01, you know the SQL pipeline: Parser → Analyzer → Rewriter → Planner → Executor.

The logical execution order lives inside Stage 5 (the Executor). The Executor doesn't process your query as written — it processes it as a **plan tree**, where each node consumes rows from child nodes. The logical execution order is the order those nodes consume and produce data:

```
                    ┌──────────────────┐
                    │  LIMIT / OFFSET  │  ← 9. Cut the result
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │    ORDER BY      │  ← 8. Sort rows
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │ Window Functions │  ← 7. Compute windows
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │    DISTINCT      │  ← 6. Remove duplicates
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │     SELECT       │  ← 5. Project columns + compute expressions
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │     HAVING       │  ← 4. Filter groups
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │    GROUP BY      │  ← 3. Form groups
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │      WHERE       │  ← 2. Filter rows
                    └────────┬─────────┘
                             │
                    ┌────────▼─────────┐
                    │  FROM + JOINs    │  ← 1. Build the working set
                    └──────────────────┘
```

Each node in this tree is a physical operator in the execution plan. The B-tree indexes (from your internals knowledge) are accessed at step 1. The buffer pool pages are read at step 1. MVCC visibility checks happen at step 1. Everything else is computed from those rows.

---

## The Execution Order Context

This topic **is** the execution order context. It is the lens through which every other SQL topic is understood. Every rule, every restriction, every "gotcha" in SQL can be traced back to this order.

When you complete this topic, you will never again need to memorise rules like:
- "You can't use a SELECT alias in WHERE"
- "You can't filter on a window function in WHERE"
- "You must GROUP BY every non-aggregate column in SELECT"

You will *derive* these rules from first principles — because you understand the order.

---

## What Is This?

The **logical execution order** (also called the **logical processing order**) is the sequence in which the SQL engine conceptually evaluates each clause of a SELECT statement. It is called "logical" because the physical execution (what the planner actually does with indexes, hash tables, etc.) can differ from this order for performance reasons — but the *results must be identical* to what logical order would produce. This order has 9 steps. It is fixed. It does not change based on how you write the query.

---

## Why Does It Matter in Production?

Not knowing this order causes real bugs and real performance problems in production:

**Bug type 1 — Wrong results, no error:**
```sql
-- Developer writes this expecting to filter on the alias:
SELECT user_id, SUM(amount) AS total_spent
FROM payments
WHERE total_spent > 1000   -- BUG: runs before SELECT — alias doesn't exist yet
GROUP BY user_id;
-- ERROR: column "total_spent" does not exist
```

**Bug type 2 — Correct results but catastrophically slow:**
```sql
-- Developer writes this, gets correct results, but performance collapses:
SELECT department, COUNT(*) as headcount
FROM employees
HAVING department = 'Engineering'  -- BUG: groups ALL departments, then filters
GROUP BY department;
-- Correct result, but wasted 99% of the computation grouping useless departments
```

**Bug type 3 — Subtly wrong results, no error:**
```sql
-- Developer thinks WHERE fires after SELECT expressions:
SELECT *, salary * 1.1 AS new_salary
FROM employees
WHERE new_salary > 50000;  -- ERROR in most databases, silent bug if it "worked"
-- In PostgreSQL: ERROR. In MySQL: sometimes works, sometimes wrong.
```

Every one of these bugs vanishes the moment you understand the order.

---

## The Complete Logical Execution Order

### Step 1: FROM + JOINs

```
WHAT HAPPENS:
The engine identifies all the tables (or subqueries, CTEs, views) involved
and builds the initial working set of rows.

For a simple query:
  FROM employees
  → Working set = all rows from employees table (subject to MVCC visibility)

For JOINs:
  FROM employees e
  JOIN departments d ON e.department_id = d.id
  → Start with cross product of employees × departments
  → Filter by ON condition: e.department_id = d.id
  → Working set = only rows where departments match

For multiple JOINs:
  FROM a JOIN b ON ... JOIN c ON ...
  → ((a × b) filtered by ON1) × c filtered by ON2
  → Each JOIN adds to the working set

WHY THIS IS FIRST:
SQL cannot filter, group, or project until it knows WHICH DATA it is working with.
Every subsequent step operates on the rows produced here.

WHAT THIS MEANS:
• Table aliases (e, d) defined in FROM are available in ALL later steps.
  (This is why you can use e.name in WHERE — e was defined in FROM.)
• Subqueries in FROM (derived tables) are evaluated here.
• CTEs referenced in FROM are resolved here.
• Views are already inlined by the Rewriter before we reach this step.

INDEXES ARE USED HERE:
The planner uses indexes (B-trees, etc.) to efficiently retrieve rows for the
FROM clause and apply JOIN conditions. This is where physical execution diverges
from logical order — but results are identical.
```

### Step 2: WHERE

```
WHAT HAPPENS:
Filter rows from the working set. Each row is tested against the WHERE condition.
Rows where the condition is TRUE pass through. FALSE and UNKNOWN (NULL) are removed.

Note: NULL comparisons produce UNKNOWN, not FALSE.
  WHERE salary > 50000 → NULL salary produces UNKNOWN → row is removed
  WHERE salary IS NULL → tests for NULL correctly → row passes through

WHY THIS IS BEFORE GROUP BY:
You want to filter individual rows BEFORE forming groups.
This is the most efficient point to reduce data volume.

WHAT THIS MEANS:
• Cannot reference SELECT aliases here — SELECT runs at step 5, after WHERE.
• Cannot use aggregate functions here — aggregates require groups (step 3).
• Cannot use window functions here — window functions run at step 7.
• CAN use subqueries — they are evaluated per-row or independently.
• CAN use table aliases from FROM (step 1).

PERFORMANCE RULE:
Push as much filtering into WHERE as possible. Every row eliminated in WHERE
is a row that doesn't need to be grouped, aggregated, or sorted later.
WHERE runs EARLY. It is cheap to filter early.

INDEX USE:
If a WHERE condition is sargable (can use an index), the planner may use the
index during the FROM+WHERE phase combined. The logical order says WHERE is
step 2, but physically the planner merges step 1 and 2 using an Index Scan
(fetch + filter in one pass).
```

### Step 3: GROUP BY

```
WHAT HAPPENS:
Rows surviving WHERE are divided into groups.
All rows with the same GROUP BY column values form one group.
After this step, the result set is a SET OF GROUPS, not a set of rows.

Example:
  Rows: [('Eng', 'Alice', 90k), ('Eng', 'Bob', 80k), ('HR', 'Carol', 70k)]
  GROUP BY department →
  Groups: {Eng: [Alice, Bob]}, {HR: [Carol]}

WHY THIS MATTERS:
After GROUP BY, each group must be represented by a SINGLE row in the result.
This means: every column in SELECT must either:
  (a) Be in the GROUP BY list (same value for all rows in group), OR
  (b) Be an aggregate function (collapses group to single value)

This is NOT an arbitrary rule. It is a logical necessity.
You cannot return "name" for a group of 2 employees — which name would you pick?

PostgreSQL EXCEPTION — Functional Dependency:
  GROUP BY user_id
  SELECT user_id, user_name  ← allowed if user_name is functionally dependent on user_id
  (i.e., user_id is a primary key, so user_name is uniquely determined)
  PostgreSQL is smart enough to allow this. MySQL strict mode is not.

WHAT THIS MEANS:
• After this step, you operate on GROUPS, not ROWS.
• HAVING (step 4) filters GROUPS.
• Aggregate functions in SELECT (step 5) collapse each GROUP to one value.
```

### Step 4: HAVING

```
WHAT HAPPENS:
Filter GROUPS (not rows) based on a condition.
This is WHERE for groups.

Example:
  HAVING COUNT(*) > 5
  → Keep only groups that have more than 5 rows
  → Groups with ≤ 5 rows are eliminated

WHY THIS IS AFTER GROUP BY:
HAVING filters on aggregate values. Aggregates cannot be computed until
groups are formed. So HAVING must come after GROUP BY.

CRITICAL DISTINCTION FROM WHERE:
  WHERE  → filters individual ROWS    → runs BEFORE grouping → can use indexes
  HAVING → filters GROUPS             → runs AFTER grouping  → cannot use indexes

PERFORMANCE RULE:
Never put a row-level filter in HAVING when it could go in WHERE.
Bad:  HAVING department = 'Engineering'    ← groups ALL departments first
Good: WHERE department = 'Engineering'     ← filters rows BEFORE grouping

WHAT THIS MEANS:
• CAN reference aggregate functions (COUNT, SUM, AVG, etc.)
• CAN reference GROUP BY columns
• CANNOT reference SELECT aliases — SELECT runs at step 5, after HAVING
  (Exception: PostgreSQL sometimes allows this as an extension — don't rely on it)
• CANNOT reference window functions — they run at step 7
```

### Step 5: SELECT

```
WHAT HAPPENS:
Compute the output columns for each row (or group, if GROUP BY was used).
This includes:
  • Column references: just picks the value
  • Expressions: salary * 1.1, CONCAT(first_name, ' ', last_name)
  • Aggregate functions: COUNT(*), SUM(amount), AVG(salary)
  • Column aliases: AS total_spent, AS full_name
  • CASE expressions

IMPORTANT: This is where column ALIASES are defined.
"AS total_spent" creates the alias total_spent here, at step 5.

WHY THIS IS STEP 5, NOT STEP 1:
SQL was designed to be readable (write SELECT first), but aliases defined in SELECT
cannot be used in WHERE (step 2), GROUP BY (step 3), or HAVING (step 4) because
those steps run before SELECT.

WHAT THIS MEANS:
• Cannot use SELECT alias in WHERE → alias doesn't exist yet at step 2
• Cannot use SELECT alias in HAVING → alias doesn't exist yet at step 4
• CAN use SELECT alias in ORDER BY → ORDER BY runs at step 8, after SELECT
• CAN use SELECT alias in window functions (in some cases, with care)

POSTGRESQL EXCEPTION:
PostgreSQL is lenient: it DOES allow GROUP BY alias_name (even though logically
GROUP BY runs before SELECT). This is a PostgreSQL extension to the SQL standard.
Standard SQL does not allow this. Be aware when writing portable SQL.

STAR (*) EXPANSION:
SELECT * is expanded here to all columns from all tables in FROM.
This is why SELECT * is dangerous: it fetches all columns even if you need one.
Extra columns = extra data from disk + extra network transfer.
```

### Step 6: DISTINCT

```
WHAT HAPPENS:
Remove duplicate rows from the SELECT output.
Two rows are duplicates if ALL their output column values are identical.

Implementation:
  • Sort-based: sort all rows by all output columns, scan for adjacent duplicates
  • Hash-based: hash all output values, track seen combinations

WHY THIS IS AFTER SELECT (not before):
DISTINCT operates on the OUTPUT columns — the result of SELECT expressions.
It cannot run before SELECT computes those expressions.

COST:
DISTINCT requires either a full sort or a hash of all output rows.
This is O(n log n) or O(n) memory.
For large result sets, DISTINCT is expensive.

PERFORMANCE RULE:
Avoid SELECT DISTINCT unless you genuinely have duplicates you want to eliminate.
Often, DISTINCT is a symptom of a JOIN that produces duplicates —
fix the JOIN instead of patching with DISTINCT.

If you need distinct values of ONE column, consider:
  SELECT DISTINCT col FROM table
  vs
  SELECT col FROM table GROUP BY col
  → Same result, similar performance, GROUP BY is more explicit
```

### Step 7: Window Functions

```
WHAT HAPPENS:
Compute window function expressions (OVER(...)) over sets of rows.
Unlike aggregates, window functions do NOT collapse rows into groups.
Each row retains its identity; the window function adds a computed value.

Examples: RANK() OVER (...), LAG() OVER (...), SUM() OVER (...)

WHY THIS IS AFTER WHERE, GROUP BY, HAVING, AND SELECT:
Window functions see the rows that survived all earlier steps.
They compute over the POST-filter, POST-group, POST-projection dataset.

This is the most counterintuitive placement for most developers.

WHAT THIS MEANS:
• Cannot use window function result in WHERE → WHERE runs before windows
• Cannot use window function result in HAVING → HAVING runs before windows
• CAN use window function result in ORDER BY → ORDER BY runs after windows
• CAN filter on window function result only by wrapping in a subquery or CTE

THE FIX PATTERN (appears in virtually every window function interview):
-- This FAILS:
SELECT name, RANK() OVER (ORDER BY salary DESC) AS rnk
FROM employees
WHERE rnk <= 10;    -- rnk doesn't exist at WHERE time!

-- This WORKS:
WITH ranked AS (
  SELECT name, RANK() OVER (ORDER BY salary DESC) AS rnk
  FROM employees
)
SELECT name, rnk FROM ranked WHERE rnk <= 10;
-- The CTE creates a "new" FROM step where rnk now exists as a column.
-- The outer WHERE runs AFTER the inner SELECT (which computed rnk).
```

### Step 8: ORDER BY

```
WHAT HAPPENS:
Sort the result set by the specified columns or expressions.

WHY THIS IS NEAR LAST:
Sorting is only meaningful for the FINAL result.
Sorting intermediate results is wasted work (the planner avoids it unless needed).

WHAT THIS MEANS:
• CAN reference SELECT aliases (SELECT ran at step 5, before ORDER BY)
• CAN reference window function results (computed at step 7)
• CAN reference columns not in SELECT (you can ORDER BY a column you don't SELECT)
  Exception: if DISTINCT is used (step 6), ORDER BY is limited to SELECT columns
  (because DISTINCT eliminates information about non-SELECT columns)

NULL ORDERING:
  ORDER BY col ASC  → NULLs appear LAST (PostgreSQL default for ASC)
  ORDER BY col DESC → NULLs appear FIRST (PostgreSQL default for DESC)
  ORDER BY col ASC NULLS FIRST → explicitly place NULLs first
  ORDER BY col DESC NULLS LAST → explicitly place NULLs last

COST:
Sorting is O(n log n). For large datasets, sorts spill to disk (external merge sort).
If the FROM step uses an index whose order matches ORDER BY, the sort is FREE
(index provides the order — no Sort node in EXPLAIN).
```

### Step 9: LIMIT / OFFSET

```
WHAT HAPPENS:
Cut the result set.
  LIMIT 10        → return at most 10 rows
  OFFSET 20       → skip the first 20 rows
  LIMIT 10 OFFSET 20 → skip 20, return next 10

WHY THIS IS LAST:
You can only limit a result that is fully computed, filtered, grouped, and sorted.
Limiting earlier would give wrong results.

WHAT THIS MEANS:
• All previous steps process ALL qualifying rows (ORDER BY sorts all, then LIMIT cuts)
  Exception: the planner can sometimes "push" LIMIT down for optimization
  (e.g., with ORDER BY + Index Scan Backward — stop early once LIMIT is satisfied)

OFFSET PERFORMANCE WARNING:
OFFSET n scans and discards the first n rows. It does not "jump" to row n.
  OFFSET 1000000 LIMIT 10 → scans 1,000,010 rows, discards first million.
  For large offsets, use keyset pagination instead:
  WHERE id > last_seen_id ORDER BY id LIMIT 10

STANDARD SYNTAX NOTE:
SQL standard uses FETCH FIRST n ROWS ONLY (instead of LIMIT).
PostgreSQL supports both. LIMIT is more common in practice.
```

---

## The Complete Order — Reference Table

```
┌──────┬──────────────────────┬───────────────────────────────────────────────────────┐
│ Step │ Clause               │ What is available at this step                        │
├──────┼──────────────────────┼───────────────────────────────────────────────────────┤
│  1   │ FROM + JOINs         │ Nothing yet — this defines what EXISTS                │
│  2   │ WHERE                │ Table aliases from FROM, column names, subqueries     │
│      │                      │ NOT: SELECT aliases, aggregates, window functions     │
│  3   │ GROUP BY             │ Everything from FROM, column names                   │
│      │                      │ NOT: SELECT aliases (standard SQL)                   │
│      │                      │ PostgreSQL EXTENSION: allows SELECT aliases in GROUP BY│
│  4   │ HAVING               │ Aggregate functions, GROUP BY columns                │
│      │                      │ NOT: SELECT aliases (standard), window functions     │
│  5   │ SELECT               │ All previous — computes output + defines aliases      │
│  6   │ DISTINCT             │ SELECT output columns only                           │
│  7   │ Window Functions     │ SELECT output (post-aggregation), window specs       │
│      │                      │ NOT: usable in WHERE/HAVING/GROUP BY directly        │
│  8   │ ORDER BY             │ SELECT aliases ✓, window function results ✓          │
│      │                      │ Columns not in SELECT ✓ (unless DISTINCT used)       │
│  9   │ LIMIT / OFFSET       │ Fully computed, sorted result                        │
└──────┴──────────────────────┴───────────────────────────────────────────────────────┘
```

---

## EXPLAIN Output for This Pattern

```sql
-- This query exercises steps 1-9:
EXPLAIN ANALYZE
SELECT
  d.name AS department,                          -- step 5
  COUNT(*) AS headcount,                         -- step 5 (aggregate)
  AVG(e.salary) AS avg_salary,                   -- step 5 (aggregate)
  RANK() OVER (ORDER BY AVG(e.salary) DESC) AS salary_rank  -- step 7
FROM employees e                                 -- step 1
JOIN departments d ON e.department_id = d.id     -- step 1
WHERE e.active = true                            -- step 2
GROUP BY d.id, d.name                            -- step 3
HAVING COUNT(*) >= 5                             -- step 4
ORDER BY salary_rank                             -- step 8
LIMIT 10;                                        -- step 9
```

```
-- EXPLAIN ANALYZE output (annotated):

Limit  (cost=312.50..312.53 rows=10 width=52) (actual time=8.420..8.423 rows=10 loops=1)
│
│  ← Step 9: LIMIT cuts after 10 rows
│
└── WindowAgg  (cost=285.00..307.50 rows=22 width=52) (actual time=8.380..8.412 rows=22 loops=1)
    │
    │  ← Step 7: Window function (RANK) computed here.
    │    Receives rows after GROUP BY + HAVING + SELECT aggregation.
    │    Note: Sort is required for RANK — see Sort node below.
    │
    └── Sort  (cost=285.00..285.55 rows=22 width=44) (actual time=8.365..8.368 rows=22 loops=1)
        │     Sort Key: (avg(e.salary)) DESC
        │
        │  ← Implicit sort needed by RANK() OVER (ORDER BY AVG(salary) DESC)
        │    The planner inserts this Sort node to feed WindowAgg.
        │
        └── HashAggregate  (cost=280.00..282.20 rows=22 width=44) (actual time=8.310..8.335 rows=22 loops=1)
            │  Group Key: d.id, d.name
            │  Filter: (count(*) >= 5)          ← Step 4: HAVING
            │  Rows Removed by Filter: 8
            │
            │  ← Step 3+4: GROUP BY forms groups, HAVING filters them.
            │    HashAggregate builds a hash table of groups.
            │    "Rows Removed by Filter: 8" = 8 departments had < 5 employees.
            │
            └── Hash Join  (cost=15.00..260.00 rows=1800 width=28) (actual time=0.245..7.820 rows=1800 loops=1)
                │  Hash Cond: (e.department_id = d.id)
                │
                │  ← Step 1 (JOIN): Hash join between employees and departments.
                │
                ├── Seq Scan on employees e  (cost=0.00..235.00 rows=1800 width=20)
                │                            (actual time=0.008..4.120 rows=1800 loops=1)
                │   Filter: (active = true)   ← Step 2: WHERE
                │   Rows Removed by Filter: 200
                │
                │  ← Step 1+2: Seq scan fetches rows, WHERE filter applied in-flight.
                │    (Planner merged step 1 and 2 into one physical operation.)
                │
                └── Hash  (cost=10.00..10.00 rows=400 width=16) (actual time=0.220..0.220 rows=400 loops=1)
                    └── Seq Scan on departments d  (cost=0.00..10.00 rows=400 width=16)
                        (actual time=0.005..0.102 rows=400 loops=1)

Planning Time: 0.285 ms
Execution Time: 8.502 ms

READ THIS TOP-TO-BOTTOM AS THE LOGICAL ORDER:
But execute BOTTOM-TO-TOP (pull model — each node pulls from its children).

LOGICAL ORDER VISIBLE IN PLAN:
Step 1: FROM+JOIN → Hash Join + Seq Scans at the bottom
Step 2: WHERE    → "Filter: (active = true)" inside Seq Scan
Step 3: GROUP BY → HashAggregate (Group Key: d.id, d.name)
Step 4: HAVING   → "Filter: (count(*) >= 5)" inside HashAggregate
Step 5: SELECT   → aggregates computed inside HashAggregate
Step 7: Window   → WindowAgg node (with Sort feeding it)
Step 8: ORDER BY → Sort node (for RANK window function ordering)
Step 9: LIMIT    → Limit node at the top

Notice: No DISTINCT step (not used). No explicit final ORDER BY on salary_rank —
the sort for the window function also serves as the output order (planner merged them).
```

---

## Example 1 — Basic: Why You Can't Use a SELECT Alias in WHERE

```sql
-- SETUP (reference tables):
-- employees(id, name, salary, department_id, active)
-- departments(id, name)

-- THE MISTAKE every developer makes at least once:
SELECT
  name,
  salary * 1.1 AS adjusted_salary   -- alias defined HERE (step 5)
FROM employees
WHERE adjusted_salary > 60000;       -- step 2 — alias doesn't exist yet!

-- ERROR: column "adjusted_salary" does not exist
-- PostgreSQL is being honest: at step 2 (WHERE), step 5 (SELECT) hasn't run.
-- The alias "adjusted_salary" does not exist in the database state at WHERE time.

-- THE FIX — Option 1: Repeat the expression in WHERE
SELECT
  name,
  salary * 1.1 AS adjusted_salary
FROM employees
WHERE salary * 1.1 > 60000;         -- repeat the expression — no alias needed
-- This works. The expression is evaluated twice (minor cost), but correct.

-- THE FIX — Option 2: Subquery (new FROM = new step 1)
SELECT name, adjusted_salary
FROM (
  SELECT name, salary * 1.1 AS adjusted_salary  -- inner SELECT (step 5)
  FROM employees
) AS sub
WHERE adjusted_salary > 60000;    -- outer WHERE (step 2 of OUTER query)
-- Now "adjusted_salary" was defined in step 5 of the inner query.
-- The outer FROM (step 1) consumes the inner query's result,
-- so "adjusted_salary" EXISTS as a column by outer step 2.

-- THE FIX — Option 3: CTE (same principle as subquery)
WITH adjusted AS (
  SELECT name, salary * 1.1 AS adjusted_salary
  FROM employees
)
SELECT name, adjusted_salary
FROM adjusted
WHERE adjusted_salary > 60000;

-- WHICH FIX TO USE?
-- Option 1: simplest, slight expression duplication — fine for simple expressions
-- Option 2/3: preferred for complex expressions to avoid repetition
-- Performance: all 3 are equivalent — the planner optimises them identically
```

---

## Example 2 — Intermediate: WHERE vs HAVING — A Real Performance Difference

```sql
-- SCENARIO: Find departments with more than 10 active employees.

-- WRONG (uses HAVING for a row-level filter):
SELECT department_id, COUNT(*) AS headcount
FROM employees
HAVING active = true        -- WRONG: 'active' is a row-level condition
   AND COUNT(*) > 10
GROUP BY department_id;

-- Actually: PostgreSQL errors here because HAVING fires after GROUP BY,
-- and 'active' is not in GROUP BY or an aggregate.
-- Even in DBs where this "works", it groups ALL rows first (including inactive),
-- producing wrong counts.

-- ALSO WRONG (uses HAVING for a row-level condition that could be WHERE):
SELECT department_id, COUNT(*) AS headcount
FROM employees
GROUP BY department_id
HAVING active = true         -- ERROR in PostgreSQL
   AND COUNT(*) > 10;

-- RIGHT:
SELECT department_id, COUNT(*) AS headcount
FROM employees
WHERE active = true          -- step 2: filter rows BEFORE grouping
GROUP BY department_id       -- step 3: group what's left (only active employees)
HAVING COUNT(*) > 10;        -- step 4: keep only groups with 10+

-- WHY IT MATTERS AT SCALE:
-- Table: 10 million employees, 2 million active, 500 departments
--
-- WRONG approach (HAVING for row filter):
--   Step 3: Group 10 million rows into 500 groups → process 10M rows
--   Step 4: Filter groups + filter by active → (if it even works correctly)
--
-- RIGHT approach (WHERE for row filter):
--   Step 2: Filter to 2 million rows first → 80% of data eliminated early
--   Step 3: Group 2 million rows into ~450 groups → process only 2M rows
--   Step 4: Filter groups → keep groups with 10+ members
--
-- Result: 5x less work in GROUP BY. On 10M rows, this is the difference between
-- 8 seconds and 1.5 seconds. On 100M rows, it's minutes vs seconds.

-- PROVE IT WITH EXPLAIN:
EXPLAIN ANALYZE
SELECT department_id, COUNT(*) AS headcount
FROM employees
WHERE active = true
GROUP BY department_id
HAVING COUNT(*) > 10;

-- Seq Scan: "Rows Removed by Filter: 8000000"  ← WHERE eliminated 8M rows early
-- HashAggregate: processes only 2M rows, not 10M
```

---

## Example 3 — Production Grade: Multiple Clauses, Order Dependencies, Window Functions

```sql
-- PRODUCTION SCENARIO:
-- Finance team wants: for each active customer, show their monthly spending,
-- their rank among all customers this month, and flag anyone spending
-- more than 3x their own average. Only show top 20 by rank.
--
-- Tables: payments(id, user_id, amount, paid_at), users(id, name, city, active)
-- Scale: payments = 50M rows, users = 2M rows

WITH monthly_stats AS (
  -- CTE = new logical scope. Has its own 9-step execution order.
  SELECT
    u.id          AS user_id,          -- step 5
    u.name        AS customer_name,    -- step 5
    u.city,                            -- step 5
    SUM(p.amount) AS monthly_total,    -- step 5 (aggregate)
    AVG(p.amount) AS avg_payment,      -- step 5 (aggregate)
    MAX(p.amount) AS max_payment       -- step 5 (aggregate)
  FROM users u                                               -- step 1
  JOIN payments p ON u.id = p.user_id                        -- step 1
  WHERE u.active = true                                      -- step 2
    AND p.paid_at >= DATE_TRUNC('month', CURRENT_DATE)       -- step 2
    AND p.paid_at < DATE_TRUNC('month', CURRENT_DATE)
                   + INTERVAL '1 month'                      -- step 2
  GROUP BY u.id, u.name, u.city                             -- step 3
  HAVING SUM(p.amount) > 0                                   -- step 4
)
SELECT
  user_id,                                                  -- step 5
  customer_name,                                            -- step 5
  city,                                                     -- step 5
  monthly_total,                                            -- step 5
  RANK() OVER (ORDER BY monthly_total DESC)    AS rank,     -- step 7
  RANK() OVER (PARTITION BY city
               ORDER BY monthly_total DESC)    AS city_rank, -- step 7
  CASE
    WHEN max_payment > avg_payment * 3 THEN 'HIGH_SPIKE'
    ELSE 'NORMAL'
  END AS spike_flag                                         -- step 5
FROM monthly_stats                                          -- step 1 (CTE as table)
ORDER BY rank                                               -- step 8
LIMIT 20;                                                   -- step 9

-- EXECUTION ORDER TRACE:
-- INNER QUERY (monthly_stats CTE):
--   1. FROM: join users + payments (Index Scan on paid_at likely)
--   2. WHERE: filter active users + current month payments
--      (eliminates ~90% of payments — only current month)
--   3. GROUP BY: group by user_id, name, city
--   4. HAVING: keep users with any payments (sum > 0)
--   5. SELECT: compute SUM, AVG, MAX per group
--
-- OUTER QUERY:
--   1. FROM: read monthly_stats CTE
--   5. SELECT: compute spike_flag CASE expression
--   7. Window: compute RANK globally + RANK per city
--      (requires sort by monthly_total DESC for global rank,
--       sort by city + monthly_total DESC for city_rank)
--   8. ORDER BY: order by global rank (already sorted — the planner may reuse)
--   9. LIMIT: return first 20

-- KEY ORDER DEPENDENCY:
-- The CASE expression uses avg_payment (computed in step 5 of the CTE).
-- The window functions use monthly_total (also from step 5 of CTE).
-- Both are available because we're in the OUTER query's step 5/7,
-- reading from the CTE's already-computed step 5.
-- The CTE is the crucial "reset" — it creates a new dataset where
-- aggregate results become plain column values.
```

---

## Wrong Query → What Breaks → Right Query

### Mistake 1: Window Function in WHERE

```sql
-- WRONG:
SELECT name, salary,
       RANK() OVER (ORDER BY salary DESC) AS rnk
FROM employees
WHERE rnk <= 5;  -- rank doesn't exist at WHERE time (step 2 vs step 7)

-- ERROR: column "rnk" does not exist

-- WHY: WHERE is step 2. Window functions are step 7.
-- "rnk" simply does not exist when WHERE is evaluated.
-- The engine has not yet computed RANK() for any row.

-- RIGHT — CTE creates a new scope:
WITH ranked AS (
  SELECT name, salary,
         RANK() OVER (ORDER BY salary DESC) AS rnk
  FROM employees
  -- At this point: steps 1-7 run for the CTE.
  -- rnk is now a computed column in the CTE's result.
)
SELECT name, salary, rnk
FROM ranked          -- step 1 of OUTER: reads CTE result (rnk exists as a column)
WHERE rnk <= 5;      -- step 2 of OUTER: rnk is now a real column

-- RIGHT — Subquery (identical logic, different syntax):
SELECT name, salary, rnk
FROM (
  SELECT name, salary,
         RANK() OVER (ORDER BY salary DESC) AS rnk
  FROM employees
) AS ranked
WHERE rnk <= 5;
```

### Mistake 2: Aggregate in WHERE

```sql
-- WRONG:
SELECT department_id, name, salary
FROM employees
WHERE salary > AVG(salary);  -- aggregate in WHERE is not allowed

-- ERROR: aggregate functions are not allowed in WHERE

-- WHY: WHERE is step 2. Aggregates require groups (step 3).
-- You cannot compute AVG(salary) before grouping — there are no groups yet.
-- More precisely: AVG needs ALL rows to compute, but WHERE filters rows one-by-one.
-- This is a circular dependency: you need all rows to compute AVG,
-- but you're supposed to be filtering rows right now.

-- RIGHT — Use a subquery or CTE:
SELECT department_id, name, salary
FROM employees
WHERE salary > (SELECT AVG(salary) FROM employees);
-- The subquery runs independently first (uncorrelated),
-- produces a single value (the average),
-- THEN the outer WHERE uses that value for each row.

-- For "above their department's average" (correlated):
SELECT e.name, e.salary, e.department_id
FROM employees e
WHERE e.salary > (
  SELECT AVG(e2.salary)
  FROM employees e2
  WHERE e2.department_id = e.department_id  -- correlated — runs per outer row
);
```

### Mistake 3: SELECT Alias in GROUP BY (Standard SQL)

```sql
-- CONTEXT: This varies by database.

-- In STANDARD SQL (and MySQL strict mode) — WRONG:
SELECT department_id, salary * 1.1 AS adjusted
FROM employees
GROUP BY adjusted;   -- "adjusted" alias not yet defined (GROUP BY is step 3, SELECT is step 5)

-- ERROR in standard SQL: column "adjusted" does not exist

-- RIGHT (standard SQL):
SELECT department_id, salary * 1.1 AS adjusted
FROM employees
GROUP BY salary * 1.1;  -- repeat the expression, not the alias

-- POSTGRESQL EXTENSION (PostgreSQL allows GROUP BY alias — non-standard):
SELECT department_id, salary * 1.1 AS adjusted
FROM employees
GROUP BY adjusted;   -- PostgreSQL allows this as an extension
-- This WORKS in PostgreSQL but NOT in standard SQL / MySQL strict mode.
-- Avoid for portable code. Use GROUP BY salary * 1.1 for clarity and portability.
```

### Mistake 4: ORDER BY with DISTINCT

```sql
-- WRONG (in most contexts):
SELECT DISTINCT department_id
FROM employees
ORDER BY salary;   -- salary is not in SELECT output!

-- ERROR: for SELECT DISTINCT, ORDER BY expressions must appear in select list

-- WHY: DISTINCT (step 6) operates on the SELECT output.
-- After DISTINCT, information about non-SELECT columns (salary) is gone.
-- PostgreSQL cannot sort by salary if each output row may correspond to
-- multiple employees with different salaries — which salary would it use?

-- RIGHT — Include salary in SELECT:
SELECT DISTINCT department_id, salary
FROM employees
ORDER BY salary;   -- salary is in SELECT output — valid

-- OR — If you only want distinct department_ids, don't sort by salary:
SELECT DISTINCT department_id
FROM employees
ORDER BY department_id;  -- department_id IS in SELECT output — valid
```

---

## Performance Profile

```
EXECUTION ORDER PERFORMANCE IMPLICATIONS:

Step 1 (FROM + JOINs):
  • ALL data retrieval happens here. This is the most expensive step.
  • Cost: O(pages in tables), reduced by index use
  • Optimization: ensure JOIN columns are indexed; reduce working set size

Step 2 (WHERE):
  • Earlier filtering = less work in every subsequent step.
  • Rule: always put as much filtering in WHERE as possible.
  • Cost: O(rows in working set), reduced dramatically with index use
  • A single WHERE on an indexed column can eliminate 99% of rows before step 3.

Step 3 (GROUP BY):
  • Groups rows using either hashing (HashAggregate) or sorting (Sort+GroupAggregate)
  • Cost: O(n log n) for sort-based, O(n) for hash-based (memory permitting)
  • Every row NOT eliminated by WHERE must be grouped here.
  • Rule: always use WHERE to reduce rows before GROUP BY.

Step 4 (HAVING):
  • Cheap relative to GROUP BY — filters already-formed groups
  • Only put conditions that REQUIRE aggregate values in HAVING.
  • Everything else belongs in WHERE.

Step 5 (SELECT):
  • Usually cheap — projecting and computing expressions per row/group
  • SELECT * is expensive: fetches all columns, some may be large (text, jsonb)
  • Expensive expressions (regex, complex functions) multiply by row count here

Step 6 (DISTINCT):
  • Sort or hash all output rows — O(n log n) or O(n) memory
  • Avoid unless genuinely needed. Usually points to a JOIN problem.
  • DISTINCT is often 10-100x more expensive than most developers expect.

Step 7 (Window Functions):
  • Must sort by PARTITION BY + ORDER BY for each window definition
  • Cost: O(n log n) per sort, O(n) memory per partition
  • Multiple different window specs = multiple sorts
  • Use WINDOW clause to reuse window definitions when ORDER BY is the same

Step 8 (ORDER BY):
  • Sort final result — O(n log n), spills to disk if result > work_mem
  • Free if an index provides the required order (no Sort node in plan)
  • Expensive if result is large — a 100MB sort with default work_mem spills to disk

Step 9 (LIMIT / OFFSET):
  • LIMIT: cheap — stops pulling rows after N
  • OFFSET: expensive — scans and discards O rows before returning N
  • Never use large OFFSETs. Use keyset pagination instead.

RULE SUMMARY FOR PERFORMANCE:
1. Filter early → heavy WHERE conditions reduce load on all later steps
2. Never put row-level conditions in HAVING (performance + correctness)
3. Avoid SELECT * in production (extra I/O, extra network, plan cache bloat)
4. Avoid SELECT DISTINCT as a fix for JOIN duplicates (find the root cause)
5. Avoid large OFFSET values (keyset pagination instead)
6. For window functions: reduce the dataset before windowing with CTE/subquery
```

---

## Node.js Integration

```javascript
// === HOW THE LOGICAL EXECUTION ORDER AFFECTS NODE.JS DEVELOPMENT ===

const { Pool } = require('pg');
const pool = new Pool();

// 1. THE ALIAS PROBLEM IN DYNAMIC QUERY BUILDERS
// Many Node.js developers hit this when building queries dynamically:

// WRONG — common bug when constructing queries from user input:
async function getUsersAboveThreshold(threshold, sortBy) {
  // Developer thinks sortBy = 'total_spent' will work in WHERE:
  const query = `
    SELECT user_id, SUM(amount) AS total_spent
    FROM payments
    WHERE total_spent > $1          -- BUG: alias doesn't exist at WHERE time
    GROUP BY user_id
    ORDER BY ${sortBy} DESC         -- This part is fine (ORDER BY is step 8)
    LIMIT 100
  `;
  return pool.query(query, [threshold]);
  // ERROR: column "total_spent" does not exist
}

// RIGHT:
async function getUsersAboveThreshold(threshold, sortBy) {
  // Option A: repeat the expression in WHERE
  const query = `
    SELECT user_id, SUM(amount) AS total_spent
    FROM payments
    GROUP BY user_id
    HAVING SUM(amount) > $1         -- HAVING is correct for post-aggregate filter
    ORDER BY total_spent DESC       -- ORDER BY alias is fine (step 8 > step 5)
    LIMIT 100
  `;
  return pool.query(query, [threshold]);
}

// 2. WINDOW FUNCTION RESULTS — THE CTE PATTERN IN NODE.JS
async function getTopUsersByRegion(limit = 10) {
  const { rows } = await pool.query(`
    WITH user_totals AS (
      SELECT
        u.id,
        u.name,
        u.city,
        SUM(p.amount) AS total_spent
      FROM users u
      JOIN payments p ON u.id = p.user_id
      WHERE u.active = true
      GROUP BY u.id, u.name, u.city
    ),
    ranked AS (
      SELECT
        *,
        RANK() OVER (PARTITION BY city ORDER BY total_spent DESC) AS city_rank
      FROM user_totals
      -- Cannot filter on city_rank here — it's computed in step 7.
      -- Must wrap in another CTE or subquery to filter.
    )
    SELECT id, name, city, total_spent, city_rank
    FROM ranked
    WHERE city_rank <= $1           -- NOW we can filter: ranked is a "new table"
    ORDER BY city, city_rank
  `, [limit]);

  return rows;
}

// 3. EXPLAINING THE ORDER TO JUNIOR DEVELOPERS (code review scenario)
// You see this in a PR:
async function getOrderStats(minRevenue) {
  return pool.query(`
    SELECT
      user_id,
      COUNT(*) AS order_count,
      SUM(total) AS revenue
    FROM orders
    HAVING revenue > $1             -- WRONG: should be HAVING SUM(total) > $1
    GROUP BY user_id                -- Also: GROUP BY should come before HAVING
  `, [minRevenue]);
  // Two bugs:
  // 1. HAVING references alias 'revenue' (step 4 can't see step 5's alias in standard SQL)
  // 2. GROUP BY after HAVING (written wrong; PostgreSQL may accept this, but it's confusing)
}

// Fix:
async function getOrderStats(minRevenue) {
  return pool.query(`
    SELECT
      user_id,
      COUNT(*) AS order_count,
      SUM(total) AS revenue
    FROM orders
    GROUP BY user_id
    HAVING SUM(total) > $1          -- Use the expression, not the alias
  `, [minRevenue]);
}
```

### ORM Comparison — How ORMs Handle Execution Order Pitfalls

```javascript
// === PRISMA ===
// Prisma's type-safe API largely prevents the classic mistakes.
// You cannot put aggregates in where() — the type system stops you.

const result = await prisma.payment.groupBy({
  by: ['userId'],
  _sum: { amount: true },
  having: {
    amount: {
      _sum: { gt: threshold }   // Correct: HAVING goes in `having`, not `where`
    }
  },
  orderBy: {
    _sum: { amount: 'desc' }
  },
  take: 100
});
// Generated SQL uses HAVING correctly — Prisma enforces the separation.

// LIMITATION: Prisma CANNOT do window functions natively.
// For any query involving RANK(), ROW_NUMBER(), LAG(), etc., you MUST use $queryRaw:
const ranked = await prisma.$queryRaw`
  WITH user_totals AS (
    SELECT user_id, SUM(amount) AS total_spent
    FROM payments
    GROUP BY user_id
  )
  SELECT user_id, total_spent,
         RANK() OVER (ORDER BY total_spent DESC) AS rnk
  FROM user_totals
  WHERE rnk <= ${limit}   -- WRONG: same alias-in-WHERE bug is possible in raw SQL!
`;
// Risk: When writing raw SQL in Prisma, you bypass type safety and can still hit
// execution order bugs. The alias-in-WHERE mistake is still possible.
// Correct raw SQL: wrap in another CTE.

// === DRIZZLE ===
// Drizzle is close to SQL — you can reason about the order more directly.
const result = await db
  .select({
    userId: payments.userId,
    totalSpent: sql<number>`sum(${payments.amount})`.as('total_spent')
  })
  .from(payments)
  .groupBy(payments.userId)
  .having(sql`sum(${payments.amount}) > ${threshold}`)  // Must use expression, not alias
  .orderBy(sql`total_spent desc`)                        // ORDER BY alias IS OK here
  .limit(100);
// Drizzle correctly maps having() → HAVING and orderBy() → ORDER BY.
// Note: in having(), you must use sql`sum(...)` not the alias (step 4 before step 5).

// === SEQUELIZE ===
// Sequelize is where developers commonly make order-related mistakes.
const result = await Payment.findAll({
  attributes: [
    'userId',
    [sequelize.fn('SUM', sequelize.col('amount')), 'totalSpent']
  ],
  group: ['userId'],
  having: sequelize.where(
    sequelize.fn('SUM', sequelize.col('amount')), { [Op.gt]: threshold }
    // Correct: using the function expression in HAVING, not the alias
  ),
  order: [[sequelize.literal('totalSpent'), 'DESC']],   // ORDER BY alias OK
  limit: 100
});
// Sequelize generates correct HAVING SQL, but the API is verbose and error-prone.
// Many Sequelize examples online incorrectly use where() for post-aggregate filters.

// === KNEX ===
// Knex maps most directly to SQL:
const result = await knex('payments')
  .select('userId', knex.raw('SUM(amount) AS total_spent'))
  .groupBy('userId')
  .havingRaw('SUM(amount) > ?', [threshold])  // Must use expression (not alias)
  .orderBy('total_spent', 'desc')              // ORDER BY alias is OK
  .limit(100);

// === VERDICT ===
// Execution order bugs are most common in:
// 1. Raw SQL strings (all ORMs when using raw/literal SQL)
// 2. Sequelize (verbose API hides what SQL is generated)
// 3. Anywhere window functions are involved (raw SQL required in all ORMs)
//
// Best protection: always run EXPLAIN in development, log generated SQL,
// and know the execution order so you catch mistakes before they ship.
```

---

## Practice Exercises

### Exercise 1 — Easy (Spot the Bug)

Each query below has an execution order mistake. For each one:
- Identify which step the mistake violates
- Explain WHY it is a mistake (using the execution order)
- Write the corrected query

**Query A:**
```sql
SELECT name, salary * 1.15 AS bonus_salary
FROM employees
WHERE bonus_salary > 70000;
```

**Query B:**
```sql
SELECT department_id, COUNT(*) AS emp_count
FROM employees
GROUP BY department_id
HAVING department_id = 5;
```

**Query C:**
```sql
SELECT DISTINCT city
FROM users
ORDER BY created_at DESC;
```

```sql
-- Write your corrected queries here
```

### Exercise 2 — Medium (Order of Evaluation)

Given this query, write out the complete logical execution order step by step. For each step, describe what the data looks like after that step completes:

```sql
SELECT
  d.name AS dept_name,
  COUNT(e.id) AS headcount,
  MAX(e.salary) AS top_salary,
  ROW_NUMBER() OVER (ORDER BY COUNT(e.id) DESC) AS size_rank
FROM departments d
LEFT JOIN employees e ON d.id = e.department_id
WHERE d.active = true
GROUP BY d.id, d.name
HAVING COUNT(e.id) > 0
ORDER BY size_rank
LIMIT 5;
```

```sql
-- Write your step-by-step execution order trace here
-- Step 1: FROM + JOIN — what does the working set look like?
-- Step 2: WHERE — what is filtered?
-- Step 3: GROUP BY — what groups are formed?
-- Step 4: HAVING — what is filtered?
-- Step 5: SELECT — what columns/expressions are computed?
-- Step 7: Window — what does ROW_NUMBER compute over?
-- Step 8: ORDER BY — what is the sort key?
-- Step 9: LIMIT — what is returned?
```

### Exercise 3 — Hard (Production Debug)

A junior developer wrote this query to get customers who spent more than the average in their city, with their rank:

```sql
SELECT
  u.id,
  u.name,
  u.city,
  SUM(p.amount) AS total_spent,
  AVG(SUM(p.amount)) OVER (PARTITION BY u.city) AS city_avg,
  RANK() OVER (PARTITION BY u.city ORDER BY SUM(p.amount) DESC) AS city_rank
FROM users u
JOIN payments p ON u.id = p.user_id
WHERE total_spent > city_avg
GROUP BY u.id, u.name, u.city
ORDER BY city_rank;
```

There are **3 distinct execution order violations** in this query. Find all 3, explain each one, and write the correct version.

```sql
-- Write your analysis and corrected query here
```

### Exercise 4 — Interview Simulation

**Interviewer asks:** "I have a query that uses `WHERE total_revenue > 1000` and I'm getting an error 'column total_revenue does not exist'. The column IS in my SELECT list. Why is this happening and how do you fix it?"

Write your response as you would in a live interview — explain the root cause using the execution order, and provide two different solutions with the trade-offs of each.

```
-- Write your interview answer here
```

---

## Interview Questions This Topic Covers

### Q: "What is the logical execution order of SQL?"

**Junior answer:** "SELECT, FROM, WHERE, GROUP BY, HAVING, ORDER BY."
*(Most juniors recite writing order, not execution order — and they get it wrong)*

**Principal answer:** "SQL is not executed in the order it's written. The logical execution order is: first FROM and JOINs build the working set, then WHERE filters individual rows, then GROUP BY forms groups, then HAVING filters groups, then SELECT computes output columns and aliases, then DISTINCT removes duplicates, then window functions run over the result set, then ORDER BY sorts, and finally LIMIT/OFFSET cuts the result. This order is fundamental because it explains every restriction in SQL: why you can't use a SELECT alias in WHERE (alias doesn't exist yet), why window function results can't be filtered in WHERE (window runs after WHERE), and why HAVING is for aggregate conditions while WHERE is for row-level conditions."

**Follow-up:** "Why can you use a SELECT alias in ORDER BY but not in WHERE?"

**Principal answer:** "Because ORDER BY is step 8, which runs after SELECT at step 5 where the alias is defined. WHERE is step 2, which runs before SELECT. The alias doesn't exist at WHERE time — the engine has not yet evaluated the SELECT expressions that create the alias. ORDER BY is downstream of SELECT in the logical order, so it can see what SELECT produced."

---

### Q: "What's the difference between WHERE and HAVING?"

**Junior answer:** "WHERE filters rows, HAVING filters after GROUP BY."

**Principal answer:** "WHERE and HAVING are different in three important ways. First, execution position: WHERE runs at step 2 before grouping, HAVING runs at step 4 after grouping. Second, what they filter: WHERE filters individual rows (can reference any column from the table), HAVING filters groups (can reference aggregate functions and GROUP BY columns). Third, index use: WHERE conditions can use indexes to scan fewer rows — it's applied as early as possible. HAVING cannot use indexes — it operates on aggregate results. The practical rule: if a condition doesn't involve an aggregate function, it belongs in WHERE. Putting row-level conditions in HAVING is not just stylistically wrong — it's a performance bug. You're forcing the engine to group all rows first, then discard them, instead of discarding them before the expensive GROUP BY."

**Follow-up:** "Can you give me an example where someone uses HAVING incorrectly and what the performance impact is?"

---

### Q: "Why can't you use a window function in a WHERE clause?"

**Junior answer:** "I think it's just not supported?"

**Principal answer:** "It's not arbitrary — it follows directly from the execution order. WHERE runs at step 2. Window functions run at step 7. The window function result simply does not exist when WHERE is evaluated. There is no 'window function result' in the data model at WHERE time — it hasn't been computed yet. The fix is to wrap the query in a CTE or subquery: in the inner query, you compute the window function (step 7 of the inner query). In the outer query, the window function result is now a plain column value (it was computed in the inner SELECT), so the outer WHERE can filter on it. This pattern — CTE to expose window function results for filtering — appears constantly in real production queries."

---

### Q: "A query works correctly but is very slow. You notice it has a HAVING clause filtering on a non-aggregate column. What do you do?"

**Junior answer:** "I'd add an index maybe?"

**Principal answer:** "This is an execution order bug masquerading as a performance problem. HAVING filters groups — it runs after GROUP BY, which means all rows were already grouped before the filter was applied. If the condition doesn't involve an aggregate, it should be in WHERE, not HAVING. Moving it to WHERE means it runs before GROUP BY, potentially eliminating a large fraction of rows early, which reduces the work GROUP BY has to do. On a 10 million row table where the condition eliminates 90% of rows, this can reduce GROUP BY's input from 10 million rows to 1 million rows — roughly a 10x improvement. I'd move the condition to WHERE, verify the result is identical using EXCEPT between old and new query results, then run EXPLAIN ANALYZE on both to confirm the performance improvement."

---

## Mental Model Checkpoint

Answer these from first principles — no looking back:

1. **A colleague says "I can use column aliases in GROUP BY in PostgreSQL, so execution order doesn't matter." How do you respond? Is the colleague right? What is the complete truth?**

2. **You write `SELECT name, SUM(salary) FROM employees`. No GROUP BY. No WHERE. How many rows does the result have? Why? Which step of execution order determines this?**

3. **`SELECT * FROM orders WHERE order_id IN (SELECT order_id FROM order_items WHERE quantity > 10)` — the subquery in WHERE: does it execute once, or once per row in orders? How does your answer connect to the execution order?**

4. **Explain why `SELECT DISTINCT city FROM users ORDER BY name` fails, but `SELECT DISTINCT city, name FROM users ORDER BY name` works. Reference the specific steps.**

5. **A window function and a HAVING clause are both in the same query. Which runs first? What does that mean for what each one can "see"?**

6. **You have a CTE: `WITH x AS (SELECT ...) SELECT * FROM x WHERE ...`. What is the execution order of the OUTER query's WHERE relative to the INNER query's SELECT? Why does this matter for solving the "can't filter window functions in WHERE" problem?**

7. **`ORDER BY salary DESC NULLS FIRST` — at which step does NULL ordering happen? Why doesn't NULL ordering matter for steps 1-7?**

---

## Quick Reference Card

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ SQL LOGICAL EXECUTION ORDER — THE CHEAT SHEET                              │
├──────┬──────────────────────┬──────────────────────────────────────────────┤
│ Step │ Clause               │ Key Rules                                    │
├──────┼──────────────────────┼──────────────────────────────────────────────┤
│  1   │ FROM + JOINs         │ Defines working set; aliases available here  │
│  2   │ WHERE                │ Row filter; can use indexes; NO aggregates;  │
│      │                      │ NO window functions; NO SELECT aliases       │
│  3   │ GROUP BY             │ Forms groups; NO SELECT aliases (standard)   │
│      │                      │ PostgreSQL allows aliases (extension)        │
│  4   │ HAVING               │ Group filter; CAN use aggregates; NO aliases │
│  5   │ SELECT               │ Computes columns + DEFINES aliases           │
│  6   │ DISTINCT             │ De-duplicates SELECT output                  │
│  7   │ Window Functions     │ Computes OVER(...); CAN see SELECT aliases   │
│  8   │ ORDER BY             │ Sorts result; CAN use SELECT aliases         │
│      │                      │ CAN sort by non-SELECT cols (unless DISTINCT)│
│  9   │ LIMIT / OFFSET       │ Cuts result; OFFSET is expensive at scale   │
└──────┴──────────────────────┴──────────────────────────────────────────────┘

CAN I USE X IN Y?

                 WHERE  GROUP BY  HAVING  SELECT  ORDER BY
SELECT alias      ✗        ✗*       ✗*      —        ✓
Aggregate fn      ✗        ✗        ✓       ✓        ✓
Window fn result  ✗        ✗        ✗       —        ✓
Column from FROM  ✓        ✓        ✓       ✓        ✓

* PostgreSQL allows SELECT aliases in GROUP BY/HAVING as an extension.
  Standard SQL does not. Don't rely on this for portable code.

COMMON MISTAKES AND FIXES:

Mistake                          Fix
──────────────────────────────   ──────────────────────────────────────────
WHERE alias_name > x             WHERE expression > x  (repeat expression)
WHERE window_fn > x              Wrap in CTE, filter outer WHERE
HAVING column = x                Move to WHERE (no aggregate = belongs in WHERE)
SELECT DISTINCT + ORDER BY col   Include col in SELECT, or remove DISTINCT
Large OFFSET                     Use keyset pagination: WHERE id > $last_id

PERFORMANCE RULES:
1. Filter early in WHERE — reduces work in all subsequent steps
2. Never use HAVING for non-aggregate conditions
3. Avoid DISTINCT as a workaround for duplicate JOINs
4. Avoid large OFFSET (scans all skipped rows)
5. ORDER BY is free when index provides the sort order
```

---

## Connected Topics

**Internals this builds on:**
- Topic 01: The SQL Processing Pipeline (this topic lives in Stage 5 of that pipeline)
- Buffer pool (rows are fetched from buffer pool during FROM/JOIN)
- B-tree indexes (used during step 1+2 combined by the planner)
- MVCC (visibility applied during step 1 as rows are fetched)

**SQL topics this is prerequisite for:**
- Topic 04: Reading EXPLAIN ANALYZE (every node maps to a logical step)
- Topic 06: WHERE Clause Mastery (deep dive into step 2)
- Topic 20: GROUP BY Fundamentals (deep dive into step 3)
- Topic 22: HAVING (deep dive into step 4)
- Topic 32: Window Functions (deep dive into step 7)
- Every topic involving query correctness and performance
