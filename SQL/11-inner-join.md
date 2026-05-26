# Topic 11 — INNER JOIN in Depth
### SQL Mastery Curriculum — Phase 3: JOINs — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

You have two lists pinned on a wall:
- **List A**: every student in school with their student ID
- **List B**: every student who passed the exam, with their student ID and their score

An INNER JOIN is: go through List A, find each student's ID in List B, and write down only the students who appear in **both** lists — with both their name (from A) and their score (from B).

Students in List A who aren't in List B? Gone.  
Entries in List B without a match in List A? Also gone.  
Only the intersection — rows with a confirmed match on both sides — make it into the result.

That's the complete definition of INNER JOIN. The word "INNER" means "only the intersection."

---

## 2. Connection to SQL Internals

INNER JOIN is the simplest join type at the engine level. It maps directly to:

```
Result = { (a, b) | a ∈ TableA, b ∈ TableB, ON_condition(a, b) = TRUE }
```

No special handling for unmatched rows (that's OUTER JOIN territory). No row duplication unless the ON condition produces multiple matches. Just: every pair where the condition is TRUE.

At the physical layer, PostgreSQL uses one of three algorithms to implement INNER JOIN:

1. **Nested Loop**: For each row in the outer table, scan (or index-probe) the inner table for matches. Cost is O(outer × inner_probe).
2. **Hash Join**: Build a hash table from the smaller table. Scan the larger table and probe the hash for matches. Cost is O(N + M) with O(smaller) memory.
3. **Merge Join**: Sort both tables on the join key (or use an existing sort order from an index), then scan both in lockstep. Cost is O(N log N + M log M) if sorting needed, O(N + M) if pre-sorted.

The planner chooses based on table sizes, available indexes, and cost model estimates. The result is always identical — only the performance differs.

---

## 3. Logical Execution Order Context

```
FROM tableA
INNER JOIN tableB ON condition   ← evaluated in FROM phase
WHERE ...                         ← applied after join
GROUP BY ...
HAVING ...
SELECT ...
ORDER BY ...
LIMIT ...
```

For INNER JOIN specifically:
- Conditions in `ON` and conditions in `WHERE` are logically equivalent — moving a filter from ON to WHERE (or vice versa) does not change the result
- The planner exploits this: it may push a WHERE predicate into the join to allow earlier filtering, reducing intermediate row counts
- This is why `WHERE a.status = 'active'` in a query with only INNER JOINs can be placed either in ON or WHERE — same output, same (or planner-adjusted) performance

This equivalence **only holds for INNER JOIN**. For OUTER JOINs (Topic 12), this distinction is critical.

---

## 4. What Is INNER JOIN?

INNER JOIN returns the set of rows that have a matching row in both tables, satisfying the ON condition.

```sql
SELECT column_list
FROM table_a [AS a]
[INNER] JOIN table_b [AS b] ON a.key = b.key;
```

`INNER` is the default — writing `JOIN` without a type keyword means `INNER JOIN`.

### Exact Semantics

Given:

```
table_a:          table_b:
id | name         a_id | score
---+------        -----+------
1  | Alice        1    | 95
2  | Bob          1    | 87
3  | Charlie      4    | 72   ← no match in table_a
```

```sql
SELECT a.name, b.score
FROM table_a a
INNER JOIN table_b b ON b.a_id = a.id;
```

```
name  | score
------+------
Alice | 95
Alice | 87
Bob   | (no rows — Bob not in table_b)
```

Wait — `Bob` has no entry in `table_b`, so he is **excluded**. `Charlie` has no entry in `table_b` either — **excluded**. The `score = 72` row in `table_b` has `a_id = 4` which doesn't exist in `table_a` — **excluded**.

Only `Alice` with `id = 1` matches `a_id = 1` in `table_b`. Alice has two matching rows (scores 95 and 87) → appears **twice**.

This is the complete behaviour:
- **Unmatched rows from either side are dropped**
- **One-to-many matches produce multiple output rows**
- **The output row count equals the number of matching pairs**

---

## 5. Why INNER JOIN Mastery Matters in Production

1. **It's the most-used JOIN** — the vast majority of production queries use INNER JOIN. Mastering its edge cases (row multiplication, NULL behaviour, algorithm selection) prevents the most common bugs.

2. **Row count reasoning** — being able to predict how many rows an INNER JOIN will return is the foundation for debugging unexpected query results and for estimating query cost.

3. **Algorithm awareness** — understanding when the planner uses Nested Loop vs Hash Join vs Merge Join lets you diagnose slow joins and know whether adding an index will actually help.

4. **NULL behaviour** — INNER JOIN silently drops rows where the join key is NULL on either side. This is correct but often surprising. Knowing this prevents "why are some rows missing?" bugs.

5. **JOIN order in multi-table queries** — the planner reorders joins for cost, but understanding INNER JOIN's commutativity and associativity helps you reason about complex multi-join queries.

---

## 6. Deep Technical Content

### 6.1 NULL Behaviour in INNER JOIN

NULL on either side of the join key causes the row to be silently excluded.

```sql
-- employees has some rows where department_id is NULL
SELECT e.name, d.name AS department
FROM employees e
INNER JOIN departments d ON d.id = e.department_id;
-- Employees with NULL department_id are excluded — no error, no warning
```

This is because the ON condition `d.id = e.department_id` evaluates to `UNKNOWN` when `department_id` is NULL (NULL comparison returns UNKNOWN, not TRUE), and INNER JOIN keeps only TRUE.

```sql
-- To include employees without departments, use LEFT JOIN:
SELECT e.name, COALESCE(d.name, 'Unassigned') AS department
FROM employees e
LEFT JOIN departments d ON d.id = e.department_id;
```

**Production impact**: Inserting rows with NULL foreign keys (due to a bug or intentional design) and then querying with INNER JOIN silently hides those rows. The query appears to work but returns incomplete data.

### 6.2 Row Multiplication — The One-to-Many Case

Any time a row in the outer table matches multiple rows in the inner table, the outer row appears multiple times in the result.

```sql
-- orders: one per order
-- order_items: many per order
SELECT o.id, o.customer_id, oi.product_id, oi.quantity
FROM orders o
INNER JOIN order_items oi ON oi.order_id = o.id;
-- Result: one row per (order, item) pair — order with 5 items appears 5 times
```

The output row count = sum of matching pairs across all rows. For a table of 10,000 orders, each with an average of 4 items, the result is ~40,000 rows.

**Detecting it**: `COUNT(*)` of the join result vs `COUNT(DISTINCT o.id)` tells you if multiplication is happening.

```sql
SELECT
  COUNT(*) AS total_rows,
  COUNT(DISTINCT o.id) AS distinct_orders
FROM orders o
INNER JOIN order_items oi ON oi.order_id = o.id;
-- If total_rows > distinct_orders, rows are multiplied
```

### 6.3 The Many-to-Many Explosion

Joining two tables where both sides have multiple matches for each key produces a result that grows quadratically.

```sql
-- tags: multiple per article
-- authors: multiple per article (co-authors)
SELECT a.title, t.name AS tag, au.name AS author
FROM articles a
INNER JOIN article_tags    at ON at.article_id = a.id
INNER JOIN tags             t  ON t.id = at.tag_id
INNER JOIN article_authors aa ON aa.article_id = a.id
INNER JOIN authors         au ON au.id = aa.author_id;

-- An article with 5 tags and 3 authors → 15 rows in result
-- An article with 10 tags and 10 authors → 100 rows
```

This is mathematically correct (all combination pairs) but often not what the developer intended. The solution is usually to aggregate tags and authors separately before or after joining.

### 6.4 Equi-Join vs Non-Equi-Join

An **equi-join** uses `=` as the comparison operator. This is by far the most common form.

```sql
-- Equi-join
JOIN orders o ON o.customer_id = c.id
```

A **non-equi-join** (theta join) uses any other comparison: `>`, `<`, `>=`, `<=`, `<>`, `BETWEEN`, `!=`.

```sql
-- Non-equi-join: match transactions to the applicable price tier
SELECT t.id, t.quantity, pt.tier_name, pt.discount_pct
FROM transactions t
INNER JOIN price_tiers pt
  ON t.quantity >= pt.min_qty AND t.quantity <= pt.max_qty;

-- Range join: match each event to its containing time window
SELECT e.id, e.event_name, w.window_name
FROM events e
INNER JOIN time_windows w
  ON e.occurred_at >= w.start_time AND e.occurred_at < w.end_time;
```

**Performance note**: Non-equi-joins typically cannot use a standard B-tree index lookup for both conditions simultaneously. The planner usually falls back to Nested Loop (with one condition as the index access and the other as a filter) or Hash/Merge with post-filter. For large tables, this can be slow — see the performance section.

### 6.5 INNER JOIN Commutativity and Associativity

INNER JOIN is both commutative and associative — this is what allows the planner to freely reorder joins.

```sql
-- Logically equivalent (planner may execute either order):
FROM a JOIN b ON b.a_id = a.id  -- a is outer, b is inner
FROM b JOIN a ON a.id = b.a_id  -- b is outer, a is inner
```

The planner uses statistics to decide which order minimises intermediate row counts. A badly chosen order (joining two large tables before filtering with a small table) is the most common cause of multi-join performance problems.

### 6.6 INNER JOIN with Multiple Conditions

Multiple conditions in ON are all required to be TRUE:

```sql
-- Match on both customer and product within a date range
SELECT c.name, p.name AS product, e.quantity
FROM customers c
INNER JOIN enrollments e
  ON e.customer_id = c.id
  AND e.active = TRUE
  AND e.expires_at > NOW()
INNER JOIN products p ON p.id = e.product_id;
```

All three ON conditions must be TRUE for the row to be included. This is equivalent to:
```sql
WHERE e.customer_id = c.id AND e.active = TRUE AND e.expires_at > NOW()
```
... but as noted earlier, putting non-key conditions in ON vs WHERE makes no difference for INNER JOIN.

### 6.7 INNER JOIN vs Subquery — Equivalence and Divergence

For many patterns, INNER JOIN and a correlated subquery produce the same result:

```sql
-- INNER JOIN form
SELECT DISTINCT c.id, c.name
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id
WHERE o.created_at >= '2025-01-01';

-- Correlated subquery form
SELECT c.id, c.name
FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders o
  WHERE o.customer_id = c.id AND o.created_at >= '2025-01-01'
);
```

The planner often rewrites one into the other (semi-join transformation). The INNER JOIN form returns duplicates if a customer has multiple orders (needs DISTINCT). The EXISTS form never duplicates because it stops at the first match. This is covered fully in Topics 19 and 27.

### 6.8 INNER JOIN with Aggregation — The GROUP BY Requirement

When INNER JOIN multiplies rows (one-to-many), aggregation must account for the fan-out.

```sql
-- WRONG: counts order_items per order, but result looks like order count
SELECT c.id, c.name, COUNT(o.id) AS order_count
FROM customers c
INNER JOIN orders o     ON o.customer_id = c.id
INNER JOIN order_items oi ON oi.order_id = o.id
GROUP BY c.id, c.name;
-- BUG: COUNT(o.id) counts order_items rows, not orders!
-- An order with 5 items is counted 5 times

-- CORRECT: count distinct orders
SELECT c.id, c.name, COUNT(DISTINCT o.id) AS order_count
FROM customers c
INNER JOIN orders o     ON o.customer_id = c.id
INNER JOIN order_items oi ON oi.order_id = o.id
GROUP BY c.id, c.name;
```

Or restructure the query to avoid the fan-out before aggregating:

```sql
-- Aggregate items first, then join
SELECT c.id, c.name, order_summary.order_count, order_summary.total_items
FROM customers c
INNER JOIN (
  SELECT o.customer_id,
    COUNT(DISTINCT o.id) AS order_count,
    COUNT(oi.id) AS total_items
  FROM orders o
  INNER JOIN order_items oi ON oi.order_id = o.id
  GROUP BY o.customer_id
) order_summary ON order_summary.customer_id = c.id;
```

### 6.9 Self-Join with INNER JOIN

A table joined to itself with INNER JOIN — the inner join semantics mean rows with no match in the "other copy" are excluded.

```sql
-- Find employees who earn more than their manager
-- INNER JOIN means: employees without a manager are excluded
SELECT e.name AS employee, e.salary, m.name AS manager, m.salary AS manager_salary
FROM employees e
INNER JOIN employees m ON m.id = e.manager_id
WHERE e.salary > m.salary;
-- Top-level employees (manager_id IS NULL) are excluded
```

### 6.10 INNER JOIN and the Planner's Join Order

For queries with 3+ tables, the planner's join order decision has a major impact on performance. The planner tries to:

1. Join tables with the most selective condition first (smallest intermediate result)
2. Use available indexes to minimise each join cost
3. Avoid building large intermediate hash tables

```sql
-- The planner is free to reorder these joins
FROM orders o
INNER JOIN customers c ON c.id = o.customer_id      -- 1M customers
INNER JOIN products p  ON p.id = oi.product_id       -- 50K products
INNER JOIN order_items oi ON oi.order_id = o.id      -- 20M items
WHERE o.status = 'completed' AND o.created_at > '2025-01-01'
-- Planner may start with the WHERE-filtered orders (most selective),
-- then join items, then customers, then products
```

You can see the chosen order in `EXPLAIN` — the outermost loop is the first table, inner loops are subsequent joins.

---

## 7. EXPLAIN — INNER JOIN Algorithms in the Plan

### Hash Join (large tables, equality join)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, c.name, o.total_amount
FROM orders o
INNER JOIN customers c ON c.id = o.customer_id;
```

```
Hash Join  (cost=35.00..2890.00 rows=30000 width=56)
           (actual time=0.8..28.4 rows=30000 loops=1)
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o  (cost=0.00..1730.00 rows=30000 width=16)
                             (actual time=0.01..8.1 rows=30000 loops=1)
  ->  Hash  (cost=22.00..22.00 rows=1000 width=48)
             (actual time=0.5..0.5 rows=1000 loops=1)
        Buckets: 1024  Batches: 1  Memory Usage: 62kB
        ->  Seq Scan on customers c  (cost=0.00..22.00 rows=1000 width=48)
                                      (actual time=0.01..0.2 rows=1000 loops=1)
Buffers: shared hit=730
Planning Time: 0.3 ms
Execution Time: 28.9 ms
```

**Reading it**:
- `Hash Cond` — the equi-join condition used to build/probe the hash table
- `Hash` node — `customers` is hashed (smaller table → hash side)
- Outer side (`orders`) is scanned and each row probes the hash
- `Batches: 1` — entire hash table fits in `work_mem` (no disk spill)

### Nested Loop Join (small outer, indexed inner)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT o.id, c.name, o.total_amount
FROM orders o
INNER JOIN customers c ON c.id = o.customer_id
WHERE o.customer_id = 42;
```

```
Nested Loop  (cost=0.43..16.5 rows=5 width=56)
             (actual time=0.08..0.14 rows=3 loops=1)
  ->  Index Scan using orders_customer_id_idx on orders o
        Index Cond: (customer_id = 42)
        (actual rows=3 loops=1)
  ->  Index Scan using customers_pkey on customers c
        Index Cond: (id = 42)
        (actual rows=1 loops=3)
Buffers: shared hit=12
Planning Time: 0.2 ms
Execution Time: 0.18 ms
```

**Reading it**:
- Outer: `orders` filtered by `customer_id = 42` → 3 rows
- Inner: for each of those 3 rows, probe `customers` by primary key → `loops=3`
- Very fast — all index lookups, minimal buffers

### Merge Join (pre-sorted or sorted inputs)

```sql
EXPLAIN (ANALYZE)
SELECT e.name, d.name AS dept
FROM employees e
INNER JOIN departments d ON d.id = e.department_id
ORDER BY e.department_id;
```

```
Merge Join  (cost=0.56..520.00 rows=1000 width=64)
            (actual time=0.05..4.2 rows=1000 loops=1)
  Merge Cond: (e.department_id = d.id)
  ->  Index Scan using employees_dept_idx on employees e
        (actual rows=1000 loops=1)
  ->  Index Scan using departments_pkey on departments d
        (actual rows=50 loops=1)
```

**Reading it**:
- Both inputs are already sorted on the join key via indexes
- No sort node needed — merge walks both sorted streams in lockstep
- Very efficient for large sorted inputs

### What to Look For in EXPLAIN

| Symptom | Likely Problem | Fix |
|---------|---------------|-----|
| Nested Loop + Seq Scan inner | No index on inner table's join column | Add index |
| Hash Join with `Batches > 1` | Hash table spilling to disk | Increase `work_mem` |
| Large `estimated rows` vs small `actual rows` | Bad statistics → wrong algorithm | Run `ANALYZE` |
| Sort node before Merge Join | Inputs not pre-sorted | Add index on join key |
| Seq Scan on large table | No usable index | Add appropriate index |

---

## 8. Query Examples

### Example 1 — Basic: Standard Equi-Join

```sql
-- Standard customer-order join — most common production pattern
SELECT
  c.id          AS customer_id,
  c.name        AS customer_name,
  c.email,
  o.id          AS order_id,
  o.total_amount,
  o.status,
  o.created_at  AS order_date
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'completed'
  AND o.created_at >= CURRENT_DATE - INTERVAL '90 days'
ORDER BY o.created_at DESC
LIMIT 100;
```

### Example 2 — Intermediate: Non-Equi-Join (Range Join)

```sql
-- Assign each order to a revenue tier based on total amount
-- price_tiers(id, tier_name, min_amount, max_amount, commission_pct)
SELECT
  o.id          AS order_id,
  o.total_amount,
  pt.tier_name,
  pt.commission_pct,
  ROUND(o.total_amount * pt.commission_pct / 100, 2) AS commission
FROM orders o
INNER JOIN price_tiers pt
  ON o.total_amount >= pt.min_amount
  AND o.total_amount < pt.max_amount
WHERE o.status = 'completed'
ORDER BY o.total_amount DESC;
```

### Example 3 — Production Grade: Multi-Join with Fan-Out Protection

```sql
-- Product performance report: revenue and unique customers
-- Avoids the fan-out aggregation bug by pre-aggregating
WITH order_metrics AS (
  SELECT
    oi.product_id,
    COUNT(DISTINCT oi.order_id)   AS order_count,
    COUNT(DISTINCT o.customer_id) AS unique_customers,
    SUM(oi.quantity)              AS units_sold,
    SUM(oi.quantity * oi.unit_price) AS revenue,
    AVG(oi.unit_price)            AS avg_unit_price
  FROM order_items oi
  INNER JOIN orders o ON o.id = oi.order_id
  WHERE o.status = 'completed'
    AND o.created_at >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '3 months')
  GROUP BY oi.product_id
)
SELECT
  p.id              AS product_id,
  p.name            AS product_name,
  c.name            AS category,
  om.order_count,
  om.unique_customers,
  om.units_sold,
  om.revenue,
  om.avg_unit_price,
  ROUND(om.revenue / NULLIF(om.unique_customers, 0), 2) AS revenue_per_customer
FROM order_metrics om
INNER JOIN products p    ON p.id = om.product_id
INNER JOIN categories c  ON c.id = p.category_id
ORDER BY om.revenue DESC
LIMIT 50;
```

---

## 9. Wrong → Right Patterns

### Wrong 1: Forgetting INNER JOIN drops NULL-key rows

```sql
-- WRONG expectation: see all orders including those with customer_id = NULL
SELECT o.id, c.name
FROM orders o
INNER JOIN customers c ON c.id = o.customer_id;
-- SILENT: orders with NULL customer_id are excluded without error or warning

-- RIGHT: if you need to detect/report orders with missing customer links:
SELECT o.id, c.name
FROM orders o
LEFT JOIN customers c ON c.id = o.customer_id
WHERE c.id IS NULL;  -- these are the orphaned orders
```

### Wrong 2: COUNT on a multiplied column after JOIN

```sql
-- WRONG: counting order_item rows, not orders
SELECT c.name, COUNT(o.id) AS orders_placed
FROM customers c
INNER JOIN orders o     ON o.customer_id = c.id
INNER JOIN order_items oi ON oi.order_id = o.id
GROUP BY c.id, c.name;
-- BUG: each order counted N times (once per item)

-- RIGHT:
SELECT c.name, COUNT(DISTINCT o.id) AS orders_placed
FROM customers c
INNER JOIN orders o     ON o.customer_id = c.id
INNER JOIN order_items oi ON oi.order_id = o.id
GROUP BY c.id, c.name;
```

### Wrong 3: Implicit INNER JOIN with accidental Cartesian product

```sql
-- WRONG: implicit syntax, missing join condition
SELECT c.name, o.total
FROM customers c, orders o
WHERE c.id = 1;  -- forgot to join on customer_id — full cartesian product with filter
-- Produces: customer 1's name × every order in the database

-- RIGHT:
SELECT c.name, o.total
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id
WHERE c.id = 1;
```

### Wrong 4: Non-equi-join producing unexpected multiple matches

```sql
-- WRONG expectation: one price tier per product
SELECT p.name, pt.tier_name
FROM products p
INNER JOIN price_tiers pt ON p.price >= pt.min_price;
-- BUG: if price_tiers has 5 tiers and product price is 100,
-- it might match tiers for min_price = 0, 25, 50, 75, 100 → 5 rows per product

-- RIGHT: each tier is a range [min, max)
SELECT p.name, pt.tier_name
FROM products p
INNER JOIN price_tiers pt
  ON p.price >= pt.min_price AND p.price < pt.max_price;
-- Now each product matches exactly one tier (assuming non-overlapping ranges)
```

### Wrong 5: Using `JOIN` when you should be filtering before joining

```sql
-- WRONG: joining 20M rows then filtering
SELECT c.name, SUM(o.total_amount) AS revenue
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id
WHERE c.signup_date >= '2024-01-01'  -- filter on customers
  AND o.created_at >= '2025-01-01'   -- filter on orders
GROUP BY c.id, c.name;

-- In practice this is fine — the planner will push predicates.
-- But if it doesn't (check EXPLAIN), pre-filter with a subquery:
SELECT c.name, SUM(o.total_amount) AS revenue
FROM (SELECT * FROM customers WHERE signup_date >= '2024-01-01') c
INNER JOIN (SELECT * FROM orders WHERE created_at >= '2025-01-01') o
  ON o.customer_id = c.id
GROUP BY c.id, c.name;
```

---

## 10. Performance Profile

| Algorithm | When Chosen | Memory | Best For |
|-----------|------------|--------|---------|
| Nested Loop | Small outer + indexed inner | O(1) | Point lookups, small result sets |
| Hash Join | Equality join, no useful index | O(smaller table) | Bulk equi-joins |
| Merge Join | Both sides sorted on join key | O(1) | Pre-sorted or indexed inputs |

### Key Performance Numbers

| Scenario | Typical Execution |
|----------|-----------------|
| Nested Loop, indexed inner, 1 outer row | < 1ms (single index probe) |
| Nested Loop, indexed inner, 1K outer rows | ~5–50ms (1K index probes) |
| Nested Loop, NO index on inner, 1K outer rows | Catastrophic — avoid |
| Hash Join, 100K × 10K, fits in work_mem | ~50–200ms |
| Hash Join with disk spill (Batches > 1) | 5–10× slower |
| Merge Join, both pre-sorted | ~50–200ms |

### The Index-on-Join-Column Rule

For INNER JOIN with Nested Loop:
- The **inner table** (the one probed in the inner loop) needs an index on the join column
- The **outer table** is scanned or filtered — an index helps if there is also a WHERE clause

```sql
-- For this query:
SELECT * FROM orders o INNER JOIN customers c ON c.id = o.customer_id WHERE o.status = 'new';

-- Needed:
CREATE INDEX ON customers(id);        -- the PK already exists — good
CREATE INDEX ON orders(status);       -- for the WHERE clause
CREATE INDEX ON orders(customer_id);  -- for the join if orders is the inner table
```

The planner may choose to scan `orders` filtered by `status` (outer) and probe `customers` by PK (inner) → only the PK index matters. But if it reverses the order, the `orders.customer_id` index is needed.

---

## 11. Node.js Integration

### 11.1 Basic INNER JOIN

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function getCompletedOrdersWithCustomers(daysBack = 90) {
  const { rows } = await pool.query(
    `SELECT
       c.id          AS customer_id,
       c.name        AS customer_name,
       o.id          AS order_id,
       o.total_amount,
       o.created_at  AS order_date
     FROM customers c
     INNER JOIN orders o ON o.customer_id = c.id
     WHERE o.status = 'completed'
       AND o.created_at >= NOW() - ($1 || ' days')::INTERVAL
     ORDER BY o.created_at DESC`,
    [daysBack]
  );
  return rows;
}
```

### 11.2 Detecting fan-out with INNER JOIN

```javascript
// Safe aggregation — pre-aggregate before joining to avoid fan-out count bugs
async function getCustomerOrderStats(customerId) {
  const { rows } = await pool.query(
    `SELECT
       c.id,
       c.name,
       c.email,
       stats.order_count,
       stats.total_spend,
       stats.avg_order_value,
       stats.last_order_at
     FROM customers c
     INNER JOIN (
       SELECT
         customer_id,
         COUNT(*)            AS order_count,
         SUM(total_amount)   AS total_spend,
         AVG(total_amount)   AS avg_order_value,
         MAX(created_at)     AS last_order_at
       FROM orders
       WHERE status = 'completed'
       GROUP BY customer_id
     ) stats ON stats.customer_id = c.id
     WHERE c.id = $1`,
    [customerId]
  );
  return rows[0] ?? null;  // null if customer has no completed orders
}
```

### 11.3 Non-equi-join for range matching

```javascript
async function classifyTransactions(userId) {
  const { rows } = await pool.query(
    `SELECT
       t.id,
       t.amount,
       t.created_at,
       rt.label       AS risk_label,
       rt.review_required
     FROM transactions t
     INNER JOIN risk_thresholds rt
       ON t.amount >= rt.min_amount
       AND t.amount < rt.max_amount
       AND rt.currency = t.currency
     WHERE t.user_id = $1
       AND t.created_at >= NOW() - INTERVAL '30 days'
     ORDER BY t.created_at DESC`,
    [userId]
  );
  return rows;
}
```

### 11.4 Handling the NULL-exclusion behaviour

```javascript
// Explicitly surface orphaned records that INNER JOIN would hide
async function auditOrphanedOrders() {
  const { rows } = await pool.query(
    `SELECT o.id, o.customer_id, o.total_amount, o.created_at
     FROM orders o
     LEFT JOIN customers c ON c.id = o.customer_id
     WHERE c.id IS NULL  -- these are orders INNER JOIN would silently exclude
     ORDER BY o.created_at DESC
     LIMIT 100`
  );
  return rows;
}
```

---

## 12. ORM Comparison

### Prisma

**Can Prisma do INNER JOIN?** — Yes, via `include` with `required: true` semantics (by default Prisma uses JOINs or separate queries depending on strategy). Prisma generates `INNER JOIN` for required relations.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// Prisma uses INNER JOIN for findMany with include on required (non-nullable) relations
const orders = await prisma.order.findMany({
  where: {
    status: 'completed',
    createdAt: { gte: new Date('2025-01-01') },
  },
  include: {
    customer: true,          // generates INNER JOIN if relation is required
    orderItems: {
      include: { product: true },
    },
  },
  orderBy: { createdAt: 'desc' },
  take: 100,
});

// Aggregate query — requires raw SQL (Prisma can't express COUNT(DISTINCT))
const stats = await prisma.$queryRaw`
  SELECT
    c.id,
    c.name,
    COUNT(DISTINCT o.id)        AS order_count,
    SUM(o.total_amount)         AS total_spend,
    MAX(o.created_at)           AS last_order_at
  FROM customers c
  INNER JOIN orders o ON o.customer_id = c.id
  WHERE o.status = 'completed'
  GROUP BY c.id, c.name
  ORDER BY total_spend DESC
  LIMIT 20
`;
```

**Where it breaks**: Prisma does not allow you to explicitly specify `INNER JOIN` vs `LEFT JOIN` on individual relations — it infers this from whether the foreign key is nullable. For non-nullable FKs it generates INNER JOIN; for nullable FKs it generates LEFT JOIN. You cannot override this without raw SQL.

**Verdict**: For standard relation-based queries, Prisma's join inference works well. For analytics, aggregations, or non-equi-joins, use `$queryRaw`.

---

### Drizzle ORM

**Can Drizzle do INNER JOIN?** — Yes, fully typed.

```typescript
import { db } from './db';
import { customers, orders, orderItems, products, categories } from './schema';
import { eq, gte, and, sql } from 'drizzle-orm';

// Simple INNER JOIN
const completedOrders = await db
  .select({
    customerId: customers.id,
    customerName: customers.name,
    orderId: orders.id,
    totalAmount: orders.totalAmount,
    orderDate: orders.createdAt,
  })
  .from(customers)
  .innerJoin(orders, eq(orders.customerId, customers.id))
  .where(
    and(
      eq(orders.status, 'completed'),
      gte(orders.createdAt, new Date('2025-01-01'))
    )
  )
  .orderBy(orders.createdAt)
  .limit(100);

// Multi-join aggregation
const productStats = await db
  .select({
    productId: products.id,
    productName: products.name,
    categoryName: categories.name,
    orderCount: sql<number>`COUNT(DISTINCT ${orderItems.orderId})`,
    revenue: sql<number>`SUM(${orderItems.quantity} * ${orderItems.unitPrice})`,
  })
  .from(orderItems)
  .innerJoin(orders, eq(orders.id, orderItems.orderId))
  .innerJoin(products, eq(products.id, orderItems.productId))
  .innerJoin(categories, eq(categories.id, products.categoryId))
  .where(eq(orders.status, 'completed'))
  .groupBy(products.id, products.name, categories.name)
  .orderBy(sql`revenue DESC`);
```

**Where it breaks**: Non-equi-joins require the `sql` template tag for the ON condition. Drizzle's type inference works perfectly for equi-joins on typed columns.

**Verdict**: Drizzle is excellent for INNER JOIN — full type inference, readable syntax, close to SQL. Use it confidently.

---

### Sequelize

**Can Sequelize do INNER JOIN?** — Yes, via `include` with `required: true`. Default is LEFT JOIN.

```javascript
const { Op } = require('sequelize');
const { Order, Customer, OrderItem, Product } = require('./models');

// INNER JOIN — must specify required: true
const orders = await Order.findAll({
  where: {
    status: 'completed',
    createdAt: { [Op.gte]: new Date('2025-01-01') },
  },
  include: [
    {
      model: Customer,
      required: true,          // ← CRITICAL: makes it INNER JOIN, not LEFT JOIN
      attributes: ['name', 'email'],
    },
  ],
  order: [['createdAt', 'DESC']],
  limit: 100,
});

// Aggregate with INNER JOIN — use raw query
const stats = await sequelize.query(
  `SELECT c.id, c.name,
          COUNT(DISTINCT o.id) AS order_count,
          SUM(o.total_amount) AS total_spend
   FROM customers c
   INNER JOIN orders o ON o.customer_id = c.id AND o.status = 'completed'
   GROUP BY c.id, c.name
   ORDER BY total_spend DESC
   LIMIT 20`,
  { type: QueryTypes.SELECT }
);
```

**Where it breaks**: Sequelize's default `include` generates `LEFT OUTER JOIN`, silently including NULLs unless `required: true` is set. This is the most common Sequelize JOIN bug — forgetting `required: true` when you intended INNER JOIN semantics.

**Verdict**: Always explicitly set `required: true` when you want INNER JOIN. For any complex aggregation, use `sequelize.query()`.

---

### TypeORM

**Can TypeORM do INNER JOIN?** — Yes, via `innerJoinAndSelect` or `innerJoin`.

```typescript
import { DataSource } from 'typeorm';
import { Order } from './entities/Order';

const dataSource = new DataSource({ /* config */ });

// INNER JOIN loading related entities
const orders = await dataSource
  .getRepository(Order)
  .createQueryBuilder('o')
  .innerJoinAndSelect('o.customer', 'c')     // INNER JOIN + loads customer
  .where('o.status = :status', { status: 'completed' })
  .andWhere('o.createdAt >= :since', { since: new Date('2025-01-01') })
  .orderBy('o.createdAt', 'DESC')
  .limit(100)
  .getMany();

// INNER JOIN without selecting related entity (filter-only)
const highValueOrders = await dataSource
  .getRepository(Order)
  .createQueryBuilder('o')
  .innerJoin('o.customer', 'c')              // INNER JOIN without select
  .where('c.tier = :tier', { tier: 'premium' })
  .andWhere('o.status = :status', { status: 'completed' })
  .getMany();

// Aggregation with INNER JOIN
const stats = await dataSource
  .createQueryBuilder()
  .select('c.id', 'customerId')
  .addSelect('c.name', 'customerName')
  .addSelect('COUNT(DISTINCT o.id)', 'orderCount')
  .addSelect('SUM(o.totalAmount)', 'totalSpend')
  .from('customers', 'c')
  .innerJoin('orders', 'o', 'o.customer_id = c.id AND o.status = :status', { status: 'completed' })
  .groupBy('c.id')
  .addGroupBy('c.name')
  .orderBy('totalSpend', 'DESC')
  .limit(20)
  .getRawMany();
```

**Where it breaks**: `innerJoinAndSelect` loads entities and applies entity mapping; `innerJoin` (without Select) is filter-only. Mixing them incorrectly leads to either missing data or N+1 queries. Raw string join conditions in the third argument to `innerJoin` have no type safety.

**Verdict**: TypeORM QueryBuilder handles INNER JOIN well. Use `innerJoinAndSelect` for entity loading, `innerJoin` for filter-only conditions. Switch to `dataSource.query()` for complex analytics.

---

### Knex.js

**Can Knex do INNER JOIN?** — Yes, fully, with `.join()` (defaults to INNER JOIN).

```javascript
const knex = require('knex')({ client: 'pg', connection: process.env.DATABASE_URL });

// Simple INNER JOIN (.join() = INNER JOIN by default)
const orders = await knex('customers as c')
  .join('orders as o', 'o.customer_id', 'c.id')
  .select('c.id as customer_id', 'c.name', 'o.id as order_id', 'o.total_amount')
  .where('o.status', 'completed')
  .where('o.created_at', '>=', new Date('2025-01-01'))
  .orderBy('o.created_at', 'desc')
  .limit(100);

// Multi-table INNER JOIN with aggregation
const productStats = await knex('order_items as oi')
  .join('orders as o',     'o.id',          'oi.order_id')
  .join('products as p',   'p.id',          'oi.product_id')
  .join('categories as c', 'c.id',          'p.category_id')
  .where('o.status', 'completed')
  .select(
    'p.id as product_id',
    'p.name as product_name',
    'c.name as category',
    knex.raw('COUNT(DISTINCT oi.order_id) AS order_count'),
    knex.raw('SUM(oi.quantity * oi.unit_price) AS revenue')
  )
  .groupBy('p.id', 'p.name', 'c.id', 'c.name')
  .orderByRaw('revenue DESC')
  .limit(50);

// Non-equi-join (range join) — requires function form
const classified = await knex('transactions as t')
  .join('risk_thresholds as rt', function() {
    this.on('t.amount', '>=', 't.amount')  // placeholder — use raw for range
  });
// Better for non-equi-join: use knex.raw for the ON clause
const classified2 = await knex('transactions as t')
  .join(
    knex.raw('risk_thresholds rt ON t.amount >= rt.min_amount AND t.amount < rt.max_amount AND rt.currency = t.currency')
  )
  .where('t.user_id', userId)
  .select('t.*', 'rt.label', 'rt.review_required');
```

**Where it breaks**: Complex ON conditions with multiple comparisons or non-equi-joins are awkward with the function form of `.join()`. The `knex.raw()` approach works but loses the structured query builder benefit.

**Verdict**: Knex is the most SQL-transparent ORM. `.join()` defaulting to INNER JOIN is correct and explicit. Use `knex.raw()` in the join argument for non-equi-join conditions.

---

### ORM Summary Table

| ORM | INNER JOIN Method | Default JOIN Type | Aggregate + JOIN | Non-Equi-Join | Verdict |
|-----|------------------|------------------|-----------------|--------------|---------|
| Prisma | `include` (inferred) | Inferred from FK nullability | Raw SQL | Raw SQL | Auto-inference works; raw for analytics |
| Drizzle | `.innerJoin()` | Must specify | `sql` template | `sql` template | Best typed support |
| Sequelize | `include` + `required: true` | LEFT JOIN (dangerous default!) | `sequelize.query()` | `literal()` | Always set `required: true` |
| TypeORM | `innerJoinAndSelect()` | N/A (explicit) | `getRawMany()` | Raw string | Explicit and clear |
| Knex | `.join()` | INNER JOIN | `knex.raw()` aggregates | `knex.raw()` in ON | Most transparent |

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given:
- `employees(id, name, salary, department_id, job_title)`
- `departments(id, name, budget, location)`

Write a query returning every employee who is in a department — `name`, `salary`, `job_title`, `department name`, `department location`. Only include employees who have a department assigned. Order by department name, then salary descending within each department.

Then: how many employees are excluded because they have no department? Write a second query to find out.

---

### Exercise 2 — Intermediate

Given:
- `orders(id, customer_id, total_amount, status, created_at)`
- `order_items(id, order_id, product_id, quantity, unit_price)`
- `products(id, name, category_id, cost_price)`
- `categories(id, name)`

Write a query returning, for each category, for completed orders in the last 6 months:
- `category_name`
- `total_orders` — count of **distinct orders** (not order items)
- `units_sold` — total quantity sold
- `gross_revenue` — sum of `quantity × unit_price`
- `total_cost` — sum of `quantity × cost_price`
- `gross_margin_pct` — `(gross_revenue - total_cost) / gross_revenue * 100`, rounded to 2dp

Only include categories that had at least 10 distinct orders. Order by `gross_margin_pct` descending.

---

### Exercise 3 — Intermediate

Given a `transactions(id, user_id, amount, currency, created_at)` table and a `risk_bands(id, label, min_amount, max_amount, currency, requires_review)` table where the bands are non-overlapping ranges per currency.

Write a non-equi-join query that:
- Assigns every transaction to its risk band
- Returns `transaction id`, `amount`, `currency`, `risk label`, `requires_review`
- Handles transactions that fall outside all defined bands (they should still appear, with `risk_label = 'undefined'` and `requires_review = TRUE`)
- Orders by `amount DESC`

Hint: what join type do you need when some rows might not match any band?

---

### Exercise 4 — Production Grade

A developer wrote this query to get the "number of products ordered per customer":

```sql
SELECT c.id, c.name, COUNT(p.id) AS products_ordered
FROM customers c
INNER JOIN orders o     ON o.customer_id = c.id
INNER JOIN order_items oi ON oi.order_id = o.id
INNER JOIN products p   ON p.id = oi.product_id
WHERE o.status = 'completed'
GROUP BY c.id, c.name
ORDER BY products_ordered DESC;
```

1. Identify all bugs in this query (there are at least 2).
2. Write the correct version that returns the **number of distinct products** each customer has ever ordered (in completed orders).
3. Extend it to also return:
   - `total_orders` (distinct completed orders)
   - `total_spend` (sum of order totals, not item prices)
   - `favourite_category` (the category name with the most distinct products ordered)
4. What index would you add to make this query fast on a 10M-row `order_items` table?

---

## 14. Interview Questions

### Junior Level

**Q: What rows does an INNER JOIN return?**

A: Only the rows where the ON condition is TRUE in both tables. Rows from either table that have no matching row in the other table are excluded. If one row matches multiple rows in the other table, it appears multiple times in the result.

---

**Q: I added an INNER JOIN to my query and now I'm getting fewer rows than I expected. What could cause this?**

A: Several causes:
1. The join key is NULL on some rows — NULL comparisons evaluate to UNKNOWN (not TRUE), so those rows are excluded.
2. The join condition is wrong — a typo or logic error means rows that should match don't.
3. The related records don't exist — some rows in the left table genuinely have no match in the right table, and INNER JOIN excludes them by definition. Use LEFT JOIN if you want to keep those rows.

---

**Q: What is the difference between `JOIN` and `INNER JOIN` in SQL?**

A: They are identical. `JOIN` without a type keyword defaults to `INNER JOIN`. Both return only matched rows. Writing `INNER JOIN` is more explicit and is preferred in production code for clarity, but functionally there is no difference.

---

### Principal Level

**Q: You have a query that joins a 50M-row events table to a 10K-row reference table. The query takes 30 seconds. EXPLAIN shows a Nested Loop Join with Seq Scan on the inner table. What's the problem and how do you fix it?**

A: The inner table (events or the reference table) doesn't have an index on the join column. With Nested Loop Join, the outer table drives the loop — for each row in the outer table, the inner table is scanned to find matches. Without an index, each inner scan is a full sequential scan. For 50M outer rows × any-row-count sequential scan = catastrophic. Fix: add an index on the join column of the inner table. The Nested Loop then becomes O(50M × O(log 10K)) = O(50M × 14) which is manageable, or the planner may switch to Hash Join which is O(N + M) with O(10K) memory. After adding the index, run `ANALYZE` and re-examine the plan — the planner may now prefer Hash Join for the full table scan case anyway.

---

**Q: When is Merge Join the fastest algorithm for an INNER JOIN, and what are its prerequisites?**

A: Merge Join is fastest when both inputs are already sorted on the join key — it scans both sorted streams simultaneously in O(N + M) without any hash table overhead. Prerequisites: (1) both tables must be sorted on the join key — either naturally (e.g., sequential IDs, index scan output) or via explicit Sort nodes; (2) the join must be an equi-join (Merge Join requires equality comparison). The advantage over Hash Join: Merge Join doesn't require memory proportional to either table size, so it works well for very large tables where `work_mem` is a constraint. The disadvantage: if neither side is pre-sorted, Sort nodes add O(N log N) overhead, which may be worse than Hash Join.

---

**Q: A query joins customers to orders in a one-to-many relationship, then aggregates. A junior developer asks why `COUNT(o.id)` gives a different result than `COUNT(DISTINCT o.id)`. Explain what's happening.**

A: When you JOIN to a one-to-many table, each row in the "one" side appears multiple times — once per matching row in the "many" side. If a customer has 5 orders and each order has 3 items, the customer appears 15 times in the JOIN result. `COUNT(o.id)` counts all 15 occurrences (including the order ID appearing 3 times per order). `COUNT(DISTINCT o.id)` counts unique values — correctly gives 5. The root cause is that joining to `order_items` multiplies each order row by its item count. The fix is either `COUNT(DISTINCT o.id)` or restructuring the query to pre-aggregate before joining (using a subquery or CTE) — which is often more efficient because it avoids the fan-out entirely.

---

## 15. Mental Model Checkpoint

1. **Table A has 1,000 rows. Table B has 5 rows. Every row in B matches exactly one row in A. How many rows does `A INNER JOIN B ON A.id = B.a_id` return?**

2. **A row in A has `foreign_key = NULL`. What does INNER JOIN do with this row? Why?**

3. **You join orders (100K rows) to order_items (500K rows). Orders have an average of 5 items each. How many rows does the INNER JOIN produce before any WHERE filtering?**

4. **EXPLAIN shows `Nested Loop` with `loops=50000` on the inner table. Is this a problem? What determines whether it's fast or slow?**

5. **You move a condition from `ON` to `WHERE` in an INNER JOIN query. Does the result change? Does performance change?**

6. **What is the danger of joining a table to itself (self-join) with INNER JOIN when some rows have a NULL in the join column?**

7. **Sequelize's `include` generates LEFT JOIN by default. You want INNER JOIN. What do you need to add? What happens if you forget?**

---

## 16. Quick Reference Card

```sql
-- Syntax (INNER is default)
SELECT ...
FROM table_a a
[INNER] JOIN table_b b ON a.key = b.key

-- NULL behaviour: rows with NULL join key are EXCLUDED
-- To find excluded rows:
SELECT * FROM a LEFT JOIN b ON b.key = a.key WHERE b.key IS NULL

-- Row multiplication: one-to-many produces multiple rows
-- Detect: COUNT(*) vs COUNT(DISTINCT a.id)
SELECT COUNT(*), COUNT(DISTINCT a.id) FROM a JOIN b ON b.a_id = a.id

-- Safe aggregation after one-to-many join:
COUNT(DISTINCT o.id)           -- count distinct orders, not rows
SUM(DISTINCT o.total_amount)   -- use sparingly; COUNT(DISTINCT) is more common fix

-- Pre-aggregate to avoid fan-out:
FROM a JOIN (SELECT a_id, COUNT(*) AS n FROM b GROUP BY a_id) agg ON agg.a_id = a.id

-- Non-equi-join (range join)
JOIN tiers t ON value >= t.min_val AND value < t.max_val

-- Multi-condition ON
JOIN b ON b.a_id = a.id AND b.active = TRUE AND b.expires_at > NOW()

-- EXPLAIN signals to watch:
-- Nested Loop + Seq Scan inner → missing index on inner join column
-- Hash Join Batches > 1       → work_mem too small, spilling to disk
-- Large estimated/actual gap  → stale statistics, run ANALYZE

-- ORM pitfall (Sequelize):
include: [{ model: B, required: true }]  -- required:true = INNER JOIN
include: [{ model: B }]                  -- default = LEFT OUTER JOIN (danger!)
```

---

## Connected Topics

- **Topic 10 — JOIN Fundamentals**: The cross-product mental model that explains all INNER JOIN behaviour.
- **Topic 12 — LEFT JOIN and RIGHT JOIN**: The critical difference when outer join semantics are needed; the ON vs WHERE trap.
- **Topic 17 — JOIN Performance Deep Dive**: Full treatment of Nested Loop, Hash Join, Merge Join — when each is used, how to fix the wrong choice.
- **Topic 18 — The N+1 Query Problem**: What happens when you avoid INNER JOIN in ORMs — the query-per-row trap.
- **Topic 19 — JOIN Alternatives**: EXISTS as an alternative to INNER JOIN for existence checks; when each is faster.
- **Topic 20 — GROUP BY Fundamentals**: After an INNER JOIN, GROUP BY collapses the multiplied rows — the two topics are inseparable in practice.
- **Topic 27 — EXISTS and NOT EXISTS**: The semi-join alternative to INNER JOIN that avoids row multiplication.
- **Topic 07 — NULL in Depth**: Why NULL join keys are silently excluded — the three-valued logic at work in ON conditions.
