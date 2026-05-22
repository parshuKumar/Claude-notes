# Topic 07 — NULL in Depth

**Phase 2 — SELECT and Filtering Foundations**
**Interview Priority: HIGH**

---

## ELI5 Analogy

Imagine a doctor's form with a field labelled "Number of previous surgeries." A patient leaves it blank. That blank is not zero — it's not "no surgeries." It means: **we don't know**. The patient might have had ten surgeries and just skipped the question, or they might have had none. The blank tells us nothing.

NULL in SQL is that blank field. It means **"the value is unknown or inapplicable."** It is not zero, not empty string, not false. It is the absence of any known value.

Now the problem: you're writing a WHERE clause to find all patients whose surgery count is different from 5. You write `WHERE surgeries != 5`. A patient with a blank form fails this check — because we don't *know* if their count is different from 5. Maybe it is, maybe it isn't. Unknown. So they're excluded from your results.

This is the core of three-valued logic, and it causes bugs in production every day.

---

## Internals Connection

NULL is not a value — it is a **marker**. In PostgreSQL's tuple storage (the heap), a NULL field is indicated by a **null bitmap** stored in the tuple header. The actual data bytes for that column aren't written to disk at all — this is a small storage optimization.

In expression evaluation, NULL propagates through operations like a virus:
- Any arithmetic involving NULL → NULL
- Any comparison involving NULL → NULL (not FALSE, NULL)
- Any string concatenation involving NULL → NULL (by default)
- Any function with a NULL argument → depends on function (most return NULL, some are NULL-safe)

The PostgreSQL executor uses a separate boolean "isnull" flag alongside every Datum (value) in its internal expression evaluation. When an expression evaluates a column, it checks this flag first. If the flag is set, the result is immediately marked as NULL without evaluating the expression.

This NULL propagation is why `NULL = NULL` returns NULL, not TRUE — the comparison function receives two "isnull" inputs and immediately returns NULL.

---

## Execution Order Context

NULL affects every clause:
- **WHERE**: UNKNOWN (NULL boolean result) rows are excluded — the same as FALSE
- **JOIN ON**: UNKNOWN in the ON condition causes rows to not match
- **GROUP BY**: NULLs are grouped together (all NULLs form one group)
- **ORDER BY**: NULLs sort last by default in ASC, first in DESC — controllable with `NULLS FIRST` / `NULLS LAST`
- **Aggregates**: NULLs are silently ignored by all aggregate functions except COUNT(*)
- **UNIQUE constraints**: Multiple NULLs are allowed in a UNIQUE column (NULL != NULL in constraint checks)
- **PRIMARY KEY**: NULLs are not allowed (PRIMARY KEY implies NOT NULL)

---

## What Is NULL?

NULL is the SQL representation of a missing or unknown value. It is defined by the SQL standard as implementing **three-valued logic** (3VL): every Boolean expression in SQL can evaluate to TRUE, FALSE, or UNKNOWN.

Key definitional properties:
1. NULL is not a value — it is an absence of value
2. NULL is not equal to anything, including itself
3. NULL is not not-equal to anything, including itself
4. Operations on NULL produce NULL (with exceptions listed below)
5. Boolean expressions that would produce TRUE or FALSE with known values produce UNKNOWN when NULL is involved

---

## Why It Matters in Production

### 1. Silent Wrong Results

The most dangerous NULL bugs produce no error — they silently return wrong data.

```sql
-- "Find all products that are NOT in the luxury category"
SELECT * FROM products WHERE category_id != 5;
-- Products where category_id IS NULL are silently excluded
-- Your "all other products" query is missing the uncategorized products
```

### 2. Aggregation Surprises

```sql
-- This looks like it counts all rows:
SELECT AVG(discount_pct) FROM products;
-- But if 3,000 of 10,000 products have NULL discount_pct,
-- AVG only averages over the 7,000 non-null rows
-- You never see a warning — the result is silently different from what you intended
```

### 3. JOIN Results with Unexpected Holes

LEFT JOINs produce NULLs for unmatched rows. Subsequent WHERE clauses that don't handle these NULLs effectively convert the LEFT JOIN into an INNER JOIN.

### 4. NOT IN + NULL = Empty Result

As established in Topic 06: `WHERE id NOT IN (subquery with NULLs)` returns zero rows. No error, no warning.

---

## Deep Technical Content

### The Truth Tables for Three-Valued Logic

**AND truth table:**

| AND | TRUE | FALSE | UNKNOWN |
|-----|------|-------|---------|
| **TRUE** | TRUE | FALSE | UNKNOWN |
| **FALSE** | FALSE | FALSE | FALSE |
| **UNKNOWN** | UNKNOWN | FALSE | UNKNOWN |

**OR truth table:**

| OR | TRUE | FALSE | UNKNOWN |
|----|------|-------|---------|
| **TRUE** | TRUE | TRUE | TRUE |
| **FALSE** | TRUE | FALSE | UNKNOWN |
| **UNKNOWN** | TRUE | UNKNOWN | UNKNOWN |

**NOT truth table:**

| NOT | Result |
|-----|--------|
| TRUE | FALSE |
| FALSE | TRUE |
| UNKNOWN | UNKNOWN |

**Critical observations from these tables:**
- `FALSE AND UNKNOWN = FALSE` — FALSE short-circuits AND (useful for null guards)
- `TRUE OR UNKNOWN = TRUE` — TRUE short-circuits OR
- `UNKNOWN AND UNKNOWN = UNKNOWN` — two unknowns don't resolve
- `NOT UNKNOWN = UNKNOWN` — negating unknown doesn't help

```sql
-- In WHERE, only TRUE passes. Both FALSE and UNKNOWN are excluded.

-- Proof:
SELECT 1 WHERE NULL = NULL;    -- 0 rows (UNKNOWN)
SELECT 1 WHERE NULL IS NULL;   -- 1 row (TRUE)
SELECT 1 WHERE NULL != NULL;   -- 0 rows (UNKNOWN)
SELECT 1 WHERE NOT (NULL = 1); -- 0 rows (NOT UNKNOWN = UNKNOWN)
```

---

### NULL in Arithmetic

```sql
SELECT NULL + 1;        -- NULL
SELECT NULL * 0;        -- NULL (NOT zero — unknown * 0 = unknown)
SELECT NULL / 2;        -- NULL
SELECT NULL || 'text';  -- NULL (string concatenation — default behavior)
SELECT NULL + NULL;     -- NULL
SELECT 5 - NULL;        -- NULL

-- Aggregate functions ignore NULLs:
SELECT SUM(NULL);       -- NULL (no rows contributed)
SELECT AVG(NULL);       -- NULL
SELECT MAX(NULL);       -- NULL
SELECT COUNT(NULL);     -- 0 (count of non-null values)
SELECT COUNT(*);        -- counts all rows including those with NULL columns
```

**The `NULL * 0 = NULL` trap:**
Mathematically, anything times zero is zero. In SQL, NULL times zero is NULL — because we don't know what the original value was, so we can't determine the result. This surprises developers who expect mathematical identity properties to hold.

