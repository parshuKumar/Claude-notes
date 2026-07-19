# Topic 26 — Subqueries in Depth
### SQL Mastery Curriculum — Phase 5: Subqueries and CTEs

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you walk into a library and ask the librarian: *"Give me every book written by the author who wrote 'The Silent Sea'."*

The librarian can't answer that in one step. She has to solve a **smaller question first**:

1. **Inner question**: "Who wrote 'The Silent Sea'?" → she looks it up → *"Author #47, Marisol Vega."*
2. **Outer question**: "Now give me every book by Author #47." → she walks to the shelf and pulls them.

That inner lookup — the question she had to answer *before* she could answer yours — is a **subquery**. It's a query nested inside another query, and its answer feeds the outer query.

Now imagine a slightly different request: *"Give me every author who has written at least one book that is currently checked out."* This time the librarian can't answer the inner question once and be done. For **each author** she considers, she has to go check: "Does *this* author have any checked-out book right now?" She re-runs the inner check for every author. That's a **correlated subquery** — the inner question depends on the outer row she's currently looking at, so it runs again and again.

And there's a third flavour: *"Give me a temporary list of the top 10 best-selling books, and then join that list to their authors."* Here the inner query produces a whole **mini-table** that the outer query treats like a real table. That's a **derived table** (a subquery in the FROM clause).

Three shapes, one idea: **a query whose result is consumed by another query.** The whole art of subqueries is knowing *where* the inner query plugs in (does it produce one value? one row? a whole table?), and *how often* it runs (once, or once per outer row).

---

## 2. Connection to SQL Internals

A subquery is not a special engine feature — it is a **planning-time construct**. When PostgreSQL parses a statement, the subquery becomes a separate node in the query tree (a `SubLink` or `SubPlan`/`InitPlan` in the internals). What happens next is where the interesting engineering lives:

1. **Subquery flattening (pull-up)**: The planner's `pull_up_subqueries()` pass tries to *dissolve* the subquery boundary and merge it into the parent query, so the whole thing can be planned as one flat join problem. A subquery in `FROM` (a derived table) or an `IN (SELECT ...)`/`EXISTS (SELECT ...)` is a prime flattening candidate. Once flattened, the planner is free to reorder joins, push predicates, and choose Hash/Merge/Nested-Loop just as if you had written a join by hand. This is the single most important internal fact about subqueries: **most of them are not executed "as subqueries" at all — they are rewritten away.**

2. **InitPlan (uncorrelated scalar subquery)**: A subquery that references no columns from the outer query and returns a single value is executed **exactly once**, before the main plan runs, and its result is cached as a constant. In `EXPLAIN` this shows up as `InitPlan 1` with a `$0` parameter. The classic case is `WHERE price > (SELECT AVG(price) FROM products)` — the average is computed once.

3. **SubPlan (correlated subquery)**: A subquery that references outer columns becomes a `SubPlan`, parameterised on those outer columns (shown as `$0`, `$1`...). It is re-executed for each outer row that reaches it — unless the planner converts it into a **semi-join** or **anti-join** (for `EXISTS`/`NOT EXISTS`/`IN`), which turns the repeated execution into a single set-based join. This conversion is the difference between O(N) sub-executions and O(N + M) hash-join behaviour.

4. **Materialization**: A derived table that cannot be flattened (e.g., it contains `LIMIT`, `DISTINCT ON`, a volatile function, or aggregation the planner chooses not to pull up) is **materialized** — executed once, its rows stashed in a tuplestore in `work_mem` (spilling to a temp file on disk if it exceeds `work_mem`), and then scanned by the parent.

Underneath all of this sit the same primitives you already know from earlier topics: B-tree index probes for correlated lookups, the buffer pool serving heap pages, hash tables for semi-joins, sorts for merge-based semi-joins, and MVCC visibility checks on every tuple the subquery touches. A subquery is just a way of *describing* a nested computation; the planner's job is to turn that description into the cheapest tree of physical operators — very often by making the subquery disappear.

---

## 3. Logical Execution Order Context

```
FROM  (including derived-table subqueries)   ← FROM-clause subqueries resolved here
JOIN
WHERE (including subquery predicates)          ← WHERE subqueries evaluated here
GROUP BY
HAVING (including subquery predicates)         ← HAVING subqueries evaluated here
SELECT (including scalar subqueries)           ← SELECT-list subqueries evaluated here
DISTINCT
window functions
ORDER BY (may reference scalar subqueries)
LIMIT
```

Subqueries appear at almost every stage, and *where* they sit determines *when* they run and *what they may reference*:

- **FROM (derived table)**: Logically resolved first, before the outer WHERE. It becomes a row source like any table. It **cannot** reference columns from other tables in the same FROM list unless you use `LATERAL` (Topic 28). A plain derived table is self-contained.
- **WHERE**: Evaluated during the row-filtering phase. A subquery here (`IN`, `EXISTS`, scalar comparison) is checked per candidate row. This is where correlation is most common and most dangerous for performance.
- **HAVING**: Evaluated after grouping. A subquery in HAVING can compare a group's aggregate against a value produced by another query — e.g., "groups whose total exceeds the global average."
- **SELECT list (scalar subquery)**: Evaluated last-ish, per output row. A scalar subquery in the SELECT list runs (logically) once per row that survives to projection — which is why an uncorrelated one gets hoisted to an InitPlan and a correlated one becomes a per-row SubPlan.

The key mental rule: **a subquery's position tells you its cardinality contract.** A scalar subquery (SELECT list, or one side of a `=`/`>` comparison) must return **at most one row and one column** or you get a runtime error. A subquery feeding `IN`/`EXISTS` may return **many rows**. A derived table may return a **full table**. Getting this contract wrong is the most common subquery bug — covered in Section 9.

---

## 4. What Is a Subquery?

A **subquery** (also called an inner query or nested query) is a complete `SELECT` statement enclosed in parentheses and embedded inside another SQL statement. Its result is consumed by the enclosing (outer) query. Subqueries are classified by **shape** (how many rows/columns they return) and by **dependency** (whether they reference the outer query).

```sql
SELECT
  u.id,
  u.name,
  (SELECT COUNT(*)                    -- ┐ scalar subquery in SELECT list
     FROM orders o                     -- │ returns exactly ONE value per outer row
    WHERE o.user_id = u.id) AS n_orders -- └── correlated: references u.id
FROM users u
WHERE u.id IN (                        -- ┐ row/table subquery feeding IN
  SELECT o.user_id                     -- │ returns a COLUMN of values (many rows)
    FROM orders o                       -- │ uncorrelated: standalone
   WHERE o.total_amount > 500          -- │
)                                       -- └──
  AND u.region = (                     -- ┐ scalar subquery on RHS of '='
    SELECT region                       -- │ MUST return exactly one row/one column
      FROM users                        -- │
     WHERE id = 1                       -- │
  );                                     -- └──
```

### The three shapes

```
Scalar subquery   →  returns 1 row, 1 column      → usable anywhere a single value is
                                                     (SELECT list, =, >, <, WHERE, HAVING)

Row subquery      →  returns 1 row, N columns      → usable with row comparison
                                                     (a, b) = (SELECT x, y FROM ...)

Table subquery    →  returns M rows, N columns     → usable in FROM (derived table),
                                                     or M rows/1 col with IN / EXISTS / ANY / ALL
```

### The two dependency classes

```
Uncorrelated  →  the subquery is self-contained; it references NO column from the
                 outer query. It could be run standalone. Executed ONCE (InitPlan /
                 materialized derived table).

Correlated    →  the subquery references at least one column from the outer query
                 (a "correlation variable"). It cannot run standalone. Logically
                 re-executed per outer row (SubPlan) — unless rewritten to a join.
```

### The four positions

```
SELECT list   →  scalar only:  SELECT (SELECT ...) AS c FROM ...
FROM          →  derived table: FROM (SELECT ...) AS d
WHERE         →  scalar / IN / EXISTS / ANY / ALL:  WHERE x IN (SELECT ...)
HAVING        →  scalar / IN / EXISTS:  HAVING SUM(x) > (SELECT ...)
```

Every real-world subquery is one point in this 3 × 2 × 4 space: a shape, a dependency class, and a position. Master the space and you master subqueries.

