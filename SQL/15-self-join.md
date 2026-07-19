# Topic 15 — Self JOIN
### SQL Mastery Curriculum — Phase 3: JOINs — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine a company org chart printed on a single sheet of paper. Every person has one line:

```
Name      | Reports To
----------+-----------
Alice     | (nobody — she's the CEO)
Bob       | Alice
Carol     | Alice
Dave      | Bob
Eve       | Bob
```

Now someone asks you: *"Print each person next to their boss's name."*

The boss's name isn't in a second document. It's on the **same sheet, a few lines up or down**. To answer, you take two photocopies of the exact same sheet. You hold copy #1 in your left hand ("the employee") and copy #2 in your right hand ("the manager"). For each person on the left copy, you look at their "Reports To" value, then scan the right copy for the person whose name matches — and you write the two names side by side.

That is a **self JOIN**: joining a table to a second copy of *itself*. There is only one physical table (`employees`), but the query pretends there are two — a "left copy" and a "right copy" — and matches rows in one copy against rows in the other.

The critical insight the ELI5 makes obvious: **you must be able to tell the two copies apart.** If both photocopies are labelled "employees", and you say "look at the Name column," you have no way to say *which* copy's Name you mean. That's why a self JOIN **requires table aliases** — you label one copy `e` (employee) and the other `m` (manager). Without those labels the query is ambiguous and Postgres rejects it. The mandatory alias isn't a style preference; it's the only way the engine can distinguish "this row's column" from "the matched row's column."

Everything else about a self JOIN — the algorithms, the NULL handling, the row multiplication — is identical to a normal join between two different tables. The *only* special thing is: the two tables happen to be the same table, so you rename them apart.

---

## 2. Connection to SQL Internals

A self JOIN is not a special join type. There is no `SELF JOIN` keyword in SQL. It is an ordinary `INNER JOIN`, `LEFT JOIN`, or `CROSS JOIN` where both sides reference the same base relation. Understanding this is the whole game: **the planner treats the two aliases as two independent scans of the same heap.**

At the physical layer, when you write:

```sql
FROM employees e
JOIN employees m ON m.id = e.manager_id
```

PostgreSQL's planner builds a range table with **two range-table entries (RTEs)** that both point at the same `pg_class` OID for `employees`. Each RTE is opened as its own scan node. The engine will:

