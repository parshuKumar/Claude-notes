# Topic 31 — Lateral Joins

### SQL Mastery Curriculum — Phase 5: Subqueries and CTEs

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a bookstore with a big binder of customers. For each customer, your manager asks you: "Go pull this person's **three most recent purchases** and staple them to their page."

Here is the important part: to answer "the three most recent purchases," you have to know **which customer you are looking at first**. You cannot pull "the three most recent purchases" in the abstract — the answer depends on the specific customer's name written at the top of the page. You flip to page 1 (Alice), then walk to the purchases shelf and grab *Alice's* top three. Then you flip to page 2 (Bob), walk back to the shelf, and grab *Bob's* top three. And so on, one customer at a time.

That "for each row on the left, go run a little query that is allowed to look at that row's values" is **exactly** what a LATERAL join is.

Now contrast this with an ordinary join, which is more like: "Here are all customers. Here are all purchases. Match them up by customer ID." In an ordinary join, the two lists are prepared **independently** — the purchases list has no idea which customer you are currently standing in front of. The subquery on the right side of a normal `JOIN` is sealed off; it cannot peek at the row on the left.

LATERAL breaks that seal. It lets the right-hand subquery **reach back and reference columns from the tables listed before it** in the `FROM` clause. That single capability — "the thing on the right can see the row on the left" — is the whole idea. Everything else in this document is a consequence of it.

The mental one-liner: **LATERAL is a `for` loop written in the `FROM` clause.** For each row the left side produces, the right side runs once, with that row's values available as parameters.

---

## 2. Connection to SQL Internals

At the engine level, a LATERAL join is almost always executed as a **Nested Loop Join with a parameterized inner side**. This is not an accident — it is the *only* general way to evaluate it, because the inner subquery's result genuinely changes per outer row.

Recall from Topic 11 how PostgreSQL implements joins with three algorithms: Nested Loop, Hash Join, and Merge Join. Hash Join and Merge Join both require the inner relation to be materializable **independently** of the outer row — you build one hash table, or sort one stream, and reuse it for every outer row. A LATERAL subquery that references the outer row cannot be pre-built that way, because its output depends on `outer.id`, `outer.created_at`, etc. So the planner falls back to the one algorithm that re-evaluates the inner side per outer row: **Nested Loop**.

Internally, the mechanism is PostgreSQL's **parameterized path**. The outer row's referenced columns become runtime parameters (you will see them in EXPLAIN as `$0`, `$1`, ...). The inner plan is a normal plan tree, but with those parameters plugged into its scan conditions. If there is a suitable **B-tree index** on the inner table matching the correlated predicate, each iteration becomes an **Index Scan** with `Index Cond: (x = $0)` — cheap, O(log N) per outer row. If there is no index, each iteration is a **Seq Scan** of the inner table, and the cost explodes to O(outer × inner) — the same catastrophic pattern we saw with un-indexed nested loops in Topic 11.

Other internals in play:

- **Buffer pool / heap pages**: because the inner side is probed once per outer row, the same inner index and heap pages are hit repeatedly. They usually stay resident in `shared_buffers` after the first few iterations, so you will see high `shared hit` and low `shared read` in `BUFFERS` output. This is why an indexed LATERAL over a warm cache is fast despite "looping."
- **The planner's `LIMIT` pushdown**: LATERAL's killer feature is top-N-per-group. When the inner subquery has `ORDER BY ... LIMIT k`, the planner can stop each inner scan after k rows using an ordered index. This is a bounded amount of work per outer row — the engine never materializes the full inner set. A window-function equivalent (Topic 32) must often sort or scan the *entire* partitioned set first.
- **Set-returning functions (SRFs)**: functions like `unnest()`, `jsonb_array_elements()`, `generate_series()`, and `regexp_matches()` are SRFs. When placed in `FROM` with LATERAL, the executor invoked them once per outer row via a **FunctionScan** node driven by the nested loop. This is the canonical way to "explode" an array column into rows.
- **MVCC**: nothing special — each inner iteration sees the same snapshot as the surrounding query. There is no per-iteration transaction; visibility is consistent across the whole nested loop.

The key internal takeaway: **LATERAL = parameterized Nested Loop.** Its performance is entirely governed by (a) how many outer rows drive the loop and (b) whether each inner iteration can use an index.

---

## 3. Logical Execution Order Context

```
FROM  table_a
      , LATERAL (subquery referencing a)   ← evaluated DURING the FROM phase,
                                              AFTER table_a rows are available
WHERE ...                                   ← applied after the lateral join
GROUP BY ...
HAVING ...
SELECT ...
DISTINCT ...
window functions
ORDER BY ...
LIMIT ...
```

The defining fact about LATERAL's position in logical execution order:

- It lives in the **FROM/JOIN phase** — the very first phase, the same phase where all joins are resolved (Topic 11, Section 3). This is *why* the lateral subquery can see `table_a`: `table_a` is produced first (or conceptually "to the left"), and the lateral subquery is evaluated with each of its rows in scope.
- **Left-to-right dependency is real here.** In an ordinary join, `A JOIN B` is commutative — the planner may reorder them (Topic 11, Section 6.5). With LATERAL, if the right side references `A`, then `A` **must** be evaluated to produce the rows that drive the right side. The lateral subquery is not commutative with the table it references. It *is* free to be reordered relative to tables it does **not** reference.
- Because LATERAL runs in the FROM phase, its subquery **cannot** reference the outer query's `SELECT`-list aliases, `GROUP BY` results, or window functions — those are computed in later phases. It can only reference the *table columns* of preceding FROM items.
- The `WHERE` of the **outer** query runs *after* the lateral join produces its combined rows. But the `WHERE`/`LIMIT` *inside* the lateral subquery runs *per outer row*, during the join. This distinction is the entire performance story: filtering inside the lateral subquery (especially `LIMIT`) prunes work per iteration; filtering in the outer `WHERE` prunes only after the rows already exist.

Compare to Topic 30's subquery-vs-CTE-vs-JOIN discussion: a scalar subquery in the `SELECT` list also runs "per row," but it can only return **one row, one column**. A LATERAL subquery in `FROM` runs per row too, but can return **many rows and many columns** — it is the generalization of the correlated subquery into the FROM clause.

---

## 4. What Is a LATERAL Join?

A **LATERAL join** is a join whose right-hand side is a subquery (or set-returning function) that is permitted to reference columns from the tables appearing to its left in the same `FROM` clause. For each row produced by the left side, the right-hand subquery is evaluated with that row's values available as inputs, and the results are joined back to that row.

Without `LATERAL`, a subquery in `FROM` is a sealed, self-contained relation: referencing an outer column raises `ERROR: invalid reference to FROM-clause entry`. `LATERAL` removes that seal.

```sql
SELECT  a.id, top.product_id, top.total
FROM    users u
CROSS JOIN LATERAL (
          SELECT o.id AS order_id, o.total
          FROM   orders o
          WHERE  o.customer_id = u.id      -- ← references u, only legal because of LATERAL
          ORDER  BY o.created_at DESC
          LIMIT  3
        ) top;
```

### Annotated Syntax Breakdown

```sql
SELECT  u.id, recent.order_id, recent.total
FROM    users u                             -- the LEFT / driving relation
│                                           │  └── produces the rows the loop iterates over
│
CROSS JOIN LATERAL (                        -- CROSS JOIN LATERAL: keep an outer row ONLY if
│   │        │                              │  the subquery returns ≥1 row (inner-join-like)
│   │        └── LATERAL keyword: unseals the subquery so it may
│   │            reference columns of `users u` above
│   └── CROSS JOIN: no ON clause; pairing is driven entirely by the
│       correlation inside the subquery
        SELECT o.id AS order_id, o.total    -- inner projection: may return MANY rows, MANY cols
        FROM   orders o                     -- inner relation, re-scanned per outer row
        WHERE  o.customer_id = u.id         -- the CORRELATION: binds inner to the current u row
        │                    │
        │                    └── u.id is a runtime parameter ($0) supplied per outer iteration
        ORDER  BY o.created_at DESC          -- per-row ordering...
        LIMIT  3                             -- ...bounded: top-3 per user. Engine stops after 3.
) recent;                                    -- mandatory alias for the lateral subquery
```

