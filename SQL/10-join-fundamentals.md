# Topic 10 — JOIN Fundamentals and Mental Model
### SQL Mastery Curriculum — Phase 3: JOINs — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine two filing cabinets. One holds customer folders. One holds order folders. Each order folder has a "customer ID" slip inside it.

A **JOIN** is you physically pulling every combination of folders from both cabinets onto a table — every customer folder paired with every order folder — and then going through the pile and throwing away every pair where the customer ID slips don't match.

What's left on the table is your result.

This sounds wasteful. And it is, conceptually. But that's exactly what a JOIN is at its logical heart: **a cross product followed by a filter**. The database engine doesn't literally do this (it uses smarter algorithms), but the *result* is identical to if it had. Understanding this mental model explains every confusing JOIN behaviour you will ever encounter.

---

## 2. Connection to SQL Internals

Every JOIN in SQL — regardless of type — is defined in terms of two operations:

1. **The Cartesian product** (cross join): pair every row from table A with every row from table B → produces `|A| × |B|` rows
2. **The ON predicate**: filter down to only rows where the condition is TRUE

```
INNER JOIN  = Cartesian product → keep rows where ON = TRUE
LEFT JOIN   = Cartesian product → keep rows where ON = TRUE
              + rows from left with no match (right columns = NULL)
RIGHT JOIN  = symmetric of LEFT JOIN
FULL JOIN   = keep all matched + all unmatched from both sides
CROSS JOIN  = Cartesian product with no filter (all pairs kept)
```

The planner knows this equivalence and uses it to choose the actual execution algorithm:

- **Nested Loop Join**: for each row in outer table, probe inner table — effectively implements the cross product with early termination
- **Hash Join**: build a hash table on the smaller side, probe with rows from the larger side
- **Merge Join**: sort both sides on the join key, scan both in lockstep

The algorithm chosen changes the *performance*, never the *result*. The logical model is always "cross product → filter."

---

## 3. Logical Execution Order Context

```
FROM table_A
JOIN table_B ON condition   ← JOIN evaluated here, in FROM phase
WHERE ...                   ← filters rows AFTER the join is built
GROUP BY ...
HAVING ...
SELECT ...
ORDER BY ...
```

The critical point: **JOINs execute in the FROM phase**, before WHERE, before GROUP BY, before SELECT. Multiple JOINs in the FROM clause are evaluated together as part of building the "working set" of rows that the rest of the query operates on.

This means:
- A condition in `ON` filters during JOIN construction
- A condition in `WHERE` filters after JOIN construction
- For INNER JOIN, these are logically equivalent — for OUTER JOIN, they are **not**

```sql
-- These produce DIFFERENT results for LEFT JOIN:
-- Version A: filter in ON
SELECT * FROM orders o
LEFT JOIN customers c ON c.id = o.customer_id AND c.country = 'US';

-- Version B: filter in WHERE
SELECT * FROM orders o
LEFT JOIN customers c ON c.id = o.customer_id
WHERE c.country = 'US';
```

Version A: keeps all orders; rows with non-US customers have NULL in customer columns.  
Version B: converts the LEFT JOIN into an INNER JOIN — rows with non-US customers are eliminated by WHERE.

This is the most common LEFT JOIN bug in production. It is covered exhaustively in Topic 12.

---

## 4. What Is a JOIN?

A JOIN combines rows from two (or more) tables based on a related column. The syntax:

```sql
SELECT ...
FROM table_a [AS alias_a]
[JOIN_TYPE] JOIN table_b [AS alias_b] ON join_condition
```

### Join Types at a Glance

```
Table A:          Table B:
┌──────┐          ┌──────┐
│  1   │          │  1   │
│  2   │          │  2   │
│  3   │          │  4   │
└──────┘          └──────┘
```

| Join Type | Rows Returned |
|-----------|--------------|
| INNER JOIN | 1, 2 (matched rows only) |
| LEFT JOIN | 1, 2, 3 (all from A; 3 has NULLs from B) |
| RIGHT JOIN | 1, 2, 4 (all from B; 4 has NULLs from A) |
| FULL OUTER JOIN | 1, 2, 3, 4 (all rows; unmatched get NULLs) |
| CROSS JOIN | Every A row × every B row = 3 × 3 = 9 rows |

---

## 5. Why JOIN Fundamentals Matter in Production

Understanding the cross-product mental model unlocks debugging superpowers:

1. **Explaining duplicate rows**: If a JOIN produces more rows than expected, a row in one table matched multiple rows in the other. The cross product model makes this obvious.

2. **Diagnosing missing rows**: If a JOIN produces fewer rows than expected, the ON condition eliminated them. For outer joins — was the filter in ON or WHERE?

3. **Understanding performance**: A JOIN of a 1M-row table with a 10K-row table starts conceptually as 10 billion rows before filtering. Whether the planner can reduce this early (via index probe) is the entire story of JOIN performance.

4. **Predicting planner choices**: The planner estimates the output cardinality of a JOIN using statistics. Wrong estimates → wrong algorithm → slow query. Understanding what the planner is doing starts with understanding what a JOIN logically produces.

5. **N+1 prevention**: The N+1 problem (Topic 18) is directly caused by not JOINing — replacing one proper JOIN with N+1 round-trip queries. Internalising the JOIN model helps you see when you need one.

---

## 6. Deep Technical Content

### 6.1 The Cross Product Foundation

The Cartesian product is the theoretical basis for all JOINs. Understanding it prevents all "where did these extra rows come from?" confusion.

