# SQL — Principal Engineer Level Mastery Curriculum
## Master Plan & Progress Tracker
### Target: Node.js + PostgreSQL Backend Developer → Staff/Principal Level

---

## Phases Overview

| Phase | Focus | Topics | Core Skill |
|-------|-------|--------|------------|
| 1 | How SQL Actually Executes | 01–04 | Understanding the engine |
| 2 | SELECT and Filtering Foundations | 05–09 | Precise data retrieval |
| 3 | JOINs — Complete Mastery | 10–19 | Combining data correctly |
| 4 | Aggregation and Grouping | 20–25 | Summarising data |
| 5 | Subqueries and CTEs | 26–31 | Query composition |
| 6 | Window Functions — Complete Mastery | 32–41 | Analytical queries |
| 7 | Indexes and Query Optimisation | 42–51 | Making queries fast |
| 8 | Transactions and Concurrency | 52–57 | Correctness under load |
| 9 | Advanced SQL Patterns | 58–67 | Real-world problem solving |
| 10 | PostgreSQL Specific Power Features | 68–73 | Platform mastery |
| 11 | SQL for Backend Developer Interviews | 74–77 | Getting hired |

---

## Complete Topic List with Descriptions and Interview Priority

### PHASE 1 — How SQL Actually Executes (Foundation)

| # | File | Topic | Description | Interview Priority |
|---|------|-------|-------------|-------------------|
| 01 | `01-how-sql-is-processed.md` | The SQL Processing Pipeline | Parse → analyse → rewrite → plan → execute — what happens at each stage and how your query becomes an execution plan | HIGH |
| 02 | `02-logical-execution-order.md` | The Logical Execution Order | FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → Window → ORDER BY → LIMIT — why this order exists and what it means for every query you write | HIGH |
| 03 | `03-the-query-planner.md` | The Query Planner in Depth | How PostgreSQL estimates cost, what statistics it uses (pg_statistic, pg_stats), why estimates go wrong, when to intervene | MEDIUM |
| 04 | `04-reading-explain-analyze.md` | Reading EXPLAIN and EXPLAIN ANALYZE | Every node type (Seq Scan, Index Scan, Index Only Scan, Bitmap Heap Scan, Hash Join, Merge Join, Nested Loop, Sort, Hash Aggregate), identifying bottlenecks, actual vs estimated rows | HIGH |

---

### PHASE 2 — SELECT and Filtering Foundations

| # | File | Topic | Description | Interview Priority |
|---|------|-------|-------------|-------------------|
| 05 | `05-select-and-projection.md` | SELECT and Projection | What SELECT actually does in the pipeline, SELECT *, aliasing, expressions, computed columns, DISTINCT cost, why SELECT * is dangerous in production | MEDIUM |
| 06 | `06-where-clause-mastery.md` | WHERE Clause Mastery | Comparison operators, NULL handling (IS NULL vs = NULL), BETWEEN, IN, LIKE, ILIKE, ANY, ALL, EXISTS — exact semantics and performance of each | HIGH |
| 07 | `07-null-in-depth.md` | NULL in Depth | Three-valued logic (TRUE/FALSE/UNKNOWN), NULL in comparisons, aggregates, JOINs, COALESCE, NULLIF, IS DISTINCT FROM — the complete picture | HIGH |
| 08 | `08-filtering-performance.md` | Filtering Performance & Sargability | Which WHERE conditions can use indexes, which cannot (functions on columns, type mismatches, leading wildcards), sargability explained completely | HIGH |
| 09 | `09-case-expressions.md` | CASE Expressions | Simple vs searched CASE, CASE in SELECT/WHERE/ORDER BY/GROUP BY, CASE in aggregates, CASE vs COALESCE vs NULLIF — when to use each | MEDIUM |

---

### PHASE 3 — JOINs — Complete Mastery