Two join flavors exist (Section 6.3 covers them fully):

```sql
FROM a CROSS JOIN LATERAL (subquery) s        -- drop outer row if subquery is empty
FROM a LEFT  JOIN LATERAL (subquery) s ON true -- keep outer row even if subquery is empty (NULLs on right)
```

The `ON true` on `LEFT JOIN LATERAL` is idiomatic: the correlation already lives *inside* the subquery, so there is nothing left to put in `ON`, but `LEFT JOIN` syntactically requires an `ON` (or `USING`). `ON true` means "always structurally join; the real matching is inside."

Note: for a **set-returning function**, `LATERAL` is *implied* and may be omitted — `FROM users u, unnest(u.tag_ids) t` works without the keyword. For a **subquery**, `LATERAL` is required.

---

## 5. Why LATERAL Mastery Matters in Production

1. **Top-N-per-group without window-function overhead.** "The 3 most recent orders per customer," "the top 5 products per category," "the latest status per device." The window-function approach (`ROW_NUMBER() OVER (PARTITION BY ... ORDER BY ...)` then filter `<= N`) computes a rank for **every** row and then throws most of them away. LATERAL with `LIMIT N` and a matching index touches only N rows per group. On large tables this is the difference between a 50ms query and a 30-second query. This is the single most common reason principal engineers reach for LATERAL.

2. **Calling set-returning functions per row.** Expanding a JSON array column, unnesting an array, running `regexp_matches` on a text column, generating a series bounded by a per-row value — all of these need the function to see the current row. Without LATERAL you resort to awkward multi-step CTEs or application-side loops (the N+1 pattern from Topic 18).

3. **Correlated aggregates that a scalar subquery cannot express.** A scalar subquery returns one column. When you need several correlated aggregates computed together per row (a running summary object, multiple stats, a whole related sub-row), LATERAL returns them in one pass instead of N separate correlated subqueries — each of which would re-scan the inner table.

4. **Readable, composable query shape.** LATERAL lets you write the per-row logic as a normal, self-contained subquery instead of contorting a window function or a `GROUP BY` with `DISTINCT ON`. Reviewers can read it top to bottom.

5. **The thing that breaks without it.** Developers who do not know LATERAL solve top-N-per-group in the application: fetch all customers, then loop and issue one query per customer. That is the N+1 query problem (Topic 18) — hundreds of round trips instead of one. Or they use a window function on a 100M-row table and wonder why the query needs 4GB of `work_mem` and spills to disk. Knowing LATERAL turns both of those into a single, index-driven statement.

---

## 6. Deep Technical Content

### 6.1 The Core Semantics — "For Each Row, Run This"

The precise evaluation model:

```
result = empty
for each row L in (left relation):
    inner_rows = evaluate(subquery, with L's columns bound as parameters)
    for each row R in inner_rows:
        emit (L, R)          -- CROSS JOIN LATERAL
    if inner_rows is empty and join is LEFT JOIN LATERAL:
        emit (L, NULLs)      -- preserve L with NULL-extended right side
```

That is the complete definition. Everything else is a special case:

- If the subquery returns **0 rows** and it is `CROSS JOIN LATERAL`, the outer row is **dropped** (behaves like INNER JOIN).
- If the subquery returns **0 rows** and it is `LEFT JOIN LATERAL ... ON true`, the outer row is **kept** with NULLs on the right (behaves like LEFT JOIN).
- If the subquery returns **k rows**, the outer row is **multiplied** into k output rows — the same fan-out we studied in Topic 11, Section 6.2.
- The subquery may reference **any** column of **any** preceding FROM item, not just the immediately preceding one.

### 6.2 Why the Seal Exists — the Non-LATERAL Error

```sql
-- This FAILS:
SELECT u.id, o.total
FROM   users u,
       (SELECT total FROM orders WHERE customer_id = u.id) o;   -- u.id not visible
-- ERROR:  invalid reference to FROM-clause entry for table "u"
-- HINT:   There is an entry for table "u", but it cannot be referenced from this part of the query.
```

By default, each item in the `FROM` list is evaluated as an independent relation; they only become correlated through join `ON` conditions and the `WHERE` clause — *after* each is independently materialized. A subquery in `FROM` is one such independent relation, so it cannot see its siblings. `LATERAL` is the explicit opt-in that says "evaluate me *in the context of* the items to my left."

```sql
-- This SUCCEEDS — only difference is the LATERAL keyword:
SELECT u.id, o.total
FROM   users u,
       LATERAL (SELECT total FROM orders WHERE customer_id = u.id) o;
```

The comma form (`FROM a, LATERAL (...)`) is equivalent to `CROSS JOIN LATERAL`. Prefer the explicit `CROSS JOIN LATERAL` / `LEFT JOIN LATERAL` spelling in production code for clarity.

### 6.3 CROSS JOIN LATERAL vs LEFT JOIN LATERAL

This is the most important behavioral distinction, exactly parallel to INNER vs LEFT JOIN (Topic 12).

```sql
-- CROSS JOIN LATERAL: customers who have ZERO orders DISAPPEAR
SELECT u.id, u.name, recent.order_id
FROM   users u
CROSS JOIN LATERAL (
    SELECT o.id AS order_id
    FROM   orders o
    WHERE  o.customer_id = u.id
    ORDER  BY o.created_at DESC
    LIMIT  3
) recent;
-- A user with no orders → subquery returns 0 rows → user dropped entirely
```

```sql
-- LEFT JOIN LATERAL: customers with ZERO orders are KEPT, with NULLs
SELECT u.id, u.name, recent.order_id
FROM   users u
LEFT JOIN LATERAL (
    SELECT o.id AS order_id
    FROM   orders o
    WHERE  o.customer_id = u.id
    ORDER  BY o.created_at DESC
    LIMIT  3
) recent ON true;
-- A user with no orders → subquery returns 0 rows → user kept, recent.order_id IS NULL
```

**The gotcha**: developers reach for `CROSS JOIN LATERAL` because it reads cleanly, then silently lose all rows whose subquery is empty. If you are building a report that must list *every* user (including zero-order users), you need `LEFT JOIN LATERAL ... ON true`. This mirrors the classic INNER-vs-LEFT reporting bug: an inner-style join hides the rows with no related data.

The `ON true` is mandatory syntax for `LEFT JOIN LATERAL`. You *can* put an additional filter there, e.g. `ON recent.total > 100`, but any predicate that references the outer row belongs *inside* the subquery for performance (so it can prune before `LIMIT`). Putting a filter in the `ON` of a `LEFT JOIN LATERAL` also has the usual outer-join subtlety from Topic 12: a false `ON` predicate does not remove the outer row, it just NULL-extends it.

### 6.4 Top-N-Per-Group — the Flagship Use Case

The canonical problem: for each category, return its 5 best-selling products.

```sql
SELECT  c.id AS category_id, c.name AS category_name,
        top.product_id, top.name AS product_name, top.units_sold
FROM    categories c
CROSS JOIN LATERAL (
    SELECT  p.id AS product_id, p.name, p.units_sold
    FROM    products p
    WHERE   p.category_id = c.id
    ORDER   BY p.units_sold DESC
    LIMIT   5
) top;
```

With an index on `products(category_id, units_sold DESC)`, each inner iteration is an **index range scan that stops after 5 rows**. If there are 200 categories, the engine reads at most ~1,000 product index entries total — regardless of whether `products` has 10 thousand or 100 million rows. This bounded cost is what makes LATERAL the correct tool.

