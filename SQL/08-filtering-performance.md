# Topic 08 — Filtering Performance & Sargability

**Phase 2 — SELECT and Filtering Foundations**
**Interview Priority: HIGH**

---

## ELI5 Analogy

A phone book is sorted alphabetically by last name. If you want to find everyone named "Smith," you flip to S and scan forward — done in seconds. That's a **sargable** condition: the index can seek directly to what you need.

Now imagine someone asks: "Find everyone whose last name backwards spells something starting with 'S'." To answer this, you must read every single entry in the phone book, reverse the name, and check. The phone book's alphabetical ordering is completely useless. That's a **non-sargable** condition — the index exists, but the query can't use it.

In SQL:
- `WHERE last_name = 'Smith'` → sargable (index lookup)
- `WHERE reverse(last_name) LIKE 'S%'` → non-sargable (full scan, index ignored)

The terrifying part: PostgreSQL will tell you nothing about this difference unless you EXPLAIN the query. Both run without error. One takes 0.2ms, the other takes 8 seconds. On a 10-million-row table.

---

## Internals Connection

PostgreSQL's index access methods (B-tree, Hash, GIN, GiST) can only serve a query if the query presents a **bound** on the indexed value in a form the access method understands.

For B-tree indexes (by far the most common), the access method understands:
- `column operator constant` — where the operator is one of `=`, `<`, `>`, `<=`, `>=`, `BETWEEN`, `IN`, `IS NULL`, `IS NOT NULL`, `LIKE 'prefix%'`

The column must appear **alone on one side of the operator** — not wrapped in a function, not cast, not concatenated with anything. This is the sargability rule.

When PostgreSQL's planner evaluates a WHERE condition, it tries to extract **indexquals** — conditions that can be pushed down into an index scan. If it can extract them, it generates an Index Scan or Bitmap Index Scan plan. If not, it falls back to a Seq Scan and applies the condition as a **Filter** (evaluated per heap page).

The planner identifies indexquals by matching expressions against index definitions. For an expression index `CREATE INDEX ON t (lower(email))`, the expression `lower(email) = $1` IS sargable because it matches the index definition exactly.

---

## Execution Order Context

```
FROM + JOINs  →  [WHERE]  →  GROUP BY  →  HAVING  →  SELECT  →  DISTINCT  →  WINDOW  →  ORDER BY  →  LIMIT
                    ↑
        Sargability affects what happens HERE
        Can the WHERE condition delegate filtering to the index,
        or must PostgreSQL read every heap page first?
```

Sargability is a property of **WHERE clause conditions** evaluated at Step 2. It determines whether PostgreSQL can use an index to avoid reading table pages entirely, or whether it must perform a sequential scan and filter rows one by one.

A non-sargable WHERE condition doesn't mean the query fails — it means performance degrades from O(log N + result_size) to O(table_size).

---

## What Is Sargability?

**Sargable** stands for **S**earch **ARG**ument **ABLE** — a condition that can be used as a search argument to an index access method.

A WHERE condition is sargable if:
1. It compares an **indexed column** (or expression matching an expression index)
2. The column appears **alone on one side** of the comparison
3. The operator is one the index access method understands
4. No function wraps the column (unless an expression index exactly matches)
5. No implicit type cast is forced on the column side

A WHERE condition is **not sargable** if:
1. A function wraps the indexed column: `WHERE upper(name) = 'ALICE'`
2. The column is cast: `WHERE id::text = '42'`
3. The column is in an arithmetic expression: `WHERE price * 1.1 > 100`
4. A leading wildcard is used: `WHERE name LIKE '%alice'`
5. A type mismatch forces implicit casting of the column: `WHERE int_col = '42'` (context-dependent)
6. The condition uses a non-indexable operator for that index type

---

## Why It Matters in Production

### The Hidden Performance Cliff

```sql
-- These two queries look nearly identical to a developer:
WHERE email = 'alice@example.com'         -- sargable: 0.1ms
WHERE lower(email) = 'alice@example.com'  -- non-sargable: 8,400ms (on 5M rows)

-- Same data, same index (email), same result set (1 row)
-- 84,000x performance difference
-- Zero indication of the problem until production load hits
```

### Sargability Determines System Capacity

On a 5-million-row users table with 1,000 concurrent API requests per second, every endpoint that hits `WHERE lower(email) = $1` instead of `WHERE email = lower($1)` is doing a full table scan. That's 1,000 full scans per second. Your database is now the bottleneck for the entire system.

The fix is one character change in the schema (lowercase emails at write time) or one line in a migration (add expression index). The diagnostics require understanding sargability.

---

## Deep Technical Content

### Category 1: Functions on Columns

Any function wrapping an indexed column makes the condition non-sargable for that column's regular index:

```sql
-- NON-SARGABLE (column is hidden inside function)
WHERE upper(last_name) = 'SMITH'
WHERE lower(email) = 'user@example.com'
WHERE TRIM(phone) = '555-1234'
WHERE LENGTH(description) > 100
WHERE EXTRACT(YEAR FROM created_at) = 2024
WHERE DATE(created_at) = '2024-01-15'
WHERE created_at::date = '2024-01-15'
WHERE TO_CHAR(created_at, 'YYYY-MM') = '2024-01'
WHERE ABS(score - 100) < 5
WHERE SUBSTRING(code, 1, 3) = 'ABC'
WHERE ROUND(price, 0) = 10

-- SARGABLE rewrites:
-- Option A: apply function to the constant (not the column)
WHERE last_name = UPPER('Smith')              -- column untouched
WHERE email = lower('User@Example.com')       -- column untouched
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'  -- range instead of EXTRACT

-- Option B: create an expression index matching the function
CREATE INDEX idx_users_email_lower ON users(lower(email));
WHERE lower(email) = 'user@example.com'       -- now matches the expression index
```

**Why expression indexes work**: When PostgreSQL evaluates `lower(email)`, it checks if any index was built on the expression `lower(email)`. If yes, it can use that index as if `lower(email)` were a plain column. The expression index stores pre-computed values of `lower(email)` in sorted order.

**The maintenance cost**: Expression indexes are maintained on every INSERT and UPDATE to the underlying column. `lower(email)` is cheap; `bcrypt(password)` would be catastrophic.

---

### Category 2: Arithmetic on Columns

