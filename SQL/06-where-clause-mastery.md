# Topic 06 — WHERE Clause Mastery

**Phase 2 — SELECT and Filtering Foundations**
**Interview Priority: HIGH**

---

## ELI5 Analogy

Imagine a warehouse with 10 million boxes. Your job: bring me only the boxes that match specific rules.

`WHERE` is the rulebook you hand to the warehouse worker before they start pulling boxes. Every rule gets evaluated for every box: TRUE (bring it), FALSE (leave it), or UNKNOWN (leave it — UNKNOWN means "not sure, skip it").

The warehouse worker has special tools:
- A scale (`=`, `<`, `>`) — compare numbers or labels
- A checklist (`IN`) — does this box's label appear in my list?
- A pattern card (`LIKE`) — does the label start with "SHIP-"?
- A range gauge (`BETWEEN`) — is the weight between 10 and 50 kg?
- A "does this even have a label?" test (`IS NULL`) — some boxes arrived with no label at all

The critical rule: **any box without a label evaluates to UNKNOWN for every label-based test, and UNKNOWN boxes are always left behind.** This is the NULL trap that ends careers.

---

## Internals Connection

WHERE is processed at **Step 2** of the 9-step logical execution order, immediately after FROM+JOINs build the working row set.

In PostgreSQL's executor, WHERE conditions become a **qual list** (qualification list) attached to the scan or join node. Each row emitted from the scan/join node passes through the qual list. If the expression evaluates to TRUE, the row is passed upstream. If FALSE or NULL, it is discarded.

Critical implementation detail: PostgreSQL **short-circuits** AND conditions. If the first condition is FALSE, subsequent conditions are not evaluated. This matters for:
1. **Performance**: Put the cheapest, most selective condition first in an AND chain
2. **Side effects**: Functions with side effects in later AND conditions may not run
3. **Null safety**: A null-safe check first can guard a subsequent comparison

The **executor's qual evaluation** runs the expression in the context of each tuple. For indexed columns, PostgreSQL may be able to use the index to skip qual evaluation entirely for most rows — this is sargability (Topic 08 goes deep on this).

---

## Execution Order Context

```
FROM + JOINs  →  [WHERE]  →  GROUP BY  →  HAVING  →  SELECT  →  DISTINCT  →  WINDOW  →  ORDER BY  →  LIMIT
                    ↑
              YOU ARE HERE
```

WHERE runs **before** GROUP BY, aggregation, SELECT, and window functions. Consequences:

- **Cannot reference SELECT aliases** — they don't exist yet
- **Cannot filter on aggregate results** — use HAVING for that
- **Cannot filter on window function results** — wrap in a subquery or CTE
- **Can filter on any column from the FROM clause** — including columns not in SELECT
- **Can filter on JOIN columns from joined tables** — this is what makes filtering post-JOIN possible

```sql
-- WHERE filters RAW rows before grouping
SELECT department_id, COUNT(*) FROM employees WHERE salary > 50000 GROUP BY department_id;
-- Only employees with salary > 50000 are counted. The rest are excluded before GROUP BY.

-- HAVING filters GROUPED rows after aggregation
SELECT department_id, COUNT(*) FROM employees GROUP BY department_id HAVING COUNT(*) > 10;
-- All employees grouped first, then groups with 10 or fewer employees excluded.

-- CANNOT use window function result in WHERE
SELECT user_id, ROW_NUMBER() OVER (ORDER BY created_at) AS rn FROM events WHERE rn < 10;
-- ERROR: column "rn" does not exist
-- Window functions run AFTER WHERE. Wrap in subquery:
SELECT * FROM (
    SELECT user_id, ROW_NUMBER() OVER (ORDER BY created_at) AS rn FROM events
) t WHERE t.rn < 10;
```

---

## What Is WHERE?

WHERE is the **row filter clause** in SQL. It specifies a search condition that each row must satisfy to be included in the result. The condition is a Boolean expression that evaluates to TRUE, FALSE, or UNKNOWN (due to NULL).

Only rows that evaluate to **TRUE** pass through. Rows that evaluate to FALSE or UNKNOWN are excluded.

This is the **three-valued logic** principle — SQL does not have a simple binary true/false system. It has three values:
- **TRUE** — row passes
- **FALSE** — row excluded
- **UNKNOWN** — row excluded (this surprises most developers)

The UNKNOWN value arises from any comparison involving NULL. This is covered in depth in Topic 07. The WHERE clause section here focuses on the mechanics of each operator and their performance characteristics.

---

## Why It Matters in Production

### 1. The WHERE Clause Determines Index Usage

The single biggest performance decision in a query is whether the WHERE clause can use an index. This depends on:
- Which column is being filtered
- Whether an index exists on that column
- **How** the filter is written — even if an index exists, a badly written WHERE clause will force a sequential scan

### 2. Wrong Operators Produce Silent Wrong Results

`WHERE status = NULL` returns zero rows — always. The correct syntax is `WHERE status IS NULL`. Developers write the wrong version and see an empty result set, assume "there are no NULLs," and ship wrong code.

### 3. LIKE with Leading Wildcard Kills Performance

`WHERE name LIKE '%smith'` cannot use a B-tree index. Full table scan every time. `WHERE name LIKE 'smith%'` can use an index. One character change: order-of-magnitude performance difference.

### 4. Type Mismatches Silently Prevent Index Use

`WHERE user_id = '42'` when `user_id` is an `INT` causes implicit casting. PostgreSQL may or may not use the index depending on the types involved. This is a hidden performance bomb that shows up only at scale.

---

## Deep Technical Content

### Comparison Operators

```sql
-- Equality
WHERE price = 9.99
WHERE status = 'active'
WHERE user_id = $1

-- Inequality
WHERE price <> 9.99    -- Standard SQL
WHERE price != 9.99    -- Also works in PostgreSQL (non-standard but widely supported)

-- Range
WHERE price < 10.00
WHERE price <= 10.00
WHERE price > 50.00
WHERE price >= 50.00

-- All comparison operators work with NULLs the same way: NULL on either side = UNKNOWN
WHERE price = NULL    -- NEVER returns rows — always UNKNOWN
WHERE price <> NULL   -- NEVER returns rows — always UNKNOWN
-- Use IS NULL / IS NOT NULL for null checks (see below)
```

**Operator precedence in WHERE (high to low):**
```
1. NOT
2. AND
3. OR
```

This matters enormously in complex conditions:
```sql
-- WRONG (likely not what you want):
WHERE status = 'active' OR status = 'pending' AND created_at > '2024-01-01'
-- Evaluated as: status = 'active' OR (status = 'pending' AND created_at > '2024-01-01')

-- RIGHT (explicit parentheses):
WHERE (status = 'active' OR status = 'pending') AND created_at > '2024-01-01'
```

**Rule**: Always use parentheses when mixing AND and OR. Never rely on precedence for readability.

---

### IS NULL / IS NOT NULL

```sql
-- The ONLY correct way to check for NULL
WHERE deleted_at IS NULL          -- rows where deleted_at is NULL
WHERE deleted_at IS NOT NULL      -- rows where deleted_at has a value

-- These NEVER work for NULL detection:
WHERE deleted_at = NULL           -- always 0 rows (NULL = NULL → UNKNOWN)
WHERE deleted_at != NULL          -- always 0 rows (NULL != NULL → UNKNOWN)
```

**Why `= NULL` always fails:** NULL represents "unknown value." Comparing an unknown value to anything — even another unknown — produces UNKNOWN, not TRUE. `NULL = NULL` is UNKNOWN, not TRUE.

**Performance:** IS NULL and IS NOT NULL can use B-tree indexes. PostgreSQL stores NULL entries in B-tree indexes (at the end). A partial index `WHERE col IS NOT NULL` is often dramatically smaller than a full index if the column is frequently NULL.