The window-function alternative (Topic 32):

```sql
SELECT category_id, product_id, name, units_sold
FROM (
    SELECT p.*,
           ROW_NUMBER() OVER (PARTITION BY category_id ORDER BY units_sold DESC) AS rn
    FROM   products p
) ranked
WHERE rn <= 5;
```

This computes `ROW_NUMBER()` for **every product row** — all 100 million — then discards those with `rn > 5`. Even with a perfect index it must read and rank every row in each partition. LATERAL reads 5 per partition. When the number of groups is small relative to the table and there is a matching index, **LATERAL wins decisively**. (When groups are many and each is small, or you need the rank value itself, window functions can be competitive or better — Section 10 quantifies this.)

**Ties**: `LIMIT 5` with `ORDER BY units_sold DESC` picks an arbitrary 5 among tied values at the boundary, just like any `LIMIT`. To make ties deterministic, add a tiebreaker: `ORDER BY units_sold DESC, id ASC`. To *keep all ties* at the boundary (return 6 rows if two tie for 5th place), LATERAL's `LIMIT` cannot do it — that is `RANK()`/`DENSE_RANK()` territory (Topic 32). This is a genuine semantic difference, not just performance.

### 6.5 Calling Set-Returning Functions Per Row

Set-returning functions in `FROM` are implicitly lateral. This is the standard way to expand array/JSON columns.

```sql
-- Expand an array column: each tag_id becomes its own row
SELECT p.id AS product_id, tag_id
FROM   products p,
       unnest(p.tag_ids) AS tag_id;     -- implicit LATERAL; unnest sees p.tag_ids
```

```sql
-- Expand a JSONB array column into rows, then join each element to a table
SELECT o.id AS order_id,
       (elem->>'product_id')::int AS product_id,
       (elem->>'qty')::int        AS qty
FROM   orders o,
       jsonb_array_elements(o.line_items) AS elem;   -- implicit LATERAL
```

```sql
-- Per-row generate_series: one row per day between each session's start and end
SELECT s.id AS session_id, d::date AS active_day
FROM   sessions s,
       generate_series(s.started_at::date, s.ended_at::date, INTERVAL '1 day') AS d;
```

The moment you need a **subquery** rather than a bare function — for example, `jsonb_array_elements` plus a filter, ordering, or a join back to another table with a `LIMIT` — you must write it as an explicit `LATERAL (SELECT ...)`:

```sql
-- Explode JSON, join each element to products, keep only in-stock, top 10 by price
SELECT o.id AS order_id, li.product_id, li.name, li.price
FROM   orders o
CROSS JOIN LATERAL (
    SELECT  (e->>'product_id')::int AS product_id, p.name, p.price
    FROM    jsonb_array_elements(o.line_items) AS e
    JOIN    products p ON p.id = (e->>'product_id')::int
    WHERE   p.in_stock
    ORDER   BY p.price DESC
    LIMIT   10
) li;
```

### 6.6 Multiple LATERAL Items and Chaining

A later LATERAL item can reference an **earlier** LATERAL item's output — they compose left to right.

```sql
SELECT u.id,
       last_order.id      AS last_order_id,
       last_order.total,
       items.product_id,  items.quantity
FROM   users u
CROSS JOIN LATERAL (
    SELECT o.id, o.total
    FROM   orders o
    WHERE  o.customer_id = u.id
    ORDER  BY o.created_at DESC
    LIMIT  1
) last_order
CROSS JOIN LATERAL (                       -- references last_order, produced above
    SELECT oi.product_id, oi.quantity
    FROM   order_items oi
    WHERE  oi.order_id = last_order.id
) items;
```

The first lateral finds each user's most-recent order; the second lateral, seeing that order's id, expands its line items. This chained shape is extremely common for "latest related record, then its children."

### 6.7 Correlated Aggregates — Beyond the Scalar Subquery

A scalar subquery (Topic 30) returns exactly one row and one column:

```sql
SELECT u.id,
       (SELECT COUNT(*) FROM orders o WHERE o.customer_id = u.id) AS order_count,
       (SELECT MAX(o.created_at) FROM orders o WHERE o.customer_id = u.id) AS last_order
FROM   users u;
-- Two separate correlated subqueries → orders is scanned/probed TWICE per user
```

LATERAL computes multiple correlated aggregates in **one** inner scan per outer row:

```sql
SELECT u.id, stats.order_count, stats.last_order, stats.lifetime_value
FROM   users u
LEFT JOIN LATERAL (
    SELECT COUNT(*)          AS order_count,
           MAX(o.created_at) AS last_order,
           SUM(o.total)      AS lifetime_value
    FROM   orders o
    WHERE  o.customer_id = u.id
) stats ON true;
-- One inner scan per user produces all three aggregates together
```

Use `LEFT JOIN LATERAL` here: an aggregate subquery over an empty set returns one row (`COUNT=0`, others NULL), so `CROSS JOIN` would technically keep the user too — but `LEFT JOIN ... ON true` documents the intent and is robust if you later add a `HAVING` or non-aggregate column that could make the subquery empty.

### 6.8 NULL Behavior

- **Correlated key is NULL**: if the outer row's correlated column is NULL (e.g. `u.id` is never NULL, but consider `orders.customer_id`), the inner predicate `WHERE o.customer_id = u.id` evaluates to UNKNOWN and matches nothing — same three-valued-logic rule as any equi-join (Topic 07). With `CROSS JOIN LATERAL` the outer row drops; with `LEFT JOIN LATERAL` it is kept with NULLs.
- **Inner returns NULL columns**: LATERAL faithfully passes through whatever the subquery projects, including NULLs. No special handling.
- **`LEFT JOIN LATERAL` NULL-extension**: when the subquery is empty, every projected column of the right side becomes NULL for that outer row — identical to LEFT JOIN's NULL-extension (Topic 12).

### 6.9 LATERAL vs DISTINCT ON

PostgreSQL's `DISTINCT ON` solves the *top-1*-per-group case compactly:

```sql
-- Latest order per user via DISTINCT ON
SELECT DISTINCT ON (o.customer_id)
       o.customer_id, o.id, o.total, o.created_at
FROM   orders o
ORDER  BY o.customer_id, o.created_at DESC;
```

This is elegant for **top-1** and often uses an index on `(customer_id, created_at DESC)` efficiently. But `DISTINCT ON` cannot do **top-N** for N > 1, and it starts from the *orders* side (so users with no orders vanish, and you cannot easily left-join-preserve them). LATERAL generalizes to any N and drives from the *users* side, making `LEFT JOIN LATERAL` preservation natural. Rule of thumb: top-1 from the child side → `DISTINCT ON`; top-N or must-preserve-parent → LATERAL.

### 6.10 What LATERAL Cannot Do

- It cannot reference tables that appear **to its right** — correlation is strictly left-to-right.
- It cannot reference the outer query's `SELECT` aliases, `GROUP BY` groups, or window results (those are later phases — Section 3).
- The subquery cannot be moved before the table it correlates to; it is not commutative with that table.
- The planner almost always uses Nested Loop for it — you do not get Hash/Merge Join across the lateral boundary, so an un-indexed correlation is genuinely O(outer × inner). LATERAL is not a substitute for a proper indexed equi-join when you actually want all matches (a plain `JOIN` may pick Hash Join and win).

---

## 7. EXPLAIN — LATERAL in the Plan