---

## 5. Why Subquery Mastery Matters in Production

1. **Correctness of cardinality**: A scalar subquery that unexpectedly returns two rows throws `ERROR: more than one row returned by a subquery used as an expression` — in production, at 2 a.m., on the one customer whose data violated your assumption. Knowing which positions demand scalar results, and how to guarantee it (`LIMIT 1`, aggregation, a unique constraint), is the difference between a query that works on your laptop and one that survives real data.

2. **The correlated-subquery performance cliff**: A correlated subquery in a SELECT list or WHERE clause can silently degrade from milliseconds to minutes as the outer row count grows, because it re-executes per row. On 10 rows nobody notices; on 10 million rows it is a full outage. Recognising the pattern in `EXPLAIN` (a `SubPlan` with high `loops`) and knowing when to rewrite it into a join or a pre-aggregated derived table is core senior skill.

3. **Knowing when the planner saves you (and when it can't)**: PostgreSQL flattens `IN`/`EXISTS` into semi-joins and pulls up simple derived tables automatically — so a "subquery" is often free. But it will **not** flatten across `LIMIT`, `DISTINCT ON`, volatile functions, or aggregates in certain positions. Understanding the flattening rules tells you which subqueries are syntactic sugar and which are optimization fences you must reason about.

4. **Readability vs. the right tool**: Subqueries, JOINs, CTEs (Topic 29), and window functions (Phase 6) often express the same result. Choosing the clearest form that also plans well — and knowing that a derived table with pre-aggregation frequently beats a correlated scalar subquery — is what separates maintainable analytics code from a query nobody dares touch.

5. **Debugging "wrong number of rows"**: Row multiplication (from JOINs, Topic 11) and row *loss* (from INNER semantics or NULL-in-`NOT IN`) both surface through subqueries. The `NOT IN (SELECT ... )` NULL trap alone — where one NULL in the subquery makes the entire predicate return zero rows — has caused countless "the report is empty and I don't know why" incidents.

---

## 6. Deep Technical Content

### 6.1 Scalar Subqueries — The One-Value Contract

A **scalar subquery** returns exactly one row and one column. It can stand in for a single value anywhere an expression is allowed.

```sql
-- In the SELECT list: attach a computed value to each row
SELECT
  p.id,
  p.name,
  p.price,
  (SELECT AVG(price) FROM products) AS overall_avg,   -- one value, same for every row
  p.price - (SELECT AVG(price) FROM products) AS diff_from_avg
FROM products p;
```

Because `(SELECT AVG(price) FROM products)` references no outer column, it is **uncorrelated** and PostgreSQL computes it **once** as an InitPlan, caching the result. The two references above share that single computation.

The **contract**: at most one row. Zero rows is allowed and yields `NULL`:

```sql
SELECT
  o.id,
  (SELECT name FROM users u WHERE u.id = o.user_id) AS user_name  -- NULL if no such user
FROM orders o;
```

Two or more rows is a **runtime error**:

```sql
-- If a user_id maps to multiple users (impossible with a PK, but possible with bad data):
SELECT (SELECT email FROM users WHERE region = 'EU');
-- ERROR:  more than one row returned by a subquery used as an expression
```

Guaranteeing scalar-ness when the logic might return many rows:

```sql
-- Latest order amount per user, guaranteed single row via ORDER BY + LIMIT 1
SELECT
  u.id,
  u.name,
  (SELECT o.total_amount
     FROM orders o
    WHERE o.user_id = u.id
    ORDER BY o.created_at DESC
    LIMIT 1) AS last_order_amount   -- correlated scalar subquery
FROM users u;
```

This one **is** correlated (`o.user_id = u.id`), so it is a per-row SubPlan. For 1M users that is 1M ordered index scans into `orders`. Fine with an index on `orders(user_id, created_at DESC)`; catastrophic without.

### 6.2 Row Subqueries — Comparing Tuples

A **row subquery** returns one row of multiple columns, compared against a row constructor.

```sql
-- Find the order that matches BOTH the max amount AND its exact timestamp
SELECT *
FROM orders
WHERE (total_amount, created_at) = (
  SELECT MAX(total_amount), MAX(created_at) FROM orders  -- one row, two columns
);
```

Row comparison applies element-wise. Ordering comparisons follow lexicographic rules:

```sql
-- (a, b) < (x, y)  is TRUE if a < x, OR (a = x AND b < y)
SELECT * FROM events
WHERE (priority, created_at) > (
  SELECT priority, created_at FROM events WHERE id = 100
);
```

Row subqueries are comparatively rare but invaluable for "match on a composite key" logic without repeating conditions. NULL handling follows three-valued logic (Topic 07): if any compared element is NULL, the row comparison can yield UNKNOWN.

### 6.3 Table Subqueries in FROM — Derived Tables

A subquery in `FROM` is a **derived table** (a.k.a. inline view). It must be aliased. It behaves like a real table for the rest of the query.

```sql
SELECT
  d.user_id,
  d.order_count,
  d.total_spend,
  u.name
FROM (
  SELECT
    user_id,
    COUNT(*)          AS order_count,
    SUM(total_amount) AS total_spend
  FROM orders
  WHERE status = 'completed'
  GROUP BY user_id
) d                                   -- ← mandatory alias
INNER JOIN users u ON u.id = d.user_id
WHERE d.total_spend > 1000;
```

**Why derived tables matter**: they let you **pre-aggregate before joining**, avoiding the fan-out row-multiplication bug from Topic 11. Instead of joining `users → orders → order_items` and then wrestling with `COUNT(DISTINCT ...)`, you collapse each table to one row per key first, then join clean one-to-one.

**Flattening**: A simple derived table (plain projection/filter/join) is *pulled up* into the parent — no separate execution. But if it contains `GROUP BY`, `DISTINCT`, `LIMIT`, or set operations, the planner usually keeps it as a `Subquery Scan` (materialized or streamed). You will see this in `EXPLAIN` as a `Subquery Scan on d` or an `Aggregate` feeding a `Hash Join`.

### 6.4 IN / NOT IN with a Subquery

`x IN (SELECT ...)` tests membership against a set (one column, many rows). PostgreSQL almost always rewrites `IN (SELECT ...)` into a **semi-join** — it stops at the first match and never multiplies rows.

```sql
-- Users who have placed at least one high-value order
SELECT u.id, u.name
FROM users u
WHERE u.id IN (
  SELECT o.user_id FROM orders o WHERE o.total_amount > 500
);
```

This is uncorrelated. The planner builds a hash of distinct `user_id`s from the subquery and semi-joins. No duplicate users even if a user has many big orders — semi-join semantics guarantee one output row per outer match.

**The `NOT IN` NULL trap** — the single most dangerous subquery gotcha:

```sql
-- Users who have NEVER placed an order — DANGEROUS if orders.user_id can be NULL
SELECT u.id FROM users u
WHERE u.id NOT IN (SELECT user_id FROM orders);
```

If **any** `user_id` in the subquery is NULL, `NOT IN` evaluates to UNKNOWN for **every** outer row, and the query returns **zero rows** — silently. This is three-valued logic (Topic 07): `x NOT IN (a, b, NULL)` is `x <> a AND x <> b AND x <> NULL`; the last term is UNKNOWN, so the whole conjunction can never be TRUE. Fix: use `NOT EXISTS` (Topic 27), or filter NULLs explicitly:

```sql
SELECT u.id FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);   -- NULL-safe
-- or
WHERE u.id NOT IN (SELECT user_id FROM orders WHERE user_id IS NOT NULL);
```

### 6.5 EXISTS / NOT EXISTS

`EXISTS (subquery)` returns TRUE as soon as the subquery yields one row; the subquery's SELECT list is irrelevant (convention: `SELECT 1`). It is almost always **correlated** and the planner turns it into a **semi-join** (`EXISTS`) or **anti-join** (`NOT EXISTS`).

```sql
SELECT u.id, u.name
FROM users u
WHERE EXISTS (
  SELECT 1 FROM orders o
  WHERE o.user_id = u.id AND o.status = 'completed'
);
```

`EXISTS` is NULL-safe (unlike `NOT IN`) because it tests row *existence*, not value equality. It is covered exhaustively in Topic 27; here the point is that `EXISTS` is a subquery pattern the planner treats as first-class semi/anti-join.

### 6.6 ANY / SOME / ALL — Quantified Comparisons

`ANY` (synonym `SOME`) and `ALL` combine a comparison operator with a subquery set.

```sql
-- price greater than AT LEAST ONE product in category 5  (= any)
SELECT * FROM products
WHERE price > ANY (SELECT price FROM products WHERE category_id = 5);

-- price greater than EVERY product in category 5         (= all)
SELECT * FROM products
WHERE price > ALL (SELECT price FROM products WHERE category_id = 5);
```

Equivalences worth memorising:
- `x = ANY (...)` ≡ `x IN (...)`
- `x <> ALL (...)` ≡ `x NOT IN (...)` (and carries the same NULL trap)
- `x > ALL (...)` ≡ `x > (SELECT MAX(...))` — often the planner rewrites it exactly so.

**Empty-set behaviour** (crucial edge case):
- `x > ALL (empty set)` → **TRUE** (vacuously; nothing to violate)
- `x > ANY (empty set)` → **FALSE** (nothing to satisfy)

This is the opposite of most people's intuition and is a frequent interview trap.

### 6.7 Correlated Subqueries — Mechanics and Cost

A **correlated** subquery references a column from the outer query. It cannot be evaluated independently; conceptually it re-runs for each outer row.

```sql
-- For each order, how many other orders did the same user place BEFORE it?
SELECT
  o.id,
  o.user_id,
  o.created_at,
  (SELECT COUNT(*)
     FROM orders o2
    WHERE o2.user_id = o.user_id           -- correlation on user_id
      AND o2.created_at < o.created_at)     -- correlation on created_at
    AS prior_orders
FROM orders o;
```

The correlation variables are `o.user_id` and `o.created_at`. Physically this is a `SubPlan` with parameters `$0, $1`, re-executed per outer row (`loops = N`). Two escape routes:

1. **Add the supporting index** so each re-execution is a cheap index range scan (`orders(user_id, created_at)`).
2. **Rewrite** using a window function (`COUNT(*) OVER (PARTITION BY user_id ORDER BY created_at)` — Phase 6) or a self-join with aggregation, converting N sub-executions into one pass.

### 6.8 Subquery Flattening / Pull-Up — What the Planner Rewrites

This is the internal behaviour that makes most subqueries free. The planner attempts, in `subquery_planner` → `pull_up_subqueries`:

- **Derived tables** (`FROM (SELECT ...)`): pulled up into the parent's join tree when they are "simple" — no `LIMIT`/`OFFSET`, no `DISTINCT ON`, no aggregation/`GROUP BY`/`HAVING` that blocks it, no volatile functions, no set operations. After pull-up the subquery is gone and the planner treats its tables as part of one flat join.
- **`IN (SELECT ...)` / `= ANY (SELECT ...)`**: converted to a **semi-join** (`JOIN_SEMI`).
- **`NOT IN` / `<> ALL`**: converted to an **anti-join** *only when NULL-safety allows* — often it cannot, which is both why `NOT IN` is slow and why it is buggy.
- **`EXISTS`**: converted to a semi-join.
- **`NOT EXISTS`**: converted to an anti-join.
- **Uncorrelated scalar subquery**: hoisted to an **InitPlan**, run once.

What **blocks** flattening (creating an "optimization fence" you must reason about):
- `LIMIT` / `OFFSET` inside the subquery
- `DISTINCT ON`
- Aggregation/`GROUP BY` in positions the planner won't pull up
- Volatile functions (e.g., `random()`, `nextval()`)
- Set operations (`UNION`/`INTERSECT`/`EXCEPT`) in the subquery
- Correlation that cannot be de-correlated

Note: unlike a `WITH` CTE (Topic 29), which historically was an *unconditional* optimization fence before PG12 and is fence-able with `MATERIALIZED` after, subqueries are **flattened aggressively by default**. If you *want* a fence, a CTE or a `LIMIT`/`OFFSET 0` trick provides one; if you want maximum planner freedom, a plain subquery/derived table is usually the better choice.

### 6.9 Subqueries in HAVING

A subquery in `HAVING` compares a group's aggregate against a value.

```sql
-- Categories whose total revenue exceeds the average category revenue
SELECT
  p.category_id,
  SUM(oi.quantity * oi.unit_price) AS revenue
FROM order_items oi
INNER JOIN products p ON p.id = oi.product_id
GROUP BY p.category_id
HAVING SUM(oi.quantity * oi.unit_price) > (
  SELECT AVG(cat_rev) FROM (
    SELECT SUM(oi2.quantity * oi2.unit_price) AS cat_rev
    FROM order_items oi2
    INNER JOIN products p2 ON p2.id = oi2.product_id
    GROUP BY p2.category_id
  ) t
);
```

The inner uncorrelated scalar subquery (average of per-category revenue) is an InitPlan run once; the outer HAVING filters groups against it.

### 6.10 Nesting Depth and Multiple Subqueries

Subqueries nest arbitrarily. Each level is a planning scope. Readability collapses fast beyond two levels — this is precisely where CTEs (Topic 29) earn their keep. But the planner treats a two-level nested subquery and the equivalent CTE-with-inline similarly once flattening applies.

```sql
-- Three levels: users → their orders → items in those orders over a threshold
SELECT u.id, u.name
FROM users u
WHERE u.id IN (
  SELECT o.user_id FROM orders o
  WHERE o.id IN (
    SELECT oi.order_id FROM order_items oi
    WHERE oi.unit_price > 1000
  )
);
```

The planner flattens both `IN`s into semi-joins, producing a three-table semi-join chain — no per-row re-execution at all.

### 6.11 Subqueries vs JOIN — When Each Wins

- **Existence check** ("users who have ordered"): `EXISTS`/`IN` semi-join beats `INNER JOIN + DISTINCT` because it stops at first match and never multiplies rows.
- **Attaching one aggregate per outer row**: a correlated scalar subquery is readable but re-executes; a `LEFT JOIN` to a pre-aggregated derived table computes it in one pass and is usually faster at scale.
- **Anti-membership** ("users who never ordered"): `NOT EXISTS` anti-join is both correct (NULL-safe) and fast; `NOT IN` is a trap.
- **Multiple aggregates from the same child**: derived table wins decisively — one scan yields COUNT, SUM, MAX together, versus one correlated subquery per metric.

### 6.12 NULL Semantics Recap Across Subquery Forms

| Form | Empty subquery result | NULL in subquery result |
|------|----------------------|------------------------|
| scalar `= (SELECT ...)` | comparison with NULL → UNKNOWN → row excluded | value is NULL → comparison UNKNOWN |
| `IN (SELECT ...)` | FALSE (no match) | can make result UNKNOWN, but never wrongly *includes* |
| `NOT IN (SELECT ...)` | TRUE | **any NULL → whole predicate UNKNOWN → zero rows** |
| `EXISTS (SELECT ...)` | FALSE | irrelevant — tests existence, NULL-safe |
| `NOT EXISTS (SELECT ...)` | TRUE | NULL-safe |
| `> ALL (SELECT ...)` | TRUE (vacuous) | NULL propagates to UNKNOWN |
| `> ANY (SELECT ...)` | FALSE | NULL may yield UNKNOWN |

---

## 7. EXPLAIN — Subqueries in the Plan

### 7.1 Uncorrelated scalar subquery → InitPlan (runs once)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, name, price
FROM products
WHERE price > (SELECT AVG(price) FROM products);
```

```
Seq Scan on products  (cost=25.88..51.75 rows=850 width=40)
                       (actual time=0.42..1.10 rows=812 loops=1)
  Filter: (price > $0)
  Rows Removed by Filter: 1738
  InitPlan 1 (returns $0)
    ->  Aggregate  (cost=25.63..25.64 rows=1 width=8)
                   (actual time=0.38..0.38 rows=1 loops=1)
          ->  Seq Scan on products products_1  (cost=0.00..23.50 rows=2550 width=6)
                                                (actual time=0.01..0.18 rows=2550 loops=1)
Buffers: shared hit=46
Planning Time: 0.20 ms
Execution Time: 1.19 ms
```

**Reading it**:
- `InitPlan 1 (returns $0)` — the average is computed **once**, before the main scan, and stored in `$0`.
- The outer `Filter: (price > $0)` reuses that constant for every row — no re-execution (`loops=1` on the InitPlan).
- This is the ideal case: an uncorrelated scalar subquery costs one extra aggregate pass, period.

### 7.2 Correlated scalar subquery → SubPlan (runs per row)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.id, u.name,
  (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS n_orders
FROM users u;
```

```
Seq Scan on users u  (cost=0.00..18620.00 rows=1000 width=48)
                     (actual time=0.05..142.7 rows=1000 loops=1)
  SubPlan 1
    ->  Aggregate  (cost=18.55..18.56 rows=1 width=8)
                   (actual time=0.14..0.14 rows=1 loops=1000)
          ->  Index Only Scan using orders_user_id_idx on orders o
                (cost=0.42..18.30 rows=8 width=0)
                (actual time=0.02..0.11 rows=9 loops=1000)
                Index Cond: (user_id = u.id)
Buffers: shared hit=9042
Planning Time: 0.22 ms
Execution Time: 143.1 ms
```

**Reading it**:
- `SubPlan 1` with `loops=1000` — the subquery re-executes **once per outer user**.
- Each execution is a cheap `Index Only Scan` (thanks to `orders_user_id_idx`), so 1000 loops = ~140ms. Without that index each loop would be a Seq Scan and total time would explode.
- The tell of a correlated subquery is always: `SubPlan` + `loops = (outer row count)` + a parameter (`u.id`) in the inner `Index Cond`.

### 7.3 IN (SELECT ...) → Hash Semi Join (flattened)

```sql
EXPLAIN (ANALYZE)
SELECT u.id, u.name
FROM users u
WHERE u.id IN (SELECT user_id FROM orders WHERE total_amount > 500);
```

```
Hash Semi Join  (cost=430.00..820.00 rows=940 width=40)
                (actual time=3.1..12.4 rows=612 loops=1)
  Hash Cond: (u.id = orders.user_id)
  ->  Seq Scan on users u  (cost=0.00..25.00 rows=1000 width=40)
                           (actual rows=1000 loops=1)
  ->  Hash  (cost=380.00..380.00 rows=4000 width=8)
            (actual time=3.0..3.0 rows=3820 loops=1)
        ->  Seq Scan on orders  (cost=0.00..380.00 rows=4000 width=8)
              Filter: (total_amount > 500)
Buffers: shared hit=290
Execution Time: 12.8 ms
```

**Reading it**:
- The `IN (SELECT ...)` became a `Hash Semi Join` — no SubPlan, no per-row loop. The subquery "disappeared" into a join.
- `Semi Join` guarantees each `users` row appears at most once regardless of how many matching orders exist. No `DISTINCT` needed.
- This is subquery flattening in action: what you wrote as a nested query planned as a single set-based join.

### 7.4 NOT EXISTS → Hash Anti Join

```sql
EXPLAIN (ANALYZE)
SELECT u.id FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

```
Hash Anti Join  (cost=430.00..900.00 rows=200 width=8)
                (actual time=4.0..9.1 rows=188 loops=1)
  Hash Cond: (u.id = o.user_id)
  ->  Seq Scan on users u  (actual rows=1000 loops=1)
  ->  Hash  (actual rows=8120 loops=1)
        ->  Seq Scan on orders o  (actual rows=8120 loops=1)
Execution Time: 9.4 ms
```

**Reading it**: `NOT EXISTS` → `Anti Join`: outer rows with **no** matching inner row are kept. NULL-safe by construction — contrast with `NOT IN` which cannot always become an anti-join.

### 7.5 Derived table → Subquery Scan / pulled-up

```sql
EXPLAIN (ANALYZE)
SELECT d.user_id, d.total_spend, u.name
FROM (SELECT user_id, SUM(total_amount) AS total_spend
      FROM orders GROUP BY user_id) d
JOIN users u ON u.id = d.user_id
WHERE d.total_spend > 1000;
```

```
Hash Join  (cost=920.00..1180.00 rows=310 width=48)
           (actual time=8.2..11.9 rows=286 loops=1)
  Hash Cond: (u.id = d.user_id)
  ->  Seq Scan on users u  (actual rows=1000 loops=1)
  ->  Hash  (actual rows=286 loops=1)
        ->  Subquery Scan on d  (cost=850.00..905.00 rows=310 width=12)
              Filter: (d.total_spend > 1000)
              ->  HashAggregate  (cost=850.00..885.00 rows=930 width=12)
                    Group Key: orders.user_id
                    ->  Seq Scan on orders  (actual rows=8120 loops=1)
Execution Time: 12.3 ms
```

**Reading it**:
- The derived table shows as `Subquery Scan on d` wrapping a `HashAggregate` — the aggregation blocked full pull-up, so it is executed as its own node and its rows feed the outer `Hash Join`.
- Note the planner pushed `d.total_spend > 1000` as a `Filter` on the Subquery Scan — it filters aggregated groups before the join.

### 7.6 Signals table

| EXPLAIN signal | Meaning | Action |
|----------------|---------|--------|
| `InitPlan N (returns $0)` | uncorrelated scalar, run **once** | ideal — leave it |
| `SubPlan N` + high `loops` | correlated, re-executing per row | ensure supporting index, or rewrite to join/window |
| `Hash Semi Join` | `IN`/`EXISTS` flattened | good — set-based, no duplicates |
| `Hash Anti Join` | `NOT EXISTS`/safe `NOT IN` | good — NULL-safe anti-membership |
| `Subquery Scan on x` | derived table not fully pulled up | check for LIMIT/DISTINCT/agg blocking flatten |
| `Materialize` above subquery | rows cached for repeated inner scans | fine if small; watch `work_mem` spill |
| SubPlan inner is `Seq Scan` | correlated lookup with no index | add index on correlation column — urgent |

---

## 8. Query Examples

### Example 1 — Basic: Uncorrelated scalar subquery in WHERE

```sql
-- Products priced above the catalog-wide average.
-- The subquery is uncorrelated → computed ONCE as an InitPlan.
SELECT
  p.id,
  p.name,
  p.price
FROM products p
WHERE p.price > (SELECT AVG(price) FROM products)   -- single value, run once
ORDER BY p.price DESC;
```

### Example 2 — Intermediate: Correlated scalar subquery + IN filter

```sql
-- For every active user who has ordered in the last 90 days,
-- attach their most recent order amount (correlated scalar subquery)
-- and restrict to users who appear in the recent-orders set (uncorrelated IN).
SELECT
  u.id,
  u.name,
  u.email,
  (SELECT o.total_amount                       -- correlated scalar subquery
     FROM orders o
    WHERE o.user_id = u.id
    ORDER BY o.created_at DESC
    LIMIT 1) AS last_order_amount              -- guaranteed one row via LIMIT 1
FROM users u
WHERE u.status = 'active'
  AND u.id IN (                                 -- uncorrelated → semi-join
    SELECT o2.user_id
      FROM orders o2
     WHERE o2.created_at >= CURRENT_DATE - INTERVAL '90 days'
  )
ORDER BY last_order_amount DESC NULLS LAST;
```

### Example 3 — Production Grade: Derived table pre-aggregation vs correlated subqueries

**Scenario**: A "customer value dashboard" over `users` (2M rows), `orders` (25M rows, indexed on `(user_id, created_at)`), and `order_items` (90M rows, indexed on `(order_id)`). We need, per active user: number of completed orders, lifetime spend, average order value, most recent order date, and distinct products purchased. The naive approach uses five correlated scalar subqueries in the SELECT list — five per-row SubPlans over 2M users. The production approach pre-aggregates each child once in a derived table and joins.

**Perf expectation**: naive form ≈ minutes (5 × 2M sub-executions); derived-table form ≈ a few seconds (three sequential scans + two hash joins). Requires indexes `orders(user_id, status, created_at)` and `order_items(order_id, product_id)`.

```sql
SELECT
  u.id,
  u.name,
  u.email,
  ostats.order_count,
  ostats.lifetime_spend,
  ostats.avg_order_value,
  ostats.last_order_at,
  COALESCE(pstats.distinct_products, 0) AS distinct_products
FROM users u
INNER JOIN (
  -- Pre-aggregate orders: one row per user, all order metrics at once.
  SELECT
    o.user_id,
    COUNT(*)                        AS order_count,
    SUM(o.total_amount)             AS lifetime_spend,
    ROUND(AVG(o.total_amount), 2)   AS avg_order_value,
    MAX(o.created_at)               AS last_order_at
  FROM orders o
  WHERE o.status = 'completed'
  GROUP BY o.user_id
) ostats ON ostats.user_id = u.id
LEFT JOIN (
  -- Pre-aggregate distinct products per user via a joined derived table.
  SELECT
    o.user_id,
    COUNT(DISTINCT oi.product_id) AS distinct_products
  FROM orders o
  INNER JOIN order_items oi ON oi.order_id = o.id
  WHERE o.status = 'completed'
  GROUP BY o.user_id
) pstats ON pstats.user_id = u.id
WHERE u.status = 'active'
ORDER BY ostats.lifetime_spend DESC
LIMIT 100;
```

```
-- EXPLAIN (ANALYZE) sketch:
Limit  (actual time=3820..3821 rows=100 loops=1)
  ->  Sort  (actual rows=100 loops=1)   Sort Key: ostats.lifetime_spend DESC
        ->  Hash Left Join  (actual rows=1.9M loops=1)
              Hash Cond: (u.id = pstats.user_id)
              ->  Hash Join  (actual rows=1.9M loops=1)
                    Hash Cond: (u.id = ostats.user_id)
                    ->  Seq Scan on users u  Filter: (status = 'active')
                    ->  Hash  (actual rows=1.9M loops=1)
                          ->  HashAggregate  Group Key: o.user_id
                                ->  Index Scan on orders o  (Index Cond: status='completed')
              ->  Hash  (actual rows=1.7M loops=1)
                    ->  HashAggregate  Group Key: o_1.user_id
                          ->  Hash Join (orders ⋈ order_items)
Execution Time: 3821 ms
```

Each child table is scanned **once**; the correlated per-row explosion is gone. This is the canonical "rewrite correlated subqueries as pre-aggregated derived tables" pattern.

---

## 9. Wrong → Right Patterns

### Wrong 1: Scalar subquery that returns multiple rows

```sql
-- WRONG: assumes one email per region, but EU has thousands of users
SELECT o.id,
       (SELECT email FROM users WHERE region = 'EU') AS contact
FROM orders o;
-- ERROR: more than one row returned by a subquery used as an expression
```

**Why it fails at the execution level**: a scalar subquery in the SELECT list is contracted to return ≤1 row. The executor evaluates it and, on seeing a second row, raises immediately. It "worked in dev" only because dev had one EU user.

```sql
-- RIGHT: make the intent explicit — pick a single, deterministic row
SELECT o.id,
       (SELECT email FROM users
         WHERE region = 'EU'
         ORDER BY id
         LIMIT 1) AS contact          -- exactly one row, deterministic
FROM o... ;
-- Or, if you truly want all of them, this is a JOIN, not a scalar subquery.
```

### Wrong 2: NOT IN with a nullable subquery column

```sql
-- WRONG: returns ZERO rows whenever any order has a NULL user_id
SELECT u.id, u.name
FROM users u
WHERE u.id NOT IN (SELECT user_id FROM orders);
```

**Why it fails**: if the subquery yields any NULL, `u.id NOT IN (..., NULL, ...)` is `... AND (u.id <> NULL)` → the `<> NULL` term is UNKNOWN → the conjunction is never TRUE → every row is filtered out. No error, just an empty result. This is three-valued logic (Topic 07).

```sql
-- RIGHT: NOT EXISTS is NULL-safe and plans to an Anti Join
SELECT u.id, u.name
FROM users u
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

### Wrong 3: Correlated scalar subquery per metric (N × M re-execution)

```sql
-- WRONG: three correlated subqueries → 3 SubPlans re-executed per user
SELECT
  u.id,
  (SELECT COUNT(*)          FROM orders o WHERE o.user_id = u.id) AS n,
  (SELECT SUM(total_amount) FROM orders o WHERE o.user_id = u.id) AS spend,
  (SELECT MAX(created_at)   FROM orders o WHERE o.user_id = u.id) AS last_at
FROM users u;
-- Scans orders THREE times per user; at 2M users this is millions of index scans.
```

**Why it's slow**: each subquery is an independent SubPlan; the planner does not merge them. For N users you pay 3N sub-executions.

```sql
-- RIGHT: one pass over orders in a derived table, joined once
SELECT u.id, s.n, s.spend, s.last_at
FROM users u
LEFT JOIN (
  SELECT user_id,
         COUNT(*)          AS n,
         SUM(total_amount) AS spend,
         MAX(created_at)   AS last_at
  FROM orders
  GROUP BY user_id
) s ON s.user_id = u.id;
```

### Wrong 4: Unaliased or mis-scoped derived table

```sql
-- WRONG: derived table without an alias
SELECT * FROM (SELECT user_id, SUM(total_amount) FROM orders GROUP BY user_id);
-- ERROR: subquery in FROM must have an alias
```

**Why it fails**: PostgreSQL requires every FROM-clause subquery to be named so its columns can be referenced.

```sql
-- RIGHT: alias the derived table (and its computed columns)
SELECT d.user_id, d.spend
FROM (SELECT user_id, SUM(total_amount) AS spend
      FROM orders GROUP BY user_id) d;
```

### Wrong 5: IN subquery with the wrong column (silent semantic bug)

```sql
-- WRONG: subquery selects id, but the intent was user_id → matches nothing sensible
SELECT * FROM users u
WHERE u.id IN (SELECT id FROM orders WHERE total_amount > 500);
-- Compares users.id against ORDER ids — coincidental matches, not "users who ordered big".
```

**Why it's wrong**: no error (both are integers), but the semantics are nonsense — you're testing whether a user's id equals some order's id.

```sql
-- RIGHT: select the correlating column
SELECT * FROM users u
WHERE u.id IN (SELECT user_id FROM orders WHERE total_amount > 500);
```

### Wrong 6: Assuming `> ALL (empty set)` is FALSE

```sql
-- WRONG mental model: "no products in category 99, so nothing qualifies"
SELECT * FROM products
WHERE price > ALL (SELECT price FROM products WHERE category_id = 99);
-- If category 99 is empty, > ALL (empty) is TRUE → returns ALL products, surprising the dev.
```

**Why**: `> ALL (empty)` is vacuously TRUE. If you meant "strictly greater than the max of a possibly-empty set and treat empty as no-match," guard it:

```sql
-- RIGHT: be explicit about the empty case
SELECT * FROM products
WHERE price > (SELECT COALESCE(MAX(price), 'Infinity') FROM products WHERE category_id = 99);
-- Empty set → MAX is NULL → COALESCE to +infinity → nothing qualifies (the usual intent).
```

---

## 10. Performance Profile

### 10.1 The cost model by subquery form

| Form | Executions | Cost driver | Scales like |
|------|-----------|-------------|-------------|
| Uncorrelated scalar (InitPlan) | 1 | one aggregate pass | O(inner) once, negligible per outer row |
| Correlated scalar (SubPlan) | N (outer rows) | inner access cost × N | O(N × inner_probe) |
| `IN`/`EXISTS` (semi-join) | 1 join | hash build + probe | O(N + M) |
| `NOT EXISTS` (anti-join) | 1 join | hash build + probe | O(N + M) |
| Derived table (materialized) | 1 | one scan + tuplestore | O(inner) + spill if > work_mem |
| Derived table (pulled up) | 0 extra | folded into parent join | same as hand-written join |

### 10.2 Scaling at 1M / 10M / 100M outer rows

- **Correlated scalar subquery**: the danger form. With a supporting index each probe is O(log inner), so total ≈ N × log(inner). At 1M rows with a good index: seconds. At 100M rows: minutes-to-hours — and if the index is missing, each probe is a full scan and the query never finishes. **Always** check `EXPLAIN` for `SubPlan` + `loops`.
- **Semi/anti-join (`IN`/`EXISTS`/`NOT EXISTS`)**: scales gracefully — a single hash join, O(N + M), memory O(smaller side). This is why converting a correlated subquery into `EXISTS` (or letting the planner do it) is the standard fix. At 100M × 25M it is a big-but-linear hash join, tunable via `work_mem`.
- **Derived-table pre-aggregation**: scales with the child table size, not the product. Aggregating 90M `order_items` once and hash-joining to 2M users is bounded and predictable; the equivalent correlated version is 2M sub-scans of a 90M table.

### 10.3 Memory and work_mem

- A materialized derived table lives in a tuplestore sized by `work_mem`; exceeding it spills to a temp file (visible as `Batches > 1` on the aggregate/hash, or temp read/write in `BUFFERS`). Raise `work_mem` for the session running heavy analytics, not globally.
- Semi/anti-joins build a hash of the inner (usually smaller) relation — same `work_mem` considerations as any Hash Join (Topic 11).

### 10.4 Index interactions

- **Correlated subquery**: index the correlation column(s), and include the aggregated/ordered column for index-only or ordered access — e.g., `orders(user_id, created_at DESC)` powers both `WHERE user_id = ? ORDER BY created_at DESC LIMIT 1` and `COUNT(*) WHERE user_id = ?`.
- **`IN`/`EXISTS` semi-join**: the planner may hash the inner or probe an index; an index on the inner join column enables a nested-loop semi-join when the outer side is tiny.
- **Derived-table aggregation**: an index matching the `GROUP BY` key can enable a cheaper `GroupAggregate` (sorted) instead of `HashAggregate`, and helps when only a filtered slice is aggregated.

### 10.5 Optimization playbook

1. See `SubPlan` with big `loops`? Rewrite the correlated subquery as a derived-table join or a window function.
2. Multiple correlated subqueries hitting the same child? Collapse into one pre-aggregated derived table.
3. `NOT IN`? Switch to `NOT EXISTS` — correctness first, then it also plans better.
4. Derived table showing `Subquery Scan` + spill? Check whether `LIMIT`/`DISTINCT` is blocking pull-up; if the fence is unintended, remove it. Raise `work_mem` if the aggregate legitimately large.
5. Uncorrelated scalar recomputed inside a loop by your app code? Push it into the SQL so it becomes a single InitPlan.

---

## 11. Node.js Integration

### 11.1 Uncorrelated scalar subquery with pg

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Products above the catalog average — the subquery runs once (InitPlan) in the DB,
// not in Node. Never fetch all products to compute the average in JS.
async function productsAboveAverage() {
  const { rows } = await pool.query(
    `SELECT id, name, price
       FROM products
      WHERE price > (SELECT AVG(price) FROM products)
      ORDER BY price DESC`
  );
  return rows;
}
```

### 11.2 Parameterised IN-subquery (semi-join) with $1

```javascript
// Users who placed a high-value order since a given date.
// The threshold and date are bound params; the subquery is a semi-join in the plan.
async function bigSpendersSince(minAmount, sinceISO) {
  const { rows } = await pool.query(
    `SELECT u.id, u.name, u.email
       FROM users u
      WHERE u.status = 'active'
        AND u.id IN (
          SELECT o.user_id
            FROM orders o
           WHERE o.total_amount > $1
             AND o.created_at >= $2
        )
      ORDER BY u.name`,
    [minAmount, sinceISO]
  );
  return rows;
}
```

### 11.3 Correlated scalar subquery — and when to avoid it

```javascript
// Correlated scalar subquery: last order amount per user.
// Fine for a single user or a small page; index orders(user_id, created_at DESC).
async function lastOrderAmount(userId) {
  const { rows } = await pool.query(
    `SELECT
       u.id,
       u.name,
       (SELECT o.total_amount
          FROM orders o
         WHERE o.user_id = u.id
         ORDER BY o.created_at DESC
         LIMIT 1) AS last_order_amount
     FROM users u
     WHERE u.id = $1`,
    [userId]
  );
  return rows[0] ?? null;
}

// For a WHOLE dashboard, do NOT run this across all users (per-row SubPlan).
// Use the pre-aggregated derived-table version instead:
async function dashboard(limit = 100) {
  const { rows } = await pool.query(
    `SELECT u.id, u.name, s.order_count, s.lifetime_spend, s.last_order_at
       FROM users u
       LEFT JOIN (
         SELECT user_id,
                COUNT(*)        AS order_count,
                SUM(total_amount) AS lifetime_spend,
                MAX(created_at) AS last_order_at
           FROM orders
          WHERE status = 'completed'
          GROUP BY user_id
       ) s ON s.user_id = u.id
      WHERE u.status = 'active'
      ORDER BY s.lifetime_spend DESC NULLS LAST
      LIMIT $1`,
    [limit]
  );
  return rows;
}
```

### 11.4 Passing an array to an IN — prefer ANY($1) over building a subquery string

```javascript
// When the "set" comes from the app (not another table), use = ANY($1::int[]).
// This avoids SQL injection and avoids constructing an IN-list string.
async function usersByIds(ids) {
  const { rows } = await pool.query(
    `SELECT id, name FROM users WHERE id = ANY($1::int[])`,
    [ids]                       // ids is a JS number[]
  );
  return rows;
}
```

**ORM note**: Every major ORM can emit `IN (SELECT ...)` / `EXISTS`, but complex correlated subqueries, quantified comparisons (`> ALL`), and derived-table pre-aggregation usually require a raw-SQL escape hatch. Use the query builder for simple membership; drop to raw SQL for analytics-grade subqueries.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do subqueries?** — Partially. Relation filters (`some`, `none`, `every`) compile to correlated `EXISTS`/`NOT EXISTS` subqueries; true scalar/derived-table subqueries need raw SQL.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// `some` → EXISTS subquery; `none` → NOT EXISTS (NULL-safe anti-join)
const bigSpenders = await prisma.user.findMany({
  where: {
    status: 'active',
    orders: { some: { totalAmount: { gt: 500 } } },   // EXISTS (...)
  },
  select: { id: true, name: true },
});

const neverOrdered = await prisma.user.findMany({
  where: { orders: { none: {} } },                     // NOT EXISTS (...)
});

// Scalar subquery / derived-table aggregation → raw SQL
const aboveAvg = await prisma.$queryRaw`
  SELECT id, name, price FROM products
  WHERE price > (SELECT AVG(price) FROM products)
`;
```

**Where it breaks**: no scalar subquery in the SELECT list, no derived tables, no `ANY`/`ALL`. **Verdict**: excellent for `EXISTS`-style relation filters; raw SQL for everything else.

---

### Drizzle ORM

**Can Drizzle do subqueries?** — Yes, first-class. `db.select()...as('sub')` produces a real subquery object usable in FROM, and `inArray`/`exists`/`sql` cover the rest.

```typescript
import { db } from './db';
import { users, orders } from './schema';
import { eq, gt, inArray, exists, sql } from 'drizzle-orm';

// IN (SELECT ...) semi-join
const buyers = await db.select({ id: users.id, name: users.name })
  .from(users)
  .where(inArray(users.id,
    db.select({ id: orders.userId }).from(orders).where(gt(orders.totalAmount, 500))
  ));

// EXISTS correlated subquery
const active = await db.select().from(users).where(
  exists(db.select({ x: sql`1` }).from(orders).where(eq(orders.userId, users.id)))
);

// Derived table in FROM
const sub = db.select({
    userId: orders.userId,
    spend: sql<number>`sum(${orders.totalAmount})`.as('spend'),
  }).from(orders).groupBy(orders.userId).as('sub');

const joined = await db.select({ id: users.id, spend: sub.spend })
  .from(users).innerJoin(sub, eq(sub.userId, users.id));
```

**Where it breaks**: `ANY`/`ALL` need the `sql` tag. **Verdict**: the best typed subquery support of any Node ORM — derived tables and correlated subqueries both expressible.

---

### Sequelize

**Can Sequelize do subqueries?** — Yes, but awkwardly, via `Sequelize.literal()` and `where` sub-conditions; derived tables need raw SQL.

```javascript
const { Sequelize, Op } = require('sequelize');

// Correlated scalar subquery in the attribute list via literal()
const users = await User.findAll({
  attributes: [
    'id', 'name',
    [Sequelize.literal(
      '(SELECT COUNT(*) FROM orders o WHERE o.user_id = "User".id)'
    ), 'orderCount'],
  ],
});

// IN (subquery) via literal
const buyers = await User.findAll({
  where: {
    id: { [Op.in]: Sequelize.literal(
      '(SELECT user_id FROM orders WHERE total_amount > 500)'
    ) },
  },
});
```

**Where it breaks**: anything non-trivial becomes a hand-written `literal()` string — no type safety, injection risk if you interpolate. **Verdict**: capable but stringly-typed; use `sequelize.query()` for real analytics.

---

### TypeORM

**Can TypeORM do subqueries?** — Yes, via the QueryBuilder's subquery API (`.subQuery()`, `getQuery()`, `.where(qb => ...)`).

```typescript
const buyers = await dataSource.getRepository(User)
  .createQueryBuilder('u')
  .where(qb => {
    const sub = qb.subQuery()
      .select('o.user_id')
      .from('orders', 'o')
      .where('o.total_amount > :amt', { amt: 500 })
      .getQuery();
    return 'u.id IN ' + sub;        // IN (subquery)
  })
  .setParameter('amt', 500)
  .getMany();

// Correlated scalar in SELECT
const withCounts = await dataSource.getRepository(User)
  .createQueryBuilder('u')
  .addSelect(
    '(SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id)',
    'u_orderCount'
  )
  .getRawAndEntities();
```

**Where it breaks**: parameter plumbing across subquery boundaries is fiddly; correlated scalar subqueries are raw strings. **Verdict**: workable QueryBuilder subquery support; verbose but explicit.

---

### Knex.js

**Can Knex do subqueries?** — Yes, cleanly. A `knex` builder can be passed anywhere a subquery is expected.

```javascript
// IN (subquery)
const buyers = await knex('users')
  .whereIn('id', knex('orders').select('user_id').where('total_amount', '>', 500))
  .select('id', 'name');

// EXISTS
const active = await knex('users as u')
  .whereExists(function () {
    this.select(1).from('orders as o').whereRaw('o.user_id = u.id');
  });

// Derived table in FROM
const sub = knex('orders')
  .select('user_id')
  .sum({ spend: 'total_amount' })
  .groupBy('user_id')
  .as('s');

const joined = await knex('users as u')
  .join(sub, 's.user_id', 'u.id')
  .select('u.id', 's.spend');

// Correlated scalar in SELECT via knex.raw
const withCount = await knex('users as u')
  .select('u.id',
    knex.raw('(SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS order_count'));
```

**Where it breaks**: correlated ON-clause correlation uses `whereRaw`; quantified `ALL`/`ANY` need `raw`. **Verdict**: the most SQL-transparent — subqueries compose naturally as builders.

---

### ORM summary table

| ORM | IN/EXISTS subquery | Scalar in SELECT | Derived table (FROM) | ANY/ALL | Verdict |
|-----|-------------------|------------------|----------------------|---------|---------|
| Prisma | `some`/`none`/`every` | raw SQL | raw SQL | raw SQL | Great for relation-filter EXISTS |
| Drizzle | `inArray`/`exists` | `sql` subquery | `.as()` subquery | `sql` | Best typed support |
| Sequelize | `Op.in` + `literal` | `literal()` | raw SQL | raw SQL | Stringly-typed, use raw |
| TypeORM | `.subQuery()` | `addSelect` raw | QueryBuilder | raw | Verbose but explicit |
| Knex | `whereIn`/`whereExists` | `knex.raw` | builder `.as()` | `raw` | Most transparent |

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given:
- `products(id, name, price, category_id)`
- `categories(id, name)`

Write a query returning every product whose `price` is **strictly greater than the average price of all products**. Return `name`, `price`, and the overall average as a column `catalog_avg`. Order by `price` descending.

Then answer: how many times does the subquery `(SELECT AVG(price) FROM products)` execute, and why?

```sql
-- Write your query here
```

---

### Exercise 2 — Intermediate (combines Topics 11 & 20)

Given:
- `users(id, name, status)`
- `orders(id, user_id, total_amount, status, created_at)`

Using a **derived table** (subquery in FROM), return for each active user their `order_count` and `total_spend` over completed orders only, but include **only users whose total_spend exceeds 1000**. Include users' names via a join. Order by `total_spend` descending.

Then: rewrite the same result using **correlated scalar subqueries** in the SELECT list instead of a derived table, and explain which version scales better on 5M users and why.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation, naive answer is wrong)