```sql
-- Index on nullable column
CREATE INDEX idx_orders_deleted_at ON orders(deleted_at);
-- This index CAN be used for:
WHERE deleted_at IS NULL        -- ✓
WHERE deleted_at IS NOT NULL    -- ✓
WHERE deleted_at = '2024-01-01' -- ✓
```

**IS DISTINCT FROM / IS NOT DISTINCT FROM:**

The null-safe equality operators — treat NULL as a comparable value:

```sql
WHERE a IS DISTINCT FROM b
-- Equivalent to: (a <> b) OR (a IS NULL AND b IS NOT NULL) OR (a IS NOT NULL AND b IS NULL)
-- Returns TRUE if a and b are different (treating NULL as an ordinary value)
-- NULL IS DISTINCT FROM NULL → FALSE (they're the same)
-- NULL IS DISTINCT FROM 'x' → TRUE (they're different)

WHERE a IS NOT DISTINCT FROM b
-- The null-safe equality: NULL IS NOT DISTINCT FROM NULL → TRUE
-- Useful for comparing two nullable columns where both being NULL means "equal"
```

```sql
-- Real use case: finding rows where a column changed
SELECT * FROM audit_log
WHERE old_value IS DISTINCT FROM new_value;
-- Handles: value → value (catches real changes)
-- Handles: NULL → value (catches setting from null)
-- Handles: value → NULL (catches clearing a value)
-- Handles: NULL → NULL (correctly excluded — no change)
-- WHERE old_value <> new_value would MISS the NULL cases
```

---

### BETWEEN

```sql
-- Inclusive range on both ends
WHERE price BETWEEN 10.00 AND 50.00
-- Equivalent to: WHERE price >= 10.00 AND price <= 50.00

WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31'
-- TRAP: this works for DATE but not for TIMESTAMP
-- '2024-12-31' as a timestamp is '2024-12-31 00:00:00'
-- Rows on 2024-12-31 at 14:30:00 are EXCLUDED

-- For timestamps, use explicit range:
WHERE created_at >= '2024-01-01' AND created_at < '2025-01-01'
-- The < on the end date is the correct pattern

-- NOT BETWEEN
WHERE price NOT BETWEEN 10.00 AND 50.00
-- Equivalent to: WHERE price < 10.00 OR price > 50.00
```

**Performance:** BETWEEN on an indexed column is range-efficient — PostgreSQL uses the index to find the start point and scans forward to the end. Same as `>= AND <=`.

**BETWEEN with NULLs:** If either bound is NULL, BETWEEN always returns UNKNOWN (false). `WHERE price BETWEEN NULL AND 50` returns nothing.

---

### IN

```sql
-- Membership test in a value list
WHERE status IN ('active', 'pending', 'processing')
-- Equivalent to: WHERE status = 'active' OR status = 'pending' OR status = 'processing'

-- NOT IN — the NULL trap (see below)
WHERE status NOT IN ('deleted', 'banned')

-- IN with subquery
WHERE user_id IN (SELECT user_id FROM admins)

-- IN with array parameter (Node.js pg pattern)
WHERE id = ANY($1)  -- $1 is passed as an array: [1, 2, 3]
-- Equivalent to IN but works with parameterized arrays in pg
```

**The NOT IN NULL Trap — Critical:**

```sql
-- This returns ZERO rows even when you expect some:
WHERE status NOT IN ('deleted', NULL)
-- Expanded: WHERE status <> 'deleted' AND status <> NULL
-- status <> NULL → UNKNOWN for every row
-- TRUE AND UNKNOWN → UNKNOWN
-- So every single row is excluded

-- Same trap with a subquery that returns NULLs:
WHERE user_id NOT IN (SELECT manager_id FROM employees)
-- If ANY manager_id is NULL in the subquery, the entire NOT IN returns nothing
-- Because: 42 NOT IN (..., NULL) expands to 42 <> NULL → UNKNOWN → excluded

-- Safe alternatives:
WHERE status NOT IN ('deleted') AND status IS NOT NULL
-- OR use NOT EXISTS (which handles NULLs correctly):
WHERE NOT EXISTS (SELECT 1 FROM banned_statuses WHERE status = o.status)
```

**Performance of IN with value list:** PostgreSQL optimizes `IN (val1, val2, ..., valN)` into a hash table lookup for large lists. For small lists (< ~8 values), it expands to OR conditions. Both can use indexes.

**Performance of IN with subquery:** The planner may transform `IN (subquery)` into a semi-join (efficient) or execute the subquery as a correlated subquery (inefficient). Use `EXISTS` when you want to guarantee semi-join behavior (Topic 27 covers this in depth).

---

### LIKE and ILIKE

```sql
-- Pattern matching
-- % = zero or more characters
-- _ = exactly one character

WHERE name LIKE 'john%'        -- starts with 'john'
WHERE name LIKE '%john%'       -- contains 'john' anywhere
WHERE name LIKE '%john'        -- ends with 'john'
WHERE name LIKE 'j_hn'         -- j, any one char, h, n
WHERE name LIKE 'jo__'         -- jo followed by exactly two chars

-- Case-insensitive (PostgreSQL extension)
WHERE name ILIKE 'john%'       -- starts with 'john', case-insensitive
WHERE name ILIKE '%john%'      -- contains 'john', case-insensitive

-- NOT LIKE / NOT ILIKE
WHERE name NOT LIKE 'test%'
WHERE name NOT ILIKE 'admin%'

-- Escaping the % and _ characters
WHERE code LIKE '10\%'  ESCAPE '\'   -- literal percent sign
WHERE code LIKE '10!%'  ESCAPE '!'   -- same, different escape char
```

**Performance rules for LIKE — the most important pattern to memorize:**

| Pattern | Index usable? | Why |
|---------|--------------|-----|
| `LIKE 'abc%'` | ✓ YES | Prefix match — index can seek to 'abc' and scan forward |
| `LIKE '%abc'` | ✗ NO | Trailing match — must check every row |
| `LIKE '%abc%'` | ✗ NO | Contains match — must check every row |
| `ILIKE 'abc%'` | ✗ NO | Case-insensitive — default index is case-sensitive |
| `LIKE 'abc'` (no wildcard) | ✓ YES | Equality — index lookup |

**Making `ILIKE` use an index:**
```sql
-- Create a case-insensitive index using lower()
CREATE INDEX idx_users_email_lower ON users(lower(email));

-- Now this can use the index:
WHERE lower(email) LIKE lower($1 || '%')
-- Or equivalently:
WHERE lower(email) = lower($1)

-- For full contains search (LIKE '%abc%'), use pg_trgm extension:
CREATE EXTENSION pg_trgm;
CREATE INDEX idx_products_name_trgm ON products USING GIN(name gin_trgm_ops);
WHERE name ILIKE '%widget%'  -- now uses the GIN trigram index
```

**Escaping in Node.js:**
```javascript
// DANGER: never interpolate user input into LIKE patterns
const searchTerm = req.query.q;  // user input

// WRONG — SQL injection AND wrong escaping:
const rows = await pool.query(`SELECT * FROM products WHERE name LIKE '%${searchTerm}%'`);

// RIGHT — parameterize AND escape LIKE metacharacters:
function escapeLike(str) {
  return str.replace(/[%_\\]/g, '\\$&');
}
const safeSearch = escapeLike(searchTerm);
const rows = await pool.query(
  'SELECT id, name FROM products WHERE name ILIKE $1',
  [`%${safeSearch}%`]
);
```

---

### ANY and ALL