**String concatenation special case:**
```sql
SELECT 'Hello' || NULL;         -- NULL (default)
SELECT CONCAT('Hello', NULL);   -- 'Hello' (CONCAT ignores NULLs)
SELECT 'Hello' || COALESCE(NULL, ''); -- 'Hello' (explicit handling)
```

`CONCAT()` is NULL-safe by design. The `||` operator is not. Use `CONCAT()` when concatenating potentially NULL values.

---

### NULL in Comparisons

Every standard comparison operator (`=`, `!=`, `<`, `>`, `<=`, `>=`) returns UNKNOWN when either operand is NULL:

```sql
SELECT NULL = NULL;   -- NULL (UNKNOWN)
SELECT NULL != NULL;  -- NULL (UNKNOWN)
SELECT NULL = 1;      -- NULL (UNKNOWN)
SELECT NULL != 1;     -- NULL (UNKNOWN)
SELECT NULL > 1;      -- NULL (UNKNOWN)
SELECT NULL < 1;      -- NULL (UNKNOWN)

-- Proof of real-world impact:
CREATE TABLE test_null (val INT);
INSERT INTO test_null VALUES (1), (2), (NULL), (NULL), (3);

SELECT * FROM test_null WHERE val = 2;     -- returns 1 row (val=2)
SELECT * FROM test_null WHERE val != 2;    -- returns 2 rows (val=1, val=3) — NULLs excluded
SELECT * FROM test_null WHERE val = NULL;  -- returns 0 rows — always UNKNOWN
SELECT * FROM test_null WHERE val IS NULL; -- returns 2 rows — NULLs found
```

The "missing rows" in `WHERE val != 2` is the most common production bug caused by NULLs. If a developer expects "all rows where val is not 2" but gets only the rows with explicit non-2 values, NULLs have been silently excluded.

---

### IS NULL and IS NOT NULL

The only correct way to test for NULL:

```sql
-- Syntax
WHERE col IS NULL          -- true only when col is NULL
WHERE col IS NOT NULL      -- true only when col has a non-NULL value

-- These are NOT expressions with an operator in the usual sense
-- They are special predicates in the SQL grammar
-- They always return TRUE or FALSE, never UNKNOWN

SELECT NULL IS NULL;       -- TRUE
SELECT NULL IS NOT NULL;   -- FALSE
SELECT 1 IS NULL;          -- FALSE
SELECT 1 IS NOT NULL;      -- TRUE
```

---

### IS DISTINCT FROM and IS NOT DISTINCT FROM

These are NULL-safe versions of `!=` and `=` respectively. They treat NULL as a known, comparable value:

```sql
-- IS DISTINCT FROM: true if values are different (NULL-safe)
SELECT NULL IS DISTINCT FROM NULL;    -- FALSE (both null = same = not distinct)
SELECT NULL IS DISTINCT FROM 1;       -- TRUE (null vs 1 = different = distinct)
SELECT 1 IS DISTINCT FROM 1;          -- FALSE (same value = not distinct)
SELECT 1 IS DISTINCT FROM 2;          -- TRUE (different values = distinct)

-- IS NOT DISTINCT FROM: true if values are the same (NULL-safe)
SELECT NULL IS NOT DISTINCT FROM NULL;   -- TRUE
SELECT NULL IS NOT DISTINCT FROM 1;      -- FALSE
SELECT 1 IS NOT DISTINCT FROM 1;         -- TRUE
SELECT 1 IS NOT DISTINCT FROM 2;         -- FALSE
```

**When to use `IS DISTINCT FROM`:**
```sql
-- Detecting genuine changes in audit queries
UPDATE products
SET price = $1
WHERE id = $2
  AND price IS DISTINCT FROM $1;  -- only update if value actually changes
                                    -- handles: NULL → value, value → NULL, value → different value
                                    -- skips: same value → same value, NULL → NULL

-- Comparing nullable columns in JOIN conditions
SELECT a.*, b.*
FROM table_a a
JOIN table_b b ON a.key IS NOT DISTINCT FROM b.key;
-- Matches even when both keys are NULL (standard = would not match)
```

---

### COALESCE

Returns the first non-NULL argument in its list:

```sql
COALESCE(val1, val2, val3, ...)
-- Returns val1 if not NULL, else val2 if not NULL, else val3, etc.
-- Returns NULL only if ALL arguments are NULL

SELECT COALESCE(NULL, NULL, 5, NULL);   -- 5
SELECT COALESCE(NULL, NULL, NULL);       -- NULL
SELECT COALESCE(1, 2, 3);               -- 1 (first non-null)

-- Common uses:
SELECT COALESCE(middle_name, '') AS middle_name        -- empty string instead of NULL
SELECT COALESCE(discount_pct, 0) AS discount           -- zero instead of NULL
SELECT COALESCE(a.override_price, b.base_price) AS price -- first non-null wins

-- In computation:
SELECT quantity * COALESCE(unit_price, 0) AS line_total
-- Without COALESCE: if unit_price is NULL, line_total is NULL
-- With COALESCE: if unit_price is NULL, treat as 0

-- COALESCE is lazy (short-circuit):
SELECT COALESCE(1, 1/0);   -- returns 1 without evaluating 1/0
-- Once a non-NULL is found, remaining arguments are NOT evaluated
```

**COALESCE for default values in expressions:**
```sql
-- Calculating effective discount
SELECT
    price,
    COALESCE(sale_price, price) AS effective_price,
    price - COALESCE(sale_price, price) AS savings,
    CASE
        WHEN sale_price IS NOT NULL
        THEN ROUND((1 - sale_price / price) * 100, 1)
        ELSE 0
    END AS discount_pct
FROM products;
```

---

### NULLIF

Returns NULL if the two arguments are equal, otherwise returns the first argument:

```sql
NULLIF(val, compare_val)
-- If val = compare_val → NULL
-- If val != compare_val → val

SELECT NULLIF(5, 5);      -- NULL (equal)
SELECT NULLIF(5, 3);      -- 5 (not equal)
SELECT NULLIF('', '');    -- NULL
SELECT NULLIF('a', '');   -- 'a'

-- Primary use case: division by zero protection
SELECT total / NULLIF(quantity, 0) AS unit_cost
-- If quantity is 0: NULLIF returns NULL, total/NULL = NULL (no error)
-- If quantity is not 0: NULLIF returns quantity, normal division

-- Converting "sentinel" values to NULL:
SELECT NULLIF(phone, 'N/A') AS phone_or_null
-- Data imported with 'N/A' for missing phones → convert to proper NULL
SELECT NULLIF(score, -1) AS score_or_null
-- -1 used as "not scored" sentinel → convert to NULL
```

**COALESCE + NULLIF combo (very useful):**
```sql
-- Problem: some rows have empty string '', some have NULL, you want NULL for both
SELECT COALESCE(NULLIF(TRIM(phone), ''), NULL) AS clean_phone
-- TRIM removes whitespace
-- NULLIF converts empty string to NULL
-- COALESCE with NULL is redundant here but makes intent clear

-- More common pattern:
SELECT NULLIF(TRIM(notes), '') AS notes_or_null
-- Empty string or whitespace-only → NULL
-- Actual text → returned as-is
```