| # | File | Topic | Description | Interview Priority |
|---|------|-------|-------------|-------------------|
| 10 | `10-join-fundamentals.md` | JOIN Fundamentals and Mental Model | What a JOIN actually does at the engine level — the cross product every JOIN starts with and how ON filters it | HIGH |
| 11 | `11-inner-join.md` | INNER JOIN in Depth | Exact semantics, what rows appear and why, execution plan (nested loop vs hash vs merge), when each algorithm is chosen | HIGH |
| 12 | `12-left-right-join.md` | LEFT JOIN and RIGHT JOIN | What "all rows from left table" means, NULLs in results, the asymmetry, the classic bug: LEFT JOIN + WHERE on right table nullifying the LEFT | HIGH |
| 13 | `13-full-outer-join.md` | FULL OUTER JOIN | When you actually need it, the pattern it enables, why it is rarely used, performance cost | LOW |
| 14 | `14-cross-join.md` | CROSS JOIN | The cartesian product, deliberate uses (generating combinations, calendar tables), accidental cross joins | MEDIUM |
| 15 | `15-self-join.md` | Self JOIN | Joining a table to itself, use cases (hierarchies, comparing rows within a table), the alias requirement | MEDIUM |
| 16 | `16-multiple-joins.md` | Multiple JOINs | Joining 3–5+ tables, join order and the planner, when the planner gets the order wrong, how to diagnose | MEDIUM |
| 17 | `17-join-performance.md` | JOIN Performance Deep Dive | Nested loop (when, index requirement), hash join (memory, batching), merge join (sort requirement), reading join algorithms in EXPLAIN | HIGH |
| 18 | `18-n-plus-one-problem.md` | The N+1 Query Problem | What it is, why ORMs cause it, detection with query logging, fixing with JOINs and eager loading, the DataLoader pattern for Node.js | HIGH |
| 19 | `19-join-alternatives.md` | JOIN Alternatives | EXISTS vs JOIN, IN vs JOIN, correlated subquery vs JOIN — when each is faster and why the planner treats them differently | HIGH |

---

### PHASE 4 — Aggregation and Grouping

| # | File | Topic | Description | Interview Priority |
|---|------|-------|-------------|-------------------|
| 20 | `20-group-by-fundamentals.md` | GROUP BY Fundamentals | What grouping does at the engine level, which columns must appear in GROUP BY, why the rule exists, the PostgreSQL functional dependency exception | HIGH |
| 21 | `21-aggregate-functions.md` | Aggregate Functions in Depth | COUNT(*) vs COUNT(col) vs COUNT(DISTINCT col), SUM, AVG, MIN, MAX, STRING_AGG, ARRAY_AGG, BOOL_AND, BOOL_OR, JSON_AGG — NULL handling for each | HIGH |
| 22 | `22-having.md` | HAVING | Filtering groups vs filtering rows, HAVING vs WHERE decision rule, HAVING with complex aggregates, performance implications | HIGH |
| 23 | `23-filter-clause.md` | FILTER Clause | Per-aggregate filtering (COUNT(*) FILTER (WHERE ...)), why this beats CASE inside aggregates, the cleaner alternative to pivot hacks | MEDIUM |
| 24 | `24-grouping-sets-rollup-cube.md` | GROUPING SETS, ROLLUP, CUBE | Computing multiple groupings in one query, the GROUPING() function, reading NULL markers, when reports need this | MEDIUM |
| 25 | `25-aggregate-performance.md` | Aggregate Performance | Hash aggregate vs sort aggregate (when each is chosen), memory use, spilling to disk, optimising GROUP BY on large tables | MEDIUM |

---

### PHASE 5 — Subqueries and CTEs

| # | File | Topic | Description | Interview Priority |
|---|------|-------|-------------|-------------------|
| 26 | `26-subqueries-in-depth.md` | Subqueries in Depth | Scalar, row, and table subqueries — correlated vs uncorrelated, where they can appear (SELECT, FROM, WHERE, HAVING), execution behaviour of each | HIGH |
| 27 | `27-exists-not-exists.md` | EXISTS and NOT EXISTS | The semi-join pattern, why EXISTS stops at first match, NOT EXISTS for anti-join, performance vs IN vs JOIN, the NULL problem with NOT IN | HIGH |
| 28 | `28-ctes.md` | CTEs (Common Table Expressions) | The WITH clause, readability and reuse, old CTE optimisation fence (pre-PG12), new default (PG12+), MATERIALIZED vs NOT MATERIALIZED | HIGH |
| 29 | `29-recursive-ctes.md` | Recursive CTEs | WITH RECURSIVE syntax, anchor + recursive member, termination condition, hierarchies, graphs, sequences, date series, cycle detection | HIGH |
| 30 | `30-subquery-vs-cte-vs-join.md` | Subquery vs CTE vs JOIN | The decision framework, performance comparison, when the planner treats them identically, when they differ, readability trade-offs | MEDIUM |
| 31 | `31-lateral-joins.md` | Lateral Joins | The LATERAL keyword, correlated subquery in FROM, top-N per group without window functions, calling functions per row, performance profile | HIGH |

