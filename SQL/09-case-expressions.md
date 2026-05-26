# Topic 09 — CASE Expressions
### SQL Mastery Curriculum — Phase 2: SELECT and Filtering Foundations

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you're a cashier applying discounts at checkout. You look at the item category and decide:

- **If** it's "Electronics" → apply 10% off  
- **If** it's "Clothing" → apply 20% off  
- **Otherwise** → no discount  

You're not filtering out rows (you're not sending customers away). You're **computing a new value for every single row** based on conditions. That computation happens column by column, row by row — and that's exactly what `CASE` does in SQL.

`CASE` is a **conditional value expression**. It doesn't filter. It doesn't group. It evaluates, and it returns a value — whatever type you specify — for every row it encounters.

---

## 2. Connection to SQL Internals

`CASE` is evaluated during the **projection phase** — the same phase where `SELECT` computes expressions. This is important:

- `CASE` in `SELECT` → runs during projection (post-filter, post-join)
- `CASE` in `WHERE` → runs during the filter phase, **before** grouping
- `CASE` in `ORDER BY` → runs during the sort phase
- `CASE` in `GROUP BY` → runs during grouping — the grouped buckets are the CASE result values
- `CASE` in aggregate arguments → runs once **per row**, then the aggregate collects the results

The planner sees `CASE` as a **scalar expression**. It cannot be indexed directly (unlike stored columns). If you filter on a `CASE` expression in `WHERE`, it is not sargable — the planner must evaluate it for every row.

One internal subtlety: **CASE short-circuits**. When a WHEN branch matches, subsequent branches are never evaluated. This is guaranteed by the SQL standard and PostgreSQL honours it — which matters when branches contain expensive subqueries or functions.

---

## 3. Logical Execution Order Context

```
FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → Window → ORDER BY → LIMIT
         ↑           ↑                        ↑                 ↑        ↑
     CASE here    CASE here                CASE here          CASE      CASE
     (filter)     (grouping)               (projection)       in        in
                                                               OVER      sort
```

A critical consequence: a `CASE` alias defined in `SELECT` **cannot be referenced in `WHERE`** (WHERE executes before SELECT). You must repeat the expression or use a subquery/CTE.

```sql
-- WRONG — alias not yet defined when WHERE runs
SELECT CASE WHEN salary > 100000 THEN 'senior' ELSE 'junior' END AS band
FROM employees
WHERE band = 'senior';  -- ERROR: column "band" does not exist

-- CORRECT — wrap in subquery
SELECT * FROM (
  SELECT *, CASE WHEN salary > 100000 THEN 'senior' ELSE 'junior' END AS band
  FROM employees
) sub
WHERE band = 'senior';
```

However, a `CASE` alias **can** be referenced in `ORDER BY` and `GROUP BY` in PostgreSQL (PostgreSQL extends the standard here as a convenience).

---

## 4. What Is CASE?

CASE is SQL's conditional expression. It comes in two syntactic forms.

### Simple CASE (equality matching)

```sql
CASE expression
  WHEN value1 THEN result1
  WHEN value2 THEN result2
  ELSE default_result
END
```

The `expression` is evaluated **once** and compared for equality to each `WHEN` value in order.

```sql
SELECT
  order_status,
  CASE order_status
    WHEN 'pending'   THEN 'Awaiting payment'
    WHEN 'paid'      THEN 'Processing'
    WHEN 'shipped'   THEN 'On the way'
    WHEN 'delivered' THEN 'Complete'
    ELSE 'Unknown status'
  END AS status_label
FROM orders;
```

**Limitation**: Simple CASE uses only `=` comparison. It cannot use `>`, `<`, `LIKE`, `IS NULL`, etc.

### Searched CASE (arbitrary conditions)

```sql
CASE
  WHEN condition1 THEN result1
  WHEN condition2 THEN result2
  ELSE default_result
END
```

Each `WHEN` is a full boolean expression — any condition valid in a WHERE clause works here.

```sql
SELECT
  salary,
  CASE
    WHEN salary < 50000                    THEN 'entry'
    WHEN salary >= 50000 AND salary < 100000 THEN 'mid'
    WHEN salary >= 100000                  THEN 'senior'
  END AS band
FROM employees;
```

### The ELSE clause

- **If ELSE is omitted** and no WHEN matches → `NULL` is returned.
- This is a common bug: silent NULLs in results from forgotten ELSE.
- Always include ELSE unless NULL is explicitly the intended default.

### Type consistency

All `THEN` and `ELSE` branches must return **compatible types**. PostgreSQL will attempt implicit coercion. If types are incompatible, you get a runtime error:

```sql
-- ERROR: integer vs text
SELECT CASE WHEN 1=1 THEN 42 ELSE 'hello' END;
-- ERROR:  CASE types integer and text cannot be matched

-- CORRECT: cast explicitly
SELECT CASE WHEN 1=1 THEN 42::text ELSE 'hello' END;
```

---

## 5. Why CASE Matters in Production

1. **Data transformation without application code** — convert codes to labels, map legacy values, normalise categories — all in SQL, all set-based, no per-row application logic.

2. **Conditional aggregation** — `COUNT(*) FILTER (WHERE ...)` is cleaner, but `SUM(CASE WHEN ... THEN 1 ELSE 0 END)` works everywhere and is the universal pivot pattern.

3. **Pivot tables** — rotating rows to columns is done entirely with CASE + GROUP BY.

4. **Custom sort orders** — ORDER BY `CASE WHEN status = 'urgent' THEN 0 ELSE 1 END` gives priority ordering that no simple column sort could achieve.

5. **Soft categorisation** — bucketing continuous values (age ranges, salary bands, score tiers) at query time without altering the schema.

6. **Safe division** — `CASE WHEN denominator = 0 THEN NULL ELSE numerator / denominator END` prevents division-by-zero errors in production.

7. **Conditional inserts/updates** — CASE in `SET` clauses of UPDATE lets you conditionally update different columns in a single pass.