---

### NULL in Aggregate Functions

All aggregate functions (except COUNT(*)) **ignore NULL values silently**:

```sql
CREATE TABLE scores (player TEXT, score INT);
INSERT INTO scores VALUES
    ('Alice', 95), ('Bob', NULL), ('Charlie', 87),
    ('Diana', NULL), ('Eve', 72);

SELECT
    COUNT(*)          AS total_rows,         -- 5 (counts all rows)
    COUNT(score)      AS scored_players,     -- 3 (ignores NULLs)
    COUNT(*) - COUNT(score) AS unscored,     -- 2 (difference = NULL count)
    SUM(score)        AS total_score,        -- 254 (95+87+72, NULLs ignored)
    AVG(score)        AS avg_score,          -- 84.67 (254/3, not 254/5)
    MIN(score)        AS min_score,          -- 72
    MAX(score)        AS max_score,          -- 95
    SUM(score) / COUNT(score) AS manual_avg  -- 84 (integer division of 254/3)
FROM scores;
```

**The AVG trap:** `AVG(score) = 84.67` but the actual average across all 5 players (treating unscored as 0) would be `254/5 = 50.8`. Neither answer is "wrong" — they answer different questions. The question is which one your business logic requires.

```sql
-- If you want NULLs to count as 0 in AVG:
SELECT AVG(COALESCE(score, 0)) AS avg_including_unscored
FROM scores;
-- Returns: 50.8 (254 / 5)
```

**COUNT(*) vs COUNT(col):**
```sql
SELECT COUNT(*)     FROM scores;  -- 5 (total rows)
SELECT COUNT(score) FROM scores;  -- 3 (non-NULL scores only)
SELECT COUNT(1)     FROM scores;  -- 5 (same as COUNT(*) — the 1 is non-null)

-- Counting NULLs directly:
SELECT COUNT(*) - COUNT(score) FROM scores;  -- 2 (NULLs = total - non-nulls)
-- OR:
SELECT COUNT(CASE WHEN score IS NULL THEN 1 END) FROM scores;  -- 2
```

---

### NULL in JOINs

NULLs in JOIN conditions cause rows not to match — because `NULL = NULL` is UNKNOWN, not TRUE:

```sql
CREATE TABLE employees (id INT, manager_id INT, name TEXT);
INSERT INTO employees VALUES
    (1, NULL, 'CEO'),
    (2, 1, 'VP'),
    (3, 1, 'VP'),
    (4, 2, 'Manager');

-- Self join to find each employee's manager
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id;
```

Result:
```
employee | manager
---------|--------
CEO      | NULL     ← manager_id is NULL, no match found, LEFT JOIN gives NULL
VP       | CEO
VP       | CEO
Manager  | VP
```

The CEO has `manager_id = NULL`. `NULL = m.id` is UNKNOWN for every row in `m`, so no match. The LEFT JOIN ensures the CEO's row still appears but with NULL for the manager columns.

**The NULL in INNER JOIN:**
```sql
SELECT e.name, m.name
FROM employees e
INNER JOIN employees m ON m.id = e.manager_id;
-- CEO is excluded entirely — INNER JOIN drops unmatched rows
```

---

### NULL in UNIQUE Constraints

SQL standard and PostgreSQL: **multiple NULLs are allowed in a UNIQUE column**. Because NULL != NULL, each NULL is considered distinct.

```sql
CREATE TABLE settings (
    user_id INT,
    key TEXT,
    value TEXT,
    UNIQUE(user_id, key)
);

-- These both succeed even though user_id is NULL:
INSERT INTO settings VALUES (NULL, 'theme', 'dark');
INSERT INTO settings VALUES (NULL, 'theme', 'light');
-- No conflict! NULL != NULL in UNIQUE check

-- This fails because (1, 'theme') already exists:
INSERT INTO settings VALUES (1, 'theme', 'dark');
INSERT INTO settings VALUES (1, 'theme', 'light');  -- ERROR: duplicate key
```

This behavior is correct by the SQL standard but counterintuitive. If you want only one NULL allowed per unique group, you need a partial unique index:

```sql
-- Allow only one NULL per key globally:
CREATE UNIQUE INDEX idx_settings_null_user
ON settings(key)
WHERE user_id IS NULL;
-- Now only one row with user_id=NULL and key='theme' is allowed
```

---

### NULL in ORDER BY

Default sort order for NULLs:
- `ORDER BY col ASC` → NULLs appear **last** (PostgreSQL default)
- `ORDER BY col DESC` → NULLs appear **first** (PostgreSQL default)

Explicit control:
```sql
ORDER BY col ASC NULLS FIRST   -- NULLs before all non-null values
ORDER BY col ASC NULLS LAST    -- NULLs after all non-null values (default for ASC)
ORDER BY col DESC NULLS FIRST  -- NULLs before all non-null values (default for DESC)
ORDER BY col DESC NULLS LAST   -- NULLs after all non-null values
```

```sql
-- Practical: show products with unknown price last
SELECT name, price FROM products ORDER BY price ASC NULLS LAST;

-- Practical: show unassigned tasks last in a task list
SELECT title, assigned_to FROM tasks ORDER BY assigned_to ASC NULLS LAST;
```

---

### NULL in GROUP BY

GROUP BY treats all NULLs as equal — they form a single group:

```sql
SELECT category_id, COUNT(*) FROM products GROUP BY category_id;
-- If category_id has NULLs, all products with NULL category_id
-- form ONE group with category_id = NULL in the output

-- Result might look like:
-- category_id | count
-- ------------|------
-- 1           | 250
-- 2           | 180
-- NULL        | 45     ← all uncategorized products grouped together
-- 3           | 95
```

This is useful — uncategorized products are aggregated together. But you need to know it happens.

---

### NULL in Window Functions

Window functions handle NULLs in their ORDER BY like regular ORDER BY. The frame boundary calculations with RANGE mode can be affected by NULLs.

For LAG/LEAD: if the offset row doesn't exist, the default value is returned (NULL by default, or the specified default):
```sql
SELECT price, LAG(price, 1, 0) OVER (ORDER BY created_at) AS prev_price
-- If there's no previous row, returns 0 (the specified default)
-- But if the previous row EXISTS and its price IS NULL, returns NULL (not 0)
-- The default only applies when no row exists, not when the row has NULL
```

---

### NULL-Related Functions Summary