```sql
-- ANY: true if condition holds for AT LEAST ONE element in the array/subquery
WHERE salary > ANY(ARRAY[50000, 60000, 70000])
-- True if salary > 50000 OR salary > 60000 OR salary > 70000
-- Simplified: salary > 50000 (true if greater than the minimum)

-- ALL: true if condition holds for ALL elements in the array/subquery
WHERE salary > ALL(ARRAY[50000, 60000, 70000])
-- True if salary > 50000 AND salary > 60000 AND salary > 70000
-- Simplified: salary > 70000 (true if greater than the maximum)

-- ANY with =  is equivalent to IN:
WHERE id = ANY(ARRAY[1, 2, 3])  -- same as WHERE id IN (1, 2, 3)

-- ANY with != is NOT equivalent to NOT IN:
WHERE id != ANY(ARRAY[1, 2, 3])  -- true if id != 1 OR id != 2 OR id != 3
                                   -- true for EVERY row! (e.g., id=1 is != 2 and != 3)

-- ALL with != is equivalent to NOT IN:
WHERE id != ALL(ARRAY[1, 2, 3])  -- true if id != 1 AND id != 2 AND id != 3
                                   -- same as NOT IN
```

**The ANY/ALL NULL behavior:**

```sql
WHERE 5 = ANY(ARRAY[1, NULL, 5])   -- TRUE (5 = 5 found)
WHERE 5 = ANY(ARRAY[1, NULL, 3])   -- UNKNOWN (no TRUE found, but NULL lurks)
WHERE 5 = ALL(ARRAY[5, NULL, 5])   -- UNKNOWN (5 = NULL is UNKNOWN, so ALL → UNKNOWN)
WHERE 5 = ALL(ARRAY[5, 5, 5])      -- TRUE
WHERE 6 = ALL(ARRAY[5, NULL, 5])   -- FALSE (6 = 5 is FALSE, short-circuits)
```

**Real use case — filtering by array parameter in Node.js:**
```sql
-- Pass an array of IDs from Node.js
WHERE user_id = ANY($1::int[])
-- In Node.js: pool.query('...WHERE user_id = ANY($1::int[])', [[1, 2, 3]])
-- This is the correct parameterized way to do IN with a variable-length list
```

---

### EXISTS

```sql
-- EXISTS: true if subquery returns at least one row
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id
      AND o.status = 'completed'
)
-- Returns true for each user that has at least one completed order
-- EXISTS stops scanning after the first match — very efficient with proper index

-- NOT EXISTS: true if subquery returns no rows
WHERE NOT EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id
)
-- Returns true for users with NO orders (anti-join)
-- NULL-safe: if all orders for a user have NULL user_id, EXISTS still returns FALSE correctly
```

**EXISTS vs IN — the difference that matters:**

```sql
-- EXISTS: semi-join, stops at first match
WHERE EXISTS (SELECT 1 FROM orders WHERE orders.user_id = users.id)

-- IN with subquery: may run as a hash semi-join or as a correlated subquery
WHERE user_id IN (SELECT user_id FROM orders)

-- For the planner, both are often transformed to the same plan
-- BUT EXISTS is more predictable and handles NULLs correctly

-- EXISTS vs NOT IN — critical difference:
-- NOT IN with NULLs in subquery = empty result (the trap)
WHERE user_id NOT IN (SELECT manager_id FROM employees)
-- If ANY manager_id is NULL → returns NOTHING

WHERE NOT EXISTS (SELECT 1 FROM employees WHERE employees.manager_id = users.id)
-- NULL-safe: only checks if a matching row exists, doesn't compare NULL
```

**The EXISTS subquery select list doesn't matter:**
```sql
-- All of these are equivalent in performance and semantics:
WHERE EXISTS (SELECT 1 FROM orders WHERE user_id = u.id)
WHERE EXISTS (SELECT * FROM orders WHERE user_id = u.id)
WHERE EXISTS (SELECT NULL FROM orders WHERE user_id = u.id)
WHERE EXISTS (SELECT 42 FROM orders WHERE user_id = u.id)
-- PostgreSQL only checks if at least one row exists — never looks at what the subquery returns
-- Convention: use SELECT 1 to signal intent clearly
```

---

### Combining Conditions: AND, OR, NOT

```sql
-- AND: both conditions must be TRUE
WHERE status = 'active' AND price < 100

-- OR: at least one condition must be TRUE
WHERE status = 'active' OR status = 'pending'

-- NOT: inverts the condition
WHERE NOT (status = 'deleted')
WHERE status != 'deleted'  -- equivalent, but NOT is clearer for complex expressions

-- Complex combination — parentheses are mandatory
WHERE (status = 'active' OR status = 'pending')
  AND (price BETWEEN 10 AND 100)
  AND NOT (category_id IN (5, 6, 7))
  AND deleted_at IS NULL
```

**Short-circuit evaluation in PostgreSQL:**

PostgreSQL evaluates AND conditions left to right and stops at the first FALSE. It evaluates OR conditions left to right and stops at the first TRUE.

However: PostgreSQL is allowed to reorder qual evaluation for optimization purposes (based on cost estimates). You **cannot** rely on left-to-right evaluation order for side effects. You CAN rely on null safety by structuring conditions carefully:

```sql
-- Null guard pattern — works because PostgreSQL evaluates this safely:
WHERE data IS NOT NULL AND data->>'status' = 'active'
-- If data IS NULL, the second condition is never reached in practice
-- But PostgreSQL might reorder these — use CASE for guaranteed order:
WHERE CASE WHEN data IS NOT NULL THEN data->>'status' = 'active' ELSE FALSE END
```

---

### String Comparison Operators

```sql
-- Standard string equality is case-sensitive
WHERE name = 'Alice'   -- does NOT match 'alice' or 'ALICE'

-- Case-insensitive equality options:
WHERE lower(name) = lower($1)              -- explicit lowercase
WHERE name ILIKE $1                         -- case-insensitive LIKE (no wildcards = equality)
WHERE name = $1 COLLATE "case_insensitive" -- custom collation (PostgreSQL 12+)

-- String ordering
WHERE name > 'M'    -- alphabetically after 'M'
WHERE name <= 'Z'

-- String prefix (fast with index)
WHERE email LIKE 'admin@%'
```

---

### Numeric Comparison Edge Cases

```sql
-- Floating point comparison (avoid =)
WHERE price = 9.99  -- may fail due to floating point representation
-- Better: use NUMERIC type for money, not FLOAT

-- Integer division in conditions
WHERE total / quantity > 100   -- integer division if both are integers
                                -- 250 / 3 = 83, not 83.33
WHERE total::numeric / quantity > 100   -- cast first to get decimal division

-- Overflow in arithmetic conditions
WHERE quantity * unit_price > 1000000
-- If quantity * unit_price overflows INTEGER, result wraps around
-- Use BIGINT or cast: WHERE quantity::bigint * unit_price > 1000000
```

---

### Date/Time Comparisons in WHERE

```sql
-- Comparing dates (safe)
WHERE order_date = '2024-01-15'
WHERE order_date BETWEEN '2024-01-01' AND '2024-01-31'

-- Comparing timestamps — the BETWEEN trap
WHERE created_at BETWEEN '2024-01-01' AND '2024-01-31'
-- '2024-01-31' becomes '2024-01-31 00:00:00'
-- Rows on 2024-01-31 after midnight are EXCLUDED

-- Correct timestamp range (exclusive upper bound):
WHERE created_at >= '2024-01-01' AND created_at < '2024-02-01'

-- Timezone-aware comparisons
WHERE created_at AT TIME ZONE 'UTC' > '2024-01-01 00:00:00'
-- or compare directly when column is TIMESTAMPTZ
WHERE created_at > '2024-01-01 00:00:00+00'

-- Using functions (breaks index use — see sargability in Topic 08):
WHERE DATE(created_at) = '2024-01-15'     -- NOT sargable
WHERE created_at::date = '2024-01-15'     -- NOT sargable
-- Correct sargable version:
WHERE created_at >= '2024-01-15' AND created_at < '2024-01-16'
```

---

## EXPLAIN Output for WHERE Conditions

### Sargable condition — uses index
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name FROM users WHERE email = 'alice@example.com';
```
```
Index Scan using idx_users_email on users
    (cost=0.43..8.45 rows=1 width=40) (actual time=0.128..0.130 rows=1 loops=1)
  Index Cond: ((email)::text = 'alice@example.com'::text)
  Buffers: shared hit=3