```sql
-- NON-SARGABLE
WHERE price * 1.1 > 100       -- column in arithmetic expression
WHERE salary + bonus > 100000
WHERE age * 12 > 360           -- months of age > 360 (age > 30)
WHERE quantity / 2 > 50

-- SARGABLE rewrites (move arithmetic to the constant side):
WHERE price > 100 / 1.1                        -- = WHERE price > 90.909...
WHERE price > 90.91                             -- or pre-compute
WHERE salary + bonus > 100000                  -- can't trivially separate if both columns
-- For multi-column arithmetic, consider a generated column or expression index:
ALTER TABLE employees ADD COLUMN total_comp NUMERIC GENERATED ALWAYS AS (salary + bonus) STORED;
CREATE INDEX idx_employees_total_comp ON employees(total_comp);
WHERE total_comp > 100000                       -- sargable via generated column
```

---

### Category 3: Type Mismatches

This is the subtle one. Type mismatches force PostgreSQL to cast one side of the comparison. The question is **which side** gets cast.

**Safe case — PostgreSQL casts the constant, not the column:**
```sql
-- Column is INT, literal is text
WHERE id = '42'
-- PostgreSQL casts '42' to INT: id = 42
-- Column is untouched → index can be used ✓

-- Column is INT, parameter is text (safe in most pg drivers)
WHERE id = $1  -- and $1 has value '42' of type text
-- PostgreSQL casts $1 to INT → sargable ✓
```

**Dangerous case — column must be cast, not the constant:**
```sql
-- Column is VARCHAR, you compare to a number
WHERE code = 42
-- PostgreSQL casts code to numeric: CAST(code AS NUMERIC) = 42
-- Column is wrapped in cast → non-sargable ✗

-- Implicit cast on timestamp
WHERE created_at = '2024-01-15'
-- '2024-01-15' could be parsed as DATE or TIMESTAMP
-- If ambiguous, PostgreSQL may cast the column → potentially non-sargable
-- Correct: WHERE created_at = '2024-01-15'::timestamp (explicit cast on constant)
```

**The parameter binding type mismatch in Node.js:**
```javascript
// pg sends parameters with type hints
// If the DB column is INT4 and you send a JS string '42':
pool.query('SELECT * FROM users WHERE id = $1', ['42'])
// pg may send $1 as TEXT type
// PostgreSQL infers and may cast $1 to INT (safe) or may require explicit cast

// SAFEST: always match types
pool.query('SELECT * FROM users WHERE id = $1', [42])      // number → INT4
pool.query('SELECT * FROM users WHERE id = $1::int', ['42'])  // explicit cast on parameter
```

**The ORM hidden type mismatch:**
```sql
-- Prisma/TypeORM schema: userId Int (PostgreSQL: user_id INTEGER)
-- ORM sends: WHERE user_id = '123' (string from URL param)
-- PostgreSQL: implicit cast — might work but generates non-optimal plan
-- Fix: parse to integer in application before passing to ORM
```

---

### Category 4: LIKE Patterns

```sql
-- SARGABLE (prefix match — PostgreSQL can range-scan the index)
WHERE name LIKE 'abc%'       -- seeks to 'abc', scans to 'abd'
WHERE name LIKE 'abc'        -- equality (no wildcard) — index lookup ✓

-- NON-SARGABLE
WHERE name LIKE '%abc'       -- suffix match — must check every row ✗
WHERE name LIKE '%abc%'      -- contains match — must check every row ✗
WHERE name LIKE '_abc'       -- single-char prefix wildcard ✗ (can't range-scan)

-- ILIKE is ALWAYS non-sargable for regular indexes
WHERE name ILIKE 'abc%'      -- case-insensitive — must check every row ✗
WHERE name ILIKE '%abc%'     -- ✗

-- Workarounds for non-sargable LIKE patterns:

-- Option A: suffix match → reverse index
CREATE INDEX idx_products_name_reversed ON products(reverse(name));
WHERE reverse(name) LIKE reverse('%smith')
-- = WHERE reverse(name) LIKE 'htims%'    → now a prefix match ✓

-- Option B: contains/ILIKE → pg_trgm trigram index
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_products_name_trgm ON products USING GIN(name gin_trgm_ops);
WHERE name ILIKE '%abc%'    -- now uses trigram GIN index ✓
WHERE name LIKE '%abc%'     -- also uses trigram index ✓
-- Minimum 3 characters for trigram to be effective
-- Less effective for short strings (< 3 chars)

-- Option C: case-insensitive prefix → lower() expression index
CREATE INDEX idx_users_name_lower ON users(lower(name));
WHERE lower(name) LIKE lower('abc') || '%'
-- or simpler:
WHERE lower(name) LIKE 'abc%'   -- sargable against lower(name) index ✓
```

**LIKE with operator class and collation:**
```sql
-- Default B-tree index uses text_ops which supports LIKE prefix matching
-- ONLY for the database's default collation (usually C or en-US)
-- If your DB uses a locale-sensitive collation (e.g. 'en_US.UTF-8'),
-- standard B-tree index may not support LIKE:

-- Check if LIKE can use the index:
EXPLAIN SELECT * FROM users WHERE name LIKE 'abc%';
-- If you see Filter instead of Index Cond, the collation is blocking it

-- Fix: use text_pattern_ops operator class
CREATE INDEX idx_users_name_pattern ON users(name text_pattern_ops);
-- Now LIKE 'prefix%' uses this index even with locale-sensitive collation
-- BUT: this index cannot be used for ORDER BY or range (<, >) operations
```

---

### Category 5: OR Conditions and Index Unions

```sql
-- Can this use indexes?
WHERE status = 'active' OR status = 'pending'
-- YES: PostgreSQL uses Bitmap Index Scan + BitmapOr
-- Scans index for 'active', scans index for 'pending', ORs the result bitmaps

WHERE id = 1 OR id = 2 OR id = 3
-- YES: treated as IN (1, 2, 3) → Bitmap Index Scan

WHERE status = 'active' OR category_id = 5
-- MAYBE: depends on selectivity
-- If both columns are indexed, PostgreSQL may use two Bitmap Index Scans + BitmapOr
-- If one is not selective enough, it may fall back to Seq Scan

WHERE status = 'active' OR user_id IS NOT NULL
-- Likely NOT: if most rows have user_id IS NOT NULL, this matches almost everything
-- Seq Scan would be faster than an index scan returning 99% of rows
```

**The OR with non-sargable condition:**
```sql
-- If ANY condition in an OR is non-sargable, the entire OR may be non-sargable
WHERE id = 5 OR lower(email) = 'test@example.com'
-- The second condition can't use a regular index on email
-- If PostgreSQL has an index on lower(email), it can BitmapOr both
-- If not, it falls back to Seq Scan (can't use half-OR with index, half without)
```

---

### Category 6: NOT Conditions