### Indexed Top-N-Per-Group (the good case)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.id, recent.id AS order_id, recent.total
FROM   users u
CROSS JOIN LATERAL (
    SELECT o.id, o.total
    FROM   orders o
    WHERE  o.customer_id = u.id
    ORDER  BY o.created_at DESC
    LIMIT  3
) recent;
```

```
Nested Loop  (cost=0.43..24500.00 rows=30000 width=20)
             (actual time=0.03..41.2 rows=28740 loops=1)
  ->  Seq Scan on users u  (cost=0.00..210.00 rows=10000 width=8)
                           (actual time=0.01..2.1 rows=10000 loops=1)
  ->  Limit  (cost=0.43..2.40 rows=3 width=20)
             (actual time=0.003..0.004 rows=2.9 loops=10000)
        ->  Index Scan using orders_customer_created_idx on orders o
              (cost=0.43..38.00 rows=57 width=20)
              (actual time=0.002..0.003 rows=2.9 loops=10000)
              Index Cond: (customer_id = u.id)
Buffers: shared hit=41210
Planning Time: 0.30 ms
Execution Time: 43.1 ms
```

**Reading it**:
- The whole thing is a **Nested Loop** — the signature of LATERAL. There is no `Hash Cond`; correlation is passed as a runtime parameter.
- Outer: `Seq Scan on users` → 10,000 rows drive the loop.
- Inner: `loops=10000` — the inner plan ran once per user.
- `Limit` node sits above the `Index Scan`: each iteration stops after 3 rows (`actual rows=2.9` average — some users have fewer than 3 orders).
- `Index Cond: (customer_id = u.id)` — `u.id` is the runtime parameter injected per iteration. The index `orders_customer_created_idx` on `(customer_id, created_at DESC)` supplies rows already in the right order, so **no Sort node** appears despite the `ORDER BY`.
- `Buffers: shared hit=41210`, zero `read` — the inner index/heap pages stay cached across iterations. This is why "looping 10,000 times" is still 43ms.

### Un-indexed Correlation (the disaster case)

```sql
-- Same query but no index on orders(customer_id, ...)
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.id, recent.id
FROM   users u
CROSS JOIN LATERAL (
    SELECT o.id FROM orders o
    WHERE o.customer_id = u.id
    ORDER BY o.created_at DESC LIMIT 3
) recent;
```

```
Nested Loop  (cost=..)  (actual time=0.5..38900.0 rows=28740 loops=1)
  ->  Seq Scan on users u  (actual rows=10000 loops=1)
  ->  Limit  (actual time=3.8..3.8 rows=2.9 loops=10000)
        ->  Sort  (actual time=3.8..3.8 rows=2.9 loops=10000)
              Sort Key: o.created_at DESC
              Sort Method: top-N heapsort  Memory: 25kB
              ->  Seq Scan on orders o  (actual rows=200 loops=10000)
                    Filter: (customer_id = u.id)
                    Rows Removed by Filter: 1999800
Buffers: shared hit=... read=...
Execution Time: 38912.0 ms
```

**Reading it**:
- Inner is now a **Seq Scan on orders** with `Filter: (customer_id = u.id)` — no index to satisfy the correlation.
- `Rows Removed by Filter: 1999800` **per loop** — each of 10,000 iterations scans the entire 2M-row `orders` table.
- A **Sort** appears because there is no ordered index to feed the `LIMIT` — top-N heapsort per iteration.
- 38.9 **seconds** vs 43 **milliseconds**. Same query, one missing index. This is the number-one LATERAL production failure.

### LEFT JOIN LATERAL (preserving zero-match outer rows)

```sql
EXPLAIN (ANALYZE)
SELECT u.id, stats.order_count
FROM   users u
LEFT JOIN LATERAL (
    SELECT COUNT(*) AS order_count
    FROM   orders o WHERE o.customer_id = u.id
) stats ON true;
```

```
Nested Loop Left Join  (cost=.. rows=10000 width=12)
                       (actual time=0.02..55.0 rows=10000 loops=1)
  ->  Seq Scan on users u  (actual rows=10000 loops=1)
  ->  Aggregate  (actual time=0.005..0.005 rows=1 loops=10000)
        ->  Index Only Scan using orders_customer_id_idx on orders o
              Index Cond: (customer_id = u.id)
              (actual rows=2.9 loops=10000)
```

**Reading it**:
- `Nested Loop **Left Join**` — note the "Left Join" tag; zero-order users are preserved.
- `rows=10000` in the output equals the number of users — every user survives, aggregate returns one row (`COUNT=0`) even when the inner is empty.
- `Aggregate` over `Index Only Scan` — the count is served from the index without touching the heap.

### What to Look For

| Symptom | Meaning | Fix |
|---|---|---|
| Nested Loop + Seq Scan inner + high "Rows Removed by Filter" | No index on the correlated column | Add index on `(correlated_col, order_col)` |
| Sort node inside the inner loop | `ORDER BY ... LIMIT` has no matching ordered index | Add composite index matching `ORDER BY` |
| `loops=` equal to a huge outer count | Too many outer rows driving the loop | Filter the outer side first / reconsider window function |
| `shared read` high (not `hit`) inside inner | Inner pages not cached — cold, or table too big to cache | More RAM / better index locality |
| Plan shows `Materialize` above inner | Rare for LATERAL; inner not truly correlated | Verify the correlation is actually used |

---

## 8. Query Examples

### Example 1 — Basic: Latest Order Per User

```sql
-- For every user who has ordered, show their single most recent order.
SELECT
  u.id            AS user_id,
  u.name,
  last_order.id   AS order_id,
  last_order.total,
  last_order.created_at
FROM users u
CROSS JOIN LATERAL (               -- users with no orders are excluded here
  SELECT o.id, o.total, o.created_at
  FROM   orders o
  WHERE  o.customer_id = u.id      -- correlation: this user's orders only
  ORDER  BY o.created_at DESC      -- newest first
  LIMIT  1                         -- just the latest
) last_order
ORDER BY last_order.created_at DESC;
```

### Example 2 — Intermediate: Top 3 Products Per Category, Preserving Empty Categories

```sql
-- Top 3 best-selling products in each category.
-- LEFT JOIN LATERAL keeps categories that have zero products.
SELECT
  c.id            AS category_id,
  c.name          AS category_name,
  top.product_id,
  top.product_name,
  top.units_sold,
  top.rank_in_category
FROM categories c
LEFT JOIN LATERAL (
  SELECT
    p.id                       AS product_id,
    p.name                     AS product_name,
    p.units_sold,
    ROW_NUMBER() OVER (ORDER BY p.units_sold DESC) AS rank_in_category
  FROM products p
  WHERE p.category_id = c.id    -- correlation
  ORDER BY p.units_sold DESC
  LIMIT 3
) top ON true                  -- empty categories kept with NULLs
ORDER BY c.name, top.rank_in_category;
```

### Example 3 — Production Grade: Per-Customer Dashboard Row

```sql
-- Customer analytics dashboard: for each active customer, in ONE pass produce
--   - lifetime stats (from a correlated aggregate LATERAL)
--   - their 3 most recent orders (from a top-N LATERAL, expanded to rows)
--
-- Table sizes:   users 500K rows, orders 40M rows, order_items 180M rows.
-- Indexes:       orders(customer_id, created_at DESC)          -- drives top-N & aggregates
--                order_items(order_id)                          -- child expansion
-- Expectation:   with the composite index warm, ~120–250ms for a 2K-customer segment.
--                Without it, minutes (per-loop Seq Scan of 40M orders).

SELECT
  u.id           AS user_id,
  u.name,
  u.email,
  stats.order_count,
  stats.lifetime_value,
  stats.last_order_at,
  recent.order_id,
  recent.order_total,
  recent.placed_at
FROM users u
-- (1) one correlated aggregate scan → several lifetime metrics together
LEFT JOIN LATERAL (
  SELECT
    COUNT(*)            AS order_count,
    SUM(o.total)        AS lifetime_value,
    MAX(o.created_at)   AS last_order_at
  FROM orders o
  WHERE o.customer_id = u.id
    AND o.status = 'completed'
) stats ON true
-- (2) top-3 recent orders, bounded by LIMIT and served by the composite index
LEFT JOIN LATERAL (
  SELECT o.id AS order_id, o.total AS order_total, o.created_at AS placed_at
  FROM   orders o
  WHERE  o.customer_id = u.id
    AND  o.status = 'completed'
  ORDER  BY o.created_at DESC
  LIMIT  3
) recent ON true
WHERE u.status = 'active'
  AND u.segment = 'enterprise'