```sql
-- Check for NULL
IS NULL
IS NOT NULL
IS DISTINCT FROM val      -- null-safe !=
IS NOT DISTINCT FROM val  -- null-safe =

-- Return NULL conditionally
NULLIF(val, match)        -- returns NULL if val = match

-- Replace NULL
COALESCE(v1, v2, ...)     -- first non-NULL
IFNULL(val, default)      -- non-standard, use COALESCE

-- Null-safe concatenation
CONCAT(s1, s2)            -- treats NULL as ''
CONCAT_WS(sep, s1, s2)   -- NULL-safe with separator

-- Aggregate behavior with NULLs
COUNT(*)                  -- counts NULLs (counts rows)
COUNT(col)                -- ignores NULLs
SUM, AVG, MIN, MAX, etc.  -- all ignore NULLs
ARRAY_AGG(col)            -- includes NULLs in array by default
STRING_AGG(col, sep)      -- ignores NULLs

-- Explicit NULL handling in CASE
CASE WHEN col IS NULL THEN 'default' ELSE col END
-- same as COALESCE but more explicit
```

---

### NULL and Indexes

B-tree indexes in PostgreSQL **store NULL entries** (at the end of the sorted structure). This means:
- `WHERE col IS NULL` can use an index scan
- `WHERE col IS NOT NULL` can use an index scan

However, if most rows have NULL in a column, a condition like `WHERE col IS NOT NULL` will return very few rows, and PostgreSQL might choose the index. But if most rows are NOT NULL (common case), `WHERE col IS NULL` is highly selective and will definitely use the index.

**Partial index without NULLs:**
```sql
-- If you only query non-null values, exclude NULLs from the index
CREATE INDEX idx_orders_completed ON orders(completed_at)
WHERE completed_at IS NOT NULL;
-- Smaller index, faster inserts (NULL rows don't touch this index)
-- WHERE completed_at > '2024-01-01' will use this index
-- WHERE completed_at IS NULL will NOT use this index (correct — NULLs not indexed)
```

---

## EXPLAIN Output for NULL Conditions

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name FROM users WHERE deleted_at IS NULL;
```
```
Index Scan using idx_users_deleted_at on users
    (cost=0.43..245.00 rows=9800 width=40) (actual time=0.052..8.234 rows=9823 loops=1)
  Index Cond: (deleted_at IS NULL)
  Buffers: shared hit=87
```
PostgreSQL pushes `IS NULL` into the index condition — it's fully sargable.

```sql
EXPLAIN SELECT id FROM products WHERE price IS NOT DISTINCT FROM target_price;
```
```
Seq Scan on products  (cost=0.00..245.00 rows=1 width=4)
  Filter: (price IS NOT DISTINCT FROM target_price)
```
`IS NOT DISTINCT FROM` on a non-indexed column → Seq Scan. If you need this on a large table, you'd need an expression index or restructure the query.

---

## Query Examples

### Example 1 — Basic: Null-Aware User Query

```sql
-- Find all active users and their last login information
-- Handle: users who never logged in (last_login_at IS NULL)
-- Handle: users with no display name (show email instead)

SELECT
    id,
    email,
    COALESCE(display_name, email) AS name,       -- fallback to email if no display name
    COALESCE(
        TO_CHAR(last_login_at, 'YYYY-MM-DD'),
        'Never'
    ) AS last_login,                              -- human-readable or 'Never'
    EXTRACT(days FROM NOW() - COALESCE(
        last_login_at,
        created_at                                -- for never-logged-in, use creation date
    )) AS days_since_activity,
    CASE
        WHEN last_login_at IS NULL THEN 'never_logged_in'
        WHEN last_login_at < NOW() - INTERVAL '90 days' THEN 'dormant'
        WHEN last_login_at < NOW() - INTERVAL '30 days' THEN 'inactive'
        ELSE 'active'
    END AS activity_status
FROM users
WHERE
    deleted_at IS NULL                          -- not soft-deleted
    AND is_active = TRUE
ORDER BY last_login_at DESC NULLS LAST;         -- never-logged-in users at the end
```

### Example 2 — Intermediate: Null-Safe Comparison in Upsert Logic

```sql
-- Log configuration changes, but only when the value actually changed
-- Must handle: NULL → value, value → NULL, value → different value

INSERT INTO config_audit (config_key, old_value, new_value, changed_by, changed_at)
SELECT
    c.key,
    c.value AS old_value,
    $2 AS new_value,
    $3 AS changed_by,
    NOW() AS changed_at
FROM config c
WHERE c.key = $1
  AND c.value IS DISTINCT FROM $2  -- only log if actually different
                                    -- handles: NULL != 'new_val', 'old_val' != NULL, NULL != NULL (FALSE)
RETURNING id;

-- Then update
UPDATE config
SET value = $2, updated_at = NOW()
WHERE key = $1
  AND value IS DISTINCT FROM $2;  -- skip update if value didn't change
```

### Example 3 — Production Grade: Reporting with NULL-Safe Aggregations

```sql
-- Sales performance report
-- Problem: some sales reps have no sales (NULL from LEFT JOIN),
--          some products have no category,
--          commission_rate may be NULL for new reps

WITH rep_sales AS (
    SELECT
        r.id AS rep_id,
        r.name AS rep_name,
        r.commission_rate,         -- may be NULL for new reps
        COUNT(s.id) AS sale_count,
        SUM(s.amount) AS total_amount,
        AVG(s.amount) AS avg_sale,
        MIN(s.sale_date) AS first_sale,
        MAX(s.sale_date) AS last_sale
    FROM sales_reps r
    LEFT JOIN sales s ON s.rep_id = r.id
        AND s.sale_date >= $1      -- filter on the joined table, not WHERE
        AND s.sale_date < $2       -- (WHERE would turn LEFT into INNER)
    WHERE r.is_active = TRUE
    GROUP BY r.id, r.name, r.commission_rate
),
rep_stats AS (
    SELECT
        rep_id,
        rep_name,
        -- Null-safe commission calculation
        COALESCE(commission_rate, 0.05) AS effective_commission_rate,
        sale_count,
        COALESCE(total_amount, 0) AS total_amount,   -- NULL from SUM when no sales
        COALESCE(avg_sale, 0) AS avg_sale,            -- NULL from AVG when no sales
        COALESCE(total_amount, 0) * COALESCE(commission_rate, 0.05) AS estimated_commission,
        -- Tenure classification
        CASE
            WHEN first_sale IS NULL THEN 'no_sales'
            WHEN first_sale < $1 - INTERVAL '1 year' THEN 'veteran'
            WHEN first_sale < $1 - INTERVAL '3 months' THEN 'established'
            ELSE 'new'
        END AS tenure_class,
        -- Days since last sale (NULL means never sold)
        CASE
            WHEN last_sale IS NULL THEN NULL
            ELSE EXTRACT(days FROM $2 - last_sale)::INT
        END AS days_since_last_sale
    FROM rep_sales
)
SELECT
    rep_name,
    sale_count,
    total_amount,
    avg_sale,
    effective_commission_rate,
    estimated_commission,
    tenure_class,
    COALESCE(days_since_last_sale::TEXT, 'Never sold') AS last_sale_info
FROM rep_stats
ORDER BY total_amount DESC NULLS LAST, rep_name;
```

---

## Wrong → Right Patterns

### Pattern 1: Comparing to NULL with =

```sql
-- WRONG
SELECT * FROM orders WHERE cancelled_at = NULL;    -- 0 rows always
SELECT * FROM orders WHERE cancelled_at != NULL;   -- 0 rows always