Planning Time: 0.312 ms
Execution Time: 0.158 ms
```
`Index Cond` = condition pushed into the index scan. 3 buffer hits = 3 pages read. Extremely fast.

### Non-sargable condition — forced sequential scan
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name FROM users WHERE lower(email) = 'alice@example.com';
```
```
Seq Scan on users  (cost=0.00..2850.00 rows=500 width=40) (actual time=0.021..45.231 rows=1 loops=1)
  Filter: (lower((email)::text) = 'alice@example.com'::text)
  Rows Removed by Filter: 99999
  Buffers: shared hit=1350
Planning Time: 0.087 ms
Execution Time: 45.279 ms
```
`Filter` (not `Index Cond`) = PostgreSQL read every row from the heap and applied the function. 1,350 pages read vs 3. **283x more I/O**.

### IN list
```sql
EXPLAIN SELECT id FROM orders WHERE status IN ('pending', 'processing', 'completed');
```
```
Bitmap Heap Scan on orders  (cost=12.45..892.32 rows=4500 width=8)
  Recheck Cond: ((status)::text = ANY ('{pending,processing,completed}'::text[]))
  ->  Bitmap Index Scan on idx_orders_status
        (cost=0.00..11.32 rows=4500 width=0)
              Index Cond: ((status)::text = ANY ('{pending,processing,completed}'::text[]))
```
PostgreSQL converted the `IN` list to `= ANY(array)`. Uses Bitmap Index Scan for multi-value lookup.

### EXISTS semi-join
```sql
EXPLAIN
SELECT u.id FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```
```
Hash Join  (cost=125.00..1250.00 rows=800 width=4)
  Hash Cond: (u.id = o.user_id)
  ->  Seq Scan on users u  (cost=0.00..350.00 rows=1000 width=4)
  ->  Hash  (cost=95.00..95.00 rows=2400 width=4)
        ->  HashAggregate  (cost=71.00..95.00 rows=2400 width=4)
              Group Key: o.user_id
              ->  Seq Scan on orders o  (cost=0.00..59.00 rows=2400 width=4)
```
PostgreSQL transformed EXISTS into a Hash Join with HashAggregate (deduplication). The `SELECT 1` subquery becomes a grouped scan.

---

## Query Examples

### Example 1 — Basic: Comprehensive WHERE Conditions

```sql
-- E-commerce product search with multiple filter types
SELECT
    id,
    name,
    price,
    stock_count,
    category_id
FROM products
WHERE
    -- Active and in stock
    is_active = TRUE
    AND deleted_at IS NULL
    AND stock_count > 0

    -- Category filter (parameterized IN)
    AND category_id = ANY($1::int[])  -- $1 = array of category IDs

    -- Price range
    AND price BETWEEN $2 AND $3

    -- Search (prefix-safe for index)
    AND (name ILIKE $4 OR $4 IS NULL)  -- $4 = optional search prefix

    -- Exclude certain statuses
    AND status NOT IN ('discontinued', 'draft')

ORDER BY
    CASE WHEN $5 = 'price_asc'  THEN price END ASC,
    CASE WHEN $5 = 'price_desc' THEN price END DESC,
    name
LIMIT 50 OFFSET $6;
```

### Example 2 — Intermediate: Null-safe Audit Query

```sql
-- Find all configuration changes where a value actually changed
-- Must handle: value → value, NULL → value, value → NULL correctly
SELECT
    a.id,
    a.table_name,
    a.record_id,
    a.column_name,
    a.old_value,
    a.new_value,
    a.changed_at,
    a.changed_by
FROM audit_log a
WHERE
    -- Only real changes — NULL-safe comparison
    a.old_value IS DISTINCT FROM a.new_value

    -- Time range
    AND a.changed_at >= $1
    AND a.changed_at < $2

    -- Specific tables (parameterized list)
    AND a.table_name = ANY($3::text[])

    -- Exclude system-generated changes
    AND a.changed_by IS NOT NULL
    AND a.changed_by NOT LIKE 'system_%'

ORDER BY a.changed_at DESC
LIMIT 100;
```

### Example 3 — Production Grade: Complex Multi-Condition Query

```sql
-- SaaS billing: find accounts that need invoice generation
-- Rules:
-- 1. Account is active
-- 2. Has usage this billing period
-- 3. Either: no invoice yet for this period, OR last invoice was < 1 hour ago (retry)
-- 4. Not suspended
-- 5. Has a valid payment method

SELECT
    a.id AS account_id,
    a.name AS account_name,
    a.billing_cycle_day,
    a.plan_id,

    -- How much usage this period
    COALESCE(u.total_units, 0) AS billable_units,

    -- Invoice status
    i.id AS last_invoice_id,
    i.created_at AS last_invoice_at

FROM accounts a

-- Current period usage (aggregated already in usage_summary)
LEFT JOIN usage_summary u
    ON u.account_id = a.id
    AND u.period_start = date_trunc('month', NOW())

-- Most recent invoice this period
LEFT JOIN LATERAL (
    SELECT id, created_at
    FROM invoices
    WHERE account_id = a.id
      AND billing_period_start = date_trunc('month', NOW())
    ORDER BY created_at DESC
    LIMIT 1
) i ON TRUE

WHERE
    -- Account must be active
    a.status = 'active'
    AND a.deleted_at IS NULL

    -- Must have usage to bill (or a minimum monthly fee)
    AND (
        COALESCE(u.total_units, 0) > 0
        OR a.has_minimum_monthly_fee = TRUE
    )

    -- Invoice condition: no invoice yet OR last invoice was > 1 hour ago (retry window)
    AND (
        i.id IS NULL
        OR i.created_at < NOW() - INTERVAL '1 hour'
    )

    -- Not suspended and not in grace period
    AND a.status NOT IN ('suspended', 'grace_period', 'cancelled')

    -- Must have a valid payment method on file
    AND EXISTS (
        SELECT 1 FROM payment_methods pm
        WHERE pm.account_id = a.id
          AND pm.is_valid = TRUE
          AND pm.expires_at > NOW()
    )

    -- Exclude test accounts
    AND a.is_test = FALSE
    AND NOT (a.name ILIKE 'test%' OR a.name ILIKE 'demo%')

ORDER BY a.id;
```

---

## Wrong → Right Patterns

### Pattern 1: NULL Comparison with = Instead of IS

```sql
-- WRONG — returns 0 rows even if deleted_at has NULLs
SELECT * FROM users WHERE deleted_at = NULL;
SELECT * FROM users WHERE deleted_at != NULL;

-- RIGHT
SELECT * FROM users WHERE deleted_at IS NULL;
SELECT * FROM users WHERE deleted_at IS NOT NULL;

-- Proof: NULL = NULL is UNKNOWN, not TRUE
-- SELECT NULL = NULL; → returns NULL (UNKNOWN)
-- SELECT NULL IS NULL; → returns TRUE
```

### Pattern 2: NOT IN with Nullable Column

```sql
-- WRONG — if manager_id has even ONE NULL, returns zero rows
SELECT * FROM employees
WHERE employee_id NOT IN (SELECT manager_id FROM employees);

-- RIGHT — option A: filter NULLs in subquery
SELECT * FROM employees
WHERE employee_id NOT IN (
    SELECT manager_id FROM employees WHERE manager_id IS NOT NULL
);

-- RIGHT — option B: use NOT EXISTS (always correct, always fast)
SELECT * FROM employees e
WHERE NOT EXISTS (
    SELECT 1 FROM employees m
    WHERE m.manager_id = e.employee_id
);
```

### Pattern 3: BETWEEN with Timestamps

```sql
-- WRONG — misses everything on the last day after midnight
WHERE created_at BETWEEN '2024-03-01' AND '2024-03-31'
-- '2024-03-31' = '2024-03-31 00:00:00' — events at 14:30 on March 31 are excluded

-- RIGHT — use half-open interval with exclusive upper bound
WHERE created_at >= '2024-03-01' AND created_at < '2024-04-01'
-- Captures everything from March 1 00:00:00 up to (but not including) April 1 00:00:00
```