---

## 6. Deep Technical Content

### 6.1 CASE in SELECT (Projection)

The standard use — compute a derived value for each row.

```sql
SELECT
  user_id,
  email,
  created_at,
  CASE
    WHEN created_at >= NOW() - INTERVAL '30 days' THEN 'new'
    WHEN created_at >= NOW() - INTERVAL '1 year'  THEN 'active'
    ELSE 'veteran'
  END AS user_segment
FROM users;
```

### 6.2 CASE in WHERE (Conditional Filtering)

Less common, but valid — when the filter condition itself needs to branch.

```sql
-- Show admins regardless of status, show regular users only if active
SELECT * FROM users
WHERE
  CASE
    WHEN role = 'admin' THEN TRUE
    ELSE is_active = TRUE
  END;
```

**Warning**: This is non-sargable. The database cannot use an index on `is_active` or `role` directly here. Use this sparingly and only when the equivalent OR expression would be harder to read or maintain.

The equivalent (more sargable) form:

```sql
SELECT * FROM users
WHERE role = 'admin' OR (role != 'admin' AND is_active = TRUE);
-- Or more simply:
SELECT * FROM users
WHERE role = 'admin' OR is_active = TRUE;
```

### 6.3 CASE in ORDER BY (Custom Sort Order)

Powerful for priority sorting and status-based ordering.

```sql
-- Priority queue: urgent first, then by created_at
SELECT * FROM tickets
ORDER BY
  CASE priority
    WHEN 'critical' THEN 1
    WHEN 'high'     THEN 2
    WHEN 'medium'   THEN 3
    WHEN 'low'      THEN 4
    ELSE 5
  END,
  created_at ASC;
```

```sql
-- NULLs last in a custom position
SELECT * FROM employees
ORDER BY
  CASE WHEN department IS NULL THEN 1 ELSE 0 END,
  department,
  last_name;
```

### 6.4 CASE in GROUP BY (Grouping by Derived Buckets)

Group rows into computed categories without materialising them.

```sql
SELECT
  CASE
    WHEN age < 18  THEN 'minor'
    WHEN age < 65  THEN 'adult'
    ELSE 'senior'
  END AS age_group,
  COUNT(*) AS count
FROM customers
GROUP BY
  CASE
    WHEN age < 18  THEN 'minor'
    WHEN age < 65  THEN 'adult'
    ELSE 'senior'
  END;
```

**PostgreSQL shortcut**: You can reference the SELECT alias in GROUP BY in PostgreSQL:

```sql
SELECT
  CASE WHEN age < 18 THEN 'minor' WHEN age < 65 THEN 'adult' ELSE 'senior' END AS age_group,
  COUNT(*) AS count
FROM customers
GROUP BY age_group;  -- PostgreSQL allows alias reference here
```

**Note**: This is a PostgreSQL extension — not standard SQL, not available in all databases.

### 6.5 CASE in Aggregate Functions (Conditional Aggregation)

The most powerful CASE pattern. Compute multiple filtered aggregates in a single pass.

```sql
SELECT
  department,
  COUNT(*)                                                           AS total,
  COUNT(CASE WHEN status = 'active'   THEN 1 END)                   AS active_count,
  COUNT(CASE WHEN status = 'inactive' THEN 1 END)                   AS inactive_count,
  SUM(CASE WHEN status = 'active' THEN salary ELSE 0 END)           AS active_payroll,
  AVG(CASE WHEN gender = 'F' THEN salary END)                       AS avg_female_salary,
  MAX(CASE WHEN hire_date >= '2023-01-01' THEN salary END)          AS max_new_hire_salary
FROM employees
GROUP BY department;
```

This replaces multiple separate queries and is computed in a **single table scan**. This is the pivot pattern.

**CASE vs FILTER for conditional aggregation**:

```sql
-- CASE approach (universal — works in all SQL databases)
COUNT(CASE WHEN status = 'active' THEN 1 END)

-- FILTER approach (SQL standard, PostgreSQL 9.4+, cleaner)
COUNT(*) FILTER (WHERE status = 'active')
```

`FILTER` is cleaner and may be slightly faster (planner has more information), but both produce identical results. `CASE` is the portable option; `FILTER` is the PostgreSQL idiomatic choice.

### 6.6 CASE for Pivot Tables

A pivot rotates distinct values from a column into separate output columns.

```sql
-- Source: sales(product_id, region, amount)
-- Goal: one row per product, columns for East/West/North revenue

SELECT
  product_id,
  SUM(CASE WHEN region = 'East'  THEN amount ELSE 0 END) AS east_revenue,
  SUM(CASE WHEN region = 'West'  THEN amount ELSE 0 END) AS west_revenue,
  SUM(CASE WHEN region = 'North' THEN amount ELSE 0 END) AS north_revenue,
  SUM(amount)                                            AS total_revenue
FROM sales
GROUP BY product_id
ORDER BY total_revenue DESC;
```

**Limitation**: The regions must be known at write time. For dynamic pivots (unknown values), you need PL/pgSQL or application-side code to build the query dynamically.

### 6.7 CASE for Safe Division

Division by zero is a runtime error in PostgreSQL (`ERROR: division by zero`).

```sql
-- Unsafe
SELECT numerator / denominator FROM metrics;

-- Safe with CASE
SELECT
  CASE
    WHEN denominator = 0 OR denominator IS NULL THEN NULL
    ELSE numerator::numeric / denominator
  END AS safe_ratio
FROM metrics;

-- Alternative using NULLIF (cleaner)
SELECT numerator::numeric / NULLIF(denominator, 0) AS safe_ratio
FROM metrics;
```

`NULLIF(denominator, 0)` returns `NULL` when `denominator = 0`, and division by `NULL` returns `NULL` — no error.

### 6.8 CASE in UPDATE (Conditional Updates)

Update different values in different rows in a single statement.