-- RIGHT
SELECT * FROM orders WHERE cancelled_at IS NULL;
SELECT * FROM orders WHERE cancelled_at IS NOT NULL;
```

### Pattern 2: Missing NULLs in Inequality Filter

```sql
-- WRONG — "all products not in category 5" but NULLs are excluded silently
SELECT * FROM products WHERE category_id != 5;
-- Returns: products where category_id is a non-null value other than 5
-- Missing: products where category_id IS NULL (uncategorized)

-- RIGHT — depending on business intent:
-- Option A: include uncategorized products
SELECT * FROM products WHERE category_id != 5 OR category_id IS NULL;

-- Option B: exclude uncategorized products intentionally
SELECT * FROM products WHERE category_id != 5 AND category_id IS NOT NULL;
-- (technically same as the WRONG query but explicit — communicates intent)
```

### Pattern 3: AVG Silently Excluding NULLs

```sql
-- WRONG (silently computes average only over non-NULL rows)
SELECT AVG(response_time_ms) AS avg_response FROM api_requests;
-- If 40% of requests have NULL response_time (timed out), this averages only 60%

-- RIGHT — depends on what you mean:
-- Option A: average only successful responses
SELECT AVG(response_time_ms) AS avg_successful_response
FROM api_requests
WHERE response_time_ms IS NOT NULL;  -- explicit, communicates intent

-- Option B: include timeouts as a high value (e.g., 30000ms)
SELECT AVG(COALESCE(response_time_ms, 30000)) AS avg_including_timeouts
FROM api_requests;

-- Option C: report both
SELECT
    COUNT(*) AS total_requests,
    COUNT(response_time_ms) AS successful_requests,
    COUNT(*) - COUNT(response_time_ms) AS failed_requests,
    AVG(response_time_ms) AS avg_successful_ms,
    AVG(COALESCE(response_time_ms, 30000)) AS avg_with_failures
FROM api_requests;
```

### Pattern 4: CONCAT with NULL

```sql
-- WRONG — any NULL component makes the whole string NULL
SELECT first_name || ' ' || last_name AS full_name FROM users;
-- If last_name IS NULL → full_name is NULL

-- RIGHT — option A: COALESCE each part
SELECT first_name || ' ' || COALESCE(last_name, '') AS full_name FROM users;
-- Downside: "John " (trailing space for users with no last name)

-- RIGHT — option B: use CONCAT (NULL-safe)
SELECT CONCAT(first_name, ' ', last_name) AS full_name FROM users;
-- CONCAT ignores NULLs: CONCAT('John', ' ', NULL) = 'John '

-- RIGHT — option C: use CONCAT_WS (separator version, ignores NULLs cleanly)
SELECT CONCAT_WS(' ', first_name, last_name) AS full_name FROM users;
-- CONCAT_WS skips NULLs AND skips the separator for them
-- CONCAT_WS(' ', 'John', NULL) = 'John' (no trailing space)
-- CONCAT_WS(' ', NULL, 'Smith') = 'Smith' (no leading space)
-- CONCAT_WS(' ', 'John', 'Smith') = 'John Smith'
```

### Pattern 5: Division by Zero via NULL

```sql
-- WRONG — error if denominator is 0
SELECT total_revenue / total_orders AS avg_order_value FROM report;
-- ERROR: division by zero if total_orders = 0

-- WRONG — another approach that still errors:
SELECT CASE WHEN total_orders = 0 THEN NULL ELSE total_revenue / total_orders END;
-- Works, but verbose

-- RIGHT — NULLIF elegantly returns NULL instead of dividing by zero
SELECT total_revenue / NULLIF(total_orders, 0) AS avg_order_value FROM report;
-- NULLIF(total_orders, 0) returns NULL when total_orders = 0
-- total_revenue / NULL = NULL (no error)

-- Also works for floating point:
SELECT 100.0 * successful_calls / NULLIF(total_calls, 0) AS success_rate FROM metrics;
```

---

## Performance Profile

| NULL Operation | Performance Notes |
|----------------|-------------------|
| `IS NULL` | Index-friendly — B-tree stores NULLs |
| `IS NOT NULL` | Index-friendly — B-tree scan from first non-null |
| `IS DISTINCT FROM val` | NOT index-friendly for indexed column — treated as expression |
| `COALESCE(col, default)` | Not sargable if col is indexed — wraps the column |
| `COALESCE(col, default) = val` | Non-sargable (function wraps column) |
| `col = val OR col IS NULL` | Sargable — two index conditions OR'd together |
| `NULLIF(col, sentinel)` | Not sargable — wraps column in function |

**The COALESCE sargability trap:**
```sql
-- WRONG for performance:
WHERE COALESCE(status, 'unknown') = 'active'
-- PostgreSQL must evaluate COALESCE(status, ...) for every row → full scan

-- RIGHT:
WHERE status = 'active'                         -- non-null case
-- or:
WHERE status = 'active' OR status IS NULL       -- include NULLs as "active"
-- or:
WHERE status IS NOT DISTINCT FROM 'active'      -- null-safe equality (cleaner but same issue)
```

---

## Node.js Integration

### Handling NULL in pg Results

```javascript
// PostgreSQL NULL comes through as JavaScript null in pg
const result = await pool.query(
  'SELECT id, name, middle_name, deleted_at FROM users WHERE id = $1',
  [userId]
);

const user = result.rows[0];
console.log(user.middle_name);   // null (not undefined, not '')
console.log(user.deleted_at);    // null if not deleted

// Safe property access patterns:
const displayName = user.middle_name ?? 'N/A';        // nullish coalescing
const isDeleted = user.deleted_at !== null;
const fullName = [user.first_name, user.middle_name, user.last_name]
  .filter(Boolean)                                      // removes null/undefined/empty
  .join(' ');
```

### Sending NULL as a Parameter

```javascript
// To send NULL to PostgreSQL:
await pool.query(
  'UPDATE users SET deleted_at = $1 WHERE id = $2',
  [null, userId]   // JavaScript null → PostgreSQL NULL
);

// Conditional NULL:
const deletedAt = shouldDelete ? new Date() : null;
await pool.query(
  'UPDATE users SET deleted_at = $1 WHERE id = $2',
  [deletedAt, userId]
);