### Pattern 4: OR with Different Precedence Than Expected

```sql
-- WRONG — OR has lower precedence than AND:
WHERE user_type = 'admin' OR user_type = 'moderator' AND is_active = TRUE
-- Actually means: user_type = 'admin' OR (user_type = 'moderator' AND is_active = TRUE)
-- Admins are returned regardless of is_active

-- RIGHT — use parentheses
WHERE (user_type = 'admin' OR user_type = 'moderator') AND is_active = TRUE
-- OR better yet, use IN:
WHERE user_type IN ('admin', 'moderator') AND is_active = TRUE
```

### Pattern 5: LIKE Leading Wildcard on a Searchable Column

```sql
-- WRONG — cannot use index, full table scan
WHERE email LIKE '%@gmail.com'

-- RIGHT for suffix match — reverse the string and use prefix index:
CREATE INDEX idx_users_email_reversed ON users(reverse(email));
WHERE reverse(email) LIKE reverse('%@gmail.com')
-- reverse('%@gmail.com') = 'moc.liamg@%'
-- Now it's a prefix match on the reversed column

-- RIGHT for contains match — use pg_trgm:
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_users_email_trgm ON users USING GIN(email gin_trgm_ops);
WHERE email ILIKE '%gmail%'  -- now uses trigram index
```

### Pattern 6: Type Mismatch in Comparison

```sql
-- Potential performance issue — implicit cast may prevent index use
WHERE user_id = '42'   -- user_id is INT, '42' is TEXT

-- PostgreSQL is smart enough here: it casts the literal '42' to INT
-- This IS safe for literal values (PostgreSQL casts the constant, not the column)

-- DANGEROUS case: parameter type mismatch
-- In Node.js with pg, parameters are typed:
pool.query('SELECT * FROM users WHERE id = $1', ['42'])
-- pg sends '42' as text type, PostgreSQL must cast
-- With proper pg type inference, this usually works, but can cause plan issues

-- SAFEST: ensure type match in Node.js
pool.query('SELECT * FROM users WHERE id = $1', [42])   // number, not string
// or cast in SQL:
pool.query('SELECT * FROM users WHERE id = $1::int', ['42'])

-- TRULY DANGEROUS: column function causing index skip
WHERE CAST(user_id AS TEXT) = $1   -- wraps column in function → no index use
WHERE user_id::text = $1           -- same problem
-- RIGHT: cast the parameter, not the column
WHERE user_id = $1::int
```

---

## Performance Profile

| Condition | Index usable | Notes |
|-----------|-------------|-------|
| `col = value` | ✓ | Index seek |
| `col != value` | Rarely | Full scan usually cheaper; partial if > 95% match value |
| `col < value`, `col > value`, `col <= value`, `col >= value` | ✓ | Range scan on B-tree |
| `col BETWEEN a AND b` | ✓ | Range scan |
| `col IS NULL` | ✓ | NULLs stored in B-tree index |
| `col IS NOT NULL` | ✓ | If table has many NULLs, very selective |
| `col IN (v1, v2, ...)` | ✓ | Bitmap or index scan |
| `col NOT IN (v1, v2)` | Rarely | Full scan often cheaper; use `!= ANY` |
| `col = ANY($1)` | ✓ | Same as IN — use for parameterized arrays |
| `col LIKE 'prefix%'` | ✓ | Prefix range scan on B-tree |
| `col LIKE '%suffix'` | ✗ | Full scan always |
| `col LIKE '%contains%'` | ✗ | Full scan (use pg_trgm GIN for this) |
| `col ILIKE 'prefix%'` | ✗ | Need `lower(col)` index |
| `EXISTS (subquery)` | ✓ | Semi-join, stops at first match |
| `NOT EXISTS (subquery)` | ✓ | Anti-join |
| `function(col) = value` | ✗ | Function hides column from index lookup |
| `col IS DISTINCT FROM val` | ✓ | Equivalent to OR conditions |
| `col::other_type = val` | ✗ | Cast on column prevents index use |

---

## Node.js Integration

### Parameterized Queries — the Non-Negotiable Rule

```javascript
// NEVER do this — SQL injection risk:
const query = `SELECT * FROM users WHERE email = '${email}'`;

// ALWAYS use parameters:
const result = await pool.query(
  'SELECT id, email, name FROM users WHERE email = $1',
  [email]
);
```

### Multiple Conditions from Dynamic Filters

```javascript
// Building dynamic WHERE clauses safely
function buildUserQuery(filters) {
  const conditions = [];
  const params = [];
  let paramIndex = 1;

  if (filters.status) {
    conditions.push(`status = $${paramIndex++}`);
    params.push(filters.status);
  }

  if (filters.minAge !== undefined) {
    conditions.push(`age >= $${paramIndex++}`);
    params.push(filters.minAge);
  }

  if (filters.maxAge !== undefined) {
    conditions.push(`age <= $${paramIndex++}`);
    params.push(filters.maxAge);
  }

  if (filters.search) {
    const escapedSearch = filters.search.replace(/[%_\\]/g, '\\$&');
    conditions.push(`name ILIKE $${paramIndex++}`);
    params.push(`%${escapedSearch}%`);
  }

  if (filters.departmentIds && filters.departmentIds.length > 0) {
    conditions.push(`department_id = ANY($${paramIndex++}::int[])`);
    params.push(filters.departmentIds);
  }

  const whereClause = conditions.length > 0
    ? 'WHERE ' + conditions.join(' AND ')
    : '';

  const sql = `
    SELECT id, name, email, status, age, department_id
    FROM users
    ${whereClause}
    ORDER BY name
    LIMIT 100
  `;

  return { sql, params };
}

// Usage
const { sql, params } = buildUserQuery({
  status: 'active',
  minAge: 25,
  departmentIds: [1, 3, 5],
  search: 'jo%hn',  // LIKE metacharacters safely escaped
});
const result = await pool.query(sql, params);
```

### Array Parameters for IN Queries

```javascript
// The correct way to pass a dynamic list of IDs
async function getUsersByIds(pool, ids) {
  if (ids.length === 0) return [];

  const result = await pool.query(
    'SELECT id, name, email FROM users WHERE id = ANY($1::int[])',
    [ids]  // pg sends this as a PostgreSQL array
  );
  return result.rows;
}

// Note: = ANY($1::int[]) is the parameterized equivalent of IN (1, 2, 3)
// Do NOT build: `WHERE id IN (${ids.join(',')})` — SQL injection risk if IDs come from user input
```

### NULL-safe Comparison in Node.js

```javascript
// When you need to check if a value changed (audit trail, conditional update)
async function hasValueChanged(pool, userId, fieldName, newValue) {
  const result = await pool.query(
    `SELECT ($1::text IS DISTINCT FROM $2::text) AS changed`,
    [await getCurrentValue(userId, fieldName), newValue]
  );
  return result.rows[0].changed;
}

// Or in a WHERE clause:
async function getChangedRecords(pool, since) {
  const result = await pool.query(`
    SELECT id, old_value, new_value, changed_at
    FROM audit_log
    WHERE changed_at > $1
      AND old_value IS DISTINCT FROM new_value
    ORDER BY changed_at DESC
  `, [since]);
  return result.rows;
}
```

### Detecting Bad WHERE Patterns in Code Review

```javascript
// Utility to detect common WHERE clause mistakes in query strings
// (for development-time validation, not production)
function auditQuery(sql) {
  const warnings = [];

  if (/=\s*NULL/i.test(sql)) {
    warnings.push('Use IS NULL instead of = NULL');
  }
  if (/!=\s*NULL|<>\s*NULL/i.test(sql)) {
    warnings.push('Use IS NOT NULL instead of != NULL or <> NULL');
  }
  if (/LIKE\s+['"]%[^%]/i.test(sql)) {
    warnings.push('LIKE with leading wildcard (%) cannot use index');
  }
  if (/NOT IN\s*\(\s*SELECT/i.test(sql)) {
    warnings.push('NOT IN with subquery is NULL-unsafe. Consider NOT EXISTS');
  }

  return warnings;
}
```

