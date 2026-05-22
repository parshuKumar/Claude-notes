# Topic 05 — SELECT and Projection

**Phase 2 — SELECT and Filtering Foundations**
**Interview Priority: MEDIUM** (but the gotchas appear in HIGH-priority questions constantly)

---

## ELI5 Analogy

Imagine you ordered a burger. The kitchen has everything: patty, bun, lettuce, tomato, pickles, onions, cheese, sauce, a full tray.

`SELECT *` means: "Give me the entire tray."
`SELECT name, price` means: "I only need the name and price label."

But here's what makes it a production concern — the kitchen still assembles the *entire tray* first, then hands you only the slips of paper you asked for. The physical work happened regardless. SELECT is the step where you decide what to *show*, not what to *compute*. Most of the work happened earlier (FROM, WHERE, GROUP BY). SELECT is the last step before sorting and pagination.

The dangerous myth: "I wrote `SELECT *` so the query can return whatever columns I need later." Reality: you locked in a contract with the table's current shape. The moment someone adds a column, drops a column, or reorders columns, every piece of code that does positional access breaks silently.

---

## Internals Connection

SELECT lives at **Step 5** of the 9-step logical execution order (covered in Topic 02). By the time SELECT executes:

1. `FROM + JOINs` — the full row source has been assembled (possibly millions of rows)
2. `WHERE` — rows have been filtered
3. `GROUP BY` — rows have been collapsed into groups
4. `HAVING` — groups have been filtered

SELECT then:
- **Projects** — picks which columns to expose
- **Computes expressions** — evaluates `price * 1.1`, `upper(name)`, CASE statements
- **Assigns aliases** — `price * 1.1 AS price_with_tax`
- **Evaluates scalar subqueries** in the SELECT list (each one runs per-output-row)

Only *after* SELECT does `DISTINCT` run (Step 6), then `Window Functions` (Step 7), then `ORDER BY` (Step 8), then `LIMIT/OFFSET` (Step 9).

In the PostgreSQL executor (Volcano model), projection is implemented via a **ProjectSet** or **Result** node that wraps the underlying scan/join plan. It evaluates the target list (the internal representation of your SELECT columns) for each tuple the child node emits.

---

## Execution Order Context

```
FROM + JOINs  →  WHERE  →  GROUP BY  →  HAVING  →  [SELECT]  →  DISTINCT  →  WINDOW  →  ORDER BY  →  LIMIT
                                                          ↑
                                                    YOU ARE HERE
```