```sql
-- Apply different discounts based on tier — one UPDATE, one table scan
UPDATE products
SET price = price * CASE tier
  WHEN 'gold'   THEN 0.85   -- 15% off
  WHEN 'silver' THEN 0.90   -- 10% off
  WHEN 'bronze' THEN 0.95   -- 5% off
  ELSE 1.00                 -- no change
END
WHERE tier IN ('gold', 'silver', 'bronze');
```

Without CASE, you'd need three separate UPDATE statements (three table scans).

### 6.9 Nesting CASE Expressions

CASE can be nested, but it gets hard to read quickly. Prefer CTEs for complex conditional logic.

```sql
-- Nested (readable only at shallow depth)
SELECT
  CASE
    WHEN country = 'US' THEN
      CASE
        WHEN state IN ('CA', 'NY', 'TX') THEN 'major_us'
        ELSE 'other_us'
      END
    WHEN country IN ('GB', 'DE', 'FR') THEN 'europe'
    ELSE 'other'
  END AS region
FROM customers;
```

### 6.10 Short-Circuit Guarantee — Why It Matters

CASE short-circuits: the first matching WHEN terminates evaluation. Branches after the match are not evaluated.

```sql
-- Safe: the division only executes when denominator != 0
-- Short-circuit guarantees this
SELECT
  CASE
    WHEN denominator = 0 THEN NULL
    ELSE numerator / denominator  -- only reached if denominator != 0
  END
FROM data;
```

This also applies to expensive subqueries in WHEN branches — place the cheap, high-selectivity conditions first.

```sql
-- Efficient: cheap check first, expensive subquery only when needed
SELECT
  CASE
    WHEN status != 'premium' THEN 'standard'
    WHEN EXISTS (SELECT 1 FROM premium_features pf WHERE pf.user_id = u.id LIMIT 1)
      THEN 'premium_active'
    ELSE 'premium_inactive'
  END AS user_type
FROM users u;
```

---

## 7. EXPLAIN — What CASE Looks Like in the Plan

CASE expressions appear as part of the output expressions in nodes. They are shown in the `Output` field when using `EXPLAIN (VERBOSE)`.

```sql
EXPLAIN (VERBOSE, COSTS OFF)
SELECT
  id,
  CASE WHEN salary > 100000 THEN 'senior' ELSE 'junior' END AS band
FROM employees;
```

```
Seq Scan on public.employees
  Output: id,
          CASE WHEN (salary > 100000) THEN 'senior'::text ELSE 'junior'::text END
```

Key observations:
- No special node for CASE — it's computed inline during the scan
- PostgreSQL shows the full CASE expression in `Output`
- No additional cost node: CASE evaluation cost is folded into the scan node's CPU cost

### CASE in WHERE — no index use

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM employees
WHERE CASE WHEN department = 'eng' THEN salary > 80000 ELSE salary > 60000 END;
```

```
Seq Scan on employees  (cost=0.00..2890.00 rows=333 width=96)
                       (actual time=0.012..18.4 rows=412 loops=1)
  Filter: CASE WHEN (department = 'eng'::text) THEN (salary > 80000)
                                               ELSE (salary > 60000) END
  Rows Removed by Filter: 588
Buffers: shared hit=540
```

The `Filter` line confirms the CASE is a row-by-row filter, not an index access. There is no `Index Cond` line.

---

## 8. Query Examples

### Example 1 — Basic: Status Labels

```sql
-- Convert status codes to human-readable labels
SELECT
  order_id,
  customer_id,
  total_amount,
  CASE status
    WHEN 1 THEN 'pending'
    WHEN 2 THEN 'processing'
    WHEN 3 THEN 'shipped'
    WHEN 4 THEN 'delivered'
    WHEN 5 THEN 'cancelled'
    ELSE 'unknown'
  END AS status_label
FROM orders
WHERE created_at >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY created_at DESC;
```

### Example 2 — Intermediate: Conditional Aggregation Report

```sql
-- Monthly cohort report: active vs churned users by signup month
SELECT
  DATE_TRUNC('month', created_at)                                      AS cohort_month,
  COUNT(*)                                                             AS total_signups,
  COUNT(CASE WHEN last_login >= NOW() - INTERVAL '30 days' THEN 1 END) AS active_30d,
  COUNT(CASE WHEN last_login <  NOW() - INTERVAL '30 days'
              AND last_login >= NOW() - INTERVAL '90 days' THEN 1 END)  AS at_risk,
  COUNT(CASE WHEN last_login <  NOW() - INTERVAL '90 days'
              OR last_login IS NULL THEN 1 END)                         AS churned,
  ROUND(
    COUNT(CASE WHEN last_login >= NOW() - INTERVAL '30 days' THEN 1 END)::numeric
    / NULLIF(COUNT(*), 0) * 100, 2
  )                                                                     AS retention_pct
FROM users
WHERE created_at >= '2024-01-01'
GROUP BY DATE_TRUNC('month', created_at)
ORDER BY cohort_month;
```

### Example 3 — Production Grade: Revenue Pivot with Running Totals

```sql
-- Product revenue breakdown by quarter with running YTD total
-- Single table scan, no JOINs
WITH quarterly_revenue AS (
  SELECT
    product_id,
    SUM(CASE WHEN EXTRACT(QUARTER FROM sale_date) = 1 THEN amount ELSE 0 END) AS q1,
    SUM(CASE WHEN EXTRACT(QUARTER FROM sale_date) = 2 THEN amount ELSE 0 END) AS q2,
    SUM(CASE WHEN EXTRACT(QUARTER FROM sale_date) = 3 THEN amount ELSE 0 END) AS q3,
    SUM(CASE WHEN EXTRACT(QUARTER FROM sale_date) = 4 THEN amount ELSE 0 END) AS q4,
    SUM(amount) AS annual_total,
    -- Flag products with any quarter > 50k
    BOOL_OR(
      CASE WHEN amount > 50000 THEN TRUE ELSE FALSE END
    ) AS had_high_revenue_quarter
  FROM sales
  WHERE EXTRACT(YEAR FROM sale_date) = 2025
  GROUP BY product_id
),
ranked AS (
  SELECT
    qr.*,
    p.name AS product_name,
    p.category,
    CASE
      WHEN annual_total >= 500000 THEN 'platinum'
      WHEN annual_total >= 200000 THEN 'gold'
      WHEN annual_total >= 50000  THEN 'silver'
      ELSE 'bronze'
    END AS revenue_tier,
    SUM(annual_total) OVER (
      PARTITION BY p.category
      ORDER BY annual_total DESC
    ) AS category_running_total
  FROM quarterly_revenue qr
  JOIN products p ON p.id = qr.product_id
)
SELECT
  product_name,
  category,
  revenue_tier,
  q1, q2, q3, q4,
  annual_total,
  category_running_total,
  -- Show trend direction
  CASE
    WHEN q4 > q3 AND q3 > q2 AND q2 > q1 THEN 'consistently_growing'
    WHEN q4 > q1 THEN 'net_positive'
    WHEN q4 < q1 THEN 'net_negative'
    ELSE 'flat'
  END AS trend