Given:
- `users(id, name)`
- `orders(id, user_id, ...)` where `user_id` **is nullable** (some orders were imported without a user link)

A teammate wrote this to find "users who have never placed an order":

```sql
SELECT id, name FROM users
WHERE id NOT IN (SELECT user_id FROM orders);
```

The report comes back **empty**, even though you know some users never ordered.

1. Explain precisely why it returns zero rows.
2. Write two correct versions: one using `NOT EXISTS`, and one that keeps `NOT IN` but is made safe.
3. State which of your two versions plans to an Anti Join and is NULL-safe by construction.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

Given:
- `employees(id, name, salary, department_id)`
- `departments(id, name)`

Write a query returning every employee who earns **more than the average salary of their own department** (a per-department comparison, not the company-wide average). Return `name`, `salary`, `department name`, and the department average as `dept_avg`.

Follow-ups to be ready for:
- Is your subquery correlated or uncorrelated? How does that change the plan?
- What index would make it fast on a 10M-row `employees` table?
- How would you rewrite it without a correlated subquery (hint: window function, Phase 6)?

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What is the difference between a correlated and an uncorrelated subquery?**

A: An **uncorrelated** subquery is self-contained — it references no columns from the outer query, so it could run on its own and is executed once (its result is reused). An example is `WHERE price > (SELECT AVG(price) FROM products)`. A **correlated** subquery references a column from the outer query (a correlation variable), so it cannot run independently; conceptually it re-executes once per outer row, like `(SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id)`. The practical consequence is performance: uncorrelated runs once; correlated runs N times unless the planner rewrites it into a join.