ORDER BY stats.lifetime_value DESC NULLS LAST, u.id, recent.placed_at DESC;
```

```
EXPLAIN (ANALYZE, BUFFERS) — abridged:

Sort  (actual time=214.0..214.9 rows=5820 loops=1)
  Sort Key: stats.lifetime_value DESC NULLS LAST, u.id, recent.placed_at DESC
  ->  Nested Loop Left Join  (actual time=0.1..205.3 rows=5820 loops=1)
        ->  Nested Loop Left Join  (actual time=0.1..120.4 rows=2000 loops=1)
              ->  Index Scan using users_segment_status_idx on users u
                    Index Cond: ((segment = 'enterprise') AND (status = 'active'))
                    (actual rows=2000 loops=1)
              ->  Aggregate  (actual rows=1 loops=2000)
                    ->  Index Scan using orders_customer_created_idx on orders o
                          Index Cond: (customer_id = u.id)
                          Filter: (status = 'completed')
                          (actual rows=32 loops=2000)
        ->  Limit  (actual rows=2.9 loops=2000)
              ->  Index Scan using orders_customer_created_idx on orders o_1
                    Index Cond: (customer_id = u.id)
                    Filter: (status = 'completed')
                    (actual rows=2.9 loops=2000)
Buffers: shared hit=98230
Execution Time: 216.4 ms
```

The two `Nested Loop Left Join` layers correspond to the two `LEFT JOIN LATERAL` items. Only 2,000 enterprise users drive both loops; each inner scan is index-served and, for the top-N, bounded by `LIMIT 3`. Total 216ms over a 40M-row orders table.

---

## 9. Wrong → Right Patterns

### Wrong 1: Referencing an outer column without LATERAL

```sql
-- WRONG: subquery in FROM tries to see u.id without LATERAL
SELECT u.id, r.total
FROM   users u,
       (SELECT o.total
        FROM   orders o
        WHERE  o.customer_id = u.id           -- u.id not in scope here
        ORDER  BY o.created_at DESC LIMIT 1) r;
-- ERROR:  invalid reference to FROM-clause entry for table "u"
-- The FROM subquery is a sealed relation; it cannot see its sibling.
```

```sql
-- RIGHT: add LATERAL to unseal the subquery
SELECT u.id, r.total
FROM   users u
CROSS JOIN LATERAL (
    SELECT o.total
    FROM   orders o
    WHERE  o.customer_id = u.id
    ORDER  BY o.created_at DESC LIMIT 1
) r;
```

### Wrong 2: CROSS JOIN LATERAL silently dropping zero-match rows

```sql
-- WRONG: "list every user with their latest order" — but users with no
-- orders VANISH because the subquery returns 0 rows for them.
SELECT u.id, u.name, last.total
FROM   users u
CROSS JOIN LATERAL (
    SELECT o.total FROM orders o
    WHERE o.customer_id = u.id
    ORDER BY o.created_at DESC LIMIT 1
) last;
-- A 500K-user table returns only the ~380K users who have ordered.
-- The report is missing every never-ordered customer — a silent data-loss bug.
```

```sql
-- RIGHT: LEFT JOIN LATERAL ... ON true preserves every user
SELECT u.id, u.name, last.total
FROM   users u
LEFT JOIN LATERAL (
    SELECT o.total FROM orders o
    WHERE o.customer_id = u.id
    ORDER BY o.created_at DESC LIMIT 1
) last ON true;
-- Never-ordered users appear with last.total = NULL
```

### Wrong 3: Filtering in the outer WHERE instead of inside the LATERAL (defeats LIMIT)

```sql
-- WRONG: the status filter is in the OUTER where, applied AFTER the LIMIT
SELECT u.id, recent.id, recent.status
FROM   users u
CROSS JOIN LATERAL (
    SELECT o.id, o.status, o.created_at
    FROM   orders o
    WHERE  o.customer_id = u.id
    ORDER  BY o.created_at DESC
    LIMIT  3                       -- takes the 3 most recent of ANY status
) recent
WHERE recent.status = 'completed'; -- then throws away non-completed ones
-- BUG: if a user's 3 newest orders are all 'cancelled', you get ZERO rows for
-- them even though they have older completed orders. The LIMIT ran first.
```

```sql
-- RIGHT: push the predicate INSIDE the lateral so LIMIT sees only completed
SELECT u.id, recent.id, recent.status
FROM   users u
CROSS JOIN LATERAL (
    SELECT o.id, o.status, o.created_at
    FROM   orders o
    WHERE  o.customer_id = u.id
      AND  o.status = 'completed'  -- filter BEFORE limiting
    ORDER  BY o.created_at DESC
    LIMIT  3
) recent;
-- Now you get each user's 3 most recent COMPLETED orders — the real intent.
```

### Wrong 4: Missing index turns LATERAL into an O(N×M) scan

```sql
-- WRONG in effect: no index on orders(customer_id, created_at)
-- The query is correct but each of 10K iterations Seq-Scans 2M orders + sorts.
SELECT u.id, r.id
FROM   users u
CROSS JOIN LATERAL (
    SELECT o.id FROM orders o
    WHERE o.customer_id = u.id
    ORDER BY o.created_at DESC LIMIT 3
) r;
-- EXPLAIN: Seq Scan on orders, "Rows Removed by Filter: 1999800" per loop → ~39s
```

```sql
-- RIGHT: add the composite index that matches BOTH the correlation and the ORDER BY
CREATE INDEX orders_customer_created_idx
    ON orders (customer_id, created_at DESC);