```sql
-- NOT equal is usually non-sargable for large result sets
WHERE status != 'deleted'
-- If 99% of rows have status != 'deleted', the index returns 99% of rows
-- PostgreSQL: "a Seq Scan would be faster than reading 99% via index"
-- Result: Seq Scan (correct choice by planner)

-- IS NOT NULL can use index if NULL rows are frequent
WHERE deleted_at IS NOT NULL
-- Index scan useful only if many rows HAVE deleted_at (i.e., many deletions)
-- If only 1% of rows are deleted, this index helps for finding deleted
-- For finding non-deleted (99%), seq scan is usually faster

-- NOT LIKE — always non-sargable
WHERE name NOT LIKE 'test%'
-- Can't use B-tree for NOT LIKE — must check every row
```

---

### Category 7: Range Conditions and Multi-Column Sargability

```sql
-- Range conditions on B-tree indexes
WHERE created_at > '2024-01-01'           -- sargable: range scan ✓
WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31'  -- sargable ✓

-- Composite index: (status, created_at)
WHERE status = 'active' AND created_at > '2024-01-01'
-- Sargable: equality on first column, range on second ✓
-- Index: seeks to status='active', then range-scans by created_at

-- IMPORTANT: range condition on first column limits second column's index use
WHERE created_at > '2024-01-01' AND status = 'active'
-- Even with index (created_at, status):
-- Range on created_at first → can't use index on status ✗
-- The planner may still use the index for created_at alone

-- Rule: equality conditions first in composite index, range conditions last
-- Index design: (status, created_at) for this query pattern, not (created_at, status)
```

---

### Category 8: Implicit Casts in PostgreSQL — The Full Reference

Not all type mismatches break sargability. PostgreSQL has a cast hierarchy:

| Column type | Comparison value | Direction | Sargable? |
|-------------|-----------------|-----------|-----------|
| `INT` | Integer literal `42` | Exact match | ✓ |
| `INT` | Text `'42'` | Cast text→int (constant cast) | ✓ |
| `TEXT` | Integer `42` | Cast text→int on column | ✗ |
| `BIGINT` | `INT` value | Upcast constant | ✓ |
| `INT` | `BIGINT` value | May downcast column | Context-dependent |
| `TIMESTAMPTZ` | `'2024-01-01'` | PostgreSQL infers type | ✓ (usually) |
| `TIMESTAMP` | `'2024-01-01'::date` | Cross-type cast | May be ✗ |
| `NUMERIC` | `FLOAT` value | May cast column to float | ✗ |
| `VARCHAR(255)` | `TEXT` value | Cast direction: text→varchar? | Usually ✓ |
| `CHAR(10)` | `'abc'` | Padding rules | Usually ✓ |

**The safest rule**: cast the parameter/constant to match the column type exactly:
```sql
-- Column: user_id BIGINT
WHERE user_id = $1::bigint    -- explicit cast on parameter ✓

-- Column: created_at TIMESTAMPTZ
WHERE created_at = $1::timestamptz   -- explicit cast ✓

-- Column: status TEXT
WHERE status = $1::text              -- explicit (usually not needed, but safe) ✓
```

---

### Category 9: Sargability in JOINs

JOIN ON conditions have the same sargability rules as WHERE:

```sql
-- SARGABLE JOIN condition (both sides are plain columns from their tables)
JOIN orders o ON o.user_id = u.id              -- ✓ index on o.user_id usable

-- NON-SARGABLE JOIN condition
JOIN orders o ON upper(o.reference) = upper(u.ref_code)  -- ✗ functions on both sides

-- NON-SARGABLE
JOIN orders o ON o.created_at::date = u.signup_date  -- ✗ cast on column

-- SARGABLE fix
JOIN orders o ON o.created_at >= u.signup_date AND o.created_at < u.signup_date + INTERVAL '1 day'
```

---

### Category 10: The COALESCE Sargability Trap (from Topic 07)

```sql
-- NON-SARGABLE: COALESCE wraps the column
WHERE COALESCE(status, 'unknown') = 'active'
-- PostgreSQL must evaluate COALESCE(status, ...) for every row

-- SARGABLE rewrites:
WHERE status = 'active'                          -- if NULLs can't be 'active'
WHERE status = 'active' OR status IS NULL        -- if NULLs should be treated as 'active'
WHERE status IS NOT DISTINCT FROM 'active'       -- null-safe equality (may not use index)
```

---

### How to Diagnose Sargability Issues

The two-step diagnosis:

**Step 1: Look for Filter vs Index Cond in EXPLAIN**
```
-- EXPLAIN output showing non-sargable condition:
Seq Scan on users
  Filter: (lower((email)::text) = 'alice@example.com'::text)
  Rows Removed by Filter: 499999
  Buffers: shared hit=2206 read=2793

-- vs sargable condition:
Index Scan using idx_users_email on users
  Index Cond: ((email)::text = 'alice@example.com'::text)
  Buffers: shared hit=4
```

`Filter` = applied after heap page read = non-sargable (or a secondary filter)
`Index Cond` = pushed into index = sargable

**Step 2: Check actual vs estimated rows for anomalies**
```
Seq Scan on orders (... rows=10000 ...) (actual ... rows=1 ...)
-- Estimated 10,000 rows but got 1
-- Mismatch = planner chose wrong plan
-- Fix: ANALYZE table, or create expression index
```

---

## EXPLAIN Output Examples

### Non-sargable vs Sargable on Same Column

```sql
-- NON-SARGABLE
EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM users WHERE lower(email) = 'alice@example.com';
```
```
Seq Scan on users  (cost=0.00..2794.00 rows=250 width=4)
                   (actual time=0.021..52.341 rows=1 loops=1)
  Filter: (lower((email)::text) = 'alice@example.com'::text)
  Rows Removed by Filter: 499999
  Buffers: shared hit=2206 read=588
Planning Time: 0.112 ms
Execution Time: 52.362 ms
```

```sql
-- After: CREATE INDEX idx_users_email_lower ON users(lower(email));

EXPLAIN (ANALYZE, BUFFERS)
SELECT id FROM users WHERE lower(email) = 'alice@example.com';
```
```
Index Scan using idx_users_email_lower on users  (cost=0.55..8.57 rows=1 width=4)
                                                  (actual time=0.034..0.035 rows=1 loops=1)
  Index Cond: (lower((email)::text) = 'alice@example.com'::text)
  Buffers: shared hit=4
Planning Time: 0.198 ms
Execution Time: 0.055 ms
```

**Before: 52ms, reading 2,794 buffers. After: 0.055ms, reading 4 buffers. 946x improvement.**

### LIKE Patterns Compared

```sql
EXPLAIN SELECT id FROM products WHERE name LIKE 'Widget%';
```
```
Index Scan using idx_products_name on products
  Index Cond: ((name >= 'Widget') AND (name < 'Widgeu'))
  -- Note: PostgreSQL converts LIKE 'Widget%' to range (>= 'Widget' AND < next prefix)
  Rows Removed by Index Recheck: 0
```