---

**Q: Where can a subquery appear in a SELECT statement?**

A: In four main places: (1) the **SELECT list**, where it must be scalar (one row, one column); (2) the **FROM clause** as a derived table (must be aliased, can return many rows/columns); (3) the **WHERE clause** as a scalar comparison, or with `IN`/`EXISTS`/`ANY`/`ALL`; and (4) the **HAVING clause** to compare a group's aggregate against another query's result. The position dictates the cardinality contract — SELECT-list and comparison subqueries must be scalar, FROM and `IN`/`EXISTS` may return sets.

---

**Q: Why is `NOT IN (SELECT ...)` considered dangerous?**

A: Because of NULLs. If the subquery returns even one NULL value, `x NOT IN (..., NULL)` expands to a conjunction containing `x <> NULL`, which evaluates to UNKNOWN, making the whole predicate UNKNOWN for every row — so the query silently returns zero rows. It's a correctness bug with no error message. The fix is `NOT EXISTS`, which tests row existence rather than value equality and is NULL-safe, or filtering the NULLs out of the subquery explicitly.

---

### Principal Level

**Q: You see a query with a correlated scalar subquery in the SELECT list running slowly on a large table. Walk me through diagnosing and fixing it.**

A: First, `EXPLAIN (ANALYZE, BUFFERS)`. The signature of the problem is a `SubPlan` node with `loops` equal to the outer row count — the subquery is re-executing per row. I check the inner access method: if it's a `Seq Scan`, the correlation column has no index and every one of those N loops is a full scan — that's the catastrophic case, and adding an index on the correlation column (e.g., `orders(user_id, created_at)`) turns each loop into a cheap index probe, often a 100×+ improvement. If the index exists and it's still slow because N itself is huge, or because there are several correlated subqueries over the same child table, the right fix is structural: rewrite as a **pre-aggregated derived table** joined once — `LEFT JOIN (SELECT user_id, COUNT(*), SUM(...), MAX(...) FROM orders GROUP BY user_id) s ON s.user_id = u.id`. That converts N per-row sub-executions into a single scan-and-aggregate plus one hash join, O(N + M) instead of O(N × probe). Where the metric is rank/running-total shaped, a window function is even cleaner. I verify by re-running EXPLAIN and confirming the `SubPlan` is gone, replaced by a `HashAggregate` + `Hash Join`.