-- Now each inner iteration is an ordered Index Scan stopping after 3 rows → ~40ms.
-- The query text does not change — only the index makes it viable.
```

### Wrong 5: Using LATERAL where a plain JOIN is better

```sql
-- WRONG (misuse): LATERAL forces a Nested Loop even when you want ALL matches
-- and a Hash Join would be far faster. This runs the inner scan per user.
SELECT u.id, o.id, o.total
FROM   users u
CROSS JOIN LATERAL (
    SELECT o.id, o.total FROM orders o WHERE o.customer_id = u.id
) o;   -- no LIMIT, no ordering, no SRF — this is just an equi-join in disguise
```

```sql
-- RIGHT: a plain INNER JOIN lets the planner pick Hash Join over the full sets
SELECT u.id, o.id, o.total
FROM   users u
INNER JOIN orders o ON o.customer_id = u.id;
-- One hash build on users, one scan of orders — O(N+M), no per-row loop.
-- Reserve LATERAL for LIMIT/top-N, SRFs, or multi-aggregate-per-row cases.
```

---

## 10. Performance Profile

### The Governing Formula

```
cost ≈ (rows on the outer/driving side) × (cost of one inner evaluation)
```

Everything about LATERAL performance reduces to shrinking one of those two factors:
- **Fewer outer rows**: filter the driving table *before* the lateral (put selective predicates on the outer table so the loop runs fewer times).
- **Cheaper inner evaluation**: an index that satisfies both the correlation predicate and the `ORDER BY`, so each iteration is a bounded index scan instead of a scan+sort.

### Scaling Table

| Scenario | Outer rows | Inner per iter | Typical time |
|---|---|---|---|
| Top-3/group, composite index, 10K outer | 10K | ~3 index rows, no sort | ~40ms |
| Top-3/group, composite index, 1M outer | 1M | ~3 index rows | ~3–6s (loop-bound) |
| Top-3/group, **no** index, 10K outer | 10K | full seq scan + sort of M rows | tens of seconds — avoid |
| Correlated aggregate, index-only scan, 100K outer | 100K | index-only aggregate | ~0.5–2s |
| SRF expansion (unnest/jsonb), 1M outer, ~5 elems each | 1M | function eval, ~5 rows | ~2–5s, CPU-bound |

### LATERAL Top-N vs Window Function ROW_NUMBER

The decision hinges on **group count vs rows-per-group** and **index availability**:

| Situation | Winner | Why |
|---|---|---|
| Few groups, many rows/group, matching index | **LATERAL** | Reads N per group; window ranks all rows |
| Many small groups, no useful index | **Window fn** | One sequential pass + sort beats millions of tiny loops |
| Need the rank/row_number value in output | **Window fn** | LATERAL discards rank; you'd re-derive it |
| Need ties preserved (all rows tied for Nth) | **Window fn** (`RANK`) | `LIMIT` can't keep ties |
| Must preserve groups with zero rows | **LATERAL** (`LEFT JOIN LATERAL`) | Drives from parent side |

Concrete: 200 categories over a 50M-row products table, top-5 each. LATERAL with `products(category_id, units_sold DESC)`: ~1,000 index rows total, single-digit ms. Window function: sort/scan all 50M rows partitioned — seconds and large `work_mem`. LATERAL wins by 100–1000×.

Flip it: 5M users, top-3 orders each, orders 40M. LATERAL loops 5M times (even at 5µs/iter that is ~25s of loop overhead). A window function doing one big partitioned sort of 40M rows may finish faster because it avoids 5M loop setups. Measure with `EXPLAIN ANALYZE`; the crossover depends on your row counts and cache.

### Memory and CPU

- **Memory**: LATERAL itself uses almost no memory — it is a nested loop, O(1) join state (contrast Hash Join's O(smaller table)). The inner `ORDER BY ... LIMIT` uses a bounded top-N heapsort (~`work_mem`-independent for small N). This is a real advantage over window functions on huge partitions, which can spill sorts to disk.
- **CPU**: dominated by loop iterations and index descents. `loops × log(inner)` index comparisons. SRF expansion is CPU-bound on the function (JSON parsing for `jsonb_array_elements` is not free).
- **Buffers**: inner index/heap pages are re-probed every iteration; after warm-up they are `shared hit`, so I/O is low. A cold cache on the first run can be much slower — the classic "second run is 10× faster" LATERAL surprise.

### Optimization Techniques Specific to LATERAL

1. **Composite index matching correlation + ORDER BY**: `(correlated_col, sort_col DESC)`. This is the single highest-leverage change — it removes both the Seq Scan and the Sort from the inner loop.
2. **Push all inner predicates inside the subquery** so `LIMIT` prunes correctly and early (Wrong #3).
3. **Filter the outer side hard** to reduce `loops`.
4. **Prefer `LEFT JOIN LATERAL ... ON true` for preservation**, `CROSS JOIN LATERAL` only when dropping empties is intended.
5. **For top-1, consider `DISTINCT ON`** (Section 6.9) — sometimes a cleaner plan.
6. **Watch cache warmth**: benchmark with a warm cache to reflect steady-state; a cold first run misleads.

---

## 11. Node.js Integration

### 11.1 Top-N-per-group with `pg`

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Each user's N most recent completed orders, in one round trip.
async function recentOrdersPerUser(segment, n = 3) {
  const { rows } = await pool.query(
    `SELECT u.id AS user_id, u.name,
            recent.order_id, recent.total, recent.placed_at
     FROM users u
     LEFT JOIN LATERAL (
       SELECT o.id AS order_id, o.total, o.created_at AS placed_at
       FROM   orders o
       WHERE  o.customer_id = u.id
         AND  o.status = 'completed'
       ORDER  BY o.created_at DESC
       LIMIT  $2                       -- N is parameterized
     ) recent ON true
     WHERE u.segment = $1
     ORDER BY u.id, recent.placed_at DESC`,
    [segment, n]                       -- $1 = segment, $2 = n
  );
  return rows;                         // users with no orders appear with NULLs
}
```

Note `LIMIT $2` — the row limit is a bound parameter, safely handled by `pg`'s `$n` binding (never string-interpolate a limit).

### 11.2 Expanding a JSONB array column per row

```javascript
// Explode order.line_items (jsonb array) into one row per line item,
// joined to products, using an implicit-lateral SRF inside an explicit LATERAL.
async function orderLineDetails(orderId) {
  const { rows } = await pool.query(
    `SELECT o.id AS order_id, li.product_id, li.name, li.qty, li.price
     FROM orders o
     CROSS JOIN LATERAL (
       SELECT (e->>'product_id')::int AS product_id,
              p.name,
              (e->>'qty')::int         AS qty,
              p.price
       FROM   jsonb_array_elements(o.line_items) AS e
       JOIN   products p ON p.id = (e->>'product_id')::int
     ) li
     WHERE o.id = $1`,
    [orderId]
  );
  return rows;
}
```

### 11.3 Correlated aggregates in one pass

```javascript
async function customerLifetimeStats(userId) {
  const { rows } = await pool.query(
    `SELECT u.id, u.name,
            stats.order_count, stats.lifetime_value, stats.last_order_at
     FROM users u
     LEFT JOIN LATERAL (
       SELECT COUNT(*)          AS order_count,
              SUM(o.total)      AS lifetime_value,
              MAX(o.created_at) AS last_order_at
       FROM   orders o
       WHERE  o.customer_id = u.id AND o.status = 'completed'
     ) stats ON true
     WHERE u.id = $1`,
    [userId]
  );
  return rows[0] ?? null;   // one row; aggregates are 0/NULL if no orders
}
```

**A note on ORMs and N+1**: the pattern LATERAL replaces is the classic ORM N+1 (Topic 18) — fetch users, then loop issuing one "recent orders" query per user. A single LATERAL query is one round trip regardless of user count. Most ORMs cannot express LATERAL natively (Section 12), so this is a prime case for the raw-SQL escape hatch.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do LATERAL?** — Not directly. Prisma's query API has no LATERAL construct. Its `take` inside a nested `include` gives "N related per parent," which under the hood Prisma implements with a **correlated LATERAL / window strategy** in recent versions — but you cannot write an arbitrary LATERAL subquery.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// "3 most recent orders per user" — Prisma can approximate via nested take+orderBy.
const users = await prisma.user.findMany({
  where: { segment: 'enterprise' },
  include: {
    orders: {
      where: { status: 'completed' },
      orderBy: { createdAt: 'desc' },
      take: 3,                       // Prisma emits a correlated/lateral-like query
    },
  },
});

// Anything beyond simple top-N (multi-aggregate-per-row, SRF/JSON expansion) → raw:
const dash = await prisma.$queryRaw`
  SELECT u.id, s.order_count, s.lifetime_value
  FROM users u
  LEFT JOIN LATERAL (
    SELECT COUNT(*) AS order_count, SUM(o.total) AS lifetime_value
    FROM orders o WHERE o.customer_id = u.id AND o.status = 'completed'
  ) s ON true
  WHERE u.segment = 'enterprise'`;
```

**Where it breaks**: correlated aggregates returning multiple columns, JSON/array SRF expansion, `LEFT JOIN LATERAL ... ON true` preservation semantics — none are expressible in the fluent API. **Verdict**: nested `take` covers plain top-N; everything else needs `$queryRaw`.

### Drizzle ORM

**Can Drizzle do LATERAL?** — Yes. Drizzle added first-class `.crossJoinLateral()` / `.leftJoinLateral()` and a `lateral()` helper.

```typescript
import { db } from './db';
import { users, orders } from './schema';
import { eq, and, desc, sql } from 'drizzle-orm';

const recent = db
  .select({ orderId: orders.id, total: orders.total, placedAt: orders.createdAt })
  .from(orders)
  .where(and(eq(orders.customerId, users.id), eq(orders.status, 'completed')))
  .orderBy(desc(orders.createdAt))
  .limit(3)
  .as('recent');