```sql
EXPLAIN SELECT id FROM products WHERE name LIKE '%Widget';
```
```
Seq Scan on products
  Filter: ((name)::text ~~ '%Widget'::text)
  Rows Removed by Filter: 49823
```

PostgreSQL explicitly converts prefix LIKE to a range condition — this is why it's sargable. Suffix LIKE has no equivalent range conversion.

---

## Query Examples

### Example 1 — Basic: Finding and Fixing a Non-sargable Condition

```sql
-- SCENARIO: API endpoint doing case-insensitive username lookup
-- Reports show this endpoint is slow at scale

-- SLOW (non-sargable):
SELECT id, username, email, role
FROM users
WHERE upper(username) = upper($1)
  AND deleted_at IS NULL;

-- EXPLAIN shows: Seq Scan with Filter — full table scan every request

-- FIX OPTION A: Normalize data at write time (preferred)
-- 1. Enforce lowercase on INSERT/UPDATE:
ALTER TABLE users ADD CONSTRAINT chk_username_lowercase
    CHECK (username = lower(username));
-- 2. Existing index on username now works:
WHERE username = lower($1)   -- plain column, sargable ✓

-- FIX OPTION B: Expression index (if can't change data)
CREATE INDEX idx_users_username_upper ON users(upper(username))
    WHERE deleted_at IS NULL;  -- partial index: only active users
WHERE upper(username) = upper($1) AND deleted_at IS NULL
-- Now sargable against the expression index ✓
-- Partial: smaller index because deleted users excluded
```

### Example 2 — Intermediate: Date Range Sargability

```sql
-- SCENARIO: Analytics query for "all orders from last month"

-- NON-SARGABLE (common mistake):
SELECT COUNT(*), SUM(total)
FROM orders
WHERE EXTRACT(YEAR FROM created_at) = 2024
  AND EXTRACT(MONTH FROM created_at) = 3;
-- Two non-sargable conditions — must scan every row to evaluate EXTRACT

-- SARGABLE:
SELECT COUNT(*), SUM(total)
FROM orders
WHERE created_at >= '2024-03-01'
  AND created_at < '2024-04-01';
-- Range condition on created_at — index scan ✓
-- Use date_trunc for dynamic "current month":
WHERE created_at >= date_trunc('month', NOW())
  AND created_at < date_trunc('month', NOW()) + INTERVAL '1 month'

-- Even better — parameterize the range from application:
-- Node.js:
const start = new Date(year, month - 1, 1);
const end = new Date(year, month, 1);
await pool.query(
  `SELECT COUNT(*), SUM(total) FROM orders
   WHERE created_at >= $1 AND created_at < $2`,
  [start, end]
);
```

### Example 3 — Production Grade: Comprehensive Sargability Audit Query

```sql
-- SCENARIO: Multi-filter search endpoint for an admin dashboard
-- Orders can be filtered by: status, date range, customer, amount range, product name

-- VERSION 1 (common ORM-generated, mostly non-sargable):
SELECT o.id, o.total, o.status, o.created_at
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE
    ($1::text IS NULL OR upper(c.name) LIKE upper($1) || '%')  -- non-sargable ILIKE equivalent
    AND ($2::text IS NULL OR o.status = $2)
    AND (EXTRACT(YEAR FROM o.created_at) = $3 OR $3 IS NULL)   -- non-sargable EXTRACT
    AND (o.total BETWEEN $4 AND $5)
ORDER BY o.created_at DESC
LIMIT 50;

-- VERSION 2 (fully sargable):
-- Requires application-side preparation of params:
SELECT o.id, o.total, o.status, o.created_at, c.name AS customer_name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE
    -- Customer name search: use pg_trgm for contains, or plain prefix
    ($1::text IS NULL OR c.name ILIKE ($1 || '%'))
    -- Note: if we have GIN trgm index on c.name, even ILIKE '%x%' is sargable
    -- CREATE INDEX idx_customers_name_trgm ON customers USING GIN(name gin_trgm_ops);

    -- Status: plain equality — sargable
    AND ($2::text IS NULL OR o.status = $2)

    -- Date range: range condition — sargable
    AND ($3::timestamptz IS NULL OR o.created_at >= $3)
    AND ($4::timestamptz IS NULL OR o.created_at < $4)
    -- Application prepares: start = first day of year, end = first day of next year

    -- Amount range: plain range — sargable
    AND ($5::numeric IS NULL OR o.total >= $5)
    AND ($6::numeric IS NULL OR o.total <= $6)

ORDER BY o.created_at DESC
LIMIT 50;

-- Index strategy for this query:
-- CREATE INDEX idx_orders_created_at ON orders(created_at DESC);
-- CREATE INDEX idx_orders_status_created ON orders(status, created_at DESC);
-- (Composite covers both status filter and sort)
-- CREATE INDEX idx_customers_name_trgm ON customers USING GIN(name gin_trgm_ops);
```

---

## Wrong → Right Patterns

### Pattern 1: Date Function on Column

```sql
-- WRONG — non-sargable
WHERE YEAR(created_at) = 2024                    -- MySQL style (error in PostgreSQL)
WHERE EXTRACT(YEAR FROM created_at) = 2024       -- works but non-sargable
WHERE created_at::date = '2024-01-15'            -- non-sargable (cast on column)
WHERE DATE_TRUNC('day', created_at) = '2024-01-15'  -- non-sargable (function on column)

-- RIGHT — sargable range
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'   -- whole year
WHERE created_at >= '2024-01-15' AND created_at < '2024-01-16'   -- single day
-- Or using date_trunc on constants:
WHERE created_at >= date_trunc('year', '2024-01-01'::timestamptz)
  AND created_at < date_trunc('year', '2024-01-01'::timestamptz) + INTERVAL '1 year'
```

### Pattern 2: Case-Insensitive String Match

```sql
-- WRONG — non-sargable (both versions)
WHERE upper(email) = upper($1)
WHERE lower(email) = lower($1)
WHERE email ILIKE $1

-- RIGHT — option A: normalize data at write time
-- Store emails as lowercase (with trigger or application constraint)
WHERE email = lower($1)    -- sargable on plain email index

-- RIGHT — option B: expression index
CREATE INDEX idx_users_email_lower ON users(lower(email));
WHERE lower(email) = lower($1)  -- sargable on expression index

-- RIGHT — option C: trigram index (for partial matches too)
CREATE INDEX idx_users_email_trgm ON users USING GIN(email gin_trgm_ops);
WHERE email ILIKE '%' || $1 || '%'  -- sargable on GIN index
```

### Pattern 3: Arithmetic on Column