---

### PHASE 6 — Window Functions — Complete Mastery

| # | File | Topic | Description | Interview Priority |
|---|------|-------|-------------|-------------------|
| 32 | `32-window-functions-fundamentals.md` | Window Functions Fundamentals | The concept of a window, how they differ from GROUP BY (all rows kept), the OVER clause, where in the execution pipeline they run | HIGH |
| 33 | `33-partition-by-order-by.md` | PARTITION BY and ORDER BY in OVER | What each does to the window, combinations, without each, the default window, how partitioning differs from grouping | HIGH |
| 34 | `34-ranking-functions.md` | Ranking Functions | ROW_NUMBER, RANK, DENSE_RANK — exact differences with tied values, when to use each, the interview trap with ties | HIGH |
| 35 | `35-ntile-percent-rank-cume-dist.md` | NTILE, PERCENT_RANK, CUME_DIST | Dividing into buckets, percentile calculations, real reporting use cases, the edge cases | MEDIUM |
| 36 | `36-lag-lead.md` | LAG and LEAD | Accessing previous/next rows, offset and default parameters, day-over-day change, finding gaps, session detection | HIGH |
| 37 | `37-first-last-nth-value.md` | FIRST_VALUE, LAST_VALUE, NTH_VALUE | Accessing specific rows in window, the LAST_VALUE frame trap — the most common window function bug that catches seniors | HIGH |
| 38 | `38-aggregate-window-functions.md` | Aggregate Functions as Window Functions | SUM OVER, AVG OVER, COUNT OVER — running totals, moving averages, cumulative distributions | HIGH |
| 39 | `39-frame-clauses.md` | Frame Clauses in Depth | ROWS vs RANGE vs GROUPS, UNBOUNDED PRECEDING, CURRENT ROW, UNBOUNDED FOLLOWING, numeric offsets, the default frame trap | HIGH |
| 40 | `40-window-function-performance.md` | Window Function Performance | Memory and sort costs, multiple windows (reuse vs recompute), the WINDOW clause for named windows, optimisation strategies | MEDIUM |
| 41 | `41-top-n-per-group.md` | Top-N Per Group | The canonical interview problem — ROW_NUMBER approach, LATERAL approach, DISTINCT ON approach, performance comparison | HIGH |

---

### PHASE 7 — Indexes and Query Optimisation

| # | File | Topic | Description | Interview Priority |
|---|------|-------|-------------|-------------------|
| 42 | `42-index-types.md` | Index Types in PostgreSQL | B-tree, Hash, GIN, GiST, SP-GiST, BRIN — what each is optimised for, when to use each, the query types each accelerates | HIGH |
| 43 | `43-index-scan-types.md` | Index Scan Types | Seq Scan, Index Scan, Index Only Scan, Bitmap Index Scan + Bitmap Heap Scan — when each is chosen, cost, how to encourage the one you want | HIGH |
| 44 | `44-composite-indexes.md` | Composite Index Strategy | Column order matters, leftmost prefix rule, covering indexes, INCLUDE clause, designing composite indexes for real query patterns | HIGH |
| 45 | `45-partial-indexes.md` | Partial Indexes | Indexing a subset of rows with WHERE on CREATE INDEX, when partial indexes dramatically outperform full indexes, real use cases | HIGH |
| 46 | `46-expression-indexes.md` | Expression Indexes | Indexing function results, why WHERE lower(email) = $1 cannot use a plain index, how to fix it, maintenance cost | MEDIUM |
| 47 | `47-index-only-scans.md` | Index-Only Scans | Answering queries entirely from the index, visibility map dependency, INCLUDE columns, designing for index-only scans | MEDIUM |
| 48 | `48-when-indexes-hurt.md` | When Indexes Hurt | Write amplification, index bloat, optimizer ignoring indexes (statistics, cost model), too many indexes, when seq scan wins | HIGH |
| 49 | `49-query-optimisation-methodology.md` | Query Optimisation Methodology | The complete workflow: identify → EXPLAIN ANALYZE → bottleneck → hypothesis → test → measure — systematic not guessing | HIGH |
| 50 | `50-statistics-and-planner.md` | Statistics and the Planner | pg_stats, ANALYZE, autovacuum, histogram, correlation, most common values — why wrong estimates happen and how to fix them | MEDIUM |
| 51 | `51-optimizer-interventions.md` | Optimizer Hints and Interventions | PostgreSQL has no hints — the alternatives: enable_* flags, CTE materialisation, statistics targets, query rewrites that guide the planner | MEDIUM |