---

**Q: Explain subquery flattening. Which subqueries does PostgreSQL rewrite, and which act as optimization fences?**

A: Flattening (pull-up) is the planner's `pull_up_subqueries` pass merging a subquery into the parent so the whole statement plans as one flat join problem, freeing it to reorder joins and push predicates. PostgreSQL flattens: simple derived tables in FROM (pure projection/filter/join) into the parent join tree; `IN (SELECT ...)` and `= ANY (SELECT ...)` into **semi-joins**; `EXISTS` into a semi-join; `NOT EXISTS` and NULL-safe `NOT IN` into **anti-joins**; and uncorrelated scalar subqueries into **InitPlans** run once. It does **not** flatten across a `LIMIT`/`OFFSET`, `DISTINCT ON`, set operations (`UNION` etc.), volatile functions, or certain aggregate placements — these become genuine `Subquery Scan` nodes that act as optimization fences (the planner can't push predicates through them). This matters because it tells you which "subqueries" are pure syntax the planner erases, versus which impose a real plan boundary you must reason about — and, conversely, how to deliberately fence the planner (a `LIMIT`, `OFFSET 0`, or a materialized CTE) when you want to force evaluation order.

---

**Q: A `NOT IN` subquery and a `NOT EXISTS` subquery look equivalent. When are they NOT equivalent, and which do you default to?**