---

## ORM Comparison

### 1. Prisma

**Can it handle WHERE conditions?** YES — comprehensive support for most conditions.

```typescript
// Equality, null checks, range
const users = await prisma.user.findMany({
  where: {
    status: 'active',                    // = 'active'
    deletedAt: null,                     // IS NULL
    price: { gte: 10, lte: 100 },       // BETWEEN equivalent
    name: { startsWith: 'John' },        // LIKE 'John%'
    name: { contains: 'john', mode: 'insensitive' }, // ILIKE '%john%'
  }
});

// IN equivalent
const products = await prisma.product.findMany({
  where: {
    categoryId: { in: [1, 2, 3] },   // IN (1, 2, 3)
    status: { notIn: ['deleted', 'draft'] },  // NOT IN — but Prisma handles NULLs safely
  }
});

// EXISTS equivalent — via relation filter
const usersWithOrders = await prisma.user.findMany({
  where: {
    orders: {
      some: { status: 'completed' }   // EXISTS (SELECT 1 FROM orders WHERE ...)
    }
  }
});

// NOT EXISTS equivalent
const usersWithoutOrders = await prisma.user.findMany({
  where: {
    orders: { none: {} }   // NOT EXISTS (SELECT 1 FROM orders WHERE user_id = ...)
  }
});

// AND / OR
const filtered = await prisma.user.findMany({
  where: {
    AND: [
      { status: 'active' },
      { OR: [{ role: 'admin' }, { role: 'moderator' }] }
    ]
  }
});
```

**Where Prisma breaks:**
- `IS DISTINCT FROM` not supported — raw SQL required
- Complex LIKE patterns with dynamic wildcards require `$queryRaw`
- `NOT IN` on relations with potential NULLs: Prisma generates correct SQL but you can't inspect it easily
- Custom SQL functions in WHERE (e.g., `WHERE date_trunc('month', created_at) = '2024-01-01'`) require `$queryRaw`

**Raw SQL escape hatch:**
```typescript
const result = await prisma.$queryRaw<User[]>`
  SELECT id, name FROM users
  WHERE lower(email) = ${email.toLowerCase()}
    AND created_at IS DISTINCT FROM updated_at
`;
```

**Verdict**: Prisma's WHERE handling is excellent for standard conditions. The `some`/`none` relation filters generate proper semi-joins. Use `$queryRaw` for `IS DISTINCT FROM`, date functions in WHERE, and complex expressions.

---

### 2. Drizzle ORM

**Can it handle WHERE conditions?** YES — very close to raw SQL.

```typescript
import { and, or, not, eq, ne, gt, gte, lt, lte, between, inArray, notInArray, isNull, isNotNull, like, ilike, sql, exists } from 'drizzle-orm';

const users = await db.select().from(usersTable).where(
  and(
    eq(usersTable.status, 'active'),
    isNull(usersTable.deletedAt),          // IS NULL
    between(usersTable.price, 10, 100),    // BETWEEN
    like(usersTable.name, 'John%'),        // LIKE
    ilike(usersTable.name, '%john%'),      // ILIKE
    inArray(usersTable.categoryId, [1, 2, 3]),  // IN
    notInArray(usersTable.status, ['deleted'])   // NOT IN
  )
);

// EXISTS
const usersWithOrders = await db.select().from(usersTable).where(
  exists(
    db.select({ one: sql`1` }).from(ordersTable)
      .where(eq(ordersTable.userId, usersTable.id))
  )
);

// IS DISTINCT FROM (using raw sql)
const changedRows = await db.select().from(auditLog).where(
  sql`${auditLog.oldValue} IS DISTINCT FROM ${auditLog.newValue}`
);

// OR with AND nesting
const complex = await db.select().from(usersTable).where(
  and(
    or(eq(usersTable.role, 'admin'), eq(usersTable.role, 'moderator')),
    eq(usersTable.isActive, true)
  )
);
```

**Where Drizzle breaks:**
- Some PostgreSQL-specific operators (`@>`, `&&`, `?`) need `sql` template tag
- Very complex correlated EXISTS subqueries may need raw SQL

**Raw SQL escape hatch:**
```typescript
const result = await db.execute(
  sql`SELECT id, name FROM users WHERE lower(email) = ${email.toLowerCase()}`
);
```

**Verdict**: Drizzle is the best ORM for WHERE clause control. Its operator imports are explicit, readable, and generate correct SQL. The `sql` template tag integrates seamlessly for anything not covered. Highly recommended for complex filtering.

---

### 3. Sequelize

**Can it handle WHERE conditions?** YES — via Op (Operators).

```javascript
const { Op } = require('sequelize');

const users = await User.findAll({
  where: {
    status: 'active',                    // = 'active'
    deletedAt: null,                     // IS NULL
    price: { [Op.between]: [10, 100] }, // BETWEEN
    name: { [Op.like]: 'John%' },       // LIKE
    name: { [Op.iLike]: '%john%' },     // ILIKE (PostgreSQL only)
    categoryId: { [Op.in]: [1, 2, 3] }, // IN
    status: { [Op.notIn]: ['deleted'] }, // NOT IN

    // AND (implicit when multiple keys)
    [Op.and]: [
      { status: 'active' },
      { [Op.or]: [{ role: 'admin' }, { role: 'moderator' }] }
    ]
  }
});

// IS NOT NULL
const active = await User.findAll({
  where: { deletedAt: { [Op.ne]: null } }  // generates IS NOT NULL
});

// NOT IN with Sequelize — watch out: Sequelize generates `NOT IN` not `!= ALL`
// If the list might contain NULLs from another query, use:
const safe = await User.findAll({
  where: {
    id: { [Op.notIn]: Sequelize.literal('(SELECT manager_id FROM employees WHERE manager_id IS NOT NULL)') }
  }
});
```

**Where Sequelize breaks:**
- `IS DISTINCT FROM` not natively supported — needs `Sequelize.literal()`
- Complex nested conditions become verbose with `Op` symbols
- `Op.notIn` with a subquery doesn't automatically add `WHERE col IS NOT NULL` — manual fix required
- Date function comparisons need `Sequelize.literal()`

**Raw SQL escape hatch:**
```javascript
const [results] = await sequelize.query(
  'SELECT id, name FROM users WHERE lower(email) = ? AND deleted_at IS NULL',
  { replacements: [email.toLowerCase()], type: QueryTypes.SELECT }
);
```

**Verdict**: Sequelize works for standard conditions but the `Op` symbol syntax is verbose and error-prone. The NOT IN + NULL trap requires manual handling. For anything involving `IS DISTINCT FROM`, date functions, or complex LIKE patterns, drop to raw SQL.

---

### 4. TypeORM

**Can it handle WHERE conditions?** YES — via `where` object or QueryBuilder.

```typescript
import { IsNull, Not, Between, In, Like, ILike, LessThan, MoreThan } from 'typeorm';

// Repository style
const users = await userRepository.find({
  where: {
    status: 'active',
    deletedAt: IsNull(),                  // IS NULL
    price: Between(10, 100),             // BETWEEN
    name: Like('John%'),                  // LIKE
    name: ILike('%john%'),               // ILIKE
    categoryId: In([1, 2, 3]),           // IN
    age: MoreThan(18),                   // > 18
  }
});

// NOT NULL
const withEmail = await userRepository.find({
  where: { email: Not(IsNull()) }        // IS NOT NULL
});

// OR conditions — requires array of where objects
const adminsOrMods = await userRepository.find({
  where: [
    { role: 'admin', isActive: true },
    { role: 'moderator', isActive: true }
  ]
  // Generates: WHERE (role = 'admin' AND is_active = true) OR (role = 'moderator' AND is_active = true)
});

// QueryBuilder for complex conditions
const result = await dataSource
  .getRepository(User)
  .createQueryBuilder('u')
  .where('u.status = :status', { status: 'active' })
  .andWhere('u.deleted_at IS NULL')
  .andWhere('u.age BETWEEN :min AND :max', { min: 18, max: 65 })
  .andWhere(qb => {
    const subQuery = qb.subQuery()
      .select('1')
      .from(Order, 'o')
      .where('o.user_id = u.id')
      .getQuery();
    return 'EXISTS ' + subQuery;
  })
  .getMany();
```