---

### PHASE 8 — Transactions and Concurrency in SQL

| # | File | Topic | Description | Interview Priority |
|---|------|-------|-------------|-------------------|
| 52 | `52-transaction-syntax.md` | Transaction Syntax in Depth | BEGIN, COMMIT, ROLLBACK, SAVEPOINT, ROLLBACK TO, RELEASE — full TCL, nested transactions (savepoints), autocommit default | HIGH |
| 53 | `53-isolation-levels.md` | Isolation Levels in SQL | The four levels, which anomalies each prevents, how to choose, PostgreSQL defaults, practical impact on concurrent queries | HIGH |
| 54 | `54-select-for-update.md` | SELECT FOR UPDATE / FOR SHARE | Pessimistic locking, SKIP LOCKED for queue processing, NOWAIT for non-blocking, lock granularity, use in Node.js workers | HIGH |
| 55 | `55-deadlocks.md` | Deadlocks in Practice | How to cause deadlocks, how to avoid them, lock ordering convention, detecting in logs, deadlock_timeout setting | HIGH |
| 56 | `56-advisory-locks.md` | Advisory Locks | pg_advisory_lock, session vs transaction scope, distributed mutex, preventing duplicate job processing, the Node.js pattern | MEDIUM |
| 57 | `57-transaction-patterns-nodejs.md` | Transaction Patterns in Node.js | Managing transactions with pg, try/catch/finally, transaction helpers, Unit of Work, handling serialization failures with retry | HIGH |

---

### PHASE 9 — Advanced SQL Patterns