FROM ranked
ORDER BY category, annual_total DESC;
```

---

## 9. Wrong → Right Patterns

### Wrong 1: Missing ELSE (silent NULLs)

```sql
-- WRONG: users with salary between 50000 and 100000 get NULL band
SELECT
  name,
  CASE
    WHEN salary < 50000  THEN 'entry'
    WHEN salary > 100000 THEN 'senior'
    -- No ELSE! 50000 <= salary <= 100000 → NULL
  END AS band
FROM employees;

-- RIGHT: explicit ELSE
SELECT
  name,
  CASE
    WHEN salary < 50000   THEN 'entry'
    WHEN salary > 100000  THEN 'senior'
    ELSE 'mid'
  END AS band
FROM employees;
```

### Wrong 2: Referencing CASE alias in WHERE

```sql
-- WRONG: alias not available in WHERE
SELECT
  id,
  CASE WHEN score >= 90 THEN 'A' WHEN score >= 80 THEN 'B' ELSE 'C' END AS grade
FROM students
WHERE grade = 'A';  -- ERROR: column "grade" does not exist

-- RIGHT: use subquery or repeat expression
SELECT * FROM (
  SELECT
    id,
    CASE WHEN score >= 90 THEN 'A' WHEN score >= 80 THEN 'B' ELSE 'C' END AS grade
  FROM students
) graded
WHERE grade = 'A';
```

### Wrong 3: CASE for NULL comparison (using = instead of IS NULL)

```sql
-- WRONG: NULL = NULL is UNKNOWN, not TRUE — this WHEN never matches
SELECT
  CASE column_value
    WHEN NULL THEN 'missing'  -- never matches!
    ELSE 'present'
  END
FROM data;

-- RIGHT: use searched CASE with IS NULL
SELECT
  CASE
    WHEN column_value IS NULL THEN 'missing'
    ELSE 'present'
  END
FROM data;
```

### Wrong 4: Type mismatch in branches

```sql
-- WRONG: mixing integer and text
SELECT CASE WHEN active THEN 1 ELSE 'inactive' END FROM users;
-- ERROR: CASE types integer and text cannot be matched

-- RIGHT: consistent types
SELECT CASE WHEN active THEN '1' ELSE 'inactive' END FROM users;
-- or
SELECT CASE WHEN active THEN 1::text ELSE 'inactive' END FROM users;
```

### Wrong 5: Overlapping WHEN conditions — order matters

```sql
-- WRONG: salary = 100000 hits 'mid' branch and never reaches 'senior'
-- because the conditions overlap and CASE stops at first match
SELECT
  CASE
    WHEN salary >= 50000  THEN 'mid'     -- matches 100000 — stops here!
    WHEN salary >= 100000 THEN 'senior'  -- never reached for salary = 100000
    ELSE 'entry'
  END
FROM employees;

-- RIGHT: order from most specific (highest) to least specific
SELECT
  CASE
    WHEN salary >= 100000 THEN 'senior'  -- checked first
    WHEN salary >= 50000  THEN 'mid'     -- only reached if < 100000
    ELSE 'entry'
  END
FROM employees;
```

### Wrong 6: Using CASE where COALESCE is cleaner

```sql
-- CLUNKY: CASE for NULL substitution
SELECT CASE WHEN name IS NULL THEN 'Anonymous' ELSE name END FROM users;