**Consequence 1**: Aliases defined in SELECT are NOT available in WHERE or HAVING (they haven't been computed yet at those steps). This is the single most common SELECT-related bug.

```sql
-- WRONG — WHERE runs before SELECT, alias doesn't exist yet
SELECT price * 1.1 AS price_with_tax
FROM products
WHERE price_with_tax > 100;  -- ERROR: column "price_with_tax" does not exist

-- RIGHT — repeat the expression
SELECT price * 1.1 AS price_with_tax
FROM products
WHERE price * 1.1 > 100;
```

**Consequence 2**: SELECT aliases ARE available in ORDER BY (PostgreSQL evaluates ORDER BY after SELECT as an extension to the SQL standard).

```sql
-- WORKS in PostgreSQL (but not in all databases)
SELECT price * 1.1 AS price_with_tax
FROM products
ORDER BY price_with_tax DESC;  -- alias resolved correctly
```

**Consequence 3**: In GROUP BY, PostgreSQL allows ordinal references (column position numbers) but NOT aliases in most contexts. Some PostgreSQL versions allow aliases in GROUP BY as an extension. Don't rely on it.

---

## What Is SELECT and Projection?

**Projection** is the relational algebra operation of selecting a *subset of columns* from a relation (table or result set). In relational theory, projection reduces the "width" of a relation — from N columns down to M columns where M ≤ N.

**SELECT** in SQL is the keyword that triggers projection. It also:
1. Performs **column selection** — choosing which columns to include
2. Performs **expression evaluation** — computing new values from existing columns
3. Performs **aliasing** — assigning output names to columns or expressions
4. Can include **scalar subqueries** — queries that return exactly one row and one column
5. Can include **aggregate functions** (when no GROUP BY is present, treats entire result as one group)
6. Is the source of **window function** evaluation (windowed expressions appear in SELECT)

The `*` syntax is syntactic sugar for "all columns from all tables in scope." PostgreSQL expands it at **analysis time** (the Analyzer stage) before the query even reaches the planner. This expansion is permanent once the query is compiled — the expanded column list is baked into the parse tree.

---

## Why It Matters in Production

### 1. SELECT * Breaks Index-Only Scans

An **Index-Only Scan** answers the query entirely from the index without touching the heap (table). This is the fastest possible scan type. It requires every column in the SELECT list to be stored in the index.

```sql
CREATE INDEX idx_products_name_price ON products(name, price);

-- Can do Index-Only Scan — both columns in index
SELECT name, price FROM products WHERE name = 'Widget';

-- CANNOT do Index-Only Scan — 'description' is not in index
SELECT * FROM products WHERE name = 'Widget';
-- Falls back to Index Scan + Heap Fetch → 10-100x slower on large tables
```

If you wrote `SELECT *` and this table has 20 columns, you forced a heap fetch for every single row. You gave up your fastest possible access pattern.

### 2. SELECT * Causes Network Overhead

Every column travels over the network. A table with `TEXT`, `JSONB`, or `BYTEA` columns can return megabytes per row. If your application needs only `id` and `status`, but you wrote `SELECT *`, you're serializing and deserializing potentially 100x more data than needed.

This compounds: PostgreSQL → pg wire protocol serialization → network transfer → Node.js deserialization → object creation → garbage collection. All for data your application never uses.

### 3. SELECT * Creates Invisible Contract with Schema

```sql
-- Your Node.js code today:
const [user] = await pool.query('SELECT * FROM users WHERE id = $1', [id]);
const name = user.rows[0].name;  // works

-- Someone adds a column 'name_hash BYTEA' next week
// SELECT * now returns name_hash — a Buffer in Node.js
// Code still "works" but GC pressure increases, response payload grows
// Logging middleware now dumps binary data to logs — compliance risk

-- Someone renames 'name' to 'full_name'
// SELECT * still works — returns full_name
// But user.rows[0].name is now undefined — silent bug
```

### 4. SELECT * in ORMs is the Default and the Trap

Every ORM defaults to `SELECT *` (or equivalent) when you do `findMany()`, `find()`, etc. Understanding this is what separates a principal engineer from a junior: knowing that `prisma.user.findMany()` is literally `SELECT * FROM users` and having a policy about when that's acceptable.

### 5. DISTINCT Has a Real Cost

`SELECT DISTINCT` triggers a deduplication step that either sorts the result set or builds a hash table. On large result sets, this is a significant CPU and memory cost. Many developers add DISTINCT defensively ("just to be safe") when the correct fix is to stop generating duplicates (usually via a JOIN fix — covered in Topic 10).

---

## Deep Technical Content

### Column Selection — Fully Qualified Names

```sql
-- When joining multiple tables, qualify ambiguous columns
SELECT
    u.id,
    u.name,
    o.id AS order_id,       -- must alias: 'id' is ambiguous
    o.created_at
FROM users u
JOIN orders o ON o.user_id = u.id;
```

PostgreSQL will error on ambiguous column references: `ERROR: column reference "id" is ambiguous`. This is a safety feature. Always qualify columns in multi-table queries.

### The `*` Expansion

`SELECT *` expands at analysis time to the full column list of all tables in the FROM clause. If two tables have the same column name, both are included — you get duplicate column names in the output.

```sql
SELECT * FROM users JOIN orders ON orders.user_id = users.id;
-- Returns: users.id, users.name, ..., orders.id, orders.user_id, ...
-- Two 'id' columns in the result! Both named 'id'.
-- In pg (Node.js), the second 'id' overwrites the first in the result object
```

This is a silent data corruption bug. You get `row.id` but can't control which table's `id` it comes from — it depends on column order, which depends on table order in FROM, which can change. **Never use `SELECT *` in a multi-table query.**

`t.*` syntax selects all columns from a specific table:

```sql
SELECT u.*, o.id AS order_id, o.total
FROM users u
JOIN orders o ON o.user_id = u.id;
-- Safer than *, but still brings all user columns
```

### Expressions in SELECT

SELECT can evaluate any expression for each output row:

```sql
SELECT
    -- Arithmetic
    price * quantity AS line_total,
    price * quantity * 0.1 AS tax,
    price * quantity * 1.1 AS total_with_tax,

    -- String operations
    upper(first_name) || ' ' || upper(last_name) AS full_name_upper,
    left(description, 100) AS description_preview,
    char_length(description) AS description_length,

    -- Conditional
    CASE
        WHEN stock_count = 0 THEN 'out_of_stock'
        WHEN stock_count < 10 THEN 'low_stock'
        ELSE 'in_stock'
    END AS stock_status,

    -- Type casting
    created_at::date AS created_date,
    created_at::text AS created_at_string,
    price::numeric(10,2) AS price_rounded,

    -- NULL handling
    COALESCE(middle_name, '') AS middle_name_safe,
    NULLIF(phone, '') AS phone_or_null,

    -- Date operations
    CURRENT_DATE - created_at::date AS days_since_created,
    date_trunc('month', created_at) AS created_month,

    -- Boolean expressions
    (stock_count > 0) AS is_available,
    (price < 10.00) AS is_cheap

FROM products;
```

Every expression is evaluated **once per output row**. Expressions in SELECT do not affect which rows are returned (that's WHERE's job) — they only affect what value is shown for each row.

### Scalar Subqueries in SELECT

A scalar subquery in the SELECT list returns exactly one column and one row. It runs **once per output row of the outer query** — this is the killer detail.

```sql
SELECT
    u.id,
    u.name,
    (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS order_count,
    (SELECT MAX(o.total) FROM orders o WHERE o.user_id = u.id) AS largest_order
FROM users u;
```

If `users` has 10,000 rows, this runs the subquery **20,000 times** (once per user per subquery). This is the scalar subquery version of the N+1 problem. The correct fix is almost always a LEFT JOIN + GROUP BY or a window function.

PostgreSQL sometimes optimizes scalar subqueries into a single scan with a HashAggregate, but you cannot rely on this. EXPLAIN ANALYZE to verify.

```sql
-- Better: one scan of orders, joined back
SELECT
    u.id,
    u.name,
    COALESCE(s.order_count, 0) AS order_count,
    s.largest_order
FROM users u
LEFT JOIN (
    SELECT user_id, COUNT(*) AS order_count, MAX(total) AS largest_order
    FROM orders
    GROUP BY user_id
) s ON s.user_id = u.id;
```

### Aliases — Complete Rules

```sql
SELECT
    price * 1.1 AS price_with_tax,   -- standard alias syntax
    price * 1.1 price_with_tax,      -- AS is optional (works, but don't do it — ambiguous to readers)
    price * 1.1 AS "Price With Tax"  -- quoted identifier — preserves case, allows spaces
FROM products;
```

**When aliases are available:**

| Clause | Can reference SELECT alias? | Notes |
|--------|----------------------------|-------|
| WHERE | NO | Runs before SELECT |
| GROUP BY | Partially | PostgreSQL extension allows it in some cases, but unreliable across SQL databases |
| HAVING | NO | Runs before SELECT |
| ORDER BY | YES | PostgreSQL explicitly evaluates ORDER BY after SELECT |
| WINDOW definitions | YES | Window clauses reference aliases defined in SELECT |
| LIMIT/OFFSET | N/A | These are not expressions |

**Alias naming rules:**
- Unquoted aliases: lowercase, no spaces, no special chars except `_`
- Quoted aliases: `"My Alias"` — case-sensitive, can contain anything
- Avoid quoting unless necessary — it creates a case-sensitive identifier that breaks if you write it in wrong case later

### DISTINCT — Full Mechanics

`SELECT DISTINCT` removes duplicate rows from the result. "Duplicate" means all selected columns have identical values.

```sql
SELECT DISTINCT status FROM orders;
-- Returns each unique status value once

SELECT DISTINCT user_id, status FROM orders;
-- Returns each unique (user_id, status) PAIR once
-- If a user has 5 'pending' orders, (user_id=42, status='pending') appears ONCE
```

**How PostgreSQL implements DISTINCT:**

Option 1 — **Sort-based deduplication**: Sort the result, then compare adjacent rows. O(N log N). Chosen when result is large or output must be sorted anyway.

Option 2 — **Hash-based deduplication**: Build a hash table of seen values. O(N) average. Chosen when result fits in `work_mem`.

```
-- EXPLAIN output for DISTINCT
HashAggregate  (cost=120.00..145.00 rows=50 width=12)
  Group Key: status
  ->  Seq Scan on orders  (cost=0.00..95.00 rows=5000 width=12)

-- Or sort-based:
Unique  (cost=420.00..445.00 rows=50 width=12)
  ->  Sort  (cost=420.00..432.50 rows=5000 width=12)
        Sort Key: status
        ->  Seq Scan on orders  (cost=0.00..95.00 rows=5000 width=12)
```

**The DISTINCT trap:** Most developers reach for DISTINCT when they see duplicates. The correct question is: **why are duplicates appearing?** Usually it's a JOIN that's creating fan-out (one-to-many JOIN without appropriate filtering). Adding DISTINCT fixes the symptom but pays a performance penalty every query. Fixing the JOIN removes duplicates at the source for free.

```sql
-- Symptom: duplicates appear
SELECT DISTINCT u.id, u.name
FROM users u
JOIN orders o ON o.user_id = u.id;
-- A user with 10 orders appears 10 times → DISTINCT removes 9 copies

-- Better: don't create duplicates in the first place
SELECT u.id, u.name
FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
-- Uses a semi-join — returns each user once, no deduplication cost
```

**DISTINCT ON** (PostgreSQL extension) — returns the first row for each distinct value of specified expressions:

```sql
-- For each user, return their most recent order
SELECT DISTINCT ON (user_id) user_id, id AS order_id, total, created_at
FROM orders
ORDER BY user_id, created_at DESC;
-- DISTINCT ON (user_id) says: keep the first row per user_id
-- ORDER BY user_id, created_at DESC ensures "first" means "most recent"
```

**DISTINCT ON is an extremely powerful PostgreSQL feature.** It's the cleanest way to get "latest row per group" without a window function subquery. But it requires:
1. The DISTINCT ON columns must appear first in ORDER BY
2. The remaining ORDER BY columns determine which row is "first"

```sql
-- ERROR: DISTINCT ON expressions must match initial ORDER BY expressions
SELECT DISTINCT ON (user_id) user_id, total
FROM orders
ORDER BY total DESC;  -- user_id must be first in ORDER BY
```

---

## EXPLAIN Output for Projection

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name, price * 1.1 AS price_with_tax
FROM products
WHERE category = 'electronics';
```

```
Seq Scan on products  (cost=0.00..245.00 rows=1200 width=52) (actual time=0.032..4.821 rows=1180 loops=1)
  Filter: ((category)::text = 'electronics'::text)
  Rows Removed by Filter: 8820
  Buffers: shared hit=95
Planning Time: 0.189 ms
Execution Time: 5.102 ms
```

Notice: there is **no separate projection node** in simple cases. PostgreSQL handles projection inline within the scan node by only materializing the requested columns from the heap tuple. The `width=52` tells you the average byte width of the projected output row — this is how wide your data is going over the wire.

When SELECT contains complex expressions, you may see a **Result** node:

```sql
EXPLAIN SELECT *, (SELECT COUNT(*) FROM orders WHERE user_id = u.id) AS order_count
FROM users u;
```

```
Seq Scan on users u  (cost=0.00..2543210.00 rows=1000 width=...)
  SubPlan 1
    ->  Aggregate  (cost=2543.21..2543.22 rows=1 width=8)
          ->  Seq Scan on orders  (cost=...)
                Filter: (user_id = u.id)
```

The `SubPlan 1` confirms: this runs once per outer row. If `users` has 1,000 rows, the orders table is scanned 1,000 times. **This is catastrophic.** EXPLAIN ANALYZE's actual timing will show it immediately.

---

## Query Examples

### Example 1 — Basic: Clean Projection

```sql
-- Scenario: Return product catalog for a listing page
-- Rule: Only select what the UI actually renders

SELECT
    id,
    name,
    slug,
    price,
    CASE
        WHEN stock_count = 0 THEN 'out_of_stock'
        WHEN stock_count < 5 THEN 'low_stock'
        ELSE 'available'
    END AS availability,
    created_at::date AS listed_date
FROM products
WHERE is_active = TRUE
  AND category_id = $1
ORDER BY name
LIMIT 50 OFFSET $2;

-- What this demonstrates:
-- ✓ Only 6 of (hypothetically) 15 columns selected
-- ✓ Computed column (availability) from expression
-- ✓ Type cast (created_at→date strips time component)
-- ✓ Enables potential Index-Only Scan on (is_active, category_id, name, id, slug, price, stock_count)
```

### Example 2 — Intermediate: Expressions and Aliases

```sql
-- Scenario: Invoice calculation with multiple computed fields

SELECT
    li.id AS line_item_id,
    p.name AS product_name,
    li.quantity,
    li.unit_price,
    li.unit_price * li.quantity AS subtotal,
    li.unit_price * li.quantity * (t.rate / 100.0) AS tax_amount,
    li.unit_price * li.quantity * (1 + t.rate / 100.0) AS line_total,

    -- Discount logic
    CASE
        WHEN li.quantity >= 100 THEN 'bulk'
        WHEN li.quantity >= 10  THEN 'volume'
        ELSE 'none'
    END AS discount_tier,

    CASE
        WHEN li.quantity >= 100 THEN li.unit_price * li.quantity * 0.85
        WHEN li.quantity >= 10  THEN li.unit_price * li.quantity * 0.95
        ELSE li.unit_price * li.quantity
    END AS discounted_subtotal

FROM line_items li
JOIN products p     ON p.id = li.product_id
JOIN tax_rates t    ON t.id = li.tax_rate_id
WHERE li.invoice_id = $1
ORDER BY p.name;

-- Key point: aliases like 'subtotal' cannot be used in other SELECT expressions
-- Must repeat the full expression — this is how SQL works
-- If you need to avoid repetition, use a CTE:

WITH base AS (
    SELECT
        li.id AS line_item_id,
        p.name AS product_name,
        li.quantity,
        li.unit_price,
        li.unit_price * li.quantity AS subtotal,
        t.rate
    FROM line_items li
    JOIN products p  ON p.id = li.product_id
    JOIN tax_rates t ON t.id = li.tax_rate_id
    WHERE li.invoice_id = $1
)
SELECT
    line_item_id,
    product_name,
    quantity,
    unit_price,
    subtotal,
    subtotal * (rate / 100.0) AS tax_amount,   -- NOW can use subtotal alias
    subtotal * (1 + rate / 100.0) AS line_total
FROM base
ORDER BY product_name;
```

### Example 3 — Production Grade: Avoiding SELECT * in API Response Query

```sql
-- Scenario: User profile API endpoint
-- BAD: What most code looks like
-- SELECT * FROM users WHERE id = $1
-- This returns: id, email, password_hash, salt, mfa_secret, reset_token,
--               reset_token_expires, failed_login_count, last_login_at,
--               created_at, updated_at, deleted_at, metadata JSONB (possibly large)
-- → Password hashes go over the wire
-- → JSONB metadata may be 50KB
-- → Heap fetch required (no index-only scan possible)
-- → Accidental data exposure risk

-- GOOD: Explicit projection for the profile endpoint
SELECT
    u.id,
    u.email,
    u.display_name,
    u.avatar_url,
    u.created_at,

    -- Computed fields the client needs
    CASE
        WHEN u.email_verified_at IS NOT NULL THEN TRUE
        ELSE FALSE
    END AS is_email_verified,

    EXTRACT(days FROM NOW() - u.created_at)::int AS member_days,

    -- Aggregate from joined table — one query instead of two
    COALESCE(s.subscription_tier, 'free') AS subscription_tier,
    s.expires_at AS subscription_expires_at,

    -- Recent activity count
    COALESCE(act.recent_count, 0) AS recent_activity_count

FROM users u
LEFT JOIN subscriptions s
    ON s.user_id = u.id
    AND s.is_active = TRUE
LEFT JOIN LATERAL (
    SELECT COUNT(*) AS recent_count
    FROM user_activity
    WHERE user_id = u.id
      AND created_at > NOW() - INTERVAL '30 days'
) act ON TRUE
WHERE u.id = $1
  AND u.deleted_at IS NULL;

-- What this achieves:
-- ✓ No sensitive columns (password_hash, mfa_secret, tokens never selected)
-- ✓ Computed values done in DB (no round trips or app-layer computation)
-- ✓ Related data in single query (no N+1)
-- ✓ LATERAL subquery for activity count (exactly one scan of user_activity)
-- ✓ Explicit column list documents the API contract
```

---

## Wrong → Right Patterns

### Pattern 1: Alias in WHERE

```sql
-- WRONG
SELECT total * 1.1 AS total_with_tax FROM orders WHERE total_with_tax > 100;
-- ERROR: column "total_with_tax" does not exist

-- RIGHT — option A: repeat the expression
SELECT total * 1.1 AS total_with_tax FROM orders WHERE total * 1.1 > 100;

-- RIGHT — option B: use a subquery/CTE if repetition is too verbose
SELECT * FROM (
    SELECT total * 1.1 AS total_with_tax FROM orders
) t WHERE t.total_with_tax > 100;
-- Note: 'SELECT *' on a derived table is acceptable — column list is explicit from inner query
```

### Pattern 2: SELECT * in Production Query

```sql
-- WRONG — in any query that runs in production
SELECT * FROM products WHERE id = $1;

-- RIGHT — explicit columns for what the caller actually needs
SELECT id, name, price, stock_count, category_id FROM products WHERE id = $1;

-- Why this matters beyond performance:
-- 1. If someone adds a 'large_blob BYTEA' column → your query suddenly returns megabytes
-- 2. Code review can immediately see what data is being accessed
-- 3. The column list IS your documentation
-- 4. Enables index-only scans
```

### Pattern 3: DISTINCT to Hide a JOIN Problem

```sql
-- WRONG — DISTINCT masking a one-to-many JOIN fan-out
SELECT DISTINCT u.id, u.name
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE o.status = 'completed';
-- Returns each user once, but reads and deduplicates ALL completed orders first

-- RIGHT — use EXISTS to get a semi-join (no fan-out, no deduplication)
SELECT u.id, u.name
FROM users u
WHERE EXISTS (
    SELECT 1 FROM orders o
    WHERE o.user_id = u.id
      AND o.status = 'completed'
);
-- EXISTS stops at first match — never reads all orders for a user
-- Index on (user_id, status) makes this extremely fast
```

### Pattern 4: Scalar Subquery in SELECT (N+1 in SQL)

```sql
-- WRONG — scalar subquery per row
SELECT
    u.id,
    u.name,
    (SELECT COUNT(*) FROM orders WHERE user_id = u.id) AS orders
FROM users u;
-- Runs orders scan once per user — O(users × orders_per_user) total I/O

-- RIGHT — aggregated join
SELECT
    u.id,
    u.name,
    COALESCE(o.order_count, 0) AS orders
FROM users u
LEFT JOIN (
    SELECT user_id, COUNT(*) AS order_count
    FROM orders
    GROUP BY user_id
) o ON o.user_id = u.id;
-- Single scan of orders, one hash aggregation, one hash join
-- O(users + orders) total — dramatically better at scale
```

### Pattern 5: Alias Collision Across Tables

```sql
-- WRONG — SELECT * from multiple tables
SELECT *
FROM users u
JOIN addresses a ON a.user_id = u.id;
-- Result has two 'id' columns, two 'created_at' columns
-- In Node.js pg: second 'id' silently overwrites first in result object

-- RIGHT — explicit projection with aliases for ambiguous columns
SELECT
    u.id       AS user_id,
    u.name     AS user_name,
    u.email,
    a.id       AS address_id,
    a.street,
    a.city,
    a.country
FROM users u
JOIN addresses a ON a.user_id = u.id;
```

---

## Performance Profile

| Pattern | Cost | Why | When It Matters |
|---------|------|-----|-----------------|
| `SELECT col1, col2` | Lowest | Minimal heap I/O, enables index-only scan | Large tables, high-frequency queries |
| `SELECT t.*` | Medium | All columns from one table, still selective | Acceptable if table is narrow |
| `SELECT *` (single table) | Medium-High | Forces heap fetch, prevents index-only scan | Acceptable for narrow tables, dev only in prod |
| `SELECT *` (multi-table join) | High | Multiple heap fetches, likely duplicate columns | Avoid in production entirely |
| `SELECT DISTINCT col` (small result) | Low | Hash dedup fits in work_mem | Fine |
| `SELECT DISTINCT col` (large result) | High | Sort or hash on full result set | Profile before using |
| `SELECT DISTINCT ON (...)` | Medium | Efficient "first per group" | Use instead of window function subquery for this pattern |
| Scalar subquery in SELECT | Very High | Runs once per output row | Almost always replace with aggregated JOIN |

**width matters in EXPLAIN**: The `width=N` in plan nodes tells you bytes per row. High width = more data over wire + more memory in work_mem for sorts.

```
Seq Scan on products (cost=0.00..245.00 rows=10000 width=8)   -- just id + int = 8 bytes per row
Seq Scan on products (cost=0.00..245.00 rows=10000 width=542) -- with JSONB column = 542 bytes per row
```

---

## Node.js Integration

### Checking What You Actually Return

```javascript
// pool.query() in pg returns a Result object
// result.fields tells you exactly what columns were returned
const result = await pool.query(
  'SELECT id, name, price FROM products WHERE id = $1',
  [productId]
);

console.log(result.fields);
// [
//   { name: 'id',    dataTypeID: 23  },  // 23 = int4
//   { name: 'name',  dataTypeID: 25  },  // 25 = text
//   { name: 'price', dataTypeID: 1700 }, // 1700 = numeric
// ]

// result.rows is an array of plain objects
const product = result.rows[0];
// { id: 1, name: 'Widget', price: '9.99' }
// Note: numeric → string in pg by default (BigNumber safety)
```

### The Duplicate Column Name Trap in pg

```javascript
// What happens with SELECT * from a join
const result = await pool.query(`
  SELECT * FROM users u JOIN addresses a ON a.user_id = u.id WHERE u.id = $1
`, [userId]);

// Both tables have 'id' and 'created_at'
// pg returns the LAST column with that name
console.log(result.rows[0].id); // address.id, not user.id — silent data corruption
```

```javascript
// Safe multi-table query helper
async function getProductWithDetails(pool, productId) {
  const { rows } = await pool.query(`
    SELECT
      p.id          AS product_id,
      p.name        AS product_name,
      p.price,
      p.stock_count,
      c.id          AS category_id,
      c.name        AS category_name,
      p.created_at  AS product_created_at
    FROM products p
    JOIN categories c ON c.id = p.category_id
    WHERE p.id = $1
  `, [productId]);

  if (rows.length === 0) return null;
  const row = rows[0];
  
  // Map flat row to nested object for API response
  return {
    id:         row.product_id,
    name:       row.product_name,
    price:      parseFloat(row.price),
    stockCount: row.stock_count,
    category: {
      id:   row.category_id,
      name: row.category_name,
    },
    createdAt: row.product_created_at,
  };
}
```

### Dynamic Column Selection (Safe Pattern)

```javascript
// Sometimes you need dynamic projection — a list of columns from user input
// NEVER interpolate column names directly — SQL injection risk
// Use an allowlist

const ALLOWED_COLUMNS = new Set([
  'id', 'name', 'price', 'stock_count', 'category_id', 'created_at'
]);

function buildProductQuery(requestedColumns) {
  // Validate every requested column against allowlist
  const columns = requestedColumns.filter(col => ALLOWED_COLUMNS.has(col));
  
  if (columns.length === 0) {
    throw new Error('No valid columns requested');
  }
  
  // Safe: all columns come from our allowlist, not user input directly
  return `SELECT ${columns.join(', ')} FROM products`;
}

// Usage
const query = buildProductQuery(['id', 'name', 'price', 'evil; DROP TABLE--']);
// 'evil; DROP TABLE--' is filtered out by allowlist check
// Result: 'SELECT id, name, price FROM products'
```

### Detecting SELECT * in Production Queries

```javascript
// Add to your query logging middleware to catch SELECT * in production
function warnOnSelectStar(sql, queryName = 'unknown') {
  // Simple heuristic — catches most cases
  if (/SELECT\s+\*/i.test(sql)) {
    console.warn(
      `[QUERY WARNING] SELECT * used in query "${queryName}". ` +
      `Consider explicit column selection for production queries.`
    );
  }
}

// Or use pg_stat_statements to audit — queries with SELECT * will show
// high total_exec_time relative to calls, and high mean rows
```

---

## ORM Comparison

### 1. Prisma

**Can it do explicit projection?** YES — via `select` option.

```typescript
// Prisma default: SELECT * equivalent
const user = await prisma.user.findUnique({ where: { id: userId } });
// Generates: SELECT id, email, name, password_hash, ... all columns

// Prisma with explicit projection
const user = await prisma.user.findUnique({
  where: { id: userId },
  select: {
    id: true,
    email: true,
    displayName: true,
    avatarUrl: true,
    createdAt: true,
  }
});
// Generates: SELECT id, email, display_name, avatar_url, created_at FROM users WHERE id = $1

// Prisma with computed fields — NOT supported in select
// Prisma cannot add SQL expressions like `price * 1.1 AS price_with_tax`
// Must compute in application layer or use $queryRaw
const product = await prisma.product.findUnique({
  where: { id },
  select: { price: true }
});
const priceWithTax = Number(product.price) * 1.1; // app layer
```

**Where it breaks:**
- Cannot compute SQL expressions in the SELECT list
- Cannot use SQL functions (UPPER, DATE_TRUNC, etc.) in projections
- DISTINCT ON is not supported
- Scalar subqueries in SELECT list not possible

**Raw SQL escape hatch:**
```typescript
const result = await prisma.$queryRaw<{
  id: number;
  name: string;
  price_with_tax: number;
}[]>`
  SELECT id, name, price * 1.1 AS price_with_tax
  FROM products
  WHERE id = ${productId}
`;
```

**Verdict**: Use Prisma `select` for column projection — it works well and generates correct SQL. Switch to `$queryRaw` the moment you need SQL expressions, DISTINCT ON, or computed columns.

---

### 2. Drizzle ORM

**Can it do explicit projection?** YES — via `.select({ ... })` syntax.

```typescript
import { db } from './db';
import { users, orders } from './schema';
import { sql, eq } from 'drizzle-orm';

// Default: selects all columns
const allUsers = await db.select().from(users);
// Generates: SELECT * FROM users (or explicit column list depending on schema definition)

// Explicit column projection
const userProfiles = await db.select({
  id:          users.id,
  email:       users.email,
  displayName: users.displayName,
}).from(users).where(eq(users.id, userId));

// Drizzle supports SQL expressions in select!
const withTax = await db.select({
  id:           products.id,
  name:         products.name,
  priceWithTax: sql<number>`${products.price} * 1.1`,
}).from(products);
// Generates: SELECT id, name, price * 1.1 FROM products

// DISTINCT
const statuses = await db.selectDistinct({ status: orders.status }).from(orders);
// Generates: SELECT DISTINCT status FROM orders
```

**Where it breaks:**
- `DISTINCT ON` requires raw SQL
- Complex expressions may need `sql` template tag
- Scalar subqueries in SELECT list require `sql` template

**Raw SQL escape hatch:**
```typescript
const result = await db.execute(
  sql`SELECT id, name, price * 1.1 AS price_with_tax FROM products WHERE id = ${productId}`
);
```

**Verdict**: Drizzle is the best ORM for explicit projection. Its `sql` template tag lets you add expressions inline without losing type safety context. The syntax closely mirrors SQL, so what you write is close to what you get.

---

### 3. Sequelize

**Can it do explicit projection?** YES — via `attributes` option.

```javascript
// Default: SELECT * equivalent
const user = await User.findOne({ where: { id: userId } });
// Generates: SELECT id, email, name, ... all mapped columns FROM users WHERE id = ?

// Explicit column projection
const user = await User.findOne({
  attributes: ['id', 'email', 'displayName'],
  where: { id: userId }
});
// Generates: SELECT id, email, display_name FROM users WHERE id = ?

// With expressions
const products = await Product.findAll({
  attributes: [
    'id',
    'name',
    [Sequelize.literal('price * 1.1'), 'priceWithTax'],
    [Sequelize.fn('UPPER', Sequelize.col('name')), 'upperName'],
  ]
});
// Generates: SELECT id, name, price * 1.1 AS priceWithTax, UPPER(name) AS upperName FROM products

// Exclude specific columns
const userWithoutPassword = await User.findOne({
  attributes: { exclude: ['passwordHash', 'mfaSecret'] },
  where: { id: userId }
});
```

**Where it breaks:**
- `Sequelize.literal()` is untyped — no compile-time safety
- DISTINCT ON not supported
- Complex expressions become verbose with `Sequelize.literal()`
- `attributes: { exclude: [...] }` still generates `SELECT *` equivalent — adds columns dynamically based on model definition

**Raw SQL escape hatch:**
```javascript
const [results] = await sequelize.query(
  'SELECT id, name, price * 1.1 AS price_with_tax FROM products WHERE id = ?',
  { replacements: [productId], type: QueryTypes.SELECT }
);
```

**Verdict**: Sequelize's `attributes` works for simple projections. The `{ exclude: [...] }` feature seems convenient but is a footgun — it fetches everything first in the query builder, and the exclusion list must stay in sync with schema. Use explicit `attributes` arrays for production queries.

---

### 4. TypeORM

**Can it do explicit projection?** YES — via `select` in QueryBuilder or Repository options.

```typescript
// Repository approach — returns full entity (SELECT * equivalent)
const user = await userRepository.findOne({ where: { id: userId } });

// Repository with select
const user = await userRepository.findOne({
  select: { id: true, email: true, displayName: true },
  where: { id: userId }
});
// Generates: SELECT user.id, user.email, user.display_name FROM users user WHERE user.id = ?

// QueryBuilder approach — more flexible
const users = await dataSource
  .getRepository(User)
  .createQueryBuilder('u')
  .select(['u.id', 'u.email', 'u.displayName'])
  .where('u.id = :id', { id: userId })
  .getMany();

// With expressions (must use addSelect with alias)
const products = await dataSource
  .getRepository(Product)
  .createQueryBuilder('p')
  .select('p.id', 'id')
  .addSelect('p.name', 'name')
  .addSelect('p.price * 1.1', 'priceWithTax')
  .getRawMany(); // getRawMany for non-entity shape results
```

**Where it breaks:**
- `select` in Repository findOne/findMany requires exact entity field names — no SQL expressions
- Expressions only possible via QueryBuilder
- DISTINCT ON requires raw query
- `getRawMany()` returns untyped objects — loses entity type safety

**Raw SQL escape hatch:**
```typescript
const result = await dataSource.query(
  'SELECT id, name, price * 1.1 AS price_with_tax FROM products WHERE id = $1',
  [productId]
);
```

**Verdict**: TypeORM's QueryBuilder can handle most projection needs but has a confusing split between `getMany()` (entity type) and `getRawMany()` (raw object) — expressions force you to `getRawMany()` and lose typing. For complex projections, prefer raw SQL in TypeORM.

---

### 5. Knex.js

**Can it do explicit projection?** YES — Knex is the closest to raw SQL among the ORMs.

```javascript
// Default — selects all (SELECT *)
const users = await knex('users');

// Explicit column selection
const users = await knex('users')
  .select('id', 'email', 'display_name as displayName')
  .where('id', userId);
// Generates: SELECT id, email, display_name as displayName FROM users WHERE id = ?

// With expressions
const products = await knex('products')
  .select(
    'id',
    'name',
    knex.raw('price * 1.1 AS price_with_tax'),
    knex.raw('UPPER(name) AS upper_name')
  )
  .where('category_id', categoryId);
// Generates: SELECT id, name, price * 1.1 AS price_with_tax, UPPER(name) AS upper_name
//            FROM products WHERE category_id = ?

// DISTINCT
const statuses = await knex('orders').distinct('status');
// Generates: SELECT DISTINCT status FROM orders

// DISTINCT ON (PostgreSQL specific)
const latestOrders = await knex('orders')
  .distinctOn('user_id')
  .select('user_id', 'id', 'total', 'created_at')
  .orderBy([{ column: 'user_id' }, { column: 'created_at', order: 'desc' }]);
// Generates: SELECT DISTINCT ON (user_id) user_id, id, total, created_at
//            FROM orders ORDER BY user_id, created_at DESC
```

**Where it breaks:**
- No type safety — column names are strings
- `knex.raw()` bypasses Knex's parameter binding — must be careful with SQL injection
- Column aliases in `.select()` must match exactly what the DB returns

**Raw SQL escape hatch:**
```javascript
const result = await knex.raw(
  'SELECT id, name, price * 1.1 AS price_with_tax FROM products WHERE id = ?',
  [productId]
);
const rows = result.rows;
```

**Verdict**: Knex is the best non-TypeScript-safe option for complex projections. Its `knex.raw()` inline expressions are more ergonomic than Sequelize's `literal()`. The `distinctOn()` support is a notable advantage over other ORMs. Use `knex.raw()` sparingly — it skips parameter binding if used for values.

---

### ORM Comparison Summary

| Feature | Prisma | Drizzle | Sequelize | TypeORM | Knex.js |
|---------|--------|---------|-----------|---------|---------|
| Explicit column select | ✓ | ✓ | ✓ | ✓ | ✓ |
| SQL expressions in SELECT | ✗ (raw SQL) | ✓ (`sql` tag) | ✓ (`literal`) | Partial (QueryBuilder) | ✓ (`raw`) |
| DISTINCT | ✗ | ✓ | ✓ | ✓ | ✓ |
| DISTINCT ON | ✗ | ✗ | ✗ | ✗ | ✓ |
| Type safety for projections | ✓ (best) | ✓ | ✗ | Partial | ✗ |
| Default behavior | SELECT * | SELECT * | SELECT * | SELECT * | SELECT * |
| Best for complex projections | `$queryRaw` | `sql` tag | `literal` + raw | QueryBuilder + raw | `.select()` + `.raw()` |

**Universal truth across all ORMs**: The default is always SELECT *. You must opt into explicit column selection in every ORM. Make it a code review requirement.

---

## Practice Exercises

### Exercise 1 — Projection Design

Given this table:
```sql
CREATE TABLE employees (
    id              SERIAL PRIMARY KEY,
    first_name      TEXT NOT NULL,
    last_name       TEXT NOT NULL,
    email           TEXT NOT NULL,
    password_hash   TEXT NOT NULL,
    department_id   INT,
    salary          NUMERIC(10,2),
    hire_date       DATE,
    is_active       BOOLEAN DEFAULT TRUE,
    profile_notes   TEXT,          -- potentially large
    created_at      TIMESTAMPTZ DEFAULT NOW(),
    updated_at      TIMESTAMPTZ DEFAULT NOW()
);
```

Write a query for a "team roster" API endpoint that returns each employee's full name (as a single column), department ID, and how many years they've been at the company (as a whole number). Exclude any security-sensitive or unnecessary columns.

---

### Exercise 2 — Fix the Aliases

Find all bugs in this query and rewrite it correctly:

```sql
SELECT
    product_id,
    SUM(quantity * unit_price) AS revenue,
    COUNT(*) AS total_orders,
    revenue / total_orders AS avg_order_value
FROM order_items
WHERE revenue > 1000
GROUP BY product_id
HAVING total_orders > 5
ORDER BY avg_order_value DESC;
```

---

### Exercise 3 — DISTINCT Diagnosis

Explain why this query is wrong and rewrite it correctly using the appropriate alternative:

```sql
SELECT DISTINCT u.id, u.name, u.email
FROM users u
JOIN user_roles ur ON ur.user_id = u.id
JOIN roles r ON r.id = ur.role_id
WHERE r.name = 'admin';
```

Your rewrite must not produce duplicate users and must not use DISTINCT.

---

### Exercise 4 — N+1 in SQL

This query works but doesn't scale. Rewrite it to use a single scan of the `reviews` table:

```sql
SELECT
    p.id,
    p.name,
    p.price,
    (SELECT AVG(rating) FROM reviews r WHERE r.product_id = p.id) AS avg_rating,
    (SELECT COUNT(*) FROM reviews r WHERE r.product_id = p.id) AS review_count,
    (SELECT MAX(created_at) FROM reviews r WHERE r.product_id = p.id) AS last_reviewed_at
FROM products p
WHERE p.is_active = TRUE;
```

---

## Interview Questions

### Junior Level

**Q: What is the difference between `SELECT *` and specifying column names?**

A: `SELECT *` returns all columns from the table, while specifying column names returns only those columns. `SELECT *` is convenient in development but has downsides: it returns unnecessary data, prevents index-only scans, and creates a fragile dependency on the table schema.

---

**Q: Can you use a SELECT alias in the WHERE clause?**

A: No. WHERE executes before SELECT in the logical execution order. The alias doesn't exist yet when WHERE evaluates. You must repeat the expression in WHERE, or use a subquery/CTE.

---

### Principal Level

**Q: A developer says "I'll add DISTINCT to the query to fix the duplicate rows." What do you say?**

A: I'd ask why duplicates are appearing. DISTINCT is a symptom fix, not a root cause fix. Duplicates in SELECT queries typically come from a JOIN creating fan-out — a one-to-many relationship where we're selecting columns only from the "one" side but joining through the "many" side. The correct fix is usually to restructure the query: use EXISTS instead of JOIN to express "does at least one matching row exist," or use a lateral subquery, or aggregate properly. DISTINCT costs a sort or hash operation on the full result set every time the query runs. Fixing the JOIN is free and often orders of magnitude faster at scale.

Beyond correctness, I'd look at whether this is a long-running query and run EXPLAIN ANALYZE to see if the deduplication shows up as a HashAggregate or Sort node with high actual time.

---

**Q: You're reviewing a PR. You see `findMany()` in a Prisma query that lists items on a page. What do you check?**

A: First, whether it has a `select` option. Without it, Prisma generates SELECT * which returns every column including potentially sensitive fields and large columns. I'd check what the API response shape needs to be, then ensure the `select` option includes exactly those columns plus any computed fields needed. I'd also verify there's no N+1 — that any related data is fetched via `include` or a separate aggregated query rather than individual follow-up queries per row. Finally, I'd check for pagination — `findMany()` without `take` on a large table will eventually cause OOM or timeout.

---

**Q: What's `DISTINCT ON` in PostgreSQL and when do you use it instead of window functions?**

A: `DISTINCT ON (expressions)` keeps exactly one row per distinct value of the specified expressions, choosing which row based on the ORDER BY clause. It's PostgreSQL-specific and not standard SQL.

I use it instead of a window function subquery for "latest row per group" patterns when:
1. I need the entire row (not just a computed rank)
2. The query is simple (one table or a simple join)
3. I'm willing to accept the PostgreSQL-specific syntax

`DISTINCT ON` is typically faster than a `ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` subquery because it avoids a subquery layer and often results in a simpler execution plan. However, it can only apply to the final SELECT — you can't use it as a building block inside a larger query the same way you can use a ranked CTE. For complex queries with multiple further filters, window functions in a CTE give more control.

---

## Mental Model Checkpoint

Answer these to verify you've internalized the topic:

1. At what step of the 9-step logical execution order does SELECT run? What has already happened before it? What happens after?

2. You write `SELECT price * 1.2 AS marked_up`. Where can you reference `marked_up` in the same query? Where can't you? Why?

3. A query uses `SELECT *` and hits a table with an index on `(user_id, status, created_at)`. The WHERE clause filters on `user_id` and `status`. Can this query use an Index-Only Scan? Why or why not?

4. `SELECT DISTINCT a, b FROM t` — what does "distinct" mean here? Is it distinct on `a` alone, `b` alone, or `(a, b)` together?

5. You have a query with `(SELECT COUNT(*) FROM orders WHERE user_id = u.id) AS cnt` in the SELECT list. The users table has 50,000 rows. Approximately how many times does the orders table get scanned? What's the fix?

6. In Node.js with `pg`, you run `SELECT * FROM users JOIN addresses ON addresses.user_id = users.id`. Both tables have an `id` column. What does `rows[0].id` return? Why?

7. `DISTINCT ON (user_id)` — what must the ORDER BY clause always include? What does the rest of ORDER BY control?

---

## Quick Reference Card

```
SELECT Execution Position
─────────────────────────
... WHERE → GROUP BY → HAVING → [SELECT] → DISTINCT → WINDOW → ORDER BY → LIMIT

Alias Availability
──────────────────
WHERE:    ✗ (runs before SELECT)
GROUP BY: Partial (PostgreSQL extension, unreliable)
HAVING:   ✗ (runs before SELECT)
ORDER BY: ✓ (runs after SELECT in PostgreSQL)
WINDOW:   ✓

SELECT * Dangers
────────────────
1. Prevents Index-Only Scans
2. Returns sensitive columns (password_hash, tokens)
3. Network overhead from unused large columns
4. Silent breakage when schema changes
5. In joins: duplicate column names → silent overwrites in pg

DISTINCT vs DISTINCT ON
───────────────────────
DISTINCT:    removes duplicate output rows (all columns compared)
DISTINCT ON: keeps first row per unique value of specified columns
             → ORDER BY must start with DISTINCT ON columns

Performance Tiers
─────────────────
Fastest:  SELECT col1, col2 → enables Index-Only Scan
Fast:     SELECT t.*        → all cols, one table
Slower:   SELECT *          → all cols, prevents Index-Only Scan
Danger:   Scalar subquery   → runs once per output row
Danger:   SELECT * multi-join → duplicate columns, all heap fetches

Scalar Subquery Fix
───────────────────
BAD:  SELECT (SELECT COUNT(*) FROM t WHERE t.id = outer.id)
GOOD: LEFT JOIN (SELECT id, COUNT(*) FROM t GROUP BY id) agg ON agg.id = outer.id

Expression References — CTE Pattern
─────────────────────────────────────
WITH base AS (
    SELECT col, expr AS alias FROM t
)
SELECT alias * 2 FROM base;  -- can reference alias here
```

---

## Connected Topics

| Topic | Connection |
|-------|-----------|
| [02-logical-execution-order.md](02-logical-execution-order.md) | Why SELECT aliases can't be used in WHERE — the execution order |
| [06-where-clause-mastery.md](06-where-clause-mastery.md) | WHERE comes before SELECT — the alias scope rule in practice |
| [09-case-expressions.md](09-case-expressions.md) | CASE in SELECT — the primary use case for computed columns |
| [17-join-performance.md](17-join-performance.md) | Why `SELECT *` in joins causes more heap fetches |
| [18-n-plus-one-problem.md](18-n-plus-one-problem.md) | Scalar subqueries in SELECT are SQL-level N+1 |
| [26-subqueries-in-depth.md](26-subqueries-in-depth.md) | Scalar subqueries in SELECT list — full mechanics |
| [41-top-n-per-group.md](41-top-n-per-group.md) | `DISTINCT ON` as an alternative to window function approach |
| [43-index-scan-types.md](43-index-scan-types.md) | Index-Only Scans — why explicit projection enables them |
| [47-index-only-scans.md](47-index-only-scans.md) | Deep dive on Index-Only Scans and the INCLUDE clause |