1. Open one scan over the `employees` heap for alias `e`.
2. Open a **second, independent scan** over the very same `employees` heap for alias `m`.
3. Feed both into a join node (Nested Loop, Hash Join, or Merge Join — Topic 11's three algorithms, unchanged).

Key internal consequences:

- **Buffer pool sharing** — Because both aliases scan the same physical pages, the shared buffer pool caches those heap pages *once*. The second scan frequently gets buffer hits for pages the first scan already loaded. This is why self JOINs are often cheaper on I/O than joining two genuinely different tables of the same size.

- **MVCC visibility** — Both aliases see the **same snapshot** (Topic on MVCC). A self JOIN is evaluated against one consistent view of the table; there's no risk of alias `e` seeing a row that alias `m` cannot. Both scans use the same transaction snapshot, so the "employee copy" and "manager copy" are always internally consistent.

- **Index reuse** — If you have an index on `employees(id)` (the primary key) and on `employees(manager_id)`, the planner can use the PK index for the `m` side probe and the `manager_id` index for filtering the `e` side. The same B-tree serves both aliases; there's only one physical index, scanned from two logical positions.

- **Statistics** — The planner's row-count and selectivity estimates for both aliases come from the *same* `pg_statistic` rows, because it's the same table. Correlation between the two sides (e.g., that `manager_id` values are a subset of `id` values) is *not* modelled by default — the planner assumes independence, which sometimes produces estimate errors on hierarchical self JOINs.

So: a self JOIN is "two scans of one heap, joined." Every internal mechanism you learned for INNER JOIN (Topic 11) and will learn for join performance (Topic 17) applies verbatim. The alias is the only new surface.

---

## 3. Logical Execution Order Context

```
FROM employees e
JOIN employees m ON m.id = e.manager_id   ← both aliases resolved in FROM/JOIN phase
WHERE e.salary > m.salary                  ← applied after the join
GROUP BY ...
HAVING ...
SELECT e.name, m.name                      ← alias-qualified projection
ORDER BY ...
LIMIT ...
```

The self JOIN lives entirely in the **FROM / JOIN phase** — the first thing logically evaluated. By the time `WHERE`, `SELECT`, or `ORDER BY` run, the two aliases already exist as distinct, fully-qualified columns. This ordering matters for self JOINs in three specific ways:

1. **Aliases are established before WHERE** — This is why you can write `WHERE e.salary > m.salary`: both `e` and `m` are already resolved relations when `WHERE` executes. You are comparing two columns *within the same conceptual row of the joined result*, one drawn from each copy.

2. **The ON vs WHERE distinction (from Topic 12) still applies** — For a self **INNER** JOIN, a predicate in `ON` and the same predicate in `WHERE` are equivalent. For a self **LEFT** JOIN (used for hierarchies where the top node has no parent), the distinction is critical: a filter on the right alias placed in `WHERE` will silently convert your LEFT JOIN back into an INNER JOIN and drop the root node. We return to this trap in Section 9.

3. **Self-referential filters need the join to have happened first** — A predicate like `e.department_id = m.department_id` (find pairs of colleagues in the same department) can only be evaluated once *both* scans have produced rows and the join has paired them. It cannot be pushed into a single scan, because it references two aliases. The planner handles this as a join qualifier or a post-join filter.

The mental model: **the two aliases come into existence together, in the FROM phase, and everything downstream treats them as if they were two separate tables that happen to share a schema.**

---

## 4. What Is a Self JOIN?

A **self JOIN** is a join in which a table is joined to another instance of itself, using table aliases to distinguish the two instances. It is used whenever a row needs to be related to *another row in the same table* — hierarchies (employee → manager), sequential comparisons (this month vs last month), or pairing rows against each other (finding duplicates, finding pairs).

```sql
SELECT
  e.name          AS employee,     -- column from the "employee" copy
  m.name          AS manager       -- column from the "manager" copy
FROM employees e                    -- ← first alias: the base table as "e"
INNER JOIN employees m             -- ← SAME table, second alias "m"
  ON m.id = e.manager_id;          -- ← ON links a row to another row in the same table
--   │    │
--   │    └── the "employee" row's manager_id
--   └── the "manager" row's own id
```

Annotated breakdown of every element:

```sql
SELECT e.name, m.name
--     │       └── MUST be alias-qualified — "name" alone is ambiguous (exists in both copies)
--     └── alias-qualified reference to the first instance
FROM employees e
--   │         └── ALIAS (mandatory in practice) — names the first logical copy "e"
--   └── the one physical table
INNER JOIN employees m
--         │         └── ALIAS for the second logical copy "m" — different from "e"
--         └── the SAME physical table named again
ON m.id = e.manager_id;
-- │  │  │  └── join key from the "e" copy
-- │  │  └── equality: standard equi-self-join (can be non-equi: >, <, <>, BETWEEN)
-- │  └── join key from the "m" copy
-- └── ON, exactly as any join — links the two copies
```

The mandatory-alias rule stated precisely: **if a query references the same table more than once in its FROM clause, every reference except at most one must be given a distinct alias, and in practice you alias all of them for clarity.** Postgres will actually reject an unaliased duplicate:

```sql
-- ERROR: table name "employees" specified more than once
SELECT name FROM employees JOIN employees ON employees.id = employees.manager_id;
```

You cannot even *write* the column references unambiguously without aliases, so the alias is not optional — it is the mechanism that makes a self JOIN expressible at all.

A self JOIN can use **any** join type:
- `INNER JOIN` — only rows with a match in the other copy (e.g., employees who *have* a manager).
- `LEFT JOIN` — all rows, with NULLs where there's no match in the other copy (e.g., all employees, including the CEO who has no manager).
- `CROSS JOIN` — every row paired with every other row (Topic 14) — used for all-pairs comparisons.

---

## 5. Why Self JOIN Mastery Matters in Production

1. **Hierarchies are everywhere** — Employee/manager reporting lines, category/parent-category trees, comment/parent-comment threads, geographic region nesting, bill-of-materials part explosions. Every one of these is an **adjacency list** (a row storing a pointer to its parent in the same table), and the single-level lookup is *always* a self JOIN. Get this wrong and your org chart, your threaded comments, or your category breadcrumbs render incorrectly.

2. **Row-to-row comparison is a self JOIN in disguise** — "Find users who signed up on the same day," "find products cheaper than another product in the same category," "find each order and the customer's previous order." Before window functions (Topic 21+), these were *only* expressible as self JOINs, and in many cases a self JOIN is still the clearest or fastest form.

3. **Duplicate detection** — The canonical "find rows that share a value but have different IDs" query is a self JOIN (`a.email = b.email AND a.id < b.id`). Data-quality pipelines, dedup jobs, and merge-candidate finders all lean on this pattern.

4. **The alias mistake is a silent correctness bug** — Forgetting to qualify a column, or accidentally using the wrong alias (`e.salary` where you meant `m.salary`), produces a query that *runs* but returns wrong data. There's no error. In a self JOIN, both aliases have identical column names, so a typo in the alias is invisible to the compiler and to the reader who isn't paying attention.

5. **Self JOINs can explode** — Because both sides scan the same (potentially large) table, an unfiltered or poorly-indexed self JOIN on a big table can produce a near-Cartesian result (Topic 14's CROSS JOIN cost model). A self JOIN on a 10M-row table with no index on the join column is a production outage waiting to happen.

6. **The "reflexive pair" trap** — Naive pair-finding self JOINs return each pair twice (A–B and B–A) and also match every row with itself (A–A). Knowing to add `a.id < b.id` is the difference between a correct dedup and one that reports every row as its own duplicate.

---

## 6. Deep Technical Content

### 6.1 The Adjacency List — Single-Level Hierarchy

The foundational self-JOIN use case. An **adjacency list** stores each node with a foreign key to its parent *in the same table*:

```sql
-- employees(id, name, salary, manager_id, department_id)
--   manager_id is a self-referencing FK: employees.manager_id -> employees.id
```

To fetch each employee alongside their direct manager:

```sql
SELECT
  e.id,
  e.name          AS employee,
  m.name          AS manager
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id;
--   ^^^^ LEFT JOIN so the CEO (manager_id IS NULL) is not dropped
```

Why `LEFT` and not `INNER` here? The top of any hierarchy has a NULL parent. With `INNER JOIN`, the ON condition `m.id = e.manager_id` evaluates to UNKNOWN when `e.manager_id` is NULL, so the root row is silently excluded (exactly the NULL behaviour from Topic 11 §6.1). If you want the *whole* org chart including the CEO, you must use `LEFT JOIN`:

```
INNER JOIN result:          LEFT JOIN result:
employee | manager          employee | manager
---------+--------          ---------+--------
Bob      | Alice            Alice    | NULL      ← CEO preserved
Carol    | Alice            Bob      | Alice
Dave     | Bob              Carol    | Alice
Eve      | Bob              Dave     | Bob
(Alice missing!)            Eve      | Bob
```

A single self JOIN retrieves exactly **one level** of the hierarchy. To go two levels (employee → manager → manager's manager), you self-join three times:

```sql
SELECT
  e.name  AS employee,
  m.name  AS manager,
  mm.name AS skip_level_manager
FROM employees e
LEFT JOIN employees m  ON m.id  = e.manager_id
LEFT JOIN employees mm ON mm.id = m.manager_id;
```

Each additional level is one more self JOIN. This is why **arbitrary-depth** hierarchies (org chart of unknown depth) cannot be done with self JOINs alone — you'd need to know the depth in advance. For unbounded depth you use a **recursive CTE** (`WITH RECURSIVE`, a later topic), which is essentially an automated repeated self JOIN until no new rows appear. The self JOIN is the primitive; recursion is the loop around it.

### 6.2 Non-Equi Self JOIN — Comparing Rows Within a Table

Self JOINs frequently use inequality (`<`, `>`, `<>`) rather than equality. "Find every employee who earns more than their manager":

```sql
SELECT
  e.name   AS employee,
  e.salary AS emp_salary,
  m.name   AS manager,
  m.salary AS mgr_salary
FROM employees e
INNER JOIN employees m ON m.id = e.manager_id   -- equi part: link to manager
WHERE e.salary > m.salary;                       -- non-equi comparison across copies
```

Here the *join* is an equi-join (link employee to their specific manager), and the inequality lives in `WHERE`. But the comparison could also be the join condition itself. "For each product, find all cheaper products in the same category":

```sql
SELECT
  p.name       AS product,
  p.price,
  cheaper.name AS cheaper_alternative,
  cheaper.price AS alt_price
FROM products p
INNER JOIN products cheaper
  ON cheaper.category_id = p.category_id   -- equi: same category
  AND cheaper.price < p.price;              -- non-equi: strictly cheaper
```

This produces one row per (product, cheaper-alternative) pair. A product with 4 cheaper alternatives in its category appears 4 times. This is standard join fan-out (Topic 11 §6.2), just against the same table.

### 6.3 Finding Pairs — The `<` Trick

To find **pairs** of related rows — two users with the same email, two orders placed at the same second — the naive query is wrong in two ways:

```sql
-- NAIVE (WRONG): find users sharing an email
SELECT a.id, b.id, a.email
FROM users a
INNER JOIN users b ON b.email = a.email;
```

This has two defects:

1. **Self-matching**: every row matches *itself* (a row's email equals its own email), so you get spurious `(1, 1)`, `(2, 2)` pairs.
2. **Mirror duplicates**: each genuine pair appears twice — `(1, 5)` and `(5, 1)` — because the join is symmetric.

The fix is a single asymmetric predicate on the primary key:

```sql
-- CORRECT: each unordered pair exactly once, no self-matches
SELECT a.id AS id1, b.id AS id2, a.email
FROM users a
INNER JOIN users b
  ON b.email = a.email
  AND a.id < b.id;          -- ← eliminates self-matches AND mirror duplicates
```

Why `a.id < b.id` works:
- It **excludes self-matches** because a row's id is never `<` itself.
- It **breaks the mirror symmetry** because for any pair {x, y} with x < y, only the ordering (a=x, b=y) satisfies `a.id < b.id`; the mirror (a=y, b=x) fails it.

Use `<` (strict) not `<=` — `<=` would re-admit self-matches. Use `<` not `<>` — `<>` (not-equal) eliminates self-matches but keeps *both* mirror orderings, halving your correctness.

### 6.4 Finding Duplicates — Group vs Self JOIN

Finding duplicate rows can be done two ways. The `GROUP BY` form (Topic 20) finds *which values* are duplicated:

```sql
-- Which emails are duplicated, and how many times?
SELECT email, COUNT(*) AS n
FROM users
GROUP BY email
HAVING COUNT(*) > 1;
```

The self JOIN form finds *which specific row pairs* are duplicates — you get the actual IDs, useful for a dedup/merge job:

```sql
-- Which specific user IDs collide on email? (for merging)
SELECT a.id AS keep_id, b.id AS dup_id, a.email
FROM users a
INNER JOIN users b ON b.email = a.email AND a.id < b.id
ORDER BY a.email, a.id;
```

The self JOIN gives you row-level granularity (the exact pairs to reconcile); `GROUP BY` gives you value-level summary. In production dedup pipelines you often use both: `GROUP BY ... HAVING` to *count* the problem, then the self JOIN to *enumerate* the specific rows to merge.

### 6.5 Sequential / Temporal Self JOIN — "Previous Row"

Before window functions, "compare each row to the previous one" was a self JOIN. "For each order, find the customer's immediately prior order":

```sql
-- For each order, the customer's previous order (by time)
SELECT
  o.id           AS order_id,
  o.created_at,
  o.total_amount,
  prev.id        AS prev_order_id,
  prev.total_amount AS prev_amount
FROM orders o
INNER JOIN orders prev
  ON prev.customer_id = o.customer_id     -- same customer
  AND prev.created_at < o.created_at;      -- strictly earlier
-- PROBLEM: this matches ALL earlier orders, not just the immediately prior one
```

This over-matches: it pairs each order with *every* earlier order by that customer. To get only the *immediately* prior order, you must add a correlated condition that no order sits between them:

```sql
SELECT
  o.id AS order_id, o.created_at, o.total_amount,
  prev.id AS prev_order_id, prev.total_amount AS prev_amount
FROM orders o
INNER JOIN orders prev
  ON prev.customer_id = o.customer_id
  AND prev.created_at < o.created_at
WHERE NOT EXISTS (                          -- no order strictly between prev and o
  SELECT 1 FROM orders mid
  WHERE mid.customer_id = o.customer_id
    AND mid.created_at > prev.created_at
    AND mid.created_at < o.created_at
);
```

This is correct but expensive — it's a self JOIN with a correlated `NOT EXISTS` subquery (itself a third reference to the same table). This is *exactly* the pattern that window functions (`LAG(...) OVER (PARTITION BY customer_id ORDER BY created_at)`) replace, running in a single ordered pass instead of an O(n²) self JOIN. **Rule of thumb: if your self JOIN uses `<` on a timestamp plus a "nothing between" condition, a window function is almost always the better tool.** The self JOIN still matters — you must recognise the pattern to know to reach for the window function.

### 6.6 The CROSS JOIN Self JOIN — All Pairs

Occasionally you genuinely want every pair of rows (Topic 14). "Compute the distance between every pair of warehouses":

```sql
SELECT
  a.id   AS warehouse_a,
  b.id   AS warehouse_b,
  earth_distance(
    ll_to_earth(a.lat, a.lng),
    ll_to_earth(b.lat, b.lng)
  ) AS meters
FROM warehouses a
CROSS JOIN warehouses b
WHERE a.id < b.id;    -- unordered pairs only; excludes self-pairs
```

For N warehouses this is N×(N−1)/2 rows after the `a.id < b.id` filter (versus N² for the raw cross product). At N = 1,000 that's ~500K rows — fine. At N = 100,000 that's ~5 billion rows — a query that will not finish. All-pairs self JOINs scale quadratically; guard them with `a.id < b.id` and a hard bound on N.

### 6.7 Self JOIN with Aggregation

You can aggregate over a self JOIN. "For each manager, count direct reports and their total salary":

```sql
SELECT
  m.id            AS manager_id,
  m.name          AS manager,
  COUNT(e.id)     AS direct_reports,
  SUM(e.salary)   AS reports_total_salary
FROM employees m
INNER JOIN employees e ON e.manager_id = m.id   -- e reports to m
GROUP BY m.id, m.name
ORDER BY direct_reports DESC;
```

Note the alias direction: `m` is the manager (the "one" side), `e` is the report (the "many" side). We `GROUP BY m` and aggregate over `e`. This is the same fan-out-aware aggregation from Topic 11 §6.8 — the "one" side is grouped, the "many" side is aggregated. Managers with zero reports are *excluded* by the `INNER JOIN`; use `LEFT JOIN employees e ON e.manager_id = m.id` and `COUNT(e.id)` (which counts non-NULLs, so 0 for childless managers) to include them.

### 6.8 Column Reference Ambiguity — The Core Alias Rule

Because both aliases expose identical column names, **every column reference in a self JOIN must be alias-qualified** or Postgres errors:

```sql
-- ERROR: column reference "name" is ambiguous
SELECT name FROM employees e JOIN employees m ON m.id = e.manager_id;

-- CORRECT: qualify everything
SELECT e.name AS employee, m.name AS manager
FROM employees e JOIN employees m ON m.id = e.manager_id;
```

This applies to `SELECT`, `WHERE`, `ORDER BY`, `GROUP BY`, and the `ON` clause. The one place you *can* sometimes omit qualification is a column that exists in neither ambiguous context — but in a self JOIN, *every* column exists in both, so qualify all of them, always. This is not merely to satisfy the parser; it's the difference between `WHERE e.salary > m.salary` (employees earning more than their manager) and `WHERE m.salary > e.salary` (the opposite) — an unqualified or misqualified reference is a silent logic inversion.

### 6.9 Choosing Alias Names — A Correctness Practice

Because the alias *is* the semantics, name aliases after the **role** each copy plays, not after the table:

```sql
-- BAD: e1/e2 give no hint which is employee, which is manager
FROM employees e1 JOIN employees e2 ON e2.id = e1.manager_id;

-- GOOD: role-based aliases document the relationship
FROM employees emp JOIN employees mgr ON mgr.id = emp.manager_id;
```

`emp`/`mgr`, `child`/`parent`, `curr`/`prev`, `a`/`b` (for symmetric pairs) all encode intent. In a self JOIN the alias is the only thing distinguishing the two roles, so a well-chosen alias name is a correctness aid, not just style.

---

## 7. EXPLAIN — Self JOIN in the Plan

A self JOIN's plan shows the **same table name twice**, distinguished by alias. This is the visual signature of a self JOIN in EXPLAIN output.

### Hash Join self-JOIN (employee → manager, full table)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT e.name AS employee, m.name AS manager
FROM employees e
INNER JOIN employees m ON m.id = e.manager_id;
```

```
Hash Join  (cost=33.50..78.20 rows=980 width=64)
           (actual time=0.42..1.85 rows=980 loops=1)
  Hash Cond: (e.manager_id = m.id)
  ->  Seq Scan on employees e  (cost=0.00..29.00 rows=1000 width=36)
                                (actual time=0.01..0.35 rows=1000 loops=1)
  ->  Hash  (cost=29.00..29.00 rows=1000 width=36)
             (actual time=0.38..0.38 rows=1000 loops=1)
        Buckets: 1024  Batches: 1  Memory Usage: 76kB
        ->  Seq Scan on employees m  (cost=0.00..29.00 rows=1000 width=36)
                                      (actual time=0.01..0.18 rows=1000 loops=1)
Buffers: shared hit=20
Planning Time: 0.28 ms
Execution Time: 1.94 ms
```

**Reading it**:
- `Seq Scan on employees e` and `Seq Scan on employees m` — the **same table appears twice**, once per alias. This is the self JOIN fingerprint.
- `Buffers: shared hit=20` — note how low this is. Both scans read the *same physical pages*; the second scan hits pages already cached by the first (Section 2's buffer-sharing point). A join of two genuinely different 1,000-row tables would show roughly double the buffer traffic.
- `Hash Cond: (e.manager_id = m.id)` — the equi-join condition linking the employee copy to the manager copy.
- The `m` side is hashed (built into the hash table), the `e` side probes it. The planner picked `m` as the build side; for equal-sized copies it's a toss-up.
- `rows=980` actual out of 1,000 employees — the 20 missing are those with NULL `manager_id` (top-level staff), correctly dropped by the INNER JOIN's UNKNOWN-on-NULL behaviour.

### Nested Loop self-JOIN (indexed, single-employee lookup)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT e.name AS employee, m.name AS manager
FROM employees e
INNER JOIN employees m ON m.id = e.manager_id
WHERE e.id = 4242;
```

```
Nested Loop  (cost=0.57..16.62 rows=1 width=64)
             (actual time=0.03..0.04 rows=1 loops=1)
  ->  Index Scan using employees_pkey on employees e
        (cost=0.29..8.31 rows=1 width=36) (actual rows=1 loops=1)
        Index Cond: (id = 4242)
  ->  Index Scan using employees_pkey on employees m
        (cost=0.29..8.30 rows=1 width=36) (actual rows=1 loops=1)
        Index Cond: (id = e.manager_id)
Buffers: shared hit=8
Planning Time: 0.22 ms
Execution Time: 0.07 ms
```

**Reading it**:
- Outer: `Index Scan ... employees e` filtered by `id = 4242` → 1 row.
- Inner: for that 1 row, `Index Scan ... employees m` on `Index Cond: (id = e.manager_id)` → probes the PK index once. `loops=1`.
- **The same PK index `employees_pkey` is used by both aliases** — there is only one physical index; both logical scans read it. `Buffers: shared hit=8` — a handful of index + heap page hits.
- Sub-millisecond. This is the ideal self-JOIN plan: point lookup on one side, indexed probe on the other.

### Non-equi self-JOIN (all cheaper products in category) — the danger plan

```sql
EXPLAIN (ANALYZE)
SELECT p.id, cheaper.id
FROM products p
INNER JOIN products cheaper
  ON cheaper.category_id = p.category_id
  AND cheaper.price < p.price;
```

```
Nested Loop  (cost=0.00..184320.00 rows=1250000 width=8)
             (actual time=0.05..842.10 rows=1180400 loops=1)
  Join Filter: (cheaper.price < p.price)
  Rows Removed by Join Filter: 1240600
  ->  Seq Scan on products p  (cost=0.00..170.00 rows=5000 width=12)
  ->  Index Scan using products_category_idx on products cheaper
        (cost=0.29..30.00 rows=250 width=12)
        Index Cond: (category_id = p.category_id)
Buffers: shared hit=421000
Execution Time: 968.30 ms
```

**Reading it**:
- `Join Filter: (cheaper.price < p.price)` with `Rows Removed by Join Filter: 1240600` — the inequality is applied *after* the index narrows to category. The index handles `category_id =` but **cannot** handle the `price <` — half the category-matched rows are then thrown away by the filter.
- `loops` on the inner index scan run once per outer product; the estimated 1.25M output rows reflect the fan-out (each product pairs with every cheaper product in its category).
- Fix hint: a composite index `products(category_id, price)` lets the index itself enforce the `price <` bound as a range, cutting `Rows Removed by Join Filter` dramatically. We revisit this in Section 10.

### What to look for in a self-JOIN EXPLAIN

| Signal | Meaning |
|--------|---------|
| Same table name twice, different alias | Confirms it's a self JOIN |
| `Buffers` lower than expected for the row count | Both aliases share cached pages — normal and good |
| `Join Filter` with high `Rows Removed` | Non-equi condition applied post-index — consider composite index |
| Inner `Seq Scan` (not Index Scan) on the second alias | Missing index on the join column — will scale terribly |
| `loops=` equal to outer row count | Nested Loop probing the inner alias once per outer row |

---

## 8. Query Examples

### Example 1 — Basic: Employee and Direct Manager

```sql
-- List every employee alongside their direct manager's name.
-- LEFT JOIN so top-level staff (manager_id IS NULL) are not dropped.
SELECT
  e.id             AS employee_id,
  e.name           AS employee_name,
  e.job_title,
  m.name           AS manager_name,       -- NULL for the CEO / top level
  m.job_title      AS manager_title
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id
ORDER BY m.name NULLS FIRST, e.name;
```

### Example 2 — Intermediate: Employees Earning More Than Their Manager

```sql
-- Salary inversion report: staff who out-earn their direct manager.
-- INNER JOIN is correct here: an employee with no manager cannot be
-- "earning more than their manager", so dropping NULL-manager rows is desired.
SELECT
  e.name           AS employee,
  e.salary         AS employee_salary,
  m.name           AS manager,
  m.salary         AS manager_salary,
  e.salary - m.salary AS salary_gap
FROM employees e
INNER JOIN employees m ON m.id = e.manager_id
WHERE e.salary > m.salary
ORDER BY salary_gap DESC;
```

### Example 3 — Production Grade: Duplicate-Customer Merge Candidates

```sql
-- CONTEXT:
--   users table: ~8,000,000 rows.
--   Goal: find pairs of accounts that are likely the same person
--   (identical normalised email OR identical phone), to feed a merge queue.
--   Indexes assumed:
--     CREATE INDEX users_email_lower_idx ON users (LOWER(email));
--     CREATE INDEX users_phone_idx       ON users (phone) WHERE phone IS NOT NULL;
--   Perf expectation: with the functional index on LOWER(email), the equi-join
--   probes the index rather than scanning 8M rows. Runs in a few seconds for the
--   email-collision set (typically a small fraction of rows collide).
--
-- The a.id < b.id predicate guarantees each unordered pair appears once and
-- excludes self-matches.

SELECT
  a.id             AS keep_user_id,      -- lower id = older account, kept
  b.id             AS merge_user_id,     -- higher id = newer account, merged away
  LOWER(a.email)   AS matched_email,
  a.created_at     AS keep_created_at,
  b.created_at     AS merge_created_at,
  CASE
    WHEN LOWER(a.email) = LOWER(b.email) AND a.phone = b.phone THEN 'email+phone'
    WHEN LOWER(a.email) = LOWER(b.email)                       THEN 'email'
    ELSE 'phone'
  END              AS match_reason
FROM users a
INNER JOIN users b
  ON a.id < b.id                          -- unordered pairs, no self-match
  AND (
        LOWER(a.email) = LOWER(b.email)   -- collide on normalised email
     OR (a.phone IS NOT NULL AND a.phone = b.phone)  -- or on phone
      )
WHERE a.status <> 'deleted'
  AND b.status <> 'deleted'
ORDER BY a.id
LIMIT 1000;
```

```
-- EXPLAIN (ANALYZE, BUFFERS) — abridged
Nested Loop  (cost=0.56..48210.30 rows=3400 width=72)
             (actual time=0.20..2140.50 rows=2870 loops=1)
  ->  Index Scan using users_email_lower_idx on users a
        (actual rows=7994000 loops=1)
  ->  Index Scan using users_email_lower_idx on users b
        Index Cond: (LOWER(b.email) = LOWER(a.email))
        Filter: (a.id < b.id AND b.status <> 'deleted')
        (actual rows=... loops=7994000)
  Buffers: shared hit=1204553 read=88210
  Execution Time: 2210.90 ms
```

The functional index on `LOWER(email)` is what makes this tractable at 8M rows — without it, the `LOWER(a.email) = LOWER(b.email)` condition forces a sequential scan on the inner alias for every outer row, turning an O(n) indexed probe into an O(n²) catastrophe.

---

## 9. Wrong → Right Patterns

### Wrong 1: INNER JOIN drops the top of the hierarchy

```sql
-- WRONG: intending to list ALL employees with their manager
SELECT e.name AS employee, m.name AS manager
FROM employees e
INNER JOIN employees m ON m.id = e.manager_id;
-- SILENT BUG: the CEO and any other manager_id-IS-NULL rows VANISH.
-- ON condition m.id = NULL evaluates to UNKNOWN → row excluded.
-- If you have 1,000 employees and 20 have no manager, you get 980 rows
-- and may not notice the 20 are missing.
```

```sql
-- RIGHT: LEFT JOIN preserves rows with no match in the manager copy
SELECT e.name AS employee, m.name AS manager   -- manager is NULL at the top
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id;
```

The execution-level cause: `e.manager_id IS NULL` makes the equi-condition UNKNOWN, and INNER JOIN keeps only TRUE. This is the single most common self-JOIN bug — the root node of every hierarchy has a NULL parent, and INNER JOIN eats it.

### Wrong 2: The LEFT JOIN → INNER JOIN silent downgrade (ON vs WHERE)

```sql
-- WRONG: want all employees, but only show manager if they're in Engineering
SELECT e.name, m.name AS manager
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id
WHERE m.department_id = 3;         -- ← filter on the RIGHT alias in WHERE
-- SILENT BUG: this converts the LEFT JOIN back to an INNER JOIN.
-- Rows where m is NULL (CEO) have m.department_id = NULL, which fails
-- "= 3", so they're dropped. The whole point of LEFT JOIN is undone.
```

```sql
-- RIGHT: put the filter on the right alias INTO the ON clause
SELECT e.name, m.name AS manager
FROM employees e
LEFT JOIN employees m
  ON m.id = e.manager_id
  AND m.department_id = 3;         -- ← filter in ON preserves unmatched e rows
-- Employees whose manager is not in dept 3 (or who have no manager) keep
-- their row, with manager_name = NULL.
```

This is the Topic 12 ON-vs-WHERE trap applied to a self JOIN. A predicate on the nullable (right) side of a LEFT JOIN belongs in `ON`, not `WHERE`, or you destroy the outer-join semantics.

### Wrong 3: Pair-finding without the asymmetric key predicate

```sql
-- WRONG: find users sharing a birthday
SELECT a.id, b.id, a.birthday
FROM users a
INNER JOIN users b ON b.birthday = a.birthday;
-- BUG 1: every user matches themselves → (1,1), (2,2), ... garbage pairs
-- BUG 2: every real pair appears twice → (1,5) AND (5,1)
-- Result is ~2× the real pair count plus one self-row per user.
```

```sql
-- RIGHT: a.id < b.id kills self-matches and mirror duplicates in one predicate
SELECT a.id AS user1, b.id AS user2, a.birthday
FROM users a
INNER JOIN users b
  ON b.birthday = a.birthday
  AND a.id < b.id;
```

### Wrong 4: Wrong alias in the comparison — silent logic inversion

```sql
-- WRONG: intending "employees earning more than their manager"
SELECT e.name, m.name AS manager
FROM employees e
INNER JOIN employees m ON m.id = e.manager_id
WHERE m.salary > e.salary;         -- ← aliases swapped!
-- No error. But this returns employees earning LESS than their manager —
-- the exact OPPOSITE of the intent. In a self JOIN both aliases have a
-- 'salary' column, so the swap is invisible.
```

```sql
-- RIGHT: e (employee) on the left of the >, m (manager) on the right
SELECT e.name, m.name AS manager
FROM employees e
INNER JOIN employees m ON m.id = e.manager_id
WHERE e.salary > m.salary;
```

The lesson: in a self JOIN there is no compiler safety net for alias direction — both copies have identical columns, so read every comparison carefully.

### Wrong 5: Unbounded all-pairs self JOIN

```sql
-- WRONG: "compare every order to every other order" with no bound
SELECT a.id, b.id, ABS(a.total_amount - b.total_amount) AS diff
FROM orders a
CROSS JOIN orders b;
-- On a 2M-row orders table this is 2M × 2M = 4 TRILLION rows.
-- The query never completes; it fills disk with temp files and is killed.
```

```sql
-- RIGHT: bound the pairing to a meaningful, small scope, and use a.id < b.id
SELECT a.id, b.id, ABS(a.total_amount - b.total_amount) AS diff
FROM orders a
INNER JOIN orders b
  ON a.customer_id = b.customer_id   -- only within the same customer
  AND a.id < b.id                     -- unordered pairs, no self-pair
WHERE a.created_at >= CURRENT_DATE - INTERVAL '30 days';
-- Now each customer's recent orders are paired among themselves — a tiny,
-- indexed, tractable set.
```

---

## 10. Performance Profile

A self JOIN has the same cost model as any join (Topic 11 §10), with one favourable twist and two specific hazards.

### The favourable twist: buffer sharing

Both aliases scan the same heap and (often) the same indexes. The shared buffer pool caches those pages once, so the *second* scan enjoys a high buffer-hit ratio. A self JOIN of a table with itself is typically cheaper on I/O than joining two distinct tables of the same size, because there's half the distinct data to fault in.

### Hazard 1: the join column must be indexed

For the adjacency-list pattern, you need an index on **both** sides of the link:

```sql
-- employees.id is the PK → already indexed (used for the manager-side probe)
CREATE INDEX employees_manager_id_idx ON employees (manager_id);
--   ^ needed so "find e where e.manager_id = m.id" (children of a manager)
--     and general manager-side joins can use an index instead of Seq Scan
```

Without `employees_manager_id_idx`, a query that drives from the manager side (`m JOIN e ON e.manager_id = m.id`) does a sequential scan of `employees` for every manager — O(managers × rows).

### Hazard 2: non-equi self JOINs and the composite index

The "cheaper product in same category" query (Section 6.2) applies `category_id =` via index but `price <` as a post-filter. A composite index lets the index enforce both:

```sql
CREATE INDEX products_category_price_idx ON products (category_id, price);
-- Now "cheaper.category_id = p.category_id AND cheaper.price < p.price"
-- becomes an index RANGE scan: seek to (p.category_id, <p.price) and read
-- backwards. Rows Removed by Join Filter drops toward zero.
```

### Scaling table

| Table size | Pattern | Behaviour |
|-----------|---------|-----------|
| 10K rows | Adjacency-list (indexed PK + manager_id) | Sub-millisecond per lookup; full-table hash self-join < 10 ms |
| 1M rows | Adjacency-list, indexed | Hash self-join ~200–500 ms; single-node lookup < 1 ms |
| 1M rows | Non-equi (price <) without composite index | Seconds — filter throws away most of each category scan |
| 1M rows | Non-equi with composite index | Tens–hundreds of ms |
| 10M rows | Pair-finding (a.id < b.id) with functional index on join key | Seconds; scales with collision count, not total rows |
| 100K rows | Unbounded CROSS JOIN self-join | ~5 billion rows after a.id<b.id — do not run |
| 2M+ rows | Unbounded CROSS JOIN self-join | Trillions of rows — never completes |

### Optimisation checklist for self JOINs

1. **Index the join column** on the alias being probed (usually the `manager_id` / parent-pointer column, since the PK side is already indexed).
2. **Composite index** `(equi_col, range_col)` for non-equi self JOINs, so the range predicate is served by the index rather than a Join Filter.
3. **Always bound all-pairs** self JOINs with `a.id < b.id` and a scope-narrowing equi-condition; never run an unbounded `CROSS JOIN` of a large table with itself.
4. **Prefer window functions** for "previous/next row" and running-comparison patterns — a `LAG`/`LEAD` over a partition is a single ordered pass versus the self JOIN's O(n²) with a "nothing between" subquery.
5. **Watch `work_mem`** — a hash self-join builds a hash table from one copy; if the table is large the build side can spill (`Batches > 1`), same as any hash join.

---

## 11. Node.js Integration

### 11.1 Adjacency-list lookup (employee + manager)

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function getEmployeeWithManager(employeeId) {
  const { rows } = await pool.query(
    `SELECT
       e.id            AS employee_id,
       e.name          AS employee_name,
       e.job_title,
       m.id            AS manager_id,
       m.name          AS manager_name    -- NULL if employee is top-level
     FROM employees e
     LEFT JOIN employees m ON m.id = e.manager_id
     WHERE e.id = $1`,
    [employeeId]
  );
  return rows[0] ?? null;
}
```

### 11.2 Salary-inversion report

```javascript
async function getSalaryInversions() {
  const { rows } = await pool.query(
    `SELECT
       e.name              AS employee,
       e.salary            AS employee_salary,
       m.name              AS manager,
       m.salary            AS manager_salary,
       e.salary - m.salary AS salary_gap
     FROM employees e
     INNER JOIN employees m ON m.id = e.manager_id
     WHERE e.salary > m.salary
     ORDER BY salary_gap DESC`
  );
  return rows;
}
```

### 11.3 Duplicate-detection with the `<` predicate

```javascript
// Parameterised so the caller can scope the search and page through results.
async function findDuplicateEmails({ limit = 500, offset = 0 } = {}) {
  const { rows } = await pool.query(
    `SELECT
       a.id          AS keep_id,
       b.id          AS dup_id,
       LOWER(a.email) AS email,
       a.created_at  AS keep_created_at,
       b.created_at  AS dup_created_at
     FROM users a
     INNER JOIN users b
       ON LOWER(b.email) = LOWER(a.email)
       AND a.id < b.id
     WHERE a.status <> 'deleted'
       AND b.status <> 'deleted'
     ORDER BY a.id
     LIMIT $1 OFFSET $2`,
    [limit, offset]
  );
  return rows;
}
```

### 11.4 Direct-report counts per manager

```javascript
async function getManagerSpans() {
  const { rows } = await pool.query(
    `SELECT
       m.id                    AS manager_id,
       m.name                  AS manager,
       COUNT(e.id)             AS direct_reports,
       COALESCE(SUM(e.salary), 0) AS reports_total_salary
     FROM employees m
     LEFT JOIN employees e ON e.manager_id = m.id   -- LEFT: keep managers with 0 reports
     GROUP BY m.id, m.name
     HAVING COUNT(e.id) > 0
     ORDER BY direct_reports DESC`
  );
  return rows;
}
```

A note on driver behaviour: `pg` returns every column by its output name. In a self JOIN the two copies share column names (`name`, `salary`), so **you must alias the columns in SQL** (`e.name AS employee`, `m.name AS manager`) — otherwise `pg` returns two properties both called `name` and the second silently overwrites the first in the result object. This is a self-JOIN-specific footgun in every JS driver: always alias colliding columns.

---

## 12. ORM Comparison

Self JOINs are where ORMs diverge most, because a self JOIN needs the *same* model aliased twice — something many ORMs express awkwardly.

### Prisma

**Can Prisma do a self JOIN?** — For adjacency-list *relations*, yes, via a self-relation in the schema. For arbitrary self JOINs (non-equi, pair-finding), no — you need `$queryRaw`.

```prisma
// schema.prisma — self-relation for the manager hierarchy
model Employee {
  id         Int        @id @default(autoincrement())
  name       String
  salary     Int
  managerId  Int?
  manager    Employee?  @relation("Reports", fields: [managerId], references: [id])
  reports    Employee[] @relation("Reports")
}
```

```typescript
// Fetch an employee with their manager (one level up)
const emp = await prisma.employee.findUnique({
  where: { id: employeeId },
  include: { manager: true },     // resolves the self-relation
});

// Non-equi / pair-finding self JOINs are NOT expressible — use raw SQL:
const inversions = await prisma.$queryRaw`
  SELECT e.name AS employee, e.salary, m.name AS manager, m.salary AS manager_salary
  FROM employees e
  INNER JOIN employees m ON m.id = e.manager_id
  WHERE e.salary > m.salary
  ORDER BY e.salary - m.salary DESC
`;
```

**Where it breaks**: Prisma models self-relations as nested object reads, which under the hood may be a JOIN or a separate query. It cannot express `e.salary > m.salary`, `a.id < b.id`, or any self JOIN whose condition isn't a plain relation traversal. **Verdict**: great for one-level hierarchy reads; raw SQL for anything comparative.

### Drizzle ORM

**Can Drizzle do a self JOIN?** — Yes, cleanly, via `alias()`, which is purpose-built for this.

```typescript
import { alias } from 'drizzle-orm/pg-core';
import { employees } from './schema';
import { eq, sql, and } from 'drizzle-orm';
import { db } from './db';

const mgr = alias(employees, 'mgr');   // ← second aliased instance of the same table

const rows = await db
  .select({
    employee:       employees.name,
    employeeSalary: employees.salary,
    manager:        mgr.name,
    managerSalary:  mgr.salary,
  })
  .from(employees)
  .innerJoin(mgr, eq(mgr.id, employees.managerId))
  .where(sql`${employees.salary} > ${mgr.salary}`);   // non-equi across aliases
```

**Where it breaks**: essentially doesn't. `alias()` is the idiomatic, type-safe way to reference a table twice; non-equi conditions use the `sql` template. **Verdict**: best-in-class self-JOIN support of the ORMs here.

### Sequelize

**Can Sequelize do a self JOIN?** — Yes, via a self-association plus `include` with an alias, or raw SQL.

```javascript
// Define the self-association
Employee.belongsTo(Employee, { as: 'manager', foreignKey: 'managerId' });
Employee.hasMany(Employee,   { as: 'reports', foreignKey: 'managerId' });

// One-level read
const emp = await Employee.findByPk(employeeId, {
  include: [{ model: Employee, as: 'manager', required: false }], // LEFT JOIN
});

// Comparative self JOIN → raw query
const [inversions] = await sequelize.query(
  `SELECT e.name AS employee, m.name AS manager
   FROM employees e INNER JOIN employees m ON m.id = e.manager_id
   WHERE e.salary > m.salary`
);
```

**Where it breaks**: the association `as` alias is mandatory (Sequelize errors without it — echoing SQL's own mandatory-alias rule). Non-equi and pair-finding self JOINs require `sequelize.query()`. Remember `required: true` for INNER-JOIN semantics vs the default LEFT JOIN. **Verdict**: workable for hierarchies; raw SQL for comparisons.

### TypeORM

**Can TypeORM do a self JOIN?** — Yes; self-referencing relations are first-class, and the QueryBuilder aliases naturally.

```typescript
// Entity with self-referencing relations
@Entity()
class Employee {
  @PrimaryGeneratedColumn() id: number;
  @Column() name: string;
  @Column() salary: number;
  @ManyToOne(() => Employee, (e) => e.reports) manager: Employee;
  @OneToMany(() => Employee, (e) => e.manager) reports: Employee[];
}

// QueryBuilder self JOIN with a non-equi condition
const inversions = await dataSource
  .getRepository(Employee)
  .createQueryBuilder('e')
  .innerJoin('e.manager', 'm')                 // self JOIN via the relation
  .where('e.salary > m.salary')                // non-equi across aliases
  .select(['e.name', 'e.salary', 'm.name'])
  .getRawMany();
```

**Where it breaks**: the relation-based `innerJoin('e.manager', 'm')` covers hierarchy joins; for pair-finding self JOINs not backed by a relation you drop to `innerJoin('employee', 'b', 'a.id < b.id AND ...')` with a raw condition string (no type safety). **Verdict**: solid for self-referencing entities; raw condition strings for exotic self JOINs.

### Knex.js

**Can Knex do a self JOIN?** — Yes, transparently — you just alias the table twice.

```javascript
const rows = await knex('employees as e')
  .leftJoin('employees as m', 'm.id', 'e.manager_id')   // ← same table, two aliases
  .select('e.name as employee', 'm.name as manager');

// Pair-finding with the < predicate
const dupes = await knex('users as a')
  .join('users as b', function () {
    this.on(knex.raw('LOWER(a.email)'), '=', knex.raw('LOWER(b.email)'))
        .andOn('a.id', '<', 'b.id');
  })
  .select('a.id as keep_id', 'b.id as dup_id', 'a.email');
```

**Where it breaks**: nothing structural — Knex mirrors SQL, so `'employees as e'` / `'employees as m'` is exactly the SQL self JOIN. Complex ON conditions use the function form or `knex.raw`. **Verdict**: the most transparent; a self JOIN looks like a self JOIN.

### ORM summary table

| ORM | Self JOIN mechanism | Non-equi / pair-finding | Verdict |
|-----|--------------------|------------------------|---------|
| Prisma | Self-relation in schema (`@relation`) | Raw SQL only | Hierarchy reads yes; comparisons need raw |
| Drizzle | `alias(table, 'x')` | `sql` template | Best typed self-JOIN support |
| Sequelize | Self-association with `as` alias | `sequelize.query()` | Works; alias mandatory; raw for comparisons |
| TypeORM | Self-referencing relation + QueryBuilder | Raw condition string | Solid for relations; raw strings otherwise |
| Knex | Alias table twice (`'t as a'`, `'t as b'`) | Function-form ON / `knex.raw` | Most transparent, closest to SQL |

Across all of them, the pattern is identical to SQL itself: **the ORM needs a way to name the same table twice.** Drizzle's `alias()` and Knex's `'t as a'` are the most direct; Prisma/Sequelize/TypeORM wrap it in a relation abstraction that handles hierarchies but forces raw SQL for comparative self JOINs.

---

## 13. Practice Exercises

### Exercise 1 — Easy

Given `employees(id, name, salary, manager_id, department_id, job_title)`:

Write a query that returns every employee's `name`, `job_title`, and their direct manager's `name` (as `manager_name`). Include employees who have no manager (their `manager_name` should be NULL). Order by `manager_name` (NULLs first), then employee `name`.

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines self JOIN + aggregation from Topic 11 §6.8)

Given `employees(id, name, salary, manager_id, department_id)` and `departments(id, name)`:

For each manager, return their `name`, their department name, the number of **direct reports**, the **average salary** of their direct reports, and how that average compares to the manager's own salary (`avg_report_salary - m.salary` as `gap`). Only include managers who have at least 3 direct reports. Order by `gap` descending (managers whose reports out-earn them most, first).

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

Given `products(id, name, category_id, price)`:

For each product, return its `name`, `price`, and the name of the **single next-cheapest product in the same category** (the product with the highest price that is still strictly less than this product's price). Products that are the cheapest in their category should still appear, with `next_cheapest = NULL`.

The naive self JOIN `ON cheaper.category_id = p.category_id AND cheaper.price < p.price` matches *all* cheaper products, not just the next one. Solve it correctly. (Hint: either a `NOT EXISTS` "nothing between" condition, or recognise that this is what a window function does in one pass — solve it both ways and compare.)

```sql
-- Write your query here (self JOIN version)

-- Write your query here (window function version)
```

### Exercise 4 — Interview Simulation

Given `users(id, email, phone, created_at, status)` with ~5M rows:

Write a query that produces a **merge queue** of duplicate accounts: pairs of non-deleted users that share either a case-insensitive email OR a phone number, where each unordered pair appears exactly once and no user is paired with itself. For each pair return the older account's id (`keep_id`), the newer account's id (`merge_id`), and the match reason (`'email'`, `'phone'`, or `'both'`). Then state which indexes you would create to make this run in seconds rather than hours.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What is a self JOIN and why do you need table aliases for it?**

A: A self JOIN is joining a table to another copy of itself, used when a row needs to relate to another row in the same table — like an employee row pointing to their manager's row via `manager_id`. You need aliases because the query references the same table twice, and every column exists in both copies. Without aliases like `e` and `m`, a reference like `name` or `salary` is ambiguous — the engine can't tell which copy you mean — and Postgres rejects the query. The alias is what makes the two copies distinguishable.

**Interviewer follow-up**: *"What join type would you use to fetch every employee including the CEO who has no manager, and why?"* — LEFT JOIN. With INNER JOIN, the CEO's `manager_id IS NULL` makes the ON condition UNKNOWN, so the row is dropped. LEFT JOIN keeps every employee, filling `manager_name` with NULL where there's no match.

---

### Junior Level

**Q: You wrote `SELECT a.id, b.id FROM users a JOIN users b ON a.email = b.email` to find duplicate emails and got way more rows than expected, including rows where a.id equals b.id. Why?**

A: Two problems. First, every row matches itself — a row's email equals its own email — so you get self-pairs like (1,1). Second, every genuine pair appears twice, once as (1,5) and once as (5,1), because the join condition is symmetric. The fix is to add `AND a.id < b.id`: it excludes self-matches (a row's id is never less than itself) and eliminates the mirror duplicate (only one of the two orderings satisfies `<`).

**Interviewer follow-up**: *"Why `<` and not `<>`?"* — `<>` (not-equal) removes the self-matches but keeps *both* mirror orderings, so you still get every pair twice. `<` does both jobs: no self-match and only one ordering per pair.

---

### Principal Level

**Q: A LEFT self JOIN of employees to their managers works, but when the team adds `WHERE m.department_id = 3` to filter to Engineering managers, the top-level employees disappear. Explain the mechanism and the fix.**

A: The `WHERE m.department_id = 3` predicate references the nullable right side of the LEFT JOIN. For a top-level employee, the manager copy `m` is all NULLs (no match), so `m.department_id` is NULL, and `NULL = 3` evaluates to UNKNOWN — the WHERE clause drops the row. This effectively downgrades the LEFT JOIN back to an INNER JOIN: any filter on the right alias in WHERE eliminates the very unmatched rows the LEFT JOIN was there to preserve. The fix is to move the predicate into the ON clause: `LEFT JOIN employees m ON m.id = e.manager_id AND m.department_id = 3`. In the ON clause the predicate participates in match-finding, not row-elimination — employees whose manager isn't in dept 3 (or who have no manager) keep their row with `manager_name = NULL`. This is the ON-vs-WHERE distinction from LEFT JOIN semantics, and it's identical whether the two tables are different or the same.

**Interviewer follow-up**: *"Does this ON-vs-WHERE distinction matter for an INNER self JOIN?"* — No. For INNER JOIN, ON and WHERE are logically equivalent — there are no unmatched rows to preserve, so moving the predicate between them can't change the result (the planner may still reorder for performance). The distinction only bites on OUTER joins.

---

### Principal Level

**Q: You have a 10M-row `orders` table and need, for each order, the customer's immediately preceding order. A colleague wrote a self JOIN with `prev.created_at < o.created_at` and a `NOT EXISTS` "nothing in between" subquery. It's correct but takes minutes. What's happening and how do you fix it?**

A: The self JOIN pairs each order with *every* earlier order by the same customer — that's a fan-out that's quadratic in each customer's order count — and the `NOT EXISTS` subquery adds a *third* correlated scan of the same table to filter down to just the adjacent pair. So you're doing O(n²)-ish work per customer plus a correlated anti-join. The fix is a window function: `LAG(...) OVER (PARTITION BY customer_id ORDER BY created_at)`. That computes the previous order in a single ordered pass — sort by `(customer_id, created_at)` once, then read adjacent rows — turning the O(n²) self JOIN into O(n log n). A supporting index on `(customer_id, created_at)` lets the sort be an index scan. This is the canonical case where recognising the self JOIN pattern is what tells you to reach for a window function instead. The self JOIN is still worth understanding, because you have to see the pattern to know it's replaceable.

**Interviewer follow-up**: *"When would you keep the self JOIN instead of the window function?"* — When the relationship isn't a simple ordered offset — e.g., pairing rows by an equality on a non-sequential key, or all-pairs comparisons within a bounded group — where there's no `PARTITION BY / ORDER BY` framing that a window function can express. Self JOINs are strictly more general; window functions are faster for the ordered-neighbour subset.

---

### Principal Level

**Q: How does a self JOIN behave differently from a join between two distinct tables, at the storage and planner level?**

A: Functionally it doesn't differ at all — the planner builds two range-table entries pointing at the same table OID and treats them as two independent scans fed into an ordinary join node, using the same three algorithms (Nested Loop, Hash, Merge). The differences are secondary: (1) buffer-pool sharing — both aliases scan the same heap pages, so the second scan gets a high cache-hit ratio and total I/O is lower than joining two same-size distinct tables; (2) both aliases share one physical set of indexes and one set of `pg_statistic` rows, so estimates for both sides come from the same statistics; (3) the planner assumes independence between the two aliases and doesn't model the correlation that they're the same data (e.g., that `manager_id` values are drawn from `id` values), which can occasionally skew row estimates on hierarchical joins. But there's no `SELF JOIN` operator — it's INNER/LEFT/CROSS JOIN with two aliases of one table.

**Interviewer follow-up**: *"If both aliases scan the same table, can they see different snapshots of the data?"* — No. Both scans run inside the same statement under the same MVCC snapshot, so they see an identical, consistent view. There's no window in which one alias could see a row the other can't.

---

## 15. Mental Model Checkpoint

1. A table has 1,000 employees; 20 have `manager_id IS NULL`. How many rows does `employees e INNER JOIN employees m ON m.id = e.manager_id` return, and how many does the LEFT JOIN version return? Why the difference?

2. You write `SELECT id, name FROM employees e JOIN employees m ON m.id = e.manager_id`. What error do you get, and what's the underlying reason the engine can't run it?

3. In a pair-finding self JOIN, explain precisely why `a.id < b.id` eliminates *both* self-matches and mirror-duplicate pairs, while `a.id <> b.id` eliminates only one of the two problems.

4. You have a LEFT self JOIN of categories to their parent categories and you add `WHERE parent.is_active = true`. What happens to top-level categories, and where should that predicate go instead?

5. A self JOIN `products p JOIN products c ON c.category_id = p.category_id AND c.price < p.price` returns 1.2M rows and EXPLAIN shows a large "Rows Removed by Join Filter." What index would you add, and what does it change in the plan?

6. Why is an unbounded `CROSS JOIN` of a 100K-row table with itself dangerous, and what two things do you add to make an all-pairs self JOIN safe?

7. You need "each order and the customer's previous order." Describe the self JOIN solution and explain why a window function is usually the better tool for this specific shape — and name one self-JOIN shape where the window function *cannot* substitute.

---

## 16. Quick Reference Card

```sql
-- SELF JOIN = join a table to another alias of itself. No "SELF JOIN" keyword.
-- Aliases are MANDATORY: both copies share column names.

-- Adjacency list: each row with its parent (hierarchy, ONE level)
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id;   -- LEFT keeps the root (NULL parent)

-- INNER drops the top of the hierarchy (manager_id IS NULL → excluded)
-- Use LEFT JOIN to keep the CEO / root node.

-- N levels up = N self joins (unbounded depth → WITH RECURSIVE instead)
FROM employees e
LEFT JOIN employees m  ON m.id  = e.manager_id
LEFT JOIN employees mm ON mm.id = m.manager_id;

-- Compare rows within a table (non-equi)
WHERE e.salary > m.salary            -- employees out-earning their manager

-- Find PAIRS / duplicates: a.id < b.id kills self-matches AND mirror dupes
SELECT a.id, b.id, a.email
FROM users a JOIN users b
  ON b.email = a.email AND a.id < b.id;

-- All-pairs within a bounded group (guard against explosion!)
FROM t a JOIN t b ON a.group_id = b.group_id AND a.id < b.id;

-- ON vs WHERE trap (LEFT self JOIN): filter on right alias goes in ON
LEFT JOIN employees m ON m.id = e.manager_id AND m.dept_id = 3   -- keeps roots
-- ...WHERE m.dept_id = 3   ← WRONG: silently downgrades to INNER JOIN

-- ALWAYS alias-qualify every column and alias colliding outputs for JS drivers
SELECT e.name AS employee, m.name AS manager   -- not bare "name" (ambiguous)

-- Indexing:
CREATE INDEX ON employees (manager_id);              -- parent-pointer join column
CREATE INDEX ON products  (category_id, price);      -- composite for non-equi self join

-- "Previous row" self JOIN (prev.created_at < o.created_at + NOT EXISTS between)
--   → prefer a window function: LAG(...) OVER (PARTITION BY ... ORDER BY ...)
```

**Perf rules of thumb**:
- Both aliases scan the same heap → buffer sharing makes I/O cheaper than joining two distinct same-size tables.
- Index the parent-pointer column; use a composite `(equi, range)` index for non-equi self JOINs.
- Never run an unbounded `CROSS JOIN` of a large table with itself — bound it and add `a.id < b.id`.
- "Previous/next row" and running comparisons → window function, not self JOIN.

**Interview one-liners**:
- "A self JOIN is INNER/LEFT/CROSS JOIN with two aliases of one table — there's no SELF JOIN keyword."
- "Aliases are mandatory because both copies share column names; the alias is what makes columns unambiguous."
- "Use LEFT JOIN for hierarchies so the root (NULL parent) survives."
- "`a.id < b.id` gives each unordered pair once and excludes self-matches."
- "A filter on the right alias in WHERE silently downgrades a LEFT self JOIN to INNER — put it in ON."

---

## Connected Topics

- **Topic 11 — INNER JOIN in Depth**: The fan-out, NULL-exclusion, and algorithm behaviour all apply verbatim to a self INNER JOIN; a self JOIN is just an INNER/LEFT/CROSS JOIN with two aliases.
- **Topic 12 — LEFT JOIN and RIGHT JOIN**: The ON-vs-WHERE trap that preserves hierarchy roots is the LEFT JOIN semantics applied to a self JOIN.
- **Topic 14 — CROSS JOIN** (previous): The all-pairs self JOIN is a CROSS JOIN of a table with itself, bounded by `a.id < b.id`.
- **Topic 16 — Multiple JOINs** (next): Multi-level hierarchy walks chain several self JOINs; the same aliasing discipline scales to N copies.
- **Topic 17 — JOIN Performance Deep Dive**: Nested Loop / Hash / Merge selection, work_mem spills, and index strategy for the two scans of one heap.
- **Topic 07 — NULL in Depth**: Why `manager_id IS NULL` drops the root under INNER JOIN — three-valued logic in the ON condition.
- **Window Functions (LAG/LEAD)**: The single-pass replacement for "previous/next row" self JOINs; recognise the self JOIN pattern to know when to switch.
- **Recursive CTEs (WITH RECURSIVE)**: The generalisation of the repeated self JOIN for arbitrary-depth hierarchy traversal.
- **Topic 20 — GROUP BY Fundamentals**: Aggregating a self JOIN (direct-report counts) collapses the fan-out exactly as with any join.