A: They diverge on NULLs, in two directions. First, if the **subquery** produces a NULL, `NOT IN` returns zero rows (the UNKNOWN-propagation trap) while `NOT EXISTS` is unaffected — it tests existence. Second, if the **outer** value is NULL, `NULL NOT IN (non-empty set)` is UNKNOWN (row excluded) whereas `NOT EXISTS` with a join predicate `o.user_id = u.id` treats a NULL outer key as simply not matching, keeping it. So they can give different results whenever NULLs are present on either side. I default to `NOT EXISTS`: it's NULL-safe by construction, it plans to an Anti Join that scales linearly, and its intent — "no matching row exists" — is clearer. I only use `NOT IN` when the subquery column is provably NOT NULL (e.g., a primary key) and even then I lean toward `NOT EXISTS` for consistency.

---

## 15. Mental Model Checkpoint

1. **A scalar subquery in the SELECT list references no outer column. How many times does it execute for a 1-million-row outer query, and what is that node called in EXPLAIN?**

2. **You write `WHERE price > ALL (SELECT price FROM products WHERE category_id = 999)` and category 999 has no products. Which rows come back, and why?**

3. **Two queries return the same users: one uses `INNER JOIN orders + DISTINCT`, the other uses `WHERE EXISTS (SELECT 1 FROM orders ...)`. What's the difference in how the engine produces the result, and which avoids row multiplication?**