```sql
-- WRONG
WHERE price * 0.9 > 100      -- 10% discounted price > 100
WHERE salary / 12 > 5000     -- monthly salary > 5000
WHERE stock_count - reserved_count > 10  -- available stock > 10

-- RIGHT — move arithmetic to constant side
WHERE price > 100 / 0.9              -- = WHERE price > 111.11
WHERE salary > 5000 * 12             -- = WHERE salary > 60000
-- Multi-column expressions: use generated column
ALTER TABLE products
    ADD COLUMN available_stock INT GENERATED ALWAYS AS (stock_count - reserved_count) STORED;
CREATE INDEX ON products(available_stock);
WHERE available_stock > 10           -- sargable ✓
```

### Pattern 4: String Padding / Trim on Column

```sql
-- WRONG
WHERE TRIM(phone) = '555-1234'      -- may strip spaces, non-sargable
WHERE LTRIM(RTRIM(code)) = 'ABC'    -- non-sargable

-- RIGHT — option A: clean data at write time
-- Add CHECK constraint: CHECK (phone = TRIM(phone))
-- Then: WHERE phone = '555-1234' -- sargable

-- RIGHT — option B: expression index
CREATE INDEX idx_products_code_trimmed ON products(TRIM(code));
WHERE TRIM(code) = 'ABC'   -- sargable on expression index

-- Lesson: messy data stored with leading/trailing spaces is a schema problem
-- Fix the schema and data, not the query
```

### Pattern 5: Checking Boolean Expression Result

```sql
-- WRONG (pointless and non-sargable in most cases)
WHERE (price > 100) = TRUE     -- comparing boolean result to TRUE
WHERE is_active IS TRUE         -- this IS actually fine! IS TRUE is a predicate
WHERE status = 'active' IS TRUE -- outer IS TRUE is redundant

-- ALSO WRONG — implicit boolean
WHERE CASE WHEN price > 100 THEN 1 ELSE 0 END = 1
-- CASE expression wrapping column — non-sargable

-- RIGHT — just use the condition directly
WHERE price > 100   -- sargable ✓
WHERE is_active = TRUE   -- or WHERE is_active  (boolean column)
```

---

## Performance Profile

### The Cost of Ignoring Sargability

| Condition type | Table size 100K | Table size 1M | Table size 10M |
|----------------|-----------------|---------------|----------------|
| Sargable (index lookup) | ~0.1ms | ~0.2ms | ~0.5ms |
| Sargable (index range, 1%) | ~1ms | ~5ms | ~25ms |
| Non-sargable (full scan, cold) | ~50ms | ~500ms | ~5,000ms |
| Non-sargable (full scan, cached) | ~5ms | ~50ms | ~500ms |

These numbers are order-of-magnitude estimates. The gap between sargable and non-sargable grows with table size — sargability matters more as data grows.

### Bitmap Index Scan — The OR Sargability Bridge

When PostgreSQL needs to combine multiple index conditions with OR, it uses Bitmap Index Scans:

```
BitmapOr
  Bitmap Index Scan on idx_orders_status  (Index Cond: status = 'active')
  Bitmap Index Scan on idx_orders_status  (Index Cond: status = 'pending')
```

This is still better than a Seq Scan because it reads only the relevant heap pages. But it's slower than a single Index Scan because of the bitmap construction overhead.

### The 5% Rule for Index Selectivity

PostgreSQL's planner will choose a Seq Scan over an Index Scan when the index returns more than roughly 5-15% of rows (configurable via `random_page_cost`). This is correct behaviour.

```sql
-- Index on 'status' with values: 90% 'active', 5% 'pending', 5% 'other'

WHERE status = 'active'   -- Returns 90% of rows → planner chooses Seq Scan
WHERE status = 'pending'  -- Returns 5% of rows → planner may choose Index Scan
WHERE status = 'other'    -- Returns 5% of rows → planner may choose Index Scan

-- The index IS sargable in all three cases
-- But the planner correctly decides Seq Scan is faster for the first case
-- This is NOT a sargability problem — it's the optimizer being correct
```

---

## Node.js Integration

### The Type-Safe Parameter Pattern

```javascript
// The root cause of type mismatch issues: URL params are always strings
app.get('/users/:id', async (req, res) => {
  const id = req.params.id;  // string '42'

  // RISKY: sending string where INT4 expected
  const result = await pool.query(
    'SELECT * FROM users WHERE id = $1',
    [id]  // pg sends as TEXT, PostgreSQL must cast
  );
});

// SAFE: parse before querying
app.get('/users/:id', async (req, res) => {
  const id = parseInt(req.params.id, 10);
  if (isNaN(id)) return res.status(400).json({ error: 'Invalid ID' });

  const result = await pool.query(
    'SELECT id, name, email FROM users WHERE id = $1',
    [id]  // pg sends as INT4 — exact type match, unambiguously sargable
  );
});
```

### Automated Sargability Audit in Development

```javascript
// Development middleware: detect non-sargable patterns in queries
// Run this in development/staging only
const SARGABILITY_WARNINGS = [
  {
    pattern: /WHERE\s+\w+\s*\(\s*\w+/i,
    message: 'Function called on column in WHERE — likely non-sargable'
  },
  {
    pattern: /WHERE.*EXTRACT\s*\(/i,
    message: 'EXTRACT on column in WHERE — use date range instead'
  },
  {
    pattern: /WHERE.*::date\s*=/i,
    message: 'Column cast to date in WHERE — use range condition instead'
  },
  {
    pattern: /LIKE\s+['"]\%[^'"]/i,
    message: 'Leading wildcard in LIKE — cannot use B-tree index'
  },
  {
    pattern: /ILIKE/i,
    message: 'ILIKE — needs lower() expression index or pg_trgm for index use'
  },
  {
    pattern: /COALESCE\s*\(\s*\w+\s*,/i,
    message: 'COALESCE on column in WHERE — non-sargable'
  },
];

function checkSargability(sql) {
  const warnings = [];
  for (const { pattern, message } of SARGABILITY_WARNINGS) {
    if (pattern.test(sql)) {
      warnings.push(message);
    }
  }
  return warnings;
}

// Integration with pg pool
const originalQuery = pool.query.bind(pool);
pool.query = function(text, values) {
  if (process.env.NODE_ENV === 'development') {
    const sql = typeof text === 'string' ? text : text.text;
    const warnings = checkSargability(sql);
    if (warnings.length > 0) {
      console.warn('[SARGABILITY WARNING]', warnings.join('; '));
      console.warn('Query:', sql.substring(0, 200));
    }
  }
  return originalQuery(text, values);
};
```

### Expression Index Creation Utility