const result = await db
  .select({ userId: users.id, name: users.name,
            orderId: recent.orderId, total: recent.total })
  .from(users)
  .leftJoinLateral(recent, sql`true`);   // LEFT JOIN LATERAL ... ON true
```

**Where it breaks**: SRF-based lateral (`jsonb_array_elements`) still needs a `sql` template. **Verdict**: best fluent LATERAL support of the ORMs; use it directly for subquery laterals, `sql` for SRFs.

### Sequelize

**Can Sequelize do LATERAL?** — No native support. `include` with `limit` on a hasMany applies a `subQuery: true` wrapper (a correlated subquery), not a general LATERAL, and it is notoriously fiddly.

```javascript
// The reliable path is a raw query:
const rows = await sequelize.query(
  `SELECT u.id, r.order_id, r.total
   FROM users u
   LEFT JOIN LATERAL (
     SELECT o.id AS order_id, o.total FROM orders o
     WHERE o.customer_id = u.id AND o.status = 'completed'
     ORDER BY o.created_at DESC LIMIT :n
   ) r ON true
   WHERE u.segment = :segment`,
  { replacements: { n: 3, segment: 'enterprise' }, type: QueryTypes.SELECT }
);
```

**Where it breaks**: everything LATERAL-shaped. `include`+`limit` sometimes silently returns wrong per-parent counts due to the join+limit interaction. **Verdict**: use `sequelize.query()` for any LATERAL need.

### TypeORM

**Can TypeORM do LATERAL?** — Not via the entity/relation API, but the QueryBuilder lets you inject raw lateral joins.

```typescript
const rows = await dataSource
  .createQueryBuilder()
  .select(['u.id AS user_id', 'r.order_id', 'r.total'])
  .from('users', 'u')
  // raw LEFT JOIN LATERAL through the query builder's join escape hatch:
  .leftJoin(
    qb => qb,                        // placeholder; TypeORM has no typed lateral
    'r', 'true')
  // In practice, use .query() for full control:
  .getRawMany();

// Idiomatic: raw query
const dash = await dataSource.query(
  `SELECT u.id, r.order_id, r.total
   FROM users u
   LEFT JOIN LATERAL (
     SELECT o.id AS order_id, o.total FROM orders o
     WHERE o.customer_id = u.id ORDER BY o.created_at DESC LIMIT $1
   ) r ON true`, [3]);
```

**Where it breaks**: no typed lateral; the QueryBuilder join helpers do not model LATERAL. **Verdict**: use `dataSource.query()`.

### Knex.js

**Can Knex do LATERAL?** — Yes, via `knex.raw()` in the join/from position; no dedicated helper, but Knex's transparency makes it clean.

```javascript
const rows = await knex('users as u')
  .select('u.id', 'r.order_id', 'r.total')
  .joinRaw(
    `LEFT JOIN LATERAL (
       SELECT o.id AS order_id, o.total
       FROM orders o
       WHERE o.customer_id = u.id AND o.status = 'completed'
       ORDER BY o.created_at DESC
       LIMIT ?
     ) r ON true`, [3])
  .where('u.segment', 'enterprise');
```

**Where it breaks**: the lateral body is a raw string — no column type inference. **Verdict**: `joinRaw` handles it cleanly; Knex is the most SQL-transparent option after Drizzle.

### ORM Summary Table

| ORM | LATERAL support | Native top-N-per-group | SRF/JSON lateral | Escape hatch |
|---|---|---|---|---|
| Prisma | No (fluent) | `include`+`take` (approx) | No | `$queryRaw` |
| Drizzle | **Yes** (`leftJoinLateral`/`crossJoinLateral`) | Yes, typed | via `sql` | `sql` template |
| Sequelize | No | `include`+`limit` (fragile) | No | `sequelize.query()` |
| TypeORM | No (typed) | No | No | `dataSource.query()` |
| Knex | via `joinRaw` | via `joinRaw` | via `joinRaw` | `knex.raw()` |

---

## 13. Practice Exercises

### Exercise 1 — Easy

Given:
- `users(id, name, email, created_at)`
- `orders(id, customer_id, total, status, created_at)`

Write a query that, for every user, returns their single most recent order's `id`, `total`, and `created_at`. Users who have never ordered must **still appear**, with NULLs for the order columns. Order the result by the order's `created_at` descending, NULLs last.

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines Topics 11, 20, 30)

Given `categories(id, name)`, `products(id, category_id, name, price, units_sold)`:

For each category, return the category name plus the **top 3 products by `units_sold`** as three separate rows, including a `rank_in_category` column (1, 2, 3). Also include, on every row, the **category's total units sold across all its products** (a correlated aggregate). Categories with no products should not appear. Order by category name, then rank.

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation — the naive answer is wrong)

Given `users(id, segment, status)` and `orders(id, customer_id, status, total, created_at)` (orders is 40M rows):

Return, for each **active enterprise** user, their 3 most recent **completed** orders. A teammate wrote:

```sql
SELECT u.id, r.id, r.status
FROM users u
CROSS JOIN LATERAL (
  SELECT o.id, o.status, o.created_at FROM orders o
  WHERE o.customer_id = u.id
  ORDER BY o.created_at DESC LIMIT 3
) r
WHERE r.status = 'completed' AND u.segment = 'enterprise' AND u.status = 'active';
```

1. Explain the two distinct bugs (one correctness, one data-loss).
2. Rewrite it correctly.
3. State the exact index you would create and why each column/order matters.

```sql
-- Write your corrected query here
```

### Exercise 4 — Interview Simulation

Given `orders(id, customer_id, line_items jsonb)` where `line_items` is a JSON array like `[{"product_id":7,"qty":2}, ...]`, and `products(id, name, price)`:

Write one query that returns, per order, the **single most expensive line item**: `order_id`, `product_id`, `name`, `qty`, and `line_total` (= qty × price). Orders with an empty `line_items` array must still appear with NULLs. You must expand the JSON, join to products, and pick the max — all inside a lateral.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1 — What does the LATERAL keyword actually do, and why is it needed?

**Junior answer**: "It lets you use a subquery in the FROM clause." (True but misses the point — you can already put subqueries in FROM.)

**Principal answer**: By default every item in the FROM list is an independent relation evaluated without knowledge of its siblings; referencing another FROM item's column raises an error. LATERAL removes that isolation, allowing the subquery (or set-returning function) to reference columns from FROM items that appear to its left. Semantically it turns the FROM clause into a `for` loop: for each row the left side produces, the right subquery is evaluated with that row's values bound as parameters, and results are joined back. The engine implements this as a parameterized Nested Loop. It exists to express top-N-per-group, per-row set-returning functions, and multi-column correlated aggregates — things a scalar subquery (one row, one column) cannot do.

**Interviewer follow-up**: "So when would a plain JOIN be better than LATERAL?" — When you want *all* matches with no per-row LIMIT/ordering/SRF: a plain equi-join lets the planner choose Hash or Merge Join (O(N+M)), whereas LATERAL forces a Nested Loop that re-scans the inner side per outer row.

### Q2 — You need the 3 most recent orders per customer over a 40M-row orders table. Compare LATERAL and a window function.

**Junior answer**: "Use `ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY created_at DESC)` and filter `<= 3`."

**Principal answer**: Both are correct; the choice is about work done. The window function assigns a row number to **every** one of the 40M rows, then discards those ranked > 3 — it must scan/sort the full partitioned set. LATERAL with `ORDER BY created_at DESC LIMIT 3` and an index on `orders(customer_id, created_at DESC)` reads **at most 3 index rows per customer** and stops — bounded work, no big sort, minimal `work_mem`. If the customer count is small relative to the table, LATERAL wins by orders of magnitude. If there are millions of customers (millions of loop iterations), the window function's single sequential pass can overtake it. I'd decide with `EXPLAIN ANALYZE`, but default to LATERAL when a matching composite index exists.

**Interviewer follow-up**: "What if two orders tie for 3rd place and you need both?" — `LIMIT` picks an arbitrary one; that requires `RANK()`/`DENSE_RANK()` semantics, which is a genuine reason to use the window function.

### Q3 — A LATERAL query runs in 40ms in staging but 39 seconds in production. Same query. What happened?

**Junior answer**: "Production has more data / is under load."

**Principal answer**: Almost certainly a missing or unusable index on the correlated column in production. LATERAL is a Nested Loop; each outer row triggers an inner evaluation. With the right composite index `(customer_id, created_at DESC)`, each iteration is a bounded Index Scan. Without it, each iteration becomes a full Seq Scan of the (larger, production-sized) inner table plus a per-iteration Sort — `EXPLAIN ANALYZE` would show `Seq Scan on orders`, a `Sort`, and a huge `Rows Removed by Filter` multiplied by `loops`. The fix is the composite index matching both the correlation predicate and the inner `ORDER BY`. A secondary suspect is cache warmth — the inner pages must be resident; a cold buffer pool makes the first run far slower, though not 1000×.

**Interviewer follow-up**: "How would you confirm it's the index and not cache?" — Run `EXPLAIN (ANALYZE, BUFFERS)`; look for `Seq Scan` + `Sort` in the inner node and high `Rows Removed by Filter` per loop (index problem) versus high `shared read` with an Index Scan already present (cache problem).

### Q4 — Explain the difference between CROSS JOIN LATERAL and LEFT JOIN LATERAL with a concrete failure.

**Junior answer**: "One is inner, one is outer."

**Principal answer**: `CROSS JOIN LATERAL` drops any outer row whose subquery returns zero rows — inner-join semantics. `LEFT JOIN LATERAL ... ON true` keeps every outer row, NULL-extending the right side when the subquery is empty — outer-join semantics. The concrete failure: a "list all users with their latest order" report built with `CROSS JOIN LATERAL` silently omits every user who has never ordered — a data-loss bug that passes casual testing (the users who *have* ordered look right) and only surfaces when someone notices the total user count is short. The fix is `LEFT JOIN LATERAL ... ON true`. The `ON true` is required syntax because the real correlation lives inside the subquery; there is nothing to put in the ON clause.

**Interviewer follow-up**: "Does putting a filter in that `ON` clause instead of inside the subquery change anything?" — Yes: a predicate in the `ON` of a LEFT JOIN LATERAL still preserves the outer row (NULL-extends when false) and, crucially for performance, runs *after* the subquery's LIMIT, so it cannot prune before the limit — push row-selective filters inside the subquery.

---

## 15. Mental Model Checkpoint

1. In one sentence, why can the right-hand subquery of a LATERAL join reference `users.id` while an ordinary FROM-subquery cannot?

2. You write `CROSS JOIN LATERAL (SELECT ... LIMIT 3)` to get each user's 3 latest orders, and a user has exactly 0 orders. What appears in the result for that user, and how does the answer change if you switch to `LEFT JOIN LATERAL ... ON true`?

3. A LATERAL top-3 query has an outer table of 10,000 rows and an inner table of 2,000,000 rows. With the right index, roughly how many inner rows are read in total? Without any index?

4. You move a `WHERE status = 'completed'` filter from *inside* the LATERAL subquery to the *outer* query's WHERE, keeping the `LIMIT 3`. Give a concrete scenario where the result set shrinks incorrectly.

5. Why does the planner almost always choose Nested Loop for a correlated LATERAL, and never Hash Join across the lateral boundary?

6. For "the single latest order per customer," compare `DISTINCT ON (customer_id)` against `CROSS JOIN LATERAL ... LIMIT 1`. When would you prefer each, and which one naturally preserves customers with no orders?

7. When is a plain window function (`ROW_NUMBER`) genuinely the better choice than LATERAL for top-N-per-group, even with a perfect index available?

---

## 16. Quick Reference Card

```sql
-- CORE SHAPE: "for each left row, run this subquery with the row's values"
FROM left_table L
CROSS JOIN LATERAL (SELECT ... WHERE x = L.col ORDER BY y LIMIT k) s
-- CROSS JOIN LATERAL  → drop L if subquery empty   (inner-join semantics)
-- LEFT JOIN LATERAL ... ON true → keep L with NULLs (outer-join semantics)