4. **A correlated subquery's inner plan shows `Seq Scan` with `loops=2000000`. What single change most likely fixes the performance, and what does the plan look like afterward?**

5. **You put a `LIMIT 10` inside a derived table and notice the planner no longer pushes your outer WHERE filter into it. Why? What property of the subquery changed?**

6. **`SELECT id FROM users WHERE id NOT IN (SELECT manager_id FROM users)` returns nothing on a table where some `manager_id` values are NULL. Explain the mechanism precisely.**

7. **When would you deliberately choose a materialized subquery (an optimization fence) over letting the planner flatten it?**

---

## 16. Quick Reference Card

```sql
-- SHAPES
(SELECT AVG(x) FROM t)              -- scalar: 1 row × 1 col (SELECT list, =, >, <)
(SELECT a, b FROM t WHERE id=1)     -- row:    1 row × N cols  → (x,y) = (SELECT a,b ...)
(SELECT x FROM t WHERE ...)         -- table:  M rows × 1 col   → IN / EXISTS / ANY / ALL
(SELECT ... GROUP BY ...) AS d      -- derived table in FROM (MUST be aliased)

-- DEPENDENCY
uncorrelated → references no outer col → runs ONCE (InitPlan / materialized)
correlated   → references outer col    → runs per-row (SubPlan) unless flattened

-- POSITIONS
SELECT (SELECT ...) AS c            -- scalar only
FROM   (SELECT ...) AS d            -- derived table, alias required
WHERE  x IN (SELECT ...)            -- semi-join
WHERE  EXISTS (SELECT 1 ...)        -- semi-join, NULL-safe
HAVING SUM(x) > (SELECT ...)        -- per-group vs scalar

-- THE NOT IN NULL TRAP  (any NULL in subquery → zero rows)
WHERE x NOT IN (SELECT c FROM t)              -- DANGEROUS if c nullable
WHERE NOT EXISTS (SELECT 1 FROM t WHERE t.c = x)   -- SAFE, plans to Anti Join

-- QUANTIFIERS
x = ANY (...)   ≡  x IN (...)
x <> ALL (...)  ≡  x NOT IN (...)            -- same NULL trap
x > ALL (...)   ≡  x > (SELECT MAX(...))
x > ALL (empty) = TRUE    (vacuous)          -- classic trap
x > ANY (empty) = FALSE

-- FLATTENING (planner rewrites these away)
IN/EXISTS      → Semi Join
NOT EXISTS     → Anti Join
scalar (uncorr)→ InitPlan (once)
simple FROM sub→ pulled up into parent join
-- BLOCKED by: LIMIT/OFFSET, DISTINCT ON, UNION, volatile funcs, some aggregates

-- PERFORMANCE RULES OF THUMB
-- correlated scalar in SELECT/WHERE → SubPlan loops=N → index the correlation col
-- many correlated subqueries, same child → collapse into ONE derived table
-- NOT IN → prefer NOT EXISTS (correctness + Anti Join)
-- attaching aggregates per row → LEFT JOIN pre-aggregated derived table, not N subqueries

-- EXPLAIN SIGNALS
-- InitPlan (returns $0)   → uncorrelated scalar, once     (ideal)
-- SubPlan + high loops    → correlated, per-row           (index or rewrite)
-- Hash Semi Join          → IN/EXISTS flattened           (good)
-- Hash Anti Join          → NOT EXISTS                    (good, NULL-safe)
-- Subquery Scan on d      → derived table not pulled up   (check LIMIT/DISTINCT/agg)

-- INTERVIEW ONE-LINERS
-- "Scalar = one row one column; a second row is a runtime error."
-- "Uncorrelated runs once (InitPlan); correlated runs per row (SubPlan)."
-- "IN/EXISTS become semi-joins; NOT EXISTS becomes an anti-join."
-- "NOT IN + any NULL = zero rows. Use NOT EXISTS."
-- "> ALL of an empty set is TRUE; > ANY of an empty set is FALSE."
-- "Rewrite N correlated subqueries as one pre-aggregated derived table."
```