// Watch out: undefined is NOT null in pg
await pool.query('UPDATE users SET name = $1', [undefined]);
// This will error or behave unexpectedly — always use null explicitly
```

### Null-Safe Comparison in Application Code

```javascript
// When building update queries, skip unchanged fields but handle NULL correctly
async function updateUser(pool, userId, updates) {
  // Get current state
  const { rows: [current] } = await pool.query(
    'SELECT name, email, phone FROM users WHERE id = $1',
    [userId]
  );

  // Check if each field actually changed (null-safe)
  const changed = Object.entries(updates).filter(([key, newVal]) => {
    const oldVal = current[key];
    // Null-safe comparison: both null = no change; different = change
    if (oldVal === null && newVal === null) return false;
    if (oldVal === null || newVal === null) return true;
    return oldVal !== newVal;
  });

  if (changed.length === 0) return { updated: false };

  const sets = changed.map(([key], i) => `${key} = $${i + 2}`);
  const values = [userId, ...changed.map(([, val]) => val)];

  await pool.query(
    `UPDATE users SET ${sets.join(', ')}, updated_at = NOW() WHERE id = $1`,
    values
  );

  return { updated: true, fields: changed.map(([key]) => key) };
}
```

### Detecting NULL Issues in Query Results

```javascript
// Development utility: warn when query returns more NULLs than expected
function checkForUnexpectedNulls(rows, expectedNonNullColumns) {
  const nullCounts = {};

  for (const row of rows) {
    for (const col of expectedNonNullColumns) {
      if (row[col] === null) {
        nullCounts[col] = (nullCounts[col] || 0) + 1;
      }
    }
  }

  const problems = Object.entries(nullCounts);
  if (problems.length > 0) {
    console.warn('[DATA QUALITY] Unexpected NULLs found:', nullCounts);
  }

  return nullCounts;
}
```

---

## ORM Comparison

### 1. Prisma

**NULL handling:** Prisma maps `null` in JavaScript directly to SQL NULL. Type safety ensures you know which fields are nullable.

```typescript
// Schema with nullable field:
// model User { phone String? }  — the ? means nullable

// Querying for NULL
const usersWithoutPhone = await prisma.user.findMany({
  where: { phone: null }          // generates: WHERE phone IS NULL
});

// Querying for NOT NULL
const usersWithPhone = await prisma.user.findMany({
  where: { phone: { not: null } } // generates: WHERE phone IS NOT NULL
});

// Setting to NULL in update
await prisma.user.update({
  where: { id: userId },
  data: { phone: null }           // generates: SET phone = NULL
});

// COALESCE in select — NOT supported in Prisma type-safe API
// Must use $queryRaw:
const users = await prisma.$queryRaw<{
  id: number;
  displayName: string;
}[]>`
  SELECT id, COALESCE(display_name, email) AS display_name FROM users WHERE id = ${userId}
`;

// IS DISTINCT FROM — raw SQL only
const changed = await prisma.$queryRaw`
  SELECT id FROM config WHERE key = ${key} AND value IS DISTINCT FROM ${newValue}
`;
```

**Where Prisma breaks:**
- `IS DISTINCT FROM` not supported in query API
- `COALESCE` in SELECT requires `$queryRaw`
- `NULLIF` requires `$queryRaw`
- Nullable fields in schema are TypeScript `string | null` — but runtime checks still needed

**Verdict**: Prisma handles null as a query value well (null in filters, null in updates). For null-manipulating SQL functions (COALESCE, NULLIF, IS DISTINCT FROM), drop to `$queryRaw`.

---

### 2. Drizzle ORM

**NULL handling:** Drizzle's `sql` template tag makes NULL operations ergonomic.

```typescript
import { isNull, isNotNull, sql, eq } from 'drizzle-orm';

// IS NULL / IS NOT NULL
const deleted = await db.select().from(users).where(isNull(users.deletedAt));
const active = await db.select().from(users).where(isNotNull(users.deletedAt));

// COALESCE in select
const result = await db.select({
  id: users.id,
  displayName: sql<string>`COALESCE(${users.displayName}, ${users.email})`,
}).from(users);

// NULLIF
const products = await db.select({
  id: products.id,
  safePrice: sql<number | null>`NULLIF(${products.price}, 0)`,
}).from(products);

// IS DISTINCT FROM
const changed = await db.select()
  .from(config)
  .where(sql`${config.value} IS DISTINCT FROM ${newValue}`);

// Setting to null in update
await db.update(users)
  .set({ phone: null })    // generates: SET phone = NULL
  .where(eq(users.id, userId));
```

**Verdict**: Drizzle handles NULL excellently. The `sql` template tag makes COALESCE, NULLIF, and IS DISTINCT FROM natural. Type definitions from the schema correctly reflect nullable columns as `T | null`.

---

### 3. Sequelize

**NULL handling:** Via `Op.is` and `Op.not` with `null`.

```javascript
const { Op } = require('sequelize');

// IS NULL
const withoutPhone = await User.findAll({
  where: { phone: null }                       // IS NULL
});
// OR explicit:
const withoutPhone2 = await User.findAll({
  where: { phone: { [Op.is]: null } }          // IS NULL
});

// IS NOT NULL
const withPhone = await User.findAll({
  where: { phone: { [Op.ne]: null } }          // IS NOT NULL
  // Note: Op.ne with null generates IS NOT NULL correctly
});

// Setting to NULL
await User.update(
  { phone: null },
  { where: { id: userId } }
);

// COALESCE — needs Sequelize.fn or literal
const users = await User.findAll({
  attributes: [
    'id',
    [Sequelize.fn('COALESCE', Sequelize.col('display_name'), Sequelize.col('email')), 'displayName']
  ]
});

// NULLIF — needs literal
const products = await Product.findAll({
  attributes: [
    'id',
    [Sequelize.literal('NULLIF(price, 0)'), 'safePrice']
  ]
});

// IS DISTINCT FROM — literal only
const changed = await Config.findAll({
  where: Sequelize.literal(`value IS DISTINCT FROM '${escapedValue}'`)
  // DANGER: manual escaping required — use queryInterface for safety
});
```

**Where Sequelize breaks:**
- `IS DISTINCT FROM` with parameters requires careful `Sequelize.literal()` + manual escaping (SQL injection risk if not careful)
- `COALESCE` with `Sequelize.fn()` is verbose and type-unsafe
- Complex NULL-handling expressions become hard to read

**Verdict**: Sequelize handles basic NULL filter conditions well. For COALESCE, NULLIF, and IS DISTINCT FROM, the API gets awkward and potentially unsafe. Use raw queries for anything beyond simple null checks.

---

### 4. TypeORM

**NULL handling:** Via `IsNull()`, `Not(IsNull())` helpers and QueryBuilder.

```typescript
import { IsNull, Not } from 'typeorm';

// IS NULL
const deleted = await userRepository.find({
  where: { deletedAt: IsNull() }             // IS NULL
});

// IS NOT NULL
const active = await userRepository.find({
  where: { deletedAt: Not(IsNull()) }        // IS NOT NULL
});

// Setting to NULL
await userRepository.update(userId, { phone: null as any });
// Note: TypeScript may need 'as any' if column is typed as string (not string | null)

// QueryBuilder for COALESCE
const users = await dataSource
  .getRepository(User)
  .createQueryBuilder('u')
  .select("COALESCE(u.displayName, u.email)", "displayName")
  .addSelect('u.id', 'id')
  .getRawMany();

// IS DISTINCT FROM
const changed = await dataSource
  .getRepository(Config)
  .createQueryBuilder('c')
  .where('c.key = :key', { key })
  .andWhere('c.value IS DISTINCT FROM :newValue', { newValue })
  .getMany();