```sql
-- Explicit CROSS JOIN — 3 rows × 3 rows = 9 rows
SELECT a.val AS a_val, b.val AS b_val
FROM (VALUES (1), (2), (3)) AS a(val)
CROSS JOIN (VALUES ('x'), ('y'), ('z')) AS b(val);
```

```
 a_val | b_val
-------+-------
     1 | x
     1 | y
     1 | z
     2 | x
     2 | y
     2 | z
     3 | x
     3 | y
     3 | z
```

Every INNER JOIN is logically this followed by a filter:

```sql
-- Logically equivalent to:
SELECT a.val AS a_val, b.val AS b_val
FROM (VALUES (1), (2), (3)) AS a(val)
CROSS JOIN (VALUES (1), (2), (4)) AS b(val)
WHERE a.val = b.val;  -- this is the ON condition
```

### 6.2 The ON Clause — What It Really Does

`ON` is a boolean predicate applied to each pair of rows from the cross product. Any valid SQL boolean expression can be an ON condition.

```sql
-- Equality join (most common — equi-join)
JOIN orders o ON o.customer_id = c.id

-- Inequality join (range join, theta join)
JOIN price_tiers pt ON pt.min_qty <= ol.quantity AND pt.max_qty >= ol.quantity

-- Multiple conditions
JOIN assignments a ON a.employee_id = e.id AND a.end_date IS NULL

-- Join on expression (avoid when possible — non-sargable)
JOIN logs l ON DATE_TRUNC('day', l.created_at) = DATE_TRUNC('day', e.event_at)
```

**Important**: `ON` is evaluated per-pair during join construction. `WHERE` is evaluated after the full join. For INNER JOIN, moving conditions between ON and WHERE produces the same result (the planner can push predicates). For OUTER JOIN, it changes results.

### 6.3 Why Outer Joins Keep Unmatched Rows

The mechanism:
1. Start with the cross product
2. Apply the ON filter
3. Check: are there rows from the "preserved" side that had no match?
4. If yes, synthesise rows: all columns from the preserved side keep their values; all columns from the "discarded" side become NULL

```sql
-- Customers with and without orders
SELECT c.id, c.name, o.id AS order_id
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id;
```

```
 id  | name    | order_id
-----+---------+---------
  1  | Alice   |   101
  1  | Alice   |   102
  2  | Bob     |   103
  3  | Charlie |   NULL    ← Charlie has no orders; order_id is NULL
```

Charlie produces one synthesised row. If Charlie had two orders, he would appear twice (once per order). If Charlie has no orders, he appears exactly once with NULLs.

### 6.4 The Duplicate Row Trap

One of the most common production bugs: unexpected row multiplication.

**Scenario**: You expect one row per customer but get more.

```sql
-- This query might return more rows than expected
SELECT c.id, c.name, t.tag_name
FROM customers c
JOIN customer_tags t ON t.customer_id = c.id;
```

If customer 1 has 3 tags, customer 1 appears 3 times. This is correct behaviour — it's the cross product at work. But developers expecting one row per customer are surprised.

**Diagnosis**: The number of output rows equals the number of matching pairs across all rows. One-to-many JOINs inherently multiply rows.