```javascript
// Utility to add missing expression indexes based on common patterns
// Run in migrations when you discover non-sargable conditions

const RECOMMENDED_INDEXES = [
  // Case-insensitive email search
  `CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_email_lower
   ON users(lower(email))`,

  // Case-insensitive username search
  `CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_users_username_lower
   ON users(lower(username)) WHERE deleted_at IS NULL`,

  // Full-text/fuzzy name search
  `CREATE EXTENSION IF NOT EXISTS pg_trgm`,
  `CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_products_name_trgm
   ON products USING GIN(name gin_trgm_ops)`,

  // Date-only filtering on timestamp column
  // Better: don't use this — use range conditions instead!
  // But if you must: CREATE INDEX ON orders((created_at::date))
];

async function applyIndexes(pool) {
  for (const sql of RECOMMENDED_INDEXES) {
    try {
      await pool.query(sql);
      console.log(`Applied: ${sql.substring(0, 60)}...`);
    } catch (err) {
      console.error(`Failed: ${err.message}`);
    }
  }
}
```

---

## ORM Comparison

### How ORMs Create Sargability Problems

All ORMs can generate non-sargable conditions — usually through:
1. Applying functions via their "helper" methods that get translated to SQL functions on columns
2. Type mismatches between ORM model field types and what gets sent to PostgreSQL
3. No way to express certain conditions without using raw SQL

### 1. Prisma

**Sargability issues:**
```typescript
// Safe — Prisma generates correct SQL:
const users = await prisma.user.findMany({
  where: {
    email: { startsWith: 'alice' }   // LIKE 'alice%' — sargable ✓
    // contains: 'alice'             // LIKE '%alice%' — non-sargable (but sometimes OK with trgm)
    // endsWith: 'alice'             // LIKE '%alice' — non-sargable
  }
});

// Prisma's `mode: 'insensitive'` generates ILIKE:
const users = await prisma.user.findMany({
  where: {
    email: { contains: 'alice', mode: 'insensitive' }
    // Generates: WHERE email ILIKE '%alice%'
    // Non-sargable unless pg_trgm GIN index exists
  }
});

// Date filtering in Prisma — this IS sargable:
const orders = await prisma.order.findMany({
  where: {
    createdAt: {
      gte: new Date('2024-01-01'),
      lt: new Date('2025-01-01')
    }
  }
});
// Generates: WHERE created_at >= $1 AND created_at < $2 — sargable ✓

// Non-sargable with Prisma — no built-in way, forces $queryRaw:
const users = await prisma.$queryRaw`
  SELECT * FROM users WHERE lower(email) = ${email.toLowerCase()}
`;
```

**Verdict**: Prisma's `startsWith` generates sargable LIKE, `contains`/`endsWith`/`mode: insensitive` are non-sargable without a trigram index. Date ranges are sargable. The ORM doesn't expose expression index usage — you must use `$queryRaw` for anything requiring expression index matching.

---

### 2. Drizzle ORM

**Sargability issues:**
```typescript
import { like, ilike, sql } from 'drizzle-orm';

// Sargable
const users = await db.select().from(usersTable)
  .where(like(usersTable.email, 'alice%'));   // LIKE 'alice%' ✓

// Non-sargable (but correct SQL)
const users2 = await db.select().from(usersTable)
  .where(ilike(usersTable.email, '%alice%'));  // ILIKE '%alice%' ✗ (unless trgm)

// Sargable expression index usage — Drizzle can match!
const users3 = await db.select().from(usersTable)
  .where(sql`lower(${usersTable.email}) = ${email.toLowerCase()}`);
// If CREATE INDEX ON users(lower(email)) exists, this IS sargable ✓

// Date range — sargable
const orders = await db.select().from(ordersTable)
  .where(
    and(
      gte(ordersTable.createdAt, startDate),
      lt(ordersTable.createdAt, endDate)
    )
  );
// Generates: WHERE created_at >= $1 AND created_at < $2 ✓
```

**Verdict**: Drizzle's `sql` template tag allows matching expression indexes exactly — it's the most flexible ORM for sargability. `ilike` generates ILIKE which needs a trgm index. Date ranges are handled correctly.

---

### 3. Sequelize

**Sargability issues:**
```javascript
const { Op } = require('sequelize');

// Sargable
const users = await User.findAll({
  where: { email: { [Op.like]: 'alice%' } }   // LIKE 'alice%' ✓
});

// Non-sargable
const users2 = await User.findAll({
  where: { email: { [Op.iLike]: '%alice%' } }  // ILIKE '%alice%' ✗
});

// Date range — sargable
const orders = await Order.findAll({
  where: {
    createdAt: {
      [Op.gte]: startDate,
      [Op.lt]: endDate
    }
  }
});
// ✓ Generates range condition

// For expression index matching:
const users3 = await User.findAll({
  where: Sequelize.where(
    Sequelize.fn('lower', Sequelize.col('email')),
    email.toLowerCase()
  )
});
// Generates: WHERE lower(email) = 'alice@example.com'
// Sargable if CREATE INDEX ON users(lower(email)) exists ✓
// But: Sequelize.where + Sequelize.fn is verbose and easy to get wrong
```

**Verdict**: Sequelize's `Sequelize.where(Sequelize.fn(...), value)` can generate expression index-matching SQL, but the syntax is verbose. `Op.iLike` with `%` wildcard is non-sargable. Raw queries needed for complex cases.

---

### 4. TypeORM

**Sargability issues:**
```typescript
import { Like, ILike, Between, MoreThanOrEqual, LessThan } from 'typeorm';

// Sargable
const users = await userRepository.find({
  where: { email: Like('alice%') }   // LIKE 'alice%' ✓
});

// Non-sargable
const users2 = await userRepository.find({
  where: { email: ILike('%alice%') }  // ILIKE '%alice%' ✗
});

// Date range — sargable
const orders = await orderRepository.find({
  where: {
    createdAt: Between(startDate, endDate)
    // Generates: WHERE created_at BETWEEN $1 AND $2 ✓
    // Note: BETWEEN is inclusive on both ends — may miss end-of-day items
    // Safer:
    // createdAt: And(MoreThanOrEqual(startDate), LessThan(endDate))
  }
});

// Expression index matching via QueryBuilder:
const users3 = await dataSource
  .getRepository(User)
  .createQueryBuilder('u')
  .where('lower(u.email) = :email', { email: email.toLowerCase() })
  .getMany();
// If lower(email) index exists: sargable ✓
// QueryBuilder parameterizes correctly ✓
```

**Verdict**: TypeORM's QueryBuilder can match expression indexes with proper string interpolation. Repository helpers `Like`/`ILike` have the same sargability limitations as Sequelize's `Op.like`. Date range: prefer `MoreThanOrEqual + LessThan` over `Between` for timestamp precision.