-- CLEAN: COALESCE
SELECT COALESCE(name, 'Anonymous') FROM users;
```

---

## 10. CASE vs COALESCE vs NULLIF — Decision Table

| Situation | Use | Reason |
|-----------|-----|--------|
| Replace NULL with a default value | `COALESCE(col, default)` | Cleaner, purpose-explicit |
| Return NULL when two values are equal | `NULLIF(col, value)` | Cleaner than `CASE WHEN col = value THEN NULL ELSE col END` |
| Safe division (avoid divide-by-zero) | `col / NULLIF(denominator, 0)` | Single expression, clear intent |
| Multiple conditions → multiple outputs | `CASE WHEN ... THEN ... END` | COALESCE can't express conditions beyond NULL checking |
| First non-NULL from a list | `COALESCE(a, b, c, d)` | CASE would be verbose |
| Convert categories to labels | `CASE WHEN ... THEN ... END` | No alternative |
| Conditional aggregation | `CASE` inside aggregate or `FILTER` | COALESCE can't filter aggregation |
| NULL-safe equality check | `IS NOT DISTINCT FROM` | Not CASE, but worth knowing |

---

## 11. Performance Profile

| Usage | Cost Profile | Notes |
|-------|-------------|-------|
| CASE in SELECT | Low | Scalar evaluation per row — negligible vs I/O cost |
| CASE in WHERE | Non-sargable, medium | Full table scan or bitmap scan — cannot use plain index |
| CASE in ORDER BY | Sort cost dominant | CASE evaluation is cheap vs sort; concern is sort, not CASE |
| CASE in GROUP BY | Hash/sort aggregate cost dominant | CASE evaluation cheap; aggregate algorithm cost dominates |
| CASE in aggregate (pivot) | One scan for N aggregates | Dramatically better than N separate queries |
| Nested CASE (3+ levels) | Linear with nesting | Minimal overhead — complexity concern is readability, not CPU |
| CASE with subquery in WHEN | Can be high | Subquery may execute per row — test with EXPLAIN ANALYZE |

**The pivot advantage**: 5 conditional aggregates in one query = 1 table scan. 5 separate queries = 5 table scans. On a 100M-row table, this is the difference between a 2s query and a 10s report.

**CASE is never the bottleneck** — the scan, sort, or aggregate algorithm is the bottleneck. Optimise indexes and query structure, not the CASE expression itself.

---

## 12. Node.js Integration

### 12.1 Basic CASE query with pg

```javascript
import pg from 'pg';
const { Pool } = pg;

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// CASE in SELECT — status labels
async function getOrdersWithLabels(customerId) {
  const { rows } = await pool.query(
    `SELECT
       order_id,
       total_amount,
       CASE status
         WHEN 'pending'   THEN 'Awaiting payment'
         WHEN 'paid'      THEN 'Processing'
         WHEN 'shipped'   THEN 'On the way'
         WHEN 'delivered' THEN 'Complete'
         ELSE 'Unknown'
       END AS status_label
     FROM orders
     WHERE customer_id = $1
     ORDER BY created_at DESC`,
    [customerId]
  );
  return rows;
}
```

### 12.2 Conditional aggregation report

```javascript
async function getUserCohortReport(startDate) {
  const { rows } = await pool.query(
    `SELECT
       DATE_TRUNC('month', created_at)                                        AS cohort_month,
       COUNT(*)                                                               AS total,
       COUNT(CASE WHEN last_login >= NOW() - INTERVAL '30 days' THEN 1 END)  AS active,
       COUNT(CASE WHEN last_login <  NOW() - INTERVAL '90 days'
                   OR  last_login IS NULL THEN 1 END)                         AS churned,
       ROUND(
         COUNT(CASE WHEN last_login >= NOW() - INTERVAL '30 days' THEN 1 END)::numeric
         / NULLIF(COUNT(*), 0) * 100, 2
       ) AS retention_pct
     FROM users
     WHERE created_at >= $1
     GROUP BY DATE_TRUNC('month', created_at)
     ORDER BY cohort_month`,
    [startDate]
  );
  return rows;
}
```

### 12.3 Dynamic CASE for configurable sorting

```javascript
// Allow API callers to specify sort priority
const PRIORITY_ORDERS = {
  urgency: `CASE priority WHEN 'critical' THEN 1 WHEN 'high' THEN 2 WHEN 'medium' THEN 3 ELSE 4 END`,
  recency: `created_at DESC`,
  severity: `CASE severity WHEN 'P1' THEN 1 WHEN 'P2' THEN 2 ELSE 3 END`,
};

async function getTickets(sortBy = 'urgency') {
  // Allowlist approach — never interpolate user input directly
  const orderExpr = PRIORITY_ORDERS[sortBy] ?? PRIORITY_ORDERS.urgency;

  const { rows } = await pool.query(
    // Safe: orderExpr is from an allowlist, not user input
    `SELECT id, title, priority, severity, created_at
     FROM tickets
     WHERE resolved_at IS NULL
     ORDER BY ${orderExpr}, created_at ASC`
  );
  return rows;
}
```

### 12.4 Bulk conditional UPDATE

```javascript
// Apply tier-based discounts in one query
async function applyTierDiscounts() {
  const { rowCount } = await pool.query(
    `UPDATE products
     SET price = price * CASE tier
       WHEN 'gold'   THEN 0.85
       WHEN 'silver' THEN 0.90
       WHEN 'bronze' THEN 0.95
       ELSE 1.00
     END
     WHERE tier IN ('gold', 'silver', 'bronze')
       AND last_discount_at < NOW() - INTERVAL '7 days'`
  );
  return rowCount; // number of rows updated
}
```

---

## 13. ORM Comparison

### Prisma

**Can Prisma do CASE?** — Partially. Basic CASE via raw is trivial; Prisma has no typed abstraction for CASE expressions in its query builder.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

// Prisma cannot express CASE in its fluent API — raw SQL required
const orders = await prisma.$queryRaw<Order[]>(
  Prisma.sql`
    SELECT
      order_id,
      CASE status
        WHEN 'pending'   THEN 'Awaiting payment'
        WHEN 'delivered' THEN 'Complete'
        ELSE 'In progress'
      END AS status_label
    FROM orders
    WHERE customer_id = ${customerId}
  `
);

// Conditional aggregation — also raw SQL required
const report = await prisma.$queryRaw<CohortRow[]>(
  Prisma.sql`
    SELECT
      DATE_TRUNC('month', created_at) AS cohort_month,
      COUNT(*) AS total,
      COUNT(CASE WHEN last_login >= NOW() - INTERVAL '30 days' THEN 1 END) AS active
    FROM users
    GROUP BY DATE_TRUNC('month', created_at)
  `
);
```

**Where it breaks**: There is no `CASE` method in Prisma's query builder. You cannot mix CASE expressions with typed Prisma queries. `$queryRaw` returns `unknown[]` unless you provide generic type parameters.

**Verdict**: Use raw SQL for any query requiring CASE expressions. Prisma offers no benefit here.

---

### Drizzle ORM

**Can Drizzle do CASE?** — Yes, fully. Drizzle exposes a `sql` template tag and has typed CASE support via `drizzle-orm`.