// QueryBuilder properly parameterizes this — safe
```

**Verdict**: TypeORM's `IsNull()` and `Not(IsNull())` are clean. The QueryBuilder handles IS DISTINCT FROM safely via parameterization. COALESCE in SELECT requires QueryBuilder + `getRawMany()`. TypeScript nullability issues (column typed as non-nullable but you want to set NULL) can cause runtime issues.

---

### 5. Knex.js

**NULL handling:** Closest to raw SQL — very natural.

```javascript
// IS NULL
const deleted = await knex('users').whereNull('deleted_at');

// IS NOT NULL
const active = await knex('users').whereNotNull('deleted_at');

// Setting to NULL
await knex('users').where('id', userId).update({ phone: null });

// COALESCE in select
const users = await knex('users')
  .select('id')
  .select(knex.raw("COALESCE(display_name, email) AS display_name"));

// NULLIF
const products = await knex('products')
  .select('id', knex.raw('NULLIF(price, 0) AS safe_price'));

// IS DISTINCT FROM
const changed = await knex('config')
  .where('key', key)
  .whereRaw('value IS DISTINCT FROM ?', [newValue]);
// knex.raw properly parameterizes — safe

// Null in dynamic conditions
const query = knex('users');
if (includeDeleted === false) {
  query.whereNull('deleted_at');
} else if (includeDeleted === 'only') {
  query.whereNotNull('deleted_at');
}
// else: include all (no condition)
```

**Verdict**: Knex is the most expressive for NULL operations. `whereNull`, `whereNotNull`, `whereRaw` with parameterization all work cleanly. The closest to writing raw SQL while still getting a fluent builder.

---

### ORM Comparison Summary

| Feature | Prisma | Drizzle | Sequelize | TypeORM | Knex.js |
|---------|--------|---------|-----------|---------|---------|
| `IS NULL` | ✓ (`null` value) | ✓ (`isNull`) | ✓ (`null` or `Op.is: null`) | ✓ (`IsNull()`) | ✓ (`.whereNull`) |
| `IS NOT NULL` | ✓ (`not: null`) | ✓ (`isNotNull`) | ✓ (`Op.ne: null`) | ✓ (`Not(IsNull())`) | ✓ (`.whereNotNull`) |
| `IS DISTINCT FROM` | ✗ (raw SQL) | ✓ (`sql` tag) | ✗ (literal, unsafe) | ✓ (QueryBuilder) | ✓ (`.whereRaw`) |
| `COALESCE` in SELECT | ✗ (raw SQL) | ✓ (`sql` tag) | ✓ (verbose) | ✓ (QueryBuilder) | ✓ (`.raw`) |
| `NULLIF` | ✗ (raw SQL) | ✓ (`sql` tag) | ✓ (literal) | ✓ (QueryBuilder) | ✓ (`.raw`) |
| Setting column to NULL | ✓ | ✓ | ✓ | Partial (typing) | ✓ |
| Type-safe nullability | ✓ (best) | ✓ | ✗ | Partial | ✗ |

---

## Practice Exercises

### Exercise 1 — Three-Valued Logic

Without running SQL, determine what each of these expressions evaluates to (TRUE, FALSE, or UNKNOWN/NULL):

```sql
1.  NULL = NULL
2.  NULL IS NULL
3.  NULL IS NOT NULL
4.  NULL = 0
5.  NULL != 0
6.  NOT (NULL = 1)
7.  TRUE AND NULL
8.  FALSE AND NULL
9.  TRUE OR NULL
10. FALSE OR NULL
11. NULL IS DISTINCT FROM NULL
12. NULL IS NOT DISTINCT FROM NULL
13. NULL IS DISTINCT FROM 'x'
14. COALESCE(NULL, NULL, 5)
15. NULLIF(5, 5)
```

---

### Exercise 2 — Fix the NULL Bugs

Find and fix all NULL-related bugs in this query:

```sql
SELECT
    e.name AS employee,
    d.name AS department,
    e.salary,
    e.manager_id,
    e.salary / e.bonus_multiplier AS effective_salary,
    e.first_name || ' ' || e.last_name AS full_name
FROM employees e
JOIN departments d ON d.id = e.department_id
WHERE e.department_id != 99
  AND e.status != 'terminated'
ORDER BY e.salary DESC;
```

Assume: `bonus_multiplier` can be 0 or NULL, `last_name` can be NULL, `department_id` can be NULL, `status` can be NULL.

---

### Exercise 3 — Aggregate NULL Behavior

Given this data:
```
scores: [10, NULL, 20, NULL, NULL, 30, 40, NULL]
```

What do each of these return?
```sql
SELECT COUNT(*) FROM t;
SELECT COUNT(score) FROM t;
SELECT SUM(score) FROM t;
SELECT AVG(score) FROM t;
SELECT AVG(COALESCE(score, 0)) FROM t;
SELECT MAX(score) FROM t;
SELECT MIN(score) FROM t;
```

Now write a single query that returns ALL of the above in one result row, plus a column showing what percentage of rows have NULL scores.

---

### Exercise 4 — NULL-Safe Upsert

Write a function `upsertUserProfile(pool, userId, profileData)` in Node.js that:
1. Updates only the fields in `profileData` that are provided AND that differ from the current value (null-safe comparison)
2. Handles the case where a field is being intentionally set to null (clearing a value)
3. Returns `{ updated: boolean, changedFields: string[] }`

The `profileData` object has these optional nullable fields: `displayName`, `bio`, `website`, `location`, `avatarUrl`

---

## Interview Questions

### Junior Level

**Q: What does NULL mean in SQL and how is it different from zero or empty string?**

A: NULL means "unknown" or "value not applicable." It is not zero (zero is a known number), not empty string (empty string is a known empty string), and not false. NULL represents the complete absence of any known value. Zero divided by two is zero — known. NULL divided by two is NULL — we can't compute from the unknown. This is why you can't use `= NULL` to check for it — comparing anything to unknown produces unknown, not true.

---

**Q: Why does `SELECT * FROM users WHERE name != 'Alice'` not return users where name is NULL?**

A: When `name` is NULL, the comparison `NULL != 'Alice'` evaluates to UNKNOWN (not TRUE, not FALSE). SQL's WHERE clause only passes rows that evaluate to TRUE. Rows that evaluate to UNKNOWN (which includes any comparison against NULL) are treated the same as FALSE and excluded. To include users with NULL names, you'd add: `OR name IS NULL`.

---

### Principal Level

**Q: A developer says COUNT(*) and COUNT(id) return the same result because id is a primary key and can't be NULL. Is this always true? When would you choose COUNT(*) over COUNT(id)?**

A: They're functionally equivalent when `id` is a NOT NULL PRIMARY KEY — you're right that in that case the results will be identical.

But I'd still choose `COUNT(*)` in most cases for several reasons:

1. **Semantic clarity**: `COUNT(*)` means "count rows." `COUNT(id)` means "count rows where id is not null." When id can't be null, they're the same, but `COUNT(*)` states intent more clearly.

2. **Performance**: `COUNT(*)` is specifically optimized in PostgreSQL. For sequential scans, the executor can use the visibility map to count visible pages without reading column values. `COUNT(id)` requires actually evaluating the id column for each row to ensure it's not NULL, even if we know it can't be null.

3. **Maintenance safety**: If the schema ever changes and id is no longer NOT NULL (rare but possible through ALTER TABLE), `COUNT(id)` silently starts returning different results. `COUNT(*)` remains correct.

4. **Clarity in aggregates**: In GROUP BY contexts, `COUNT(*)` vs `COUNT(col)` is a meaningful distinction you want to be explicit about. Always using `COUNT(*)` when you mean "count rows" avoids accidental use of `COUNT(col)` with a nullable column that gives unexpectedly different results.

---

**Q: You're debugging a query that joins users to subscriptions with a LEFT JOIN, but the result seems to behave like an INNER JOIN — every user in the result has a non-null subscription. What's the most likely cause?**

A: The most common cause is a **WHERE clause condition on the right-side table that filters out NULL rows**, effectively converting the LEFT JOIN back into an INNER JOIN.

```sql
-- This looks like a LEFT JOIN but behaves like INNER JOIN:
SELECT u.*, s.plan_id
FROM users u
LEFT JOIN subscriptions s ON s.user_id = u.id
WHERE s.plan_id != 'free';    -- ← this is the problem
```

When a user has no subscription, the LEFT JOIN produces a row with `s.plan_id = NULL`. Then `NULL != 'free'` evaluates to UNKNOWN, and the WHERE clause excludes the row — exactly like an INNER JOIN.

The fixes:
```sql
-- Fix A: put the filter in the ON clause (applies before the outer join)
SELECT u.*, s.plan_id
FROM users u
LEFT JOIN subscriptions s ON s.user_id = u.id AND s.plan_id != 'free';
-- Now unsubscribed users appear with NULL plan_id (not filtered out)