---

### 5. Knex.js

**Sargability issues:**
```javascript
// Sargable
const users = await knex('users')
  .where('email', 'like', 'alice%');    // LIKE 'alice%' ✓

// Non-sargable
const users2 = await knex('users')
  .where('email', 'ilike', '%alice%');  // ILIKE '%alice%' ✗

// Expression index matching — most natural in Knex:
const users3 = await knex('users')
  .whereRaw('lower(email) = ?', [email.toLowerCase()]);
// Sargable if lower(email) index exists ✓
// Knex parameterizes correctly ✓

// Date range — sargable
const orders = await knex('orders')
  .where('created_at', '>=', startDate)
  .where('created_at', '<', endDate);
// Generates: WHERE created_at >= $1 AND created_at < $2 ✓

// The cleanest ORM for arbitrary sargable conditions
const results = await knex('products')
  .whereRaw('lower(name) like ?', [namePrefix.toLowerCase() + '%'])
  .where('price', '>', 10)
  .where('price', '<', 100)
  .whereNotNull('category_id');
```

**Verdict**: Knex.js gives the most control over sargability. `whereRaw` with parameterization is clean, safe, and can match any expression index. Highly recommended when expression index usage is important.

---

### ORM Comparison Summary

| Condition | Prisma | Drizzle | Sequelize | TypeORM | Knex.js |
|-----------|--------|---------|-----------|---------|---------|
| `LIKE 'prefix%'` (sargable) | ✓ `startsWith` | ✓ `like()` | ✓ `Op.like` | ✓ `Like()` | ✓ `.where('like')` |
| `LIKE '%suffix'` (non-sargable) | ✓ generates it | ✓ generates it | ✓ generates it | ✓ generates it | ✓ generates it |
| Expression index matching | `$queryRaw` | ✓ `sql` tag | ✓ (verbose) | ✓ QueryBuilder | ✓ `whereRaw` |
| Date range (sargable) | ✓ | ✓ | ✓ | ✓ | ✓ |
| Warns about non-sargable | ✗ | ✗ | ✗ | ✗ | ✗ |
| Type-safe for column names | ✓ | ✓ | ✗ | Partial | ✗ |

**Universal truth**: No ORM warns you about sargability. They all silently generate non-sargable SQL. You must EXPLAIN your queries and understand the output.

---

## Practice Exercises

### Exercise 1 — Sargability Audit

For each of the following WHERE conditions, determine: (a) is it sargable? (b) why or why not? (c) how would you rewrite it to be sargable?

```sql
1. WHERE UPPER(country_code) = 'US'
2. WHERE CAST(order_id AS TEXT) LIKE '100%'
3. WHERE price / quantity > 50
4. WHERE DATE_TRUNC('month', created_at) = '2024-03-01'
5. WHERE email LIKE '%@gmail.com'
6. WHERE COALESCE(discount, 0) > 0
7. WHERE LENGTH(description) < 200
8. WHERE status IN ('active', 'pending')
9. WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31'
10. WHERE TRIM(LOWER(username)) = 'admin'
```

---

### Exercise 2 — Expression Index Design

Given this table and query, design the minimal set of indexes needed to make ALL WHERE conditions sargable:

```sql
CREATE TABLE product_reviews (
    id SERIAL PRIMARY KEY,
    product_id INT NOT NULL,
    reviewer_email TEXT NOT NULL,
    body TEXT NOT NULL,
    rating SMALLINT,
    is_verified BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMPTZ DEFAULT NOW()
);

-- Query 1: Find recent verified reviews for a product
WHERE product_id = $1
  AND is_verified = TRUE
  AND created_at > NOW() - INTERVAL '30 days'

-- Query 2: Find reviews by email (case-insensitive)
WHERE lower(reviewer_email) = lower($1)

-- Query 3: Search review text for keyword (contains match)
WHERE body ILIKE '%' || $1 || '%'

-- Query 4: Find reviews in a rating range for products in a category
WHERE product_id = ANY($1::int[])
  AND rating BETWEEN $2 AND $3
```

Write all necessary `CREATE INDEX` statements with justification for each.

---

### Exercise 3 — EXPLAIN Reading

Given this EXPLAIN ANALYZE output, identify: (a) which condition is non-sargable, (b) why, (c) what the fix is:

```
Seq Scan on orders  (cost=0.00..12543.00 rows=150 width=68)
                    (actual time=0.034..421.231 rows=47 loops=1)
  Filter: (((status)::text = 'completed'::text) AND
           (date_part('year'::text, (created_at)::timestamp) = '2024'::double precision))
  Rows Removed by Filter: 499953
  Buffers: shared hit=4234 read=2104
Planning Time: 1.234 ms
Execution Time: 421.452 ms
```

---

### Exercise 4 — ORM Sargability Fix

This TypeORM query generates non-sargable SQL. Rewrite it to be fully sargable:

```typescript
const invoices = await invoiceRepository.find({
  where: {
    status: Not('cancelled'),
    customer: {
      email: ILike(`%${searchTerm}%`)
    }
  },
  relations: ['customer'],
  order: { createdAt: 'DESC' },
  take: 20
});
```

Assume: `customer.email` has a `pg_trgm` GIN index on `lower(email)`. Rewrite using QueryBuilder to make the email search use the trigram index.

---

## Interview Questions

### Junior Level

**Q: What is a sargable condition in SQL?**

A: A sargable condition is one where the database engine can use an index to find matching rows directly, rather than scanning the entire table. "Sargable" comes from "Search ARGument ABLE." For example, `WHERE email = 'alice@example.com'` is sargable if there's an index on `email` — the database can look up the value directly in the index. But `WHERE lower(email) = 'alice@example.com'` is not sargable for a regular index on `email`, because the `lower()` function wraps the column, preventing the index from being used.

---

**Q: Why does `WHERE EXTRACT(YEAR FROM created_at) = 2024` hurt performance compared to a date range?**

A: `EXTRACT(YEAR FROM created_at)` applies a function to the `created_at` column. Even if there's a B-tree index on `created_at`, PostgreSQL can't use it because the indexed value is `created_at` — not the year extracted from it. PostgreSQL must read every row, apply the EXTRACT function, and check the result. This is a full sequential scan.

The sargable rewrite is: `WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'`. This presents `created_at` untouched on one side of the comparison, so PostgreSQL can use the index to seek to January 1st 2024 and scan forward to January 1st 2025.

---

### Principal Level

**Q: A developer has added `CREATE INDEX idx_users_email ON users(email)` but the query `WHERE lower(email) = $1` still does a full table scan. They're confused because the index exists. What do you tell them?**