**Where TypeORM breaks:**
- `IS DISTINCT FROM` not supported
- OR conditions in object syntax require an array of full condition objects — awkward for nested logic
- Complex LIKE with dynamic escaping needs raw SQL
- `Not(In([...]))` generates `NOT IN` — has the NULL trap just like raw SQL

**Raw SQL escape hatch:**
```typescript
const result = await dataSource.query(
  `SELECT id, name FROM users WHERE lower(email) = $1 AND deleted_at IS NULL`,
  [email.toLowerCase()]
);
```

**Verdict**: TypeORM's repository `where` helpers work for simple conditions. The `QueryBuilder` handles complex WHERE clauses well but is verbose. The OR condition syntax (array of objects) is non-obvious. For `IS DISTINCT FROM` and complex correlated subqueries, use raw SQL.

---

### 5. Knex.js

**Can it handle WHERE conditions?** YES — closest to SQL syntax.

```javascript
// Standard conditions
const users = await knex('users')
  .where('status', 'active')                           // =
  .whereNull('deleted_at')                             // IS NULL
  .whereNotNull('email')                               // IS NOT NULL
  .whereBetween('price', [10, 100])                   // BETWEEN
  .whereIn('category_id', [1, 2, 3])                  // IN
  .whereNotIn('status', ['deleted', 'draft'])          // NOT IN
  .where('name', 'like', 'John%')                     // LIKE
  .where('name', 'ilike', '%john%')                   // ILIKE (PostgreSQL)
  .where('age', '>=', 18);                            // >=

// OR conditions
const result = await knex('users')
  .where('role', 'admin')
  .orWhere('role', 'moderator');

// Complex nested AND/OR
const complex = await knex('users')
  .where(function() {
    this.where('role', 'admin').orWhere('role', 'moderator')
  })
  .andWhere('is_active', true);

// EXISTS
const withOrders = await knex('users').where(
  knex.raw('EXISTS (SELECT 1 FROM orders WHERE orders.user_id = users.id AND orders.status = ?)', ['completed'])
);

// IS DISTINCT FROM (raw)
const changed = await knex('audit_log')
  .whereRaw('old_value IS DISTINCT FROM new_value');

// NOT IN safe version with Knex
const safe = await knex('users')
  .whereNotIn('id',
    knex('employees').select('manager_id').whereNotNull('manager_id')
  );
```

**Where Knex breaks:**
- `whereNotIn` has the same NULL trap as SQL — use `.whereNotNull()` on subquery explicitly
- No type safety on column names
- `orWhere` chaining can produce unexpected grouping without explicit wrapper functions

**Raw SQL escape hatch:**
```javascript
const result = await knex.raw(
  'SELECT id, name FROM users WHERE lower(email) = ? AND deleted_at IS NULL',
  [email.toLowerCase()]
);
const rows = result.rows;
```

**Verdict**: Knex is the most expressive ORM for WHERE clauses. Its `.whereRaw()` integration is clean. The chained `.where().orWhere()` syntax requires care with grouping. Use wrapper functions (`this.where(...).orWhere(...)`) to control AND/OR precedence explicitly.

---

### ORM Comparison Summary

| Feature | Prisma | Drizzle | Sequelize | TypeORM | Knex.js |
|---------|--------|---------|-----------|---------|---------|
| IS NULL | ✓ (null value) | ✓ (`isNull`) | ✓ (null value) | ✓ (`IsNull()`) | ✓ (`.whereNull`) |
| IS NOT NULL | ✓ (`not: null`) | ✓ (`isNotNull`) | ✓ (`Ne(null)`) | ✓ (`Not(IsNull())`) | ✓ (`.whereNotNull`) |
| IN | ✓ | ✓ | ✓ (`Op.in`) | ✓ (`In()`) | ✓ (`.whereIn`) |
| NOT IN (null-safe) | ✓ (safe) | ✓ | Manual | Manual | Manual |
| BETWEEN | ✓ | ✓ | ✓ (`Op.between`) | ✓ (`Between()`) | ✓ (`.whereBetween`) |
| LIKE | ✓ | ✓ | ✓ (`Op.like`) | ✓ (`Like()`) | ✓ |
| ILIKE | ✓ (`mode: insensitive`) | ✓ | ✓ (`Op.iLike`) | ✓ (`ILike()`) | ✓ |
| IS DISTINCT FROM | ✗ (raw SQL) | ✓ (`sql` tag) | ✗ (raw SQL) | ✗ (raw SQL) | ✓ (`.whereRaw`) |
| EXISTS | ✓ (relation filters) | ✓ | Partial | ✓ (QueryBuilder) | ✓ (`.whereRaw`) |
| Complex AND/OR | ✓ | ✓ | ✓ (verbose) | ✓ (verbose) | ✓ |
| Date functions in WHERE | ✗ (raw SQL) | ✓ (`sql` tag) | ✗ (literal) | ✗ (raw) | ✓ (`.whereRaw`) |

---

## Practice Exercises

### Exercise 1 — NULL Trap

Given this table:
```sql
CREATE TABLE tasks (
    id          SERIAL PRIMARY KEY,
    title       TEXT NOT NULL,
    assigned_to INT,           -- NULL means unassigned
    completed_at TIMESTAMPTZ,  -- NULL means not completed
    priority    TEXT
);
```

Write a query that returns all **unassigned tasks that are not yet completed**, ordered by priority (HIGH first, then MEDIUM, then LOW, then NULL).

Then explain what happens if you accidentally write `WHERE assigned_to = NULL AND completed_at = NULL`.

---

### Exercise 2 — NOT IN vs NOT EXISTS

Table: `users` (id, name, email)
Table: `banned_users` (user_id, banned_at, reason — where user_id can sometimes be NULL due to a data import bug)

Write the query to find all users who are NOT in the banned_users table. Write it:
1. Using NOT IN (and explain the exact scenario where it breaks)
2. Using NOT EXISTS (explain why this is safe)

---

### Exercise 3 — Operator Performance Audit

For each of the following WHERE conditions, identify if it can use a B-tree index on `col` and explain why:

```sql
WHERE col = 'value'
WHERE col LIKE 'val%'
WHERE col LIKE '%val'
WHERE col ILIKE 'val%'
WHERE lower(col) = 'val'
WHERE col IS NULL
WHERE col IS NOT NULL
WHERE col BETWEEN 10 AND 50
WHERE col IN (1, 2, 3)
WHERE col != ALL(ARRAY[1, 2, 3])
WHERE CAST(col AS TEXT) = 'value'
```

---

### Exercise 4 — Dynamic Safe Query Builder

Write a Node.js function `buildOrderSearch(filters)` that:
1. Accepts an object with optional fields: `status` (string), `minTotal` (number), `maxTotal` (number), `customerEmail` (string — may contain LIKE metacharacters), `createdAfter` (Date), `productIds` (array of numbers)
2. Builds a parameterized SQL query (no string interpolation of values)
3. Handles each filter as optional (only adds condition if field is provided)
4. Returns `{ sql, params }`

The base query should be: `SELECT id, status, total, customer_id, created_at FROM orders`

---

## Interview Questions

### Junior Level

**Q: What is the difference between `WHERE col = NULL` and `WHERE col IS NULL`?**