```typescript
import { db } from './db';
import { orders, users } from './schema';
import { sql, eq, gte } from 'drizzle-orm';

// CASE in SELECT using sql template tag
const ordersWithLabels = await db.select({
  orderId: orders.orderId,
  totalAmount: orders.totalAmount,
  statusLabel: sql<string>`
    CASE ${orders.status}
      WHEN 'pending'   THEN 'Awaiting payment'
      WHEN 'delivered' THEN 'Complete'
      ELSE 'In progress'
    END
  `.as('status_label'),
}).from(orders);

// Conditional aggregation
const cohortReport = await db.select({
  cohortMonth: sql`DATE_TRUNC('month', ${users.createdAt})`.as('cohort_month'),
  total: sql<number>`COUNT(*)`.as('total'),
  active: sql<number>`COUNT(CASE WHEN ${users.lastLogin} >= NOW() - INTERVAL '30 days' THEN 1 END)`.as('active'),
}).from(users)
  .groupBy(sql`DATE_TRUNC('month', ${users.createdAt})`);
```

**Where it breaks**: Complex nested CASE with subqueries requires full `sql` template usage. Type inference for CASE return types must be manually specified via generics.

**Verdict**: Drizzle handles CASE well via the `sql` template tag. Use it — no need to drop to raw pool queries.

---

### Sequelize

**Can Sequelize do CASE?** — Partially, via `literal()`. No typed CASE builder.

```javascript
const { Op, literal, fn, col } = require('sequelize');
const { Order, User } = require('./models');

// CASE using literal() — no type safety
const orders = await Order.findAll({
  attributes: [
    'orderId',
    'totalAmount',
    [
      literal(`CASE status
        WHEN 'pending'   THEN 'Awaiting payment'
        WHEN 'delivered' THEN 'Complete'
        ELSE 'In progress'
      END`),
      'statusLabel',
    ],
  ],
  where: { customerId },
});

// Conditional aggregation — also via literal
const report = await User.findAll({
  attributes: [
    [fn('DATE_TRUNC', 'month', col('created_at')), 'cohortMonth'],
    [fn('COUNT', col('*')), 'total'],
    [
      literal(`COUNT(CASE WHEN last_login >= NOW() - INTERVAL '30 days' THEN 1 END)`),
      'active',
    ],
  ],
  group: [fn('DATE_TRUNC', 'month', col('created_at'))],
});
```

**Where it breaks**: `literal()` strings are raw SQL with no validation, no type safety, no column reference tracking. Column name changes silently break these queries. Nested aggregates can hit Sequelize's unpredictable SQL generation.

**Verdict**: Use `sequelize.query()` for queries with significant CASE logic. `literal()` works for simple cases but is fragile in complex queries.

---

### TypeORM

**Can TypeORM do CASE?** — Partially, via `QueryBuilder` and raw expressions.

```typescript
import { DataSource } from 'typeorm';
import { Order } from './entities/Order';

const dataSource = new DataSource({ /* config */ });

// CASE in QueryBuilder using addSelect with raw expression
const orders = await dataSource
  .getRepository(Order)
  .createQueryBuilder('order')
  .select('order.orderId', 'orderId')
  .addSelect('order.totalAmount', 'totalAmount')
  .addSelect(
    `CASE order.status
      WHEN 'pending'   THEN 'Awaiting payment'
      WHEN 'delivered' THEN 'Complete'
      ELSE 'In progress'
    END`,
    'statusLabel'
  )
  .where('order.customerId = :customerId', { customerId })
  .getRawMany();

// Conditional aggregation
const report = await dataSource
  .getRepository(User)
  .createQueryBuilder('user')
  .select(`DATE_TRUNC('month', user.createdAt)`, 'cohortMonth')
  .addSelect('COUNT(*)', 'total')
  .addSelect(
    `COUNT(CASE WHEN user.lastLogin >= NOW() - INTERVAL '30 days' THEN 1 END)`,
    'active'
  )
  .groupBy(`DATE_TRUNC('month', user.createdAt)`)
  .getRawMany();
```

**Where it breaks**: Raw string expressions have no type safety. `getRawMany()` returns `any[]`. TypeORM's entity mapping does not apply to raw expression results.

**Verdict**: TypeORM's QueryBuilder handles CASE adequately for moderate complexity. For large pivot queries, drop to `dataSource.query()` for clarity.

---

### Knex.js

**Can Knex do CASE?** — Fully, via `knex.raw()` interpolated within query builder, or full raw queries.

```javascript
const knex = require('knex')({ client: 'pg', connection: process.env.DATABASE_URL });

// CASE using knex.raw() in select
const orders = await knex('orders')
  .select(
    'order_id',
    'total_amount',
    knex.raw(`
      CASE status
        WHEN 'pending'   THEN 'Awaiting payment'
        WHEN 'delivered' THEN 'Complete'
        ELSE 'In progress'
      END AS status_label
    `)
  )
  .where('customer_id', customerId)
  .orderBy('created_at', 'desc');

// Conditional aggregation pivot
const report = await knex('users')
  .select(
    knex.raw(`DATE_TRUNC('month', created_at) AS cohort_month`),
    knex.raw(`COUNT(*) AS total`),
    knex.raw(`COUNT(CASE WHEN last_login >= NOW() - INTERVAL '30 days' THEN 1 END) AS active`),
    knex.raw(`COUNT(CASE WHEN last_login < NOW() - INTERVAL '90 days' OR last_login IS NULL THEN 1 END) AS churned`)
  )
  .where('created_at', '>=', startDate)
  .groupByRaw(`DATE_TRUNC('month', created_at)`)
  .orderByRaw(`DATE_TRUNC('month', created_at)`);
```

**Where it breaks**: `knex.raw()` strings are untyped. Complex queries with many CASE expressions become harder to maintain than equivalent raw SQL strings. No structural validation.

**Verdict**: Knex handles CASE cleanly. For pivot-heavy queries, a full `knex.raw()` string is often more readable than many `.select(knex.raw(...))` calls interspersed through the query builder.

---

### ORM Summary Table