| # | File | Topic | Description | Interview Priority |
|---|------|-------|-------------|-------------------|
| 58 | `58-set-operations.md` | Set Operations | UNION vs UNION ALL (why ALL is almost always correct), INTERSECT, EXCEPT — semantics, performance, deduplication cost | MEDIUM |
| 59 | `59-upsert-patterns.md` | Upsert Patterns | INSERT ON CONFLICT DO NOTHING/DO UPDATE, the EXCLUDED table, concurrency trap, idempotent writes for Node.js APIs | HIGH |
| 60 | `60-bulk-operations.md` | Bulk Operations | INSERT multiple rows, COPY for bulk import, UPDATE FROM, DELETE USING, batch sizes, transaction sizing for bulk ops | HIGH |
| 61 | `61-pivot-crosstab.md` | Pivot and Crosstab | Rotating rows to columns, CASE + GROUP BY pivot, crosstab() function, dynamic pivots (the limitation) | MEDIUM |
| 62 | `62-generating-data.md` | Generating Data | generate_series(), calendar table pattern, filling time-series gaps, generating test data with SQL | LOW |
| 63 | `63-date-time-mastery.md` | Date and Time Mastery | timestamp vs timestamptz (the production disaster), date arithmetic, date_trunc, extract, interval, timezone handling | HIGH |
| 64 | `64-json-jsonb.md` | JSON and JSONB in PostgreSQL | JSONB vs JSON, operators (->, ->>, #>, @>, ?), GIN indexing, when JSONB is appropriate vs schema smell | HIGH |
| 65 | `65-array-operations.md` | Array Operations | PostgreSQL arrays, operators, ANY/ALL with arrays, unnest(), array_agg(), when arrays solve real problems | MEDIUM |
| 66 | `66-full-text-search.md` | Full-Text Search | tsvector, tsquery, @@, GIN indexes for FTS, ts_rank, when PostgreSQL FTS vs Elasticsearch | MEDIUM |
| 67 | `67-materialized-views.md` | Materialized Views | CREATE/REFRESH MATERIALIZED VIEW, CONCURRENTLY, indexing, refresh strategy problem, when they are the right answer | MEDIUM |

---

### PHASE 10 — PostgreSQL Specific Power Features

| # | File | Topic | Description | Interview Priority |
|---|------|-------|-------------|-------------------|
| 68 | `68-explain-options.md` | EXPLAIN Options Deep Dive | ANALYZE, BUFFERS, FORMAT JSON, TIMING, SUMMARY — what each adds, buffers output and I/O vs CPU bound diagnosis | MEDIUM |
| 69 | `69-pg-stat-statements.md` | pg_stat_statements | Enabling the extension, useful columns (mean_exec_time, calls, total_exec_time), finding slowest queries, query normalisation | HIGH |
| 70 | `70-table-partitioning.md` | Table Partitioning | Declarative partitioning (RANGE, LIST, HASH), partition pruning, partition-wise joins/aggregates, attach/detach for time-series | HIGH |
| 71 | `71-schema-design-patterns.md` | Schema Design Patterns | Schemas as namespaces, public schema default, multi-tenant patterns (row-level vs schema-level vs database-level) | MEDIUM |
| 72 | `72-row-level-security.md` | Row Level Security (RLS) | Enabling RLS, CREATE POLICY, USING/CHECK clauses, multi-tenant security pattern, performance implications | MEDIUM |
| 73 | `73-extensions.md` | Essential Extensions | pg_stat_statements, pgcrypto, uuid-ossp, pg_trgm (fuzzy search), postgis concept, timescaledb concept | LOW |

---

### PHASE 11 — SQL for Backend Developer Interviews

| # | File | Topic | Description | Interview Priority |
|---|------|-------|-------------|-------------------|
| 74 | `74-top-20-interview-questions.md` | The 20 Most Common SQL Interview Questions | Principal-level answers, follow-up questions interviewers ask, mistakes that eliminate candidates | HIGH |
| 75 | `75-live-coding-patterns.md` | Live Coding Patterns | How to approach unseen SQL problems, thinking-out-loud framework, edge cases (NULLs, ties, empty tables), optimising after correctness | HIGH |
| 76 | `76-system-design-sql.md` | System Design SQL Questions | "Design the schema for X", "how would you query Y efficiently", "diagnose this slow query" — frameworks for each type | HIGH |
| 77 | `77-canonical-hard-problems.md` | Canonical Hard SQL Problems | Top-N per group, running totals, gaps and islands, consecutive streaks, hierarchies, sessionisation, deduplication, median, pivots, self-referencing | HIGH |

---

## Progress Tracker

### Phase 1 — How SQL Actually Executes
- [x] 01 — The SQL Processing Pipeline
- [x] 02 — The Logical Execution Order
- [x] 03 — The Query Planner in Depth
- [x] 04 — Reading EXPLAIN and EXPLAIN ANALYZE

### Phase 2 — SELECT and Filtering Foundations
- [x] 05 — SELECT and Projection
- [x] 06 — WHERE Clause Mastery
- [x] 07 — NULL in Depth
- [x] 08 — Filtering Performance & Sargability
- [x] 09 — CASE Expressions

### Phase 3 — JOINs — Complete Mastery
- [x] 10 — JOIN Fundamentals and Mental Model
- [x] 11 — INNER JOIN in Depth
- [ ] 12 — LEFT JOIN and RIGHT JOIN
- [ ] 13 — FULL OUTER JOIN
- [ ] 14 — CROSS JOIN
- [ ] 15 — Self JOIN
- [ ] 16 — Multiple JOINs
- [ ] 17 — JOIN Performance Deep Dive
- [ ] 18 — The N+1 Query Problem
- [ ] 19 — JOIN Alternatives

### Phase 4 — Aggregation and Grouping
- [ ] 20 — GROUP BY Fundamentals
- [ ] 21 — Aggregate Functions in Depth
- [ ] 22 — HAVING
- [ ] 23 — FILTER Clause
- [ ] 24 — GROUPING SETS, ROLLUP, CUBE
- [ ] 25 — Aggregate Performance

### Phase 5 — Subqueries and CTEs
- [ ] 26 — Subqueries in Depth
- [ ] 27 — EXISTS and NOT EXISTS
- [ ] 28 — CTEs (Common Table Expressions)
- [ ] 29 — Recursive CTEs
- [ ] 30 — Subquery vs CTE vs JOIN
- [ ] 31 — Lateral Joins

### Phase 6 — Window Functions — Complete Mastery
- [ ] 32 — Window Functions Fundamentals
- [ ] 33 — PARTITION BY and ORDER BY in OVER
- [ ] 34 — Ranking Functions
- [ ] 35 — NTILE, PERCENT_RANK, CUME_DIST
- [ ] 36 — LAG and LEAD
- [ ] 37 — FIRST_VALUE, LAST_VALUE, NTH_VALUE
- [ ] 38 — Aggregate Functions as Window Functions
- [ ] 39 — Frame Clauses in Depth
- [ ] 40 — Window Function Performance
- [ ] 41 — Top-N Per Group

### Phase 7 — Indexes and Query Optimisation
- [ ] 42 — Index Types in PostgreSQL
- [ ] 43 — Index Scan Types
- [ ] 44 — Composite Index Strategy
- [ ] 45 — Partial Indexes
- [ ] 46 — Expression Indexes
- [ ] 47 — Index-Only Scans
- [ ] 48 — When Indexes Hurt
- [ ] 49 — Query Optimisation Methodology
- [ ] 50 — Statistics and the Planner
- [ ] 51 — Optimizer Hints and Interventions

### Phase 8 — Transactions and Concurrency
- [ ] 52 — Transaction Syntax in Depth
- [ ] 53 — Isolation Levels in SQL
- [ ] 54 — SELECT FOR UPDATE / FOR SHARE
- [ ] 55 — Deadlocks in Practice
- [ ] 56 — Advisory Locks
- [ ] 57 — Transaction Patterns in Node.js

### Phase 9 — Advanced SQL Patterns
- [ ] 58 — Set Operations
- [ ] 59 — Upsert Patterns
- [ ] 60 — Bulk Operations
- [ ] 61 — Pivot and Crosstab
- [ ] 62 — Generating Data
- [ ] 63 — Date and Time Mastery
- [ ] 64 — JSON and JSONB in PostgreSQL
- [ ] 65 — Array Operations
- [ ] 66 — Full-Text Search
- [ ] 67 — Materialized Views

### Phase 10 — PostgreSQL Specific Power Features
- [ ] 68 — EXPLAIN Options Deep Dive
- [ ] 69 — pg_stat_statements
- [ ] 70 — Table Partitioning
- [ ] 71 — Schema Design Patterns
- [ ] 72 — Row Level Security (RLS)
- [ ] 73 — Essential Extensions

### Phase 11 — SQL for Backend Developer Interviews
- [ ] 74 — The 20 Most Common SQL Interview Questions
- [ ] 75 — Live Coding Patterns
- [ ] 76 — System Design SQL Questions
- [ ] 77 — Canonical Hard SQL Problems

---

## Interview Readiness Map

> "To confidently answer questions about X, complete these topics."

| Interview Question Category | Required Topics | Phase |
|----------------------------|-----------------|-------|
| **"Explain the SQL execution order"** | 01, 02 | 1 |
| **"How do you read an EXPLAIN plan?"** | 03, 04, 43, 68 | 1, 7, 10 |
| **"What's the difference between WHERE and HAVING?"** | 02, 06, 22 | 1, 2, 4 |
| **"Explain NULL behaviour in SQL"** | 07, 27 | 2, 5 |
| **"Why is this query slow?"** | 04, 08, 17, 48, 49, 50 | 1, 2, 3, 7 |
| **"What index would you add?"** | 42, 43, 44, 45, 46, 47, 48 | 7 |
| **"Explain JOIN types and when to use each"** | 10, 11, 12, 13, 14, 15 | 3 |
| **"What's an N+1 problem?"** | 18, 19 | 3 |
| **"Write a query for top-N per group"** | 34, 41, 31 | 5, 6 |
| **"Explain window functions"** | 32, 33, 34, 38, 39 | 6 |
| **"What are CTEs? When to use them?"** | 28, 29, 30 | 5 |
| **"How do you handle transactions?"** | 52, 53, 57 | 8 |
| **"How do you prevent deadlocks?"** | 54, 55 | 8 |
| **"Design a schema for multi-tenancy"** | 71, 72, 76 | 10, 11 |
| **"How would you handle bulk imports?"** | 60, 59 | 9 |
| **"timestamp vs timestamptz?"** | 63 | 9 |
| **"When to use JSONB vs normalised columns?"** | 64 | 9 |
| **"How do you find slow queries in production?"** | 69, 49, 68 | 7, 10 |
| **"Explain table partitioning"** | 70 | 10 |
| **"Solve: gaps and islands / consecutive streaks"** | 36, 38, 39, 77 | 6, 11 |
| **"UNION vs UNION ALL?"** | 58 | 9 |
| **"Correlated subquery vs JOIN?"** | 19, 26, 27 | 3, 5 |
| **"How do you handle concurrent writes?"** | 53, 54, 55, 56, 59 | 8, 9 |
| **"What's a materialized view? When to use?"** | 67 | 9 |
| **"Live code: write a complex query"** | 75, 77 | 11 |
| **"System design: schema + queries for X"** | 76 | 11 |

---

## File Naming Convention

All files live in: `c:\Users\Parsh\Desktop\NOTES\SQL\`

Format: `XX-topic-name.md` where XX is the zero-padded topic number.

---

## Depth Guarantee

Every single topic in this curriculum is covered **exhaustively** — no surface-level summaries, no "we'll skip this for brevity." Every edge case, every gotcha, every subtle behaviour is explained until there is zero ambiguity. If a concept has 5 variations, all 5 are shown. If a function behaves differently with NULLs, empty sets, or ties — that's explicitly demonstrated.

The goal: after completing any topic, you should have **no remaining doubts** about that concept. You should be able to explain it to someone else, write it under interview pressure, debug it in production at 3am, and review a junior's code using it.

---

## ORM Comparison — How ORMs Handle Each SQL Pattern

Every topic includes an **ORM comparison section** showing how the SQL pattern is handled (or mishandled) by the major Node.js ORMs:

| ORM | Approach | When Raw SQL is Needed |
|-----|----------|----------------------|
| **Prisma** | Type-safe query builder, declarative relations | Window functions, CTEs, complex aggregations, lateral joins, advisory locks |
| **Drizzle** | SQL-like syntax, closer to raw SQL | Recursive CTEs, advanced window frames, some PostgreSQL-specific features |
| **Sequelize** | Classic ORM with models and associations | Nearly all advanced patterns — window functions, CTEs, upserts with complex logic |
| **TypeORM** | Repository/Active Record patterns | Window functions, recursive CTEs, partition operations, advanced locking |
| **Knex.js** | Query builder (not ORM) — closest to raw SQL | Recursive CTEs, some window function syntax, PostgreSQL-specific extensions |

For each topic, the ORM section will show:
1. **Can the ORM do this?** — Yes fully / Partially / No (raw SQL required)
2. **ORM code** — How it looks if the ORM supports it
3. **Where the ORM breaks** — The exact limitation or wrong output
4. **The raw SQL escape hatch** — How to drop to raw SQL in each ORM
5. **Verdict** — "Use the ORM" or "Use raw SQL" with reasoning

This teaches you exactly when to trust your ORM and when to write raw SQL — a decision principal engineers make daily.

---

## How to Use This Curriculum

1. **Sequential**: Topics build on each other. Do them in order.
2. **Each topic**: Read → understand → run proof queries → complete 4 exercises.
3. **Exercises**: Paste answers back. Each gets PASS or NEEDS WORK.
4. **Commands**:
   - `START` → Begin Topic 01
   - `NEXT` → Generate next topic from plan
   - `PROGRESS` → Show checklist with completed topics ticked
   - `REDO [topic#]` → Regenerate a topic completely
5. **Interview prep shortcut**: If time-limited, prioritise all HIGH topics first.

---

## Statistics

- **Total topics**: 77
- **HIGH priority topics**: 45
- **MEDIUM priority topics**: 23
- **LOW priority topics**: 9
- **Estimated completion**: Self-paced (each topic = deep study + exercises)

---

*This plan is the contract. Every topic will be generated following the mandatory template with all 14 teaching rules applied.*