A: `WHERE col = NULL` never returns any rows. NULL represents an unknown value, and comparing anything to an unknown value produces UNKNOWN, not TRUE. Only TRUE rows pass WHERE. The correct way to check for NULL is `WHERE col IS NULL`, which uses the special IS NULL predicate that specifically tests whether the value is null.

---

**Q: Why does `WHERE status NOT IN ('deleted', 'banned')` fail when the status column has NULL values in some rows?**

A: NOT IN expands to a series of `!=` comparisons joined by AND. For a row with `status = NULL`, the condition becomes `NULL != 'deleted' AND NULL != 'banned'`, which evaluates to `UNKNOWN AND UNKNOWN = UNKNOWN`. WHERE excludes UNKNOWN rows. So rows with NULL status are excluded even though NULL is not 'deleted' or 'banned'. The fix is to add `AND status IS NOT NULL` or to use NOT EXISTS instead.

---

### Principal Level

**Q: You're optimizing a slow query. The WHERE clause is `WHERE EXTRACT(MONTH FROM created_at) = 6`. What's wrong with it and how do you fix it?**

A: The condition wraps the column in a function (`EXTRACT`), which prevents PostgreSQL from using an index on `created_at`. Even if there's a perfectly good index, PostgreSQL must evaluate the function for every row — a full sequential scan.

This is a **non-sargable condition** (the term comes from "Search ARGument ABLE" — can the condition use an index?). Any time a function or expression wraps the indexed column, the index is bypassed.

The fix is to rewrite the condition to isolate the column on one side:
```sql
-- For "all records in June of any year":
WHERE EXTRACT(MONTH FROM created_at) = 6  -- can't use index

-- Option A: Range condition (sargable, uses index)
-- Not straightforward for "any year" — need OR or generate_series

-- Option B: Expression index
CREATE INDEX idx_orders_month ON orders(EXTRACT(MONTH FROM created_at));
-- Now the original condition can use this expression index

-- Option C: If you want a specific year + month (most common production case):
WHERE created_at >= '2024-06-01' AND created_at < '2024-07-01'
-- Always sargable, always uses index if one exists on created_at
```

At principal level, I'd also ask: is this for a monthly aggregation in a reporting query? If so, `date_trunc('month', created_at)` on an expression index or pre-computed column in a materialized view may be better than range conditions in an ad-hoc query.

---

**Q: A developer writes `WHERE NOT EXISTS (SELECT 1 FROM orders WHERE orders.user_id = users.id)` and says it's equivalent to `WHERE users.id NOT IN (SELECT user_id FROM orders)`. Are they equivalent? When do they differ?**

A: They produce the same result **only** when `user_id` in orders is never NULL. They differ when NULLs exist in the subquery column.

- `NOT IN`: Expands to `id != val1 AND id != val2 AND ... AND id != NULL`. The `id != NULL` term evaluates to UNKNOWN. TRUE AND UNKNOWN = UNKNOWN. Every row is excluded — the entire result set is empty.

- `NOT EXISTS`: Checks whether a matching row exists. NULL values in the subquery column don't match (NULL = user_id evaluates to UNKNOWN, which EXISTS treats as "no match found"), so the EXISTS correctly returns FALSE for each non-matching user. NOT EXISTS then returns TRUE, and the row is included.

Practically: if `orders.user_id` has a NOT NULL constraint, they're identical in behavior (the planner may generate the same plan). If there's any chance of NULLs — which you should assume unless there's a constraint — NOT EXISTS is always safer.

The correct mental model: `NOT IN` is semantically fragile. `NOT EXISTS` is semantically robust. Prefer NOT EXISTS for anti-joins in production code.

---

## Mental Model Checkpoint

1. WHERE runs at Step 2 of the logical execution order. Name three things you **cannot** filter on in WHERE because of this, and what clause you should use for each instead.

2. `WHERE col = NULL` returns 0 rows. `WHERE col IS NULL` returns rows where col is null. What does `WHERE col IS NOT DISTINCT FROM NULL` return?

3. You have `WHERE a OR b AND c`. What is the actual evaluation order? Rewrite it with explicit parentheses to show what PostgreSQL computes.

4. A query has `WHERE user_id NOT IN (SELECT manager_id FROM org_chart)`. The org_chart table has a CEO row where manager_id is NULL. How many rows does your WHERE clause return? Why?

5. `LIKE 'abc%'` can use an index. `ILIKE 'abc%'` cannot (with a standard index). What are the two ways to make `ILIKE 'abc%'` use an index?

6. What does `WHERE price = ANY(ARRAY[10, 20, 30])` return? What does `WHERE price != ANY(ARRAY[10, 20, 30])` return? Are they complementary?

7. You need to find records where `old_status` and `new_status` are different, handling the case where either might be NULL (NULL → 'active' is a real change, 'active' → NULL is a real change, NULL → NULL is NOT a change). What operator do you use?

---

## Quick Reference Card

```
WHERE Evaluation
────────────────
Position: Step 2 (after FROM+JOINs, before GROUP BY)
Three values: TRUE (passes) | FALSE (excluded) | UNKNOWN (excluded)
UNKNOWN arises from: any comparison with NULL

NULL Checks
───────────
IS NULL          → correct
IS NOT NULL      → correct
= NULL           → always UNKNOWN (0 rows)
!= NULL          → always UNKNOWN (0 rows)
IS DISTINCT FROM     → null-safe !=
IS NOT DISTINCT FROM → null-safe =

NOT IN NULL Trap
─────────────────
WHERE id NOT IN (SELECT col FROM t)
If col has ANY NULL → returns 0 rows
Fix: Add WHERE col IS NOT NULL in subquery
OR: Use NOT EXISTS instead (always safe)

LIKE Index Rules
────────────────
'prefix%'  → index ✓
'%suffix'  → no index ✗
'%middle%' → no index ✗ (use pg_trgm GIN)
ILIKE      → no index ✗ (use lower(col) index)

Operator Precedence
────────────────────
NOT > AND > OR
Always use parentheses when mixing AND/OR

Sargable vs Non-Sargable
──────────────────────────
col = val           → sargable (index)
function(col) = val → NOT sargable (full scan)
col::type = val     → NOT sargable (full scan)
col = val::type     → sargable (cast the literal)

Array IN Pattern (Node.js)
───────────────────────────
WHERE id = ANY($1::int[])   -- parameterized IN
params: [[1, 2, 3]]          -- pg sends as array

BETWEEN Timestamp Trap
───────────────────────
BETWEEN '2024-01-01' AND '2024-01-31' -- misses 2024-01-31 after midnight
CORRECT: >= '2024-01-01' AND < '2024-02-01'
```

---

## Connected Topics

| Topic | Connection |
|-------|-----------|
| [02-logical-execution-order.md](02-logical-execution-order.md) | WHERE is Step 2 — why you can't reference SELECT aliases |
| [07-null-in-depth.md](07-null-in-depth.md) | Three-valued logic — the full story behind UNKNOWN in WHERE |
| [08-filtering-performance.md](08-filtering-performance.md) | Sargability — which WHERE conditions can use indexes and exactly why |
| [09-case-expressions.md](09-case-expressions.md) | Using CASE inside WHERE for complex conditional logic |
| [19-join-alternatives.md](19-join-alternatives.md) | EXISTS vs IN vs JOIN — deep comparison of semi-join patterns |
| [22-having.md](22-having.md) | WHERE vs HAVING — filtering rows vs filtering groups |
| [27-exists-not-exists.md](27-exists-not-exists.md) | EXISTS semantics deep dive — semi-joins and anti-joins |
| [42-index-types.md](42-index-types.md) | Which index types support which WHERE conditions |
| [44-composite-indexes.md](44-composite-indexes.md) | Designing indexes for multi-column WHERE clauses |
| [63-date-time-mastery.md](63-date-time-mastery.md) | Date range patterns in WHERE — BETWEEN traps with timestamps |