A: The index on `email` exists, but the query asks for `lower(email)` — a function applied to the column. PostgreSQL's B-tree index stores values of `email` in sorted order. When the query presents `lower(email)`, the planner looks for an index on the expression `lower(email)`. Finding none, it falls back to a Seq Scan.

There are two fixes depending on the situation:

**Fix 1 — Expression index** (when you can't control data storage):
```sql
CREATE INDEX idx_users_email_lower ON users(lower(email));
```
Now `lower(email)` matches the index definition. PostgreSQL can do an index lookup.

**Fix 2 — Normalize at write time** (the preferred production approach):
Enforce lowercase emails at write time — application constraint, trigger, or `CHECK (email = lower(email))`. Then the query becomes `WHERE email = lower($1)` — the column is untouched, the function is applied to the parameter (which is just a constant operation), and the plain `email` index works.

Fix 2 is better for two reasons: (1) the plain index is simpler to maintain; (2) normalizing data at write time prevents inconsistency — you can't accidentally store mixed-case emails if you have the constraint.

I'd also check whether the developer is calling `lower($1)` in the application before sending the parameter. If they're comparing `lower(email)` to a possibly-uppercase input, they need EITHER the expression index approach OR normalize both sides.

---

**Q: Your DBA says "just add more indexes." A senior dev says "indexes on every column will make everything fast." How do you respond?**

A: I'd push back on both assumptions.

First: indexes are not free. Each index adds write amplification — every INSERT, UPDATE, and DELETE must update all indexes on the table. A table with 15 indexes requires updating 15 separate B-tree structures on every write. For write-heavy tables (like event logs, audit tables, queues), excessive indexes degrade write throughput significantly.

Second: the right question is not "which columns should I index" but "which query patterns should I optimize." I start from EXPLAIN ANALYZE output and pg_stat_statements to find the actual slow queries. For each slow query, I identify:
1. What condition is being filtered on?
2. Is that condition sargable?
3. What's the cardinality/selectivity?
4. Would an index actually be used (selectivity > 5% threshold)?

If the condition is non-sargable, an index on the raw column doesn't help — I need an expression index or to rewrite the query. If the column has low cardinality (like a boolean `is_active`), a plain index on that column alone may not be used anyway.

The optimal index strategy is: minimal indexes that cover the critical query patterns, designed as composite indexes when queries filter on multiple columns, with INCLUDE for covering indexes when Index-Only Scans are achievable.

---

## Mental Model Checkpoint

1. Define sargability in one sentence. What does "Search ARGument ABLE" mean practically?

2. You have `CREATE INDEX ON products(price)`. Will `WHERE price * 1.1 > 100` use this index? What's the correct rewrite?

3. `WHERE created_at::date = '2024-01-15'` — is this sargable? Write the sargable equivalent.

4. You create `CREATE INDEX ON users(lower(email))`. Now you run `WHERE email = 'Alice@example.com'`. Does this query use the new index? Why or why not?

5. `WHERE status = 'active' OR lower(email) LIKE 'admin%'` — the status column has an index, and there's a `lower(email)` expression index. Can PostgreSQL use both indexes for this OR condition?

6. A query runs in 2ms in development (10,000 rows) but 45 seconds in production (5 million rows). The WHERE clause hasn't changed. What's the most likely explanation? What's your first diagnostic step?

7. `WHERE COALESCE(discount_pct, 0) > 0` vs `WHERE discount_pct > 0 AND discount_pct IS NOT NULL` vs `WHERE discount_pct IS NOT NULL AND discount_pct != 0` — which of these can use an index on `discount_pct`? Are they semantically equivalent?

---

## Quick Reference Card

```
Sargability Rule
────────────────
Sargable: column OPERATOR constant  (column must be alone, untouched)
Non-sargable: function(column) OPERATOR constant
Non-sargable: column OPERATOR function(column)  (if both sides have index columns)

Always Non-Sargable
────────────────────
function(col) = val         → expression index needed
col::other_type = val       → cast on column → full scan
col * n > val               → arithmetic on column → full scan
EXTRACT(...FROM col) = val  → use range condition instead
DATE(col) = val             → use range condition instead
COALESCE(col, def) = val    → rewrite to col = val OR col IS NULL
LIKE '%suffix'              → no B-tree solution (use reverse index or trgm)
ILIKE anything              → needs lower() expression index or trgm GIN

Always Sargable
────────────────
col = val                   → index seek
col < val, col > val        → range scan
col BETWEEN a AND b         → range scan
col IN (v1, v2, v3)         → bitmap scan or multiple seeks
col IS NULL                 → index scan (NULLs in B-tree)
col IS NOT NULL             → index scan
col LIKE 'prefix%'          → range scan (B-tree converts to range)
lower(col) = val            → sargable IF CREATE INDEX ON t(lower(col)) exists

Diagnosis
──────────
EXPLAIN output:
  Index Cond: → sargable ✓
  Filter: → non-sargable (or post-index filter) ✗

Fix Options
────────────
1. Apply function to constant, not column: lower(col) = lower($1) → col = lower($1)
2. Rewrite as range: EXTRACT(YEAR FROM col) = 2024 → col >= '2024-01-01' AND col < '2025-01-01'
3. Expression index: CREATE INDEX ON t(lower(col)) → lower(col) = val IS sargable
4. Generated column: ALTER TABLE t ADD col2 AS (expr) STORED; CREATE INDEX ON t(col2)
5. Normalize at write time: store data in queryable form from the start
```

---

## Connected Topics

| Topic | Connection |
|-------|-----------|
| [06-where-clause-mastery.md](06-where-clause-mastery.md) | LIKE patterns, IS NULL, type comparison — the conditions whose sargability this topic explains |
| [07-null-in-depth.md](07-null-in-depth.md) | COALESCE in WHERE is non-sargable — the specific NULL-handling sargability trap |
| [42-index-types.md](42-index-types.md) | Which index types support which operators — the foundation of sargability |
| [43-index-scan-types.md](43-index-scan-types.md) | Index Scan vs Seq Scan vs Bitmap — what EXPLAIN shows when conditions are/aren't sargable |
| [44-composite-indexes.md](44-composite-indexes.md) | Multi-column sargability — equality first, range last in composite index design |
| [45-partial-indexes.md](45-partial-indexes.md) | Partial expression indexes — combining expression indexes with WHERE predicates |
| [46-expression-indexes.md](46-expression-indexes.md) | The complete guide to expression indexes — the fix for most non-sargable conditions |
| [49-query-optimisation-methodology.md](49-query-optimisation-methodology.md) | The systematic workflow for identifying and fixing non-sargable conditions in production |
| [63-date-time-mastery.md](63-date-time-mastery.md) | Date function sargability — DATE_TRUNC, EXTRACT, cast patterns for timestamps |