| ORM | CASE in SELECT | Conditional Aggregation | Typed? | Recommendation |
|-----|---------------|------------------------|--------|---------------|
| Prisma | `$queryRaw` only | `$queryRaw` only | No | Raw SQL |
| Drizzle | `sql` template | `sql` template | Partial | Use `sql` tag |
| Sequelize | `literal()` | `literal()` | No | `sequelize.query()` for complex |
| TypeORM | `addSelect(raw)` | `addSelect(raw)` | No | QueryBuilder for simple; `dataSource.query()` for complex |
| Knex | `knex.raw()` | `knex.raw()` | No | Mix builder + raw works well |

---

## 14. Practice Exercises

### Exercise 1 — Basic

Given a `products` table with columns: `id`, `name`, `price`, `stock_count`, `category`.

Write a query that returns each product with a `stock_status` column containing:
- `'out_of_stock'` when `stock_count = 0`
- `'low_stock'` when `stock_count` between 1 and 10 (inclusive)
- `'in_stock'` when `stock_count > 10`

Also add an `affordability` column:
- `'budget'` when price < 25
- `'mid_range'` when price between 25 and 100
- `'premium'` when price > 100

Order results by: premium first, then mid_range, then budget (use CASE in ORDER BY).

---

### Exercise 2 — Intermediate

Given a `support_tickets` table: `id`, `created_at`, `resolved_at` (nullable), `priority` (`'low'`/`'medium'`/`'high'`/`'critical'`), `agent_id`.

Write a single query that returns one row per `agent_id` with these columns:
- `total_tickets` — all tickets
- `open_tickets` — where `resolved_at IS NULL`
- `resolved_tickets` — where `resolved_at IS NOT NULL`
- `critical_resolved` — resolved critical tickets
- `avg_resolution_hours` — average hours to resolve (only for resolved tickets, NULL-safe)
- `performance_tier`:
  - `'excellent'` if resolved_tickets >= 50 AND avg resolution < 4 hours
  - `'good'` if resolved_tickets >= 20 AND avg resolution < 8 hours
  - `'needs_improvement'` otherwise

---

### Exercise 3 — Intermediate

Given a `sales` table: `id`, `sale_date`, `product_id`, `region` (`'north'`/`'south'`/`'east'`/`'west'`), `amount`.

Write a pivot query returning one row per `product_id` with columns:
- `product_id`
- `north_revenue`, `south_revenue`, `east_revenue`, `west_revenue`
- `total_revenue`
- `strongest_region` — the region name with the highest revenue for that product (use CASE comparing the four values)
- `growth_flag` — `'growing'` if Q4 revenue > Q1 revenue, `'declining'` if Q4 < Q1, `'stable'` otherwise (Q1 = Jan–Mar, Q4 = Oct–Dec, use sale_date)

---

### Exercise 4 — Production Grade

You have a `users` table: `id`, `email`, `created_at`, `last_login`, `plan` (`'free'`/`'pro'`/`'enterprise'`), `country`.

Write a query that produces a marketing segmentation report with **one row per user** containing:
- `id`, `email`
- `plan_label` — `'Free'`, `'Pro ($29/mo)'`, `'Enterprise'`
- `engagement_segment`:
  - `'champion'` — pro/enterprise AND last_login within 7 days
  - `'loyal'` — last_login within 30 days
  - `'at_risk'` — last_login between 30 and 90 days ago
  - `'dormant'` — last_login > 90 days ago or never logged in
- `upgrade_candidate` — `TRUE` if free plan AND last_login within 14 days AND country IN ('US', 'GB', 'CA', 'AU'), else `FALSE`
- `priority_score` — numeric:
  - enterprise champion: 100
  - pro champion: 80
  - enterprise loyal: 70
  - pro loyal: 60
  - free upgrade_candidate: 50
  - everything else: 10

Order by `priority_score` DESC, then `last_login` DESC NULLS LAST.

---

## 15. Interview Questions

### Junior Level

**Q: What is the difference between a simple CASE and a searched CASE?**

A: A simple CASE evaluates one expression and compares it to a list of values using equality (`=`). A searched CASE evaluates each WHEN as an independent boolean condition and allows any comparison operators (`>`, `<`, `LIKE`, `IS NULL`, etc.). Simple CASE is shorthand for searched CASE with equality comparisons.

---

**Q: What happens if no WHEN matches and there is no ELSE?**

A: `NULL` is returned. This is a common source of bugs — data that doesn't match any condition silently produces NULL instead of a default value. Always include ELSE unless NULL is explicitly the intended output.

---

**Q: Can you use a CASE alias defined in SELECT inside a WHERE clause?**

A: No. WHERE executes before SELECT in the logical processing order, so aliases defined in SELECT are not yet available when WHERE is evaluated. You must either repeat the CASE expression in WHERE or wrap the query in a subquery/CTE and filter on the alias in the outer query.

---

### Principal Level

**Q: CASE in WHERE is non-sargable. When is it still acceptable to use it?**

A: When the alternative — an equivalent OR/AND expression — would be equally non-sargable, or when the query has low frequency, the table is small, or there is no appropriate index to leverage anyway. The key is knowing *why* it's non-sargable (the planner cannot determine a key range for an index scan from a conditional expression that yields different columns based on row values) and making the trade-off explicitly. For high-frequency queries on large tables, rewrite to sargable form or use a partial index that materialises the condition.

---

**Q: Explain the performance trade-off of the conditional aggregation (CASE in aggregate) pivot pattern versus running N separate queries.**

A: Conditional aggregation — `SUM(CASE WHEN region = 'North' THEN amount END)` repeated for each region — executes in a single sequential scan with O(N) memory for the hash aggregate. N separate queries each require their own sequential scan, their own query planning cycle, and their own round-trip from the application. For a 100M-row table, one query at 2 seconds is categorically better than 10 queries at 2 seconds each. The trade-off is query complexity: a single pivot query with 15 CASE expressions is harder to modify than individual queries. In practice, pivot queries should be wrapped in a view or function to manage this complexity.