-- Fix B: include NULL in the WHERE condition explicitly
SELECT u.*, s.plan_id
FROM users u
LEFT JOIN subscriptions s ON s.user_id = u.id
WHERE s.plan_id != 'free' OR s.plan_id IS NULL;
-- Users with no subscription (plan_id IS NULL) pass the WHERE clause
```

The choice between A and B depends on intent: Fix A returns all users (even those without any matching subscription row), Filter B is more explicit. I teach this as "the LEFT JOIN WHERE trap" — it's one of the most common JOIN bugs in production code.

---

## Mental Model Checkpoint

1. What are the three possible results of any Boolean expression in SQL? Which two cause a row to be excluded from WHERE?

2. `NULL + 5 = ?` and `NULL * 0 = ?` — explain why neither gives an intuitive mathematical result.

3. You have a table with 1,000 rows. 200 have `status = 'active'`, 600 have `status = 'inactive'`, and 200 have `status = NULL`. A query runs `WHERE status != 'active'`. How many rows does it return? Why?

4. `COALESCE(a, b)` vs `CASE WHEN a IS NOT NULL THEN a ELSE b END` — are they equivalent? Is there any scenario where they differ?

5. `AVG(score)` on a column with 5 values: `[10, NULL, 20, NULL, 30]`. What does it return? What would `AVG(COALESCE(score, 0))` return? Which one is "correct"?

6. You JOIN two tables where the JOIN column is nullable. Multiple rows in both tables have NULL in the join column. How many rows match between them? Why?

7. A UNIQUE constraint is on column `email`. You insert 3 rows all with `email = NULL`. Does PostgreSQL allow this? Why or why not?

---

## Quick Reference Card

```
Three-Valued Logic
───────────────────
TRUE AND NULL  = UNKNOWN    FALSE AND NULL = FALSE
TRUE OR NULL   = TRUE       FALSE OR NULL  = UNKNOWN
NOT UNKNOWN    = UNKNOWN

NULL Propagation
────────────────
NULL + anything      = NULL
NULL * anything      = NULL (even * 0)
NULL || 'string'     = NULL (use CONCAT or CONCAT_WS)
NULL = anything      = UNKNOWN (never TRUE)
NULL != anything     = UNKNOWN (never TRUE)
NULL IS NULL         = TRUE  ← the only way to detect NULL
NULL IS NOT NULL     = FALSE

NULL-Safe Operators
───────────────────
IS NULL / IS NOT NULL        → always TRUE or FALSE
IS DISTINCT FROM             → null-safe !=
IS NOT DISTINCT FROM         → null-safe =

NULL Handling Functions
────────────────────────
COALESCE(a, b, c)     → first non-null; lazy evaluation
NULLIF(val, match)    → NULL if val=match, else val
CONCAT(a, b)          → NULL-safe concatenation (ignores NULLs)
CONCAT_WS(sep, a, b)  → NULL-safe with separator

Aggregate Functions and NULL
─────────────────────────────
COUNT(*)    → counts ALL rows (including NULL columns)
COUNT(col)  → counts only non-NULL values
SUM/AVG/MIN/MAX → all IGNORE NULLs silently

NULL in ORDER BY
────────────────
ASC:  NULLs LAST by default  (override: NULLS FIRST)
DESC: NULLs FIRST by default (override: NULLS LAST)

NULL in UNIQUE constraints
───────────────────────────
Multiple NULLs allowed in a UNIQUE column
NULL != NULL in constraint evaluation

NULL in Node.js pg
───────────────────
PostgreSQL NULL → JavaScript null (not undefined)
JavaScript null → PostgreSQL NULL
JavaScript undefined → error/unexpected behavior
Use null explicitly, never undefined
```

---

## Connected Topics

| Topic | Connection |
|-------|-----------|
| [06-where-clause-mastery.md](06-where-clause-mastery.md) | NULL operators in WHERE — IS NULL, IS DISTINCT FROM, NOT IN trap |
| [08-filtering-performance.md](08-filtering-performance.md) | COALESCE in WHERE is non-sargable — the performance consequence of NULL handling |
| [09-case-expressions.md](09-case-expressions.md) | CASE WHEN x IS NULL THEN — CASE as a flexible NULL-handling tool |
| [12-left-right-join.md](12-left-right-join.md) | LEFT JOIN produces NULLs for unmatched rows — the WHERE-kills-LEFT-JOIN trap |
| [21-aggregate-functions.md](21-aggregate-functions.md) | NULL behavior in every aggregate function — the complete reference |
| [27-exists-not-exists.md](27-exists-not-exists.md) | NOT EXISTS is NULL-safe; NOT IN is not — the critical difference |
| [36-lag-lead.md](36-lag-lead.md) | LAG/LEAD default parameter — when no row exists vs when the row has NULL |
| [59-upsert-patterns.md](59-upsert-patterns.md) | IS DISTINCT FROM in upsert logic — only update when value actually changed |
| [63-date-time-mastery.md](63-date-time-mastery.md) | NULL timestamps in date arithmetic — COALESCE patterns for date handling |