**Fix options**:
- Accept the multiplication (it's correct for many use cases)
- Use `STRING_AGG` or `ARRAY_AGG` with `GROUP BY` to collapse
- Use a subquery to aggregate before joining
- Reconsider whether a JOIN is the right tool

```sql
-- Collapsed: one row per customer, tags as array
SELECT c.id, c.name, ARRAY_AGG(t.tag_name) AS tags
FROM customers c
JOIN customer_tags t ON t.customer_id = c.id
GROUP BY c.id, c.name;
```

### 6.5 JOIN on Non-Key Columns

JOINs don't require foreign keys or even indexed columns. They work on any condition. But the performance characteristics change dramatically.

```sql
-- Join on an unindexed column — sequential scan + nested loop or hash join
SELECT e.name, d.name AS dept
FROM employees e
JOIN departments d ON d.name = e.department_name;  -- string match, no FK

-- Join on a range — requires range-aware algorithm (nested loop with index, or merge)
SELECT p.product_id, t.tier_name
FROM purchases p
JOIN price_tiers t ON p.quantity BETWEEN t.min_qty AND t.max_qty;
```

PostgreSQL will find the best algorithm given available indexes and statistics. But absence of an index on the join column is the most common cause of slow JOINs.

### 6.6 Self-Referential Joins

A table can join to itself. This requires aliases to distinguish the two "instances."

```sql
-- Find all employees and their manager's name
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id;
```

The planner sees this as two separate logical table instances even though they're the same physical table. Index access on `e.manager_id` works exactly as it would for a two-table join.

### 6.7 USING Clause — Shorthand for Equi-Joins

When the join columns have the same name in both tables, `USING` can replace `ON`:

```sql
-- ON syntax
SELECT * FROM orders o JOIN customers c ON c.id = o.customer_id;

-- USING syntax (only works when column names match)
SELECT * FROM orders o JOIN customers c USING (customer_id);
```

`USING` also deduplicates the join column in `SELECT *` — only one `customer_id` column appears, not two. This is a semantic difference from `ON`.

**Pitfall**: `USING` column deduplication means you cannot qualify the column with a table alias in the SELECT. Prefer `ON` in production code for clarity.

### 6.8 NATURAL JOIN — The Footgun

`NATURAL JOIN` automatically joins on all columns with matching names. **Never use this in production code.**

```sql
-- DANGER: joins on ALL matching column names
SELECT * FROM orders NATURAL JOIN customers;
-- If both have 'created_at', it joins on customer_id AND created_at
-- Silent bug if column names drift or a new column is added
```

`NATURAL JOIN` is fragile — adding a column to either table that happens to match a column name in the other table silently changes query semantics. Always use explicit `ON`.

### 6.9 Implicit vs Explicit JOIN Syntax

Old SQL used implicit join syntax (comma-separated FROM with WHERE conditions):

```sql
-- OLD implicit syntax (SQL-89 style) — avoid
SELECT c.name, o.total
FROM customers c, orders o
WHERE c.id = o.customer_id
  AND o.total > 100;
```

Modern explicit syntax is always preferred:

```sql
-- MODERN explicit syntax — use this
SELECT c.name, o.total
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE o.total > 100;
```

The implicit syntax:
- Makes it easy to accidentally produce a Cartesian product (forget the WHERE condition)
- Cannot express OUTER JOINs cleanly
- Is harder to read when multiple tables are joined
- The query engine treats them identically, but explicit syntax is unambiguous

### 6.10 Join Order in the Query vs Join Order in the Plan

The order you write JOINs in the query is **not** the order the planner executes them. The planner is free to reorder joins to minimise cost.

```sql
-- You write this order
FROM a JOIN b ON ... JOIN c ON ... JOIN d ON ...

-- Planner may execute: a→c→d→b if that's cheaper
```

You can see the planner's chosen order in `EXPLAIN`. For queries with more than 8 tables, PostgreSQL uses a genetic algorithm (GEQO) to find a good join order because exhaustive search is too expensive.

This matters when:
- The planner makes a bad estimate and chooses the wrong join order
- You have join_collapse_limit or from_collapse_limit set
- You're debugging a slow multi-join query

---

## 7. EXPLAIN — What Joins Look Like in the Plan

```sql
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT c.name, COUNT(o.id) AS order_count
FROM customers c
JOIN orders o ON o.customer_id = c.id
GROUP BY c.id, c.name;
```

```
HashAggregate  (cost=2840.00..2940.00 rows=1000 width=40)
               (actual time=28.4..31.2 rows=987 loops=1)
  Group Key: c.id, c.name
  ->  Hash Join  (cost=55.00..2690.00 rows=30000 width=36)
                 (actual time=0.8..22.1 rows=30000 loops=1)
        Hash Cond: (o.customer_id = c.id)
        ->  Seq Scan on orders o  (cost=0.00..1730.00 rows=30000 width=8)
                                  (actual time=0.01..8.2 rows=30000 loops=1)
        ->  Hash  (cost=42.00..42.00 rows=1000 width=36)
                  (actual time=0.6..0.6 rows=1000 loops=1)
              Buckets: 1024  Batches: 1  Memory Usage: 62kB
              ->  Seq Scan on customers c  (cost=0.00..42.00 rows=1000 width=36)
                                           (actual time=0.01..0.2 rows=1000 loops=1)
Buffers: shared hit=730 read=0
Planning Time: 0.3 ms
Execution Time: 31.6 ms
```

Key observations:
- **Hash Join** chosen — builds a hash table on `customers` (smaller), probes with `orders`
- **Hash Cond** shows the join condition: `o.customer_id = c.id`
- The inner table (`customers`) is scanned first and loaded into memory (the `Hash` node)
- The outer table (`orders`) is scanned and each row probes the hash table
- Two Seq Scans — neither table has a useful index for this full aggregation

### Nested Loop Join

```sql
-- With an index on orders.customer_id:
EXPLAIN SELECT c.name, o.total
FROM customers c
JOIN orders o ON o.customer_id = c.id
WHERE c.id = 42;
```

```
Nested Loop  (cost=0.43..18.5 rows=3 width=48)
  ->  Index Scan using customers_pkey on customers c
        Index Cond: (id = 42)
  ->  Index Scan using orders_customer_id_idx on orders o
        Index Cond: (customer_id = 42)
```

Here:
- The outer loop finds one row (customer 42) via primary key index
- For that one row, the inner index scan finds matching orders
- Loops = 1 (outer) → only one inner probe needed

---

## 8. Query Examples

### Example 1 — Basic: Equi-Join with Projection

```sql
-- Orders with customer names — standard equi-join
SELECT
  o.id          AS order_id,
  c.name        AS customer_name,
  c.email       AS customer_email,
  o.total_amount,
  o.created_at  AS order_date
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY o.created_at DESC;
```

### Example 2 — Intermediate: Multi-Table Join with Aggregation

```sql
-- Sales report: category → product → revenue, with customer count
SELECT
  cat.name                         AS category,
  p.name                           AS product,
  COUNT(DISTINCT o.customer_id)    AS unique_customers,
  COUNT(oi.id)                     AS total_orders,
  SUM(oi.quantity * oi.unit_price) AS revenue,
  AVG(oi.unit_price)               AS avg_unit_price
FROM categories cat
JOIN products p          ON p.category_id = cat.id
JOIN order_items oi      ON oi.product_id = p.id
JOIN orders o            ON o.id = oi.order_id
WHERE o.status = 'completed'
  AND o.created_at >= DATE_TRUNC('year', CURRENT_DATE)
GROUP BY cat.id, cat.name, p.id, p.name
ORDER BY revenue DESC;
```

### Example 3 — Production Grade: Detecting the Duplicate Row Trap

```sql
-- WRONG — if a customer has multiple email addresses, rows multiply
-- This is the duplicate row trap in action
SELECT
  c.id,
  c.name,
  e.address AS email   -- customer_emails is one-to-many
FROM customers c
JOIN customer_emails e ON e.customer_id = c.id
WHERE c.account_status = 'active';
-- PROBLEM: customer with 3 emails appears 3 times

-- CORRECT — decide what you actually want:

-- Option A: all email rows (accept multiplication — often correct)
SELECT c.id, c.name, e.address AS email, e.is_primary
FROM customers c
JOIN customer_emails e ON e.customer_id = c.id AND e.is_primary = TRUE
WHERE c.account_status = 'active';

-- Option B: one row per customer, primary email only
SELECT c.id, c.name,
  (SELECT e2.address FROM customer_emails e2
   WHERE e2.customer_id = c.id AND e2.is_primary = TRUE
   LIMIT 1) AS primary_email
FROM customers c
WHERE c.account_status = 'active';

-- Option C: one row per customer, all emails as array
SELECT
  c.id,
  c.name,
  ARRAY_AGG(e.address ORDER BY e.is_primary DESC) AS emails
FROM customers c
JOIN customer_emails e ON e.customer_id = c.id
WHERE c.account_status = 'active'
GROUP BY c.id, c.name;
```

---

## 9. Wrong → Right Patterns

### Wrong 1: Accidental Cartesian Product (Missing ON)

```sql
-- WRONG: forgot ON condition — 1000 customers × 30000 orders = 30M rows
SELECT c.name, o.total
FROM customers c
JOIN orders o;  -- ERROR in modern SQL (ON required for explicit JOIN)

-- With implicit syntax, this silently produces 30M rows:
SELECT c.name, o.total
FROM customers c, orders o;  -- missing WHERE — full cartesian product

-- RIGHT:
SELECT c.name, o.total
FROM customers c
JOIN orders o ON o.customer_id = c.id;
```

### Wrong 2: ON vs WHERE for Outer Joins

```sql
-- WRONG: intending to show all customers, but WHERE converts LEFT to INNER
SELECT c.name, o.total
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'completed';  -- eliminates customers with no completed orders
-- Customers with NO orders (o.* = NULL) are removed by WHERE

-- RIGHT: filter in ON to preserve unmatched customers
SELECT c.name, o.total
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id AND o.status = 'completed';
-- Customers with no completed orders appear with NULL in o.total
```

### Wrong 3: Selecting Non-Aggregated Columns After JOIN

```sql
-- WRONG: products.name is not in GROUP BY or aggregated
SELECT c.name, p.name, COUNT(o.id)
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
GROUP BY c.name;  -- ERROR: p.name must appear in GROUP BY or aggregate

-- RIGHT:
GROUP BY c.id, c.name, p.id, p.name;
```

### Wrong 4: Joining Without Understanding Fan-Out

```sql
-- WRONG expectation: developer expects one row per order
SELECT o.id, o.total, oi.product_id, oi.quantity
FROM orders o
JOIN order_items oi ON oi.order_id = o.id;
-- REALITY: one row per order_item — an order with 5 items appears 5 times

-- Not wrong per se, but you must know this is happening
-- Aggregate if you want one row per order:
SELECT o.id, o.total,
  COUNT(oi.id) AS item_count,
  SUM(oi.quantity) AS total_units
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
GROUP BY o.id, o.total;
```

### Wrong 5: Using NATURAL JOIN

```sql
-- WRONG: fragile, implicit, changes behaviour when schema changes
SELECT * FROM orders NATURAL JOIN customers;

-- RIGHT: explicit ON condition always
SELECT o.id, c.name
FROM orders o
JOIN customers c ON c.id = o.customer_id;
```

---

## 10. Performance Profile

| Scenario | Algorithm | Complexity | Notes |
|----------|-----------|------------|-------|
| Small outer, indexed inner | Nested Loop | O(outer × log(inner)) | Best for point lookups |
| Large tables, equality join | Hash Join | O(N + M) | Memory limited by work_mem |
| Pre-sorted inputs or sort merge | Merge Join | O(N log N + M log M) | Avoids random I/O |
| No index, low selectivity | Seq Scan + Hash Join | O(N + M) | Planner default for full scans |
| Cartesian product (CROSS JOIN) | Nested Loop | O(N × M) | Dangerous at scale |

**Index impact on JOIN performance**:

| Index Situation | Expected Plan |
|-----------------|--------------|
| Index on join column of inner table | Nested Loop with Index Scan |
| No index on either join column | Hash Join (or Merge Join with sort) |
| Both tables sorted on join key | Merge Join |
| Small inner table | Nested Loop (inner in memory) |
| Large outer + small inner | Hash Join (hash the small one) |

**The join selectivity rule**: A JOIN that reduces rows significantly (high selectivity) is fast. A JOIN that barely filters (e.g., joining on a low-cardinality column) produces many rows and is expensive. Always check `EXPLAIN ANALYZE` for actual vs estimated row counts after the join node.

---

## 11. Node.js Integration

### 11.1 Basic JOIN with pg

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function getOrdersWithCustomer(since) {
  const { rows } = await pool.query(
    `SELECT
       o.id          AS order_id,
       c.name        AS customer_name,
       c.email,
       o.total_amount,
       o.status,
       o.created_at
     FROM orders o
     JOIN customers c ON c.id = o.customer_id
     WHERE o.created_at >= $1
     ORDER BY o.created_at DESC`,
    [since]
  );
  return rows;
}
```

### 11.2 Detecting fan-out in results

```javascript
// When you JOIN to a one-to-many table, handle multiplication explicitly
async function getProductsWithTags(categoryId) {
  const { rows } = await pool.query(
    `SELECT
       p.id,
       p.name,
       p.price,
       t.name AS tag
     FROM products p
     JOIN product_tags pt ON pt.product_id = p.id
     JOIN tags t ON t.id = pt.tag_id
     WHERE p.category_id = $1
     ORDER BY p.id, t.name`,
    [categoryId]
  );

  // Collapse in JS: group tags per product
  const productMap = new Map();
  for (const row of rows) {
    if (!productMap.has(row.id)) {
      productMap.set(row.id, { id: row.id, name: row.name, price: row.price, tags: [] });
    }
    productMap.get(row.id).tags.push(row.tag);
  }
  return [...productMap.values()];
}
```

### 11.3 Collapsing in SQL (preferred over JS for large result sets)

```javascript
async function getProductsWithTagsCollapsed(categoryId) {
  const { rows } = await pool.query(
    `SELECT
       p.id,
       p.name,
       p.price,
       ARRAY_AGG(t.name ORDER BY t.name) AS tags
     FROM products p
     JOIN product_tags pt ON pt.product_id = p.id
     JOIN tags t           ON t.id = pt.tag_id
     WHERE p.category_id = $1
     GROUP BY p.id, p.name, p.price
     ORDER BY p.name`,
    [categoryId]
  );
  return rows; // tags is already a JS array (pg deserialises PG arrays)
}
```

### 11.4 Parameterised JOIN conditions

```javascript
// Safe: all dynamic values are parameters, never interpolated
async function getOrdersByRegion(region, startDate, endDate) {
  const { rows } = await pool.query(
    `SELECT
       o.id,
       c.name AS customer,
       r.name AS region,
       o.total_amount
     FROM orders o
     JOIN customers c ON c.id = o.customer_id
     JOIN regions r    ON r.id = c.region_id
     WHERE r.code      = $1
       AND o.created_at BETWEEN $2 AND $3
     ORDER BY o.total_amount DESC`,
    [region, startDate, endDate]
  );
  return rows;
}
```

---

## 12. ORM Comparison

### Prisma

**Can Prisma do JOINs?** — Yes, via `include` and `select` with nested relations. For complex multi-table JOINs, Prisma generates the SQL automatically from relation definitions.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// Simple JOIN — Prisma uses include for relations
const orders = await prisma.order.findMany({
  where: { createdAt: { gte: new Date('2025-01-01') } },
  include: {
    customer: {
      select: { name: true, email: true },
    },
  },
  orderBy: { createdAt: 'desc' },
});
// orders[0].customer.name works — typed

// Multi-level JOIN (orders → items → products → categories)
const ordersWithProducts = await prisma.order.findMany({
  include: {
    customer: true,
    orderItems: {
      include: {
        product: {
          include: { category: true },
        },
      },
    },
  },
});
```

**Where it breaks**: Prisma generates separate queries for nested `include` in some cases (N+1 risk). Use `include` with awareness — Prisma may use a JOIN or a batched subquery depending on the relation type. Complex GROUP BY with JOINs requires `$queryRaw`.

```typescript
// Aggregate JOIN — requires raw SQL
const report = await prisma.$queryRaw`
  SELECT cat.name, COUNT(DISTINCT o.customer_id) AS customers, SUM(oi.quantity * oi.unit_price) AS revenue
  FROM categories cat
  JOIN products p    ON p.category_id = cat.id
  JOIN order_items oi ON oi.product_id = p.id
  JOIN orders o      ON o.id = oi.order_id
  GROUP BY cat.id, cat.name
  ORDER BY revenue DESC
`;
```

**Verdict**: Use Prisma's `include` for standard relational fetching. Drop to raw SQL for aggregation, complex join conditions, or cross-table analytics.

---

### Drizzle ORM

**Can Drizzle do JOINs?** — Yes, fully. Drizzle has typed join methods.

```typescript
import { db } from './db';
import { orders, customers, orderItems, products, categories } from './schema';
import { eq, gte, and, sql } from 'drizzle-orm';

// Simple JOIN
const result = await db
  .select({
    orderId: orders.id,
    customerName: customers.name,
    email: customers.email,
    totalAmount: orders.totalAmount,
  })
  .from(orders)
  .innerJoin(customers, eq(customers.id, orders.customerId))
  .where(gte(orders.createdAt, new Date('2025-01-01')))
  .orderBy(orders.createdAt);

// Multi-table JOIN with aggregation
const revenue = await db
  .select({
    categoryName: categories.name,
    revenue: sql<number>`SUM(${orderItems.quantity} * ${orderItems.unitPrice})`.as('revenue'),
    uniqueCustomers: sql<number>`COUNT(DISTINCT ${orders.customerId})`.as('unique_customers'),
  })
  .from(categories)
  .innerJoin(products, eq(products.categoryId, categories.id))
  .innerJoin(orderItems, eq(orderItems.productId, products.id))
  .innerJoin(orders, eq(orders.id, orderItems.orderId))
  .groupBy(categories.id, categories.name)
  .orderBy(sql`revenue DESC`);
```

**Where it breaks**: LEFT JOIN combined with complex aggregation occasionally requires `sql` template usage for the full expression. Recursive or self-referential joins need explicit aliases.

**Verdict**: Drizzle handles JOINs well with full type inference on joined columns. Use it confidently for standard join patterns.

---

### Sequelize

**Can Sequelize do JOINs?** — Yes, via associations and `include`. But the SQL generated is often suboptimal.

```javascript
const { Op } = require('sequelize');
const { Order, Customer, OrderItem, Product, Category } = require('./models');

// Simple JOIN via include
const orders = await Order.findAll({
  where: { createdAt: { [Op.gte]: new Date('2025-01-01') } },
  include: [
    {
      model: Customer,
      attributes: ['name', 'email'],
    },
  ],
  order: [['createdAt', 'DESC']],
});

// Multi-level include (generates LEFT JOINs by default, not INNER)
const deep = await Order.findAll({
  include: [
    { model: Customer },
    {
      model: OrderItem,
      include: [{ model: Product, include: [{ model: Category }] }],
    },
  ],
});
```

**Where it breaks**: Sequelize uses `LEFT OUTER JOIN` for `include` by default — you must explicitly set `required: true` to get INNER JOIN behaviour. Aggregation with `include` produces unreliable SQL (often wrong GROUP BY). Complex join conditions require `literal()`.

```javascript
// Force INNER JOIN behaviour
include: [{ model: Customer, required: true }]

// Complex aggregation — use raw query
const result = await sequelize.query(
  `SELECT cat.name, SUM(oi.quantity * oi.unit_price) AS revenue
   FROM categories cat
   JOIN products p ON p.category_id = cat.id
   JOIN order_items oi ON oi.product_id = p.id
   JOIN orders o ON o.id = oi.order_id
   GROUP BY cat.id, cat.name`,
  { type: QueryTypes.SELECT }
);
```

**Verdict**: Sequelize associations work for simple cases. Be aware that `include` defaults to LEFT JOIN. For any aggregation involving JOINs, use `sequelize.query()`.

---

### TypeORM

**Can TypeORM do JOINs?** — Yes, via QueryBuilder with `leftJoinAndSelect`, `innerJoinAndSelect`.

```typescript
import { DataSource } from 'typeorm';

const dataSource = new DataSource({ /* config */ });

// Simple JOIN using QueryBuilder
const orders = await dataSource
  .getRepository(Order)
  .createQueryBuilder('o')
  .innerJoinAndSelect('o.customer', 'c')
  .where('o.createdAt >= :since', { since: new Date('2025-01-01') })
  .orderBy('o.createdAt', 'DESC')
  .getMany();
// orders[0].customer.name — typed

// Multi-table join with aggregation
const revenue = await dataSource
  .createQueryBuilder()
  .select('cat.name', 'categoryName')
  .addSelect('SUM(oi.quantity * oi.unitPrice)', 'revenue')
  .from(Category, 'cat')
  .innerJoin('cat.products', 'p')
  .innerJoin('p.orderItems', 'oi')
  .innerJoin('oi.order', 'o')
  .groupBy('cat.id')
  .addGroupBy('cat.name')
  .orderBy('revenue', 'DESC')
  .getRawMany();
```

**Where it breaks**: TypeORM's relation-based joins require relation definitions in entity classes. `getMany()` loads related entities; `getRawMany()` returns plain objects. Complex join conditions not expressible as relations need `leftJoin('table', 'alias', 'condition')` with raw condition strings.

**Verdict**: TypeORM QueryBuilder handles joins adequately. Use `getMany()` for entity loading, `getRawMany()` for analytical queries. For very complex multi-joins, raw SQL is cleaner.

---

### Knex.js

**Can Knex do JOINs?** — Yes, fully. Knex has explicit join methods.

```javascript
const knex = require('knex')({ client: 'pg', connection: process.env.DATABASE_URL });

// Simple JOIN
const orders = await knex('orders as o')
  .join('customers as c', 'c.id', 'o.customer_id')
  .select('o.id as order_id', 'c.name as customer_name', 'c.email', 'o.total_amount')
  .where('o.created_at', '>=', '2025-01-01')
  .orderBy('o.created_at', 'desc');

// Multi-table JOIN with aggregation
const revenue = await knex('categories as cat')
  .join('products as p',    'p.category_id', 'cat.id')
  .join('order_items as oi', 'oi.product_id', 'p.id')
  .join('orders as o',       'o.id',          'oi.order_id')
  .select('cat.name as category')
  .sum('oi.unit_price as revenue')
  .countDistinct('o.customer_id as unique_customers')
  .groupBy('cat.id', 'cat.name')
  .orderByRaw('revenue DESC');

// Complex join condition (range join)
const tiered = await knex('purchases as p')
  .join('price_tiers as t', function() {
    this.on('p.quantity', '>=', 't.min_qty')
        .andOn('p.quantity', '<=', 't.max_qty');
  })
  .select('p.id', 'p.quantity', 't.tier_name', 't.discount_pct');
```

**Where it breaks**: Complex join conditions require the function form of `.join()`. Knex doesn't validate that joined tables are actually related — accidental Cartesian products are possible if conditions are omitted.

**Verdict**: Knex handles all join types cleanly and stays close to SQL. The closest to raw SQL of all ORMs reviewed. Good default for join-heavy queries.

---

### ORM Summary Table

| ORM | Simple JOIN | Multi-table JOIN | Aggregation + JOIN | Complex ON | Verdict |
|-----|------------|------------------|--------------------|-----------|---------|
| Prisma | `include` (typed) | Nested `include` | Raw SQL required | Raw SQL | Use `include` for relations; raw for analytics |
| Drizzle | `.innerJoin()` | Chained joins | `sql` template | `sql` template | Strong — use builder |
| Sequelize | `include` (LEFT by default) | Nested `include` | Raw SQL required | `literal()` | Watch LEFT vs INNER default |
| TypeORM | QueryBuilder | QueryBuilder chain | `getRawMany()` | Raw string | Builder ok; raw for complex |
| Knex | `.join()` | Chained `.join()` | `.sum()`, `.count()` | Function form | Best JOIN support of all |

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given:
- `employees(id, name, department_id, salary, manager_id)`
- `departments(id, name, location)`

Write a query returning every employee's `name`, `salary`, their `department name`, and their `department location`. Include employees who have no department assigned (NULL `department_id`). Order by department name, then by salary descending within each department.

---

### Exercise 2 — Intermediate

Given:
- `products(id, name, price, category_id)`
- `order_items(id, order_id, product_id, quantity, unit_price)`
- `orders(id, customer_id, status, created_at)`
- `categories(id, name)`

Write a single query that returns, for each category:
- `category_name`
- `total_orders` — count of distinct orders that include at least one product from this category
- `total_revenue` — sum of `quantity × unit_price` for completed orders only
- `avg_item_price` — average `unit_price` across all order_items for this category
- `top_product_count` — the number of distinct products from this category that appear in any order

Only include categories that have at least one order.

---

### Exercise 3 — Intermediate

Using only a `employees(id, name, salary, manager_id)` table, write a self-join query that returns:
- `employee_name`
- `employee_salary`
- `manager_name` (NULL for top-level employees with no manager)
- `salary_difference` — employee's salary minus manager's salary (NULL if no manager)
- `earns_more_than_manager` — TRUE if the employee earns more than their manager, FALSE otherwise, NULL if no manager

Order by `salary_difference` DESC NULLS LAST.

---

### Exercise 4 — Production Grade

You're debugging a report that should show "customers with their most recent order and total lifetime spend." A junior developer wrote this:

```sql
SELECT c.id, c.name, o.created_at AS last_order, SUM(o.total_amount) AS lifetime_spend
FROM customers c
JOIN orders o ON o.customer_id = c.id
ORDER BY o.created_at DESC;
```

Identify every bug in this query (there are at least 3). Then write the correct version that returns:
- One row per customer
- Their most recent order date
- Their total lifetime spend
- Also include customers who have never placed an order (show NULL for last_order and 0 for lifetime_spend)
- Order by lifetime_spend DESC, then by name for ties

---

## 14. Interview Questions

### Junior Level

**Q: What is an INNER JOIN? What rows does it return?**

A: An INNER JOIN returns only the rows where the ON condition is TRUE in both tables. Rows from either table that have no matching row in the other table are excluded from the result. It is the default JOIN type — writing `JOIN` without a type keyword means INNER JOIN.

---

**Q: What is the difference between ON and WHERE for a JOIN?**

A: For INNER JOIN, `ON` and `WHERE` produce the same result — you can put conditions in either place. For OUTER JOINs (LEFT, RIGHT, FULL), the placement matters: a condition in `ON` is evaluated during join construction and preserves unmatched rows from the preserved side. A condition in `WHERE` is evaluated after join construction and eliminates rows — including those with NULL values synthesised by the outer join — which effectively converts the OUTER JOIN into an INNER JOIN.

---

**Q: Why might a JOIN return more rows than the table you're joining from?**

A: Because of the one-to-many (or many-to-many) nature of the relationship. The JOIN conceptually starts with a Cartesian product — every row from table A paired with every row from table B — and then filters on the ON condition. If one row in table A matches 5 rows in table B, that row from table A appears 5 times in the result. This is correct behaviour; it is the cross product foundation of JOIN semantics.

---

### Principal Level

**Q: Explain how the query planner chooses between Nested Loop, Hash Join, and Merge Join. What signals in EXPLAIN ANALYZE tell you the planner made the wrong choice?**

A: The planner uses cost estimates based on table statistics to choose the algorithm. Nested Loop is chosen when the outer table is small and the inner table has an index on the join key — cost is O(outer × inner_index_lookup). Hash Join is chosen for equality joins where neither side is indexed or when both sides are large — it builds a hash table on the smaller side (bounded by `work_mem`) and probes with the larger side. Merge Join is chosen when both sides are already sorted on the join key (e.g., joining on a primary key after an index scan) — it avoids random I/O by merging in order.

The wrong choice is signalled by a large discrepancy between `estimated rows` and `actual rows` in EXPLAIN ANALYZE, particularly at the node just below the join. If the planner estimated 10 rows but got 100,000, it likely chose Nested Loop when Hash Join would have been faster. Fix approaches: run ANALYZE to update statistics, increase `statistics target` for the join column, or — as a last resort — disable the chosen algorithm with `SET enable_nestloop = off` to force a different plan.

---

**Q: A developer reports that a LEFT JOIN is returning no rows from the left table when they expect all left rows to be present. What is the most likely cause?**

A: The most likely cause is a `WHERE` clause filtering on a column from the right table. A LEFT JOIN preserves all left-table rows, but if the right-table columns are NULL for unmatched rows, and the WHERE clause requires a non-NULL value from the right table (e.g., `WHERE right_table.col = 'x'`), those NULLs fail the condition and the rows are eliminated. The LEFT JOIN is effectively converted into an INNER JOIN by the WHERE filter. The fix is to move the condition from WHERE to the ON clause: `ON right_table.id = left_table.fk AND right_table.col = 'x'`. This way, unmatched rows still produce NULLs and are preserved.

---

**Q: You have a JOIN between a 50M-row table and a 10K-row table. The query takes 45 seconds. Walk through your diagnostic process.**

A: 
1. Run `EXPLAIN (ANALYZE, BUFFERS)` and look at the join node's algorithm — is it Nested Loop, Hash Join, or Merge Join?
2. Check `actual rows` vs `estimated rows` at the join node — a large mismatch indicates bad statistics, which may have caused a suboptimal algorithm choice.
3. Check buffer usage (`Buffers: shared hit` vs `read`) — if most are `read` (disk I/O), the problem is I/O-bound, not CPU-bound.
4. For Nested Loop on a 50M-row table: check whether the inner scan has an index. If not, the planner is doing 50M × sequential scans of the 10K table — catastrophic. Add an index on the join column of the 10K table.
5. For Hash Join: check if `Batches > 1` — this means work_mem is too small, causing the hash table to spill to disk. Increase `work_mem` for this session.
6. Check if the join is on the right columns — verify the ON condition uses indexed columns, no functions wrapping them (sargability), and types match (implicit cast can prevent index use).
7. If statistics are stale: run `ANALYZE table_name` and re-explain.

---

## 15. Mental Model Checkpoint

1. **You write `FROM A JOIN B ON A.id = B.a_id`. Table A has 1,000 rows. Table B has 10 rows, and every row in B matches a row in A. How many rows does the result have at most? Could it be more than 1,000?**

2. **You have a LEFT JOIN query that returns NULL for all right-table columns on some rows. Then you add `WHERE right_table.name = 'active'`. What happens to those NULL rows?**

3. **What is the logical difference between `JOIN b ON b.x = a.x AND b.status = 'active'` and `JOIN b ON b.x = a.x WHERE b.status = 'active'` when the JOIN is a LEFT JOIN?**

4. **A query returns 50,000 rows but you only have 10,000 customers and 8,000 orders. What does this tell you about the relationship between the tables being joined?**

5. **The planner chose a Hash Join for your query. You see `Batches: 4` in EXPLAIN ANALYZE. What does this mean and what is the fix?**

6. **You need to join on `DATE_TRUNC('day', event_time) = DATE_TRUNC('day', log_time)`. Why is this a problem and how do you fix it?**

7. **NATURAL JOIN automatically joins on all matching column names. What is the danger of this and why should you never use it in production code?**

---

## 16. Quick Reference Card

```sql
-- JOIN types
INNER JOIN  → matched rows only
LEFT JOIN   → all from left + NULLs for unmatched right
RIGHT JOIN  → all from right + NULLs for unmatched left
FULL OUTER JOIN → all rows from both, NULLs for unmatched
CROSS JOIN  → every row × every row (Cartesian product)

-- Basic syntax
SELECT ...
FROM table_a a
[INNER|LEFT|RIGHT|FULL OUTER|CROSS] JOIN table_b b ON a.id = b.a_id

-- USING shorthand (same column name in both tables)
JOIN table_b USING (shared_column_name)

-- Multi-table
FROM a
JOIN b ON b.a_id = a.id
JOIN c ON c.b_id = b.id
JOIN d ON d.a_id = a.id   -- can reference any already-joined table

-- Self-join (aliases required)
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id

-- Range join (non-equi-join)
JOIN price_tiers pt ON p.qty BETWEEN pt.min_qty AND pt.max_qty

-- ON vs WHERE for outer joins
LEFT JOIN b ON b.id = a.b_id AND b.status = 'x'  -- filter in ON: preserves left rows
LEFT JOIN b ON b.id = a.b_id WHERE b.status = 'x' -- filter in WHERE: converts to INNER

-- Collapse one-to-many with GROUP BY
SELECT a.id, ARRAY_AGG(b.value) AS values
FROM a JOIN b ON b.a_id = a.id
GROUP BY a.id

-- Diagnose join performance
EXPLAIN (ANALYZE, BUFFERS) SELECT ...;
-- Look for: actual vs estimated rows, Hash Batches > 1, Nested Loop without index

-- Implicit cast danger in ON
ON a.user_id = b.user_id   -- both must be same type; mismatch = seq scan
ON a.user_id::text = b.user_id  -- non-sargable — avoid
```

---

## Connected Topics

- **Topic 11 — INNER JOIN in Depth**: The mechanics of the most common join type, execution algorithms, exact semantics.
- **Topic 12 — LEFT JOIN and RIGHT JOIN**: The ON vs WHERE trap in full detail; NULL semantics in outer join results.
- **Topic 17 — JOIN Performance Deep Dive**: Nested loop, hash join, merge join — when each is chosen, how to read them in EXPLAIN, how to fix wrong choices.
- **Topic 18 — The N+1 Query Problem**: What happens when you don't JOIN — the ORM trap and how to diagnose it.
- **Topic 08 — Filtering Performance & Sargability**: Join conditions on functions, type mismatches — same sargability rules apply to ON as to WHERE.
- **Topic 26 — Subqueries in Depth**: Correlated subqueries as an alternative to JOIN in some patterns.
- **Topic 31 — Lateral Joins**: The LATERAL keyword as a more powerful correlated join.
- **Topic 19 — JOIN Alternatives**: EXISTS vs JOIN, IN vs JOIN — when each is faster.