---

## Connected Topics

- **Topic 07 — NULL in Depth**: three-valued logic is the root of the `NOT IN` trap, scalar-vs-NULL comparisons yielding UNKNOWN, and quantifier NULL propagation.
- **Topic 11 — INNER JOIN in Depth**: subqueries and joins are interchangeable for many patterns; flattening rewrites `IN`/`EXISTS` into the very semi-joins built on join internals; pre-aggregated derived tables cure the fan-out bug from that topic.
- **Topic 20 — GROUP BY Fundamentals**: derived tables exist largely to pre-aggregate before joining; HAVING subqueries compare group aggregates.
- **Topic 25 — Aggregate Performance** (previous): the cost of the aggregate inside a scalar/derived subquery is exactly the cost analysed there; InitPlan vs per-row aggregation is the crux.
- **Topic 27 — EXISTS and NOT EXISTS** (next): the deep dive into the semi-join/anti-join subquery forms introduced here, including why `EXISTS` beats `IN` and `NOT IN` on correctness and speed.
- **Topic 28 — LATERAL Joins**: the correlated derived table — a FROM-clause subquery allowed to reference earlier FROM items, merging the derived-table and correlated-subquery ideas.
- **Topic 29 — Common Table Expressions (CTEs)**: the readable, named alternative to nested subqueries, with explicit control over the materialization fence that subqueries leave to the planner.
- **Internals — Planner (subquery_planner, pull_up_subqueries)**: the flattening machinery that decides whether your subquery becomes a join, an InitPlan, a SubPlan, or a Subquery Scan.