-- TOP-N-PER-GROUP (flagship use case)
FROM categories c
CROSS JOIN LATERAL (
  SELECT * FROM products p
  WHERE p.category_id = c.id
  ORDER BY p.units_sold DESC
  LIMIT 5
) top;
-- Index that makes it fast: (category_id, units_sold DESC)

-- SET-RETURNING FUNCTIONS: LATERAL is IMPLICIT (keyword optional)
FROM orders o, jsonb_array_elements(o.line_items) e   -- explode JSON per row
FROM products p, unnest(p.tag_ids) tag                -- explode array per row

-- CORRELATED MULTI-AGGREGATE in one inner scan
LEFT JOIN LATERAL (
  SELECT COUNT(*) c, SUM(total) s, MAX(created_at) m
  FROM orders o WHERE o.customer_id = u.id
) stats ON true;

-- GOTCHAS
-- 1. Outer-column reference in FROM without LATERAL → ERROR (invalid reference)
-- 2. CROSS JOIN LATERAL silently drops zero-match outer rows → use LEFT JOIN LATERAL ... ON true
-- 3. Filter in OUTER where runs AFTER inner LIMIT → push predicates INSIDE the subquery
-- 4. No index on correlated col → O(outer × inner) Seq-Scan-per-loop disaster
-- 5. LATERAL forces Nested Loop → for "all matches, no LIMIT", a plain JOIN (Hash) is faster

-- EXPLAIN SIGNALS
-- Nested Loop + inner Index Scan (Index Cond: col = $0), Limit node, no Sort → GOOD
-- Nested Loop + inner Seq Scan + Sort + "Rows Removed by Filter" ×loops     → MISSING INDEX

-- INTERVIEW ONE-LINERS
-- "LATERAL is a for-loop in the FROM clause."
-- "Top-N-per-group: LATERAL reads N per group; window ROW_NUMBER ranks ALL rows."
-- "CROSS = inner (drop empties), LEFT ... ON true = outer (keep empties)."
-- "It's always a parameterized Nested Loop — index the correlated column."
-- "Set-returning functions in FROM are implicitly lateral."
```

---

## Connected Topics

- **Topic 30 — Subquery vs CTE vs JOIN**: LATERAL is the FROM-clause generalization of the correlated subquery — read that comparison first; LATERAL extends "one row, one column" scalar subqueries to "many rows, many columns."
- **Topic 32 — Window Functions Fundamentals**: the primary alternative for top-N-per-group; `ROW_NUMBER`/`RANK` vs LATERAL `LIMIT` is the key trade-off (all-rows ranking vs bounded per-group reads, tie handling).
- **Topic 11 — INNER JOIN in Depth**: LATERAL is executed as a parameterized Nested Loop; the index-on-inner-column rule and fan-out/row-multiplication mechanics carry over directly.
- **Topic 12 — LEFT JOIN and RIGHT JOIN**: `CROSS JOIN LATERAL` vs `LEFT JOIN LATERAL ... ON true` is exactly the INNER-vs-LEFT preservation distinction, including the ON-clause subtlety.
- **Topic 18 — The N+1 Query Problem**: LATERAL collapses the "fetch parents, then loop one child-query per parent" anti-pattern into a single statement.
- **Topic 07 — NULL in Depth**: three-valued logic governs the correlated predicate; NULL keys match nothing and interact with CROSS/LEFT preservation.
- **Topic 17 — JOIN Performance Deep Dive**: Nested Loop cost model, parameterized index paths, and buffer-cache reuse across inner iterations explain LATERAL's performance profile.
```