---

**Q: How does PostgreSQL's short-circuit guarantee on CASE interact with volatile functions? Give a concrete example.**

A: PostgreSQL guarantees that CASE short-circuits — branches after the first match are never evaluated. This means a volatile function (one whose result can vary per call, like `random()`, `NOW()`, or a function with side effects) in a WHEN branch will only execute for rows where all preceding conditions were false. This is safe and intentional: place cheap, high-selectivity conditions early to skip expensive evaluation. An important application is safe division: `CASE WHEN denominator = 0 THEN NULL ELSE num / denominator END` is safe precisely because the division is never evaluated when the guard condition matches — there is no "evaluate all branches, then pick one" behaviour.

---

**Q: You have a query that uses CASE to compute a `user_tier` column, and you need to filter for `user_tier = 'gold'`. What are the three ways to do this, and when would you choose each?**

A: 
1. **Subquery/CTE**: Wrap in `SELECT * FROM (...) WHERE user_tier = 'gold'`. The planner may or may not push the filter down depending on cost — in PostgreSQL this is often equivalent to option 2 but adds clarity.
2. **Repeat the CASE in WHERE**: `WHERE CASE WHEN ... THEN 'gold' ... END = 'gold'`. Avoids a subquery layer. Non-sargable, but explicit.
3. **Generated column**: `ALTER TABLE users ADD COLUMN user_tier TEXT GENERATED ALWAYS AS (CASE WHEN ...) STORED` — then `WHERE user_tier = 'gold'` IS sargable and an index on `user_tier` can be used. This is the right choice when the filter is frequent and the table is large. The trade-off: write amplification on INSERT/UPDATE.

---

## 16. Mental Model Checkpoint

1. **Where in the execution order does CASE in SELECT run, and what does that mean for referencing its alias in WHERE?**

2. **A searched CASE has no ELSE. A user's `account_type` value is `'trial'` and none of the WHEN conditions match it. What does CASE return?**

3. **You write: `CASE WHEN denominator = 0 THEN NULL ELSE value / denominator END`. Is the division evaluated when `denominator = 0`? What SQL feature guarantees this?**

4. **What is the performance advantage of `SUM(CASE WHEN region = 'East' THEN amount ELSE 0 END), SUM(CASE WHEN region = 'West' THEN amount ELSE 0 END)` in a single query versus two separate `SUM` queries?**

5. **When should you prefer `COALESCE(col, default)` over `CASE WHEN col IS NULL THEN default ELSE col END`?**

6. **You have a query with `CASE priority WHEN 'high' THEN 1 WHEN 'medium' THEN 2 ELSE 3 END` in ORDER BY. The `priority` column has an index. Does the sort use the index? Why or why not?**

7. **What is the difference in NULL handling between `COUNT(CASE WHEN status = 'active' THEN 1 END)` and `COUNT(CASE WHEN status = 'active' THEN 1 ELSE 0 END)`?**

---

## 17. Quick Reference Card

```sql
-- Simple CASE (equality match)
CASE expression
  WHEN value1 THEN result1
  WHEN value2 THEN result2
  ELSE default       -- always include ELSE
END

-- Searched CASE (arbitrary conditions)
CASE
  WHEN condition1 THEN result1
  WHEN condition2 THEN result2
  ELSE default
END

-- CASE in aggregate (conditional aggregation / pivot)
SUM(CASE WHEN region = 'East' THEN amount ELSE 0 END)
COUNT(CASE WHEN status = 'active' THEN 1 END)  -- NULL ELSE: NULLs not counted
AVG(CASE WHEN gender = 'F' THEN salary END)    -- NULL ELSE: AVG ignores NULLs

-- CASE vs FILTER (PostgreSQL 9.4+)
COUNT(CASE WHEN status = 'active' THEN 1 END)  -- portable
COUNT(*) FILTER (WHERE status = 'active')      -- PostgreSQL idiomatic, cleaner

-- CASE in ORDER BY (custom sort)
ORDER BY CASE priority WHEN 'critical' THEN 1 WHEN 'high' THEN 2 ELSE 3 END

-- CASE in UPDATE
UPDATE t SET price = price * CASE tier WHEN 'gold' THEN 0.85 ELSE 1 END

-- Safe division (prefer NULLIF)
numerator / NULLIF(denominator, 0)

-- COALESCE = CASE WHEN IS NULL (use COALESCE)
COALESCE(col, 'default')

-- NULL trap: simple CASE cannot match NULL
CASE col WHEN NULL THEN ...   -- NEVER matches
CASE WHEN col IS NULL THEN ... -- CORRECT

-- Alias not available in WHERE
SELECT CASE ... END AS band FROM t WHERE band = 'x'  -- ERROR
SELECT * FROM (SELECT CASE ... END AS band FROM t) s WHERE band = 'x'  -- CORRECT

-- PostgreSQL: alias usable in GROUP BY (extension)
GROUP BY band   -- OK in PostgreSQL after SELECT ... AS band
```

---

## Connected Topics

- **Topic 07 — NULL in Depth**: CASE has specific NULL behaviour — the NULL trap in simple CASE, ELSE omission producing NULL, COALESCE as NULL-specific CASE shorthand.
- **Topic 06 — WHERE Clause Mastery**: CASE in WHERE and its non-sargability; OR as the sargable alternative.
- **Topic 08 — Filtering Performance & Sargability**: Why CASE in WHERE cannot use indexes; the generated column solution.
- **Topic 23 — FILTER Clause**: The PostgreSQL-specific aggregate filter syntax that competes with CASE inside aggregates.
- **Topic 61 — Pivot and Crosstab**: CASE + GROUP BY is the manual pivot pattern; `crosstab()` is the function-based alternative.
- **Topic 20 — GROUP BY Fundamentals**: CASE in GROUP BY creates derived grouping buckets.
- **Topic 46 — Expression Indexes**: If you filter frequently on a CASE result, a generated column + index is the correct solution.
