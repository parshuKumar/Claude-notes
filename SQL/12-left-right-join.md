# Topic 12 — LEFT JOIN and RIGHT JOIN

### SQL Mastery Curriculum — Phase 3: JOINs — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

You're a teacher taking attendance. You have two lists:

- **List A (the roster)**: every student enrolled in your class — one line per student.
- **List B (the homework)**: every homework submission that came in, tagged with a student's name and a grade.

You want to fill out a report card grid that has **one row for every student on the roster**, whether or not they turned in homework.

- For a student who submitted homework → write their name and their grade.
- For a student who submitted **nothing** → still write their name, but leave the grade cell **blank**.

That blank cell is a `NULL`. The roster is the "left" table, and because you insisted on keeping *every* roster student, this is a **LEFT JOIN**. The homework list is optional context bolted on to the left. Where it exists, you get a grade; where it doesn't, you get a blank — but the student never disappears.

An INNER JOIN (Topic 11) would have quietly dropped every student who didn't submit homework. LEFT JOIN's entire reason to exist is: **"keep every row on my side of the join, matched or not."**

RIGHT JOIN is the exact same idea, just with the roles flipped: keep every row from the table on the *right*. That's the whole story. LEFT and RIGHT are mirror images — and as we'll see, real engineers almost always pick LEFT and never look back.

---

## 2. Connection to SQL Internals

An OUTER JOIN is an INNER JOIN plus a **null-extension step**. Formally:

```
A LEFT JOIN B ON p  =  (A INNER JOIN B ON p)  ∪  { (a, NULL) | a ∈ A, no b ∈ B satisfies p(a, b) }
```

The engine computes the matched pairs exactly as it would for an INNER JOIN, then walks the outer side (`A`) a second time — conceptually — and for every outer row that produced **zero** matches, it emits one row with all of B's columns set to NULL. That NULL-extension is the defining mechanic. RIGHT JOIN is identical with A and B swapped.

At the physical layer, PostgreSQL implements this with the same three join algorithms as INNER JOIN, but each is taught to preserve unmatched outer rows:

1. **Nested Loop (Left) Join**: For each outer row, probe the inner. If the inner probe yields no match, still emit the outer row with NULLs. The planner marks this node `Nested Loop Left Join`.
2. **Hash (Left) Join**: Build a hash on the inner (B) side, scan the outer (A) side probing the hash. A "matched" flag rides along; unmatched outer rows are emitted null-extended. Node: `Hash Left Join`.
3. **Merge (Left) Join**: Sort both sides, walk in lockstep, and when the outer key has no counterpart in the inner stream, emit the outer row with NULLs. Node: `Merge Left Join`.

Two internal consequences matter enormously:

- **Join order is no longer free.** INNER JOIN is commutative and associative, so the planner reorders joins at will (Topic 11, §6.5). OUTER JOIN is **not** commutative — `A LEFT JOIN B` ≠ `B LEFT JOIN A`. The planner's freedom to reorder is constrained by the `join_collapse_limit` and the outer-join semantics; it cannot arbitrarily swap the preserved side. This is why a chain of LEFT JOINs can produce worse plans than the equivalent INNER JOINs — the planner has fewer legal orderings to choose from.

- **The `ON` predicate and the `WHERE` predicate are no longer interchangeable.** For INNER JOIN they were logically equivalent. For OUTER JOIN they are semantically different operations at different phases of execution. This single fact is the source of the most common OUTER JOIN bug in production, covered in depth in §6.3 and §9.

RIGHT JOIN is not a distinct engine primitive — Postgres internally rewrites `A RIGHT JOIN B` into `B LEFT JOIN A` before planning. There is no `Hash Right Join`-with-different-semantics; you'll literally see the planner flip the sides. That's your first hint that RIGHT JOIN is syntactic sugar you rarely need.

---

## 3. Logical Execution Order Context

```
FROM tableA
LEFT JOIN tableB ON <join-condition>   ← evaluated in the FROM/JOIN phase, produces null-extended rows
WHERE <filter>                          ← applied AFTER the join, to the null-extended result set
GROUP BY ...
HAVING ...
SELECT ...
DISTINCT ...
window functions ...
ORDER BY ...
LIMIT ...
```

The critical timeline for LEFT JOIN:

1. **JOIN phase** runs first. The `ON` condition decides *which B rows match each A row*, and *which A rows get null-extended*. Filters placed in `ON` participate in match-making, not row-elimination — a non-matching `ON` predicate produces NULLs, it does not delete the A row.

2. **WHERE phase** runs *after* the null-extended result already exists. Any predicate here is applied to rows that may already contain NULLs from B. A predicate like `WHERE b.status = 'active'` is evaluated against those NULLs — and `NULL = 'active'` is `UNKNOWN`, which WHERE treats as "reject." So every null-extended row is silently discarded, **collapsing your LEFT JOIN back into an INNER JOIN.**

This ordering is the entire ballgame. Compare with INNER JOIN, where (Topic 11, §3) moving a predicate between `ON` and `WHERE` changed nothing. Here it changes everything:

| Predicate on B goes in… | Effect |
|---|---|
| `ON` | Restricts *what counts as a match*; unmatched A rows survive with NULLs. LEFT JOIN semantics preserved. |
| `WHERE` | Filters the *post-join* rowset; null-extended rows fail the test and vanish. LEFT JOIN degrades to INNER JOIN. |

Remember from Topic 09 (WHERE fundamentals): WHERE keeps only rows where the predicate is `TRUE`, discarding both `FALSE` and `UNKNOWN`. That "discard UNKNOWN" rule is precisely what destroys the outer rows. Hold this thought — it recurs in every section below.

---

## 4. What Is LEFT JOIN and RIGHT JOIN?

A **LEFT JOIN** (a.k.a. `LEFT OUTER JOIN`) returns every row from the left table, paired with matching rows from the right table where the `ON` condition holds; where no right-side match exists, the right table's columns are returned as `NULL`. A **RIGHT JOIN** does the mirror image, preserving every row from the right table.

```sql
SELECT column_list
FROM table_a a                       -- the LEFT table (preserved in full)
LEFT [OUTER] JOIN table_b b          -- the RIGHT table (optional; NULLs where absent)
  ON a.key = b.key;                  -- match condition
--     │            │
--     │            └── right-side key; when no b satisfies this, b.* becomes NULL
--     └── left-side key; every distinct a row is guaranteed to appear at least once
```

Annotated breakdown of every clause and keyword:

```sql
SELECT
  a.id,                              -- always populated (left row always present)
  b.score                            -- NULL when this a-row had no matching b-row
FROM table_a AS a                    -- FROM names the preserved (left) relation
LEFT                                 -- ← the direction: preserve the LEFT input
     OUTER                           -- ← optional noise word; LEFT JOIN == LEFT OUTER JOIN
           JOIN table_b AS b         -- ← the optional (right) relation
  ON b.a_id = a.id                   -- ← ON: the match predicate, evaluated in JOIN phase
                                     --    lives here → controls matching, keeps unmatched A
                                     --    (contrast: putting b-filters in WHERE kills outer rows)
;
```

For RIGHT JOIN, only the direction word changes:

```sql
SELECT a.id, b.score
FROM table_a a
RIGHT [OUTER] JOIN table_b b         -- ← preserve the RIGHT input (table_b) in full
  ON b.a_id = a.id;                  -- every b row appears; a.* is NULL where unmatched
```

`OUTER` is a pure noise word — `LEFT JOIN` and `LEFT OUTER JOIN` are byte-for-byte equivalent in behavior. Unlike INNER JOIN, the direction keyword (`LEFT`/`RIGHT`) is **mandatory** — there is no bare `OUTER JOIN` default.

### Exact Semantics

Given:

```
table_a (roster):     table_b (homework):
id | name             a_id | score
---+--------          -----+------
1  | Alice            1    | 95
2  | Bob              1    | 87
3  | Charlie          4    | 72   ← a_id=4 does not exist in table_a
```

```sql
SELECT a.id, a.name, b.score
FROM table_a a
LEFT JOIN table_b b ON b.a_id = a.id;
```

```
id | name    | score
---+---------+------
1  | Alice   | 95      ← Alice matched twice…
1  | Alice   | 87      ← …so Alice appears twice
2  | Bob     | NULL    ← Bob had no homework → kept, score NULL
3  | Charlie | NULL    ← Charlie had no homework → kept, score NULL
```

Read this carefully against the INNER JOIN result for the *same data* (Topic 11, §4), which returned only the two Alice rows. The differences:

- **Bob and Charlie survive.** INNER dropped them; LEFT keeps them with `score = NULL`.
- **The orphan `a_id = 4` row in B is still gone.** LEFT JOIN preserves the *left* table's unmatched rows, not the right's. That B row has no home in A and disappears — exactly as it did under INNER JOIN. (To keep *both* sides' orphans you need FULL OUTER JOIN — Topic 13.)
- **Alice still multiplies.** Matching two B rows still yields two output rows. Outer-join-ness does not suppress fan-out; it only adds unmatched-row preservation on top.

The RIGHT JOIN mirror on the same data:

```sql
SELECT a.id, a.name, b.score
FROM table_a a
RIGHT JOIN table_b b ON b.a_id = a.id;
```

```
id   | name  | score
-----+-------+------
1    | Alice | 95
1    | Alice | 87
NULL | NULL  | 72     ← the orphan b-row (a_id=4) survives; a.* is NULL
```

Now Bob and Charlie are gone (they're on the un-preserved left side) and the orphan homework row survives with `a.id`/`a.name` set to NULL. This is precisely `table_b LEFT JOIN table_a` with the columns reordered — which is why RIGHT JOIN earns its reputation as the confusing, avoidable sibling.

The complete behavior:

- **Every row of the preserved side appears at least once**, matched or null-extended.
- **Unmatched preserved rows are null-extended** on all columns of the other side.
- **Unmatched rows of the *other* side are dropped** (that's what makes it a one-sided outer join, not a FULL join).
- **One-to-many matches still multiply** the preserved row.

---

## 5. Why LEFT JOIN Mastery Matters in Production

1. **The silent-INNER-JOIN bug is the single most common JOIN error in real codebases.** A `WHERE` predicate on the right table — often added months later by a different engineer for an unrelated reason — quietly converts a LEFT JOIN into an INNER JOIN and starts dropping rows. Nothing errors. Nothing warns. A dashboard that should show "all 10,000 users and their order counts" silently starts showing "the 6,200 users who have ordered." Revenue reports undercount. "Users with zero activity" reports return empty. Mastering the `ON` vs `WHERE` distinction is the difference between a correct report and a subtly, expensively wrong one.

2. **"Find the missing / orphaned / never-happened" queries depend on it.** *Which users never placed an order? Which products were never sold? Which employees have no manager assigned?* Every one of these is a LEFT JOIN with an `IS NULL` anti-join filter. There is no INNER-JOIN way to express "the absence of a match." This is a staple of both production analytics and interviews.

3. **Optional relationships model reality.** Not every order has a coupon. Not every user has a profile photo. Not every session converted to a payment. Modeling optional relationships with INNER JOIN throws away exactly the rows you care about. LEFT JOIN is how you say "attach this if it exists."

4. **RIGHT JOIN in someone else's code is a comprehension tax.** You *will* encounter RIGHT JOIN in legacy queries and generated ORM SQL. Being able to instantly rewrite `A RIGHT JOIN B` as `B LEFT JOIN A` in your head is essential for reading and reviewing SQL.

5. **NULLs from LEFT JOIN propagate into aggregates and arithmetic.** `SUM` skips NULLs, `COUNT(col)` skips NULLs, `COUNT(*)` doesn't, and `NULL + 5` is NULL. A LEFT JOIN followed by a careless aggregate produces wrong totals. Knowing which aggregate ignores which NULL is production-critical (§6.6).

---

## 6. Deep Technical Content

### 6.1 The Preserved Side and the Null-Extended Side

The mental model that never fails you:

- The **preserved side** (LEFT table in a LEFT JOIN, RIGHT table in a RIGHT JOIN) contributes **every** row, no matter what.
- The **optional side** contributes real values where the `ON` matches, and `NULL` for *all its columns* where it doesn't.

```sql
-- Every user, with their order count (0 for users who never ordered)
SELECT u.id, u.name, o.id AS order_id, o.total_amount
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
-- users with no orders: one row, order_id = NULL, total_amount = NULL
```

A user with zero orders yields exactly **one** row (null-extended). A user with three orders yields **three** rows. A user with orders is never null-extended — the null extension only happens when the match count is zero.

### 6.2 LEFT vs RIGHT — The Asymmetry, and Why LEFT Wins

`A LEFT JOIN B` and `A RIGHT JOIN B` are *not* the same query — they preserve opposite sides. But `A RIGHT JOIN B` **is** identical to `B LEFT JOIN A` (modulo column ordering in `SELECT *`):

```sql
-- These three produce the same rows:
SELECT * FROM orders o RIGHT JOIN users u ON o.user_id = u.id;   -- preserve users
SELECT * FROM users u  LEFT  JOIN orders o ON o.user_id = u.id;  -- preserve users
-- (column order differs between the two SELECT * forms, but the row set is identical)
```

Because every RIGHT JOIN can be mechanically rewritten as a LEFT JOIN by swapping the table order, and because humans read top-to-bottom / left-to-right, the universal style convention is: **always write LEFT JOIN; put the table you want to preserve first.** Reasons LEFT is strictly better as a convention:

- **Readability**: the preserved (most important) table is named first, matching the reading order.
- **Chainability**: `A LEFT JOIN B LEFT JOIN C` reads as "start with A, optionally attach B, optionally attach C." A mix of LEFT and RIGHT in one query forces you to mentally track which side each join preserves — a frequent source of bugs.
- **Consistency with the planner**: since Postgres rewrites RIGHT into LEFT internally anyway, writing LEFT keeps your mental model aligned with the physical plan.

The only time RIGHT JOIN is defensible is deep inside a mechanically generated query where flipping table order is inconvenient. In hand-written SQL, treat RIGHT JOIN as a code smell.

### 6.3 The Classic Bug — WHERE on the Optional Table Silently Downgrades to INNER JOIN

This is the headline gotcha of the entire topic. Study it until it's reflexive.

```sql
-- INTENT: every user, plus their count of completed orders (0 if none)
-- BUG: the filter on the optional table is in WHERE
SELECT u.id, u.name, COUNT(o.id) AS completed_orders
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.status = 'completed'          -- ← disaster
GROUP BY u.id, u.name;
```

What actually happens, phase by phase:

1. JOIN phase: LEFT JOIN correctly produces every user, null-extending users with no orders (`o.status = NULL`, `o.id = NULL`).
2. WHERE phase: `o.status = 'completed'` is evaluated. For a null-extended row, `o.status` is `NULL`, so `NULL = 'completed'` → `UNKNOWN` → **row discarded**. Every user with zero orders is thrown away. Users whose only orders are non-completed are *also* thrown away.
3. Result: only users with at least one completed order remain. Your "every user" report now silently excludes everyone else. **You have written an INNER JOIN with extra steps.**

The fix is to move the optional-table predicate into the `ON` clause, where it participates in match-making instead of post-join filtering:

```sql
-- CORRECT: predicate on optional table lives in ON
SELECT u.id, u.name, COUNT(o.id) AS completed_orders
FROM users u
LEFT JOIN orders o
  ON o.user_id = u.id
  AND o.status = 'completed'          -- ← part of the match condition
GROUP BY u.id, u.name;
-- Users with no completed orders: kept, COUNT(o.id) = 0
```

Now `o.status = 'completed'` decides *which orders count as a match*. A user with no completed orders simply has zero matches and is null-extended (surviving with `COUNT(o.id) = 0`, because `COUNT(col)` ignores the NULL). The LEFT JOIN semantics are preserved.

**The rule, memorized:** *A filter on the optional (null-extended) table belongs in `ON`. A filter on the preserved table belongs in `WHERE`.* Putting an optional-table filter in `WHERE` is almost always a bug — unless you deliberately want INNER-JOIN semantics, in which case you should write INNER JOIN and say what you mean.

There is one legitimate `WHERE` use on the optional table — the intentional anti-join (§6.4). The distinguishing feature: it filters *for* `IS NULL`, not for a specific value.

### 6.4 The Anti-Join — "Find Rows With No Match"

The canonical, correct use of a `WHERE` clause on the optional table is `IS NULL` to find *unmatched* preserved rows. This is called an **anti-join**.

```sql
-- Users who have NEVER placed an order
SELECT u.id, u.name, u.email
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;                    -- keep only the null-extended (unmatched) rows
```

How it works: the LEFT JOIN null-extends users with no orders (`o.id = NULL`). The `WHERE o.id IS NULL` then keeps *exactly* those rows and discards everyone who matched. You are deliberately filtering for the absence of a match.

Critical detail: the `IS NULL` check must be on a column that is **guaranteed non-NULL for matched rows** — a primary key or a `NOT NULL` column of the optional table. If you check a nullable column, you'll also catch rows that matched but happen to have a NULL there:

```sql
-- WRONG: coupon_code is nullable even for matched orders
SELECT u.id FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.coupon_code IS NULL;   -- catches users-with-no-orders AND users-whose-orders-had-no-coupon

-- RIGHT: o.id is the PK, non-NULL for every real order row
WHERE o.id IS NULL;
```

Anti-joins pair naturally with `NOT EXISTS` (Topic 27), which the planner often executes identically. The LEFT-JOIN-IS-NULL form is the classic; `NOT EXISTS` is frequently clearer and handles the "nullable column" trap automatically.

### 6.5 Chaining Multiple LEFT JOINs

LEFT JOINs chain left-to-right, and **once a row is on the null-extended path, it stays there** through subsequent LEFT JOINs.

```sql
SELECT u.id, u.name, o.id AS order_id, oi.product_id, p.name AS product
FROM users u
LEFT JOIN orders o       ON o.user_id = u.id
LEFT JOIN order_items oi ON oi.order_id = o.id
LEFT JOIN products p     ON p.id = oi.product_id;
```

For a user with no orders: `o.*` is NULL. The next join `oi.order_id = o.id` becomes `oi.order_id = NULL` → `UNKNOWN` → no match → `oi.*` is NULL too. Then `p.id = NULL` → no match → `p.*` NULL. The user survives all the way through with a trailing string of NULLs. This is the correct and desired behavior: the optional chain gracefully null-extends the whole way down.

**Trap:** if *any* join in the chain is written as INNER JOIN (or bare `JOIN`), it acts as a filter that can prune the null-extended rows:

```sql
-- BUG: the INNER JOIN in the middle drops users with no orders
FROM users u
LEFT JOIN  orders o       ON o.user_id = u.id
INNER JOIN order_items oi ON oi.order_id = o.id   -- ← kills users with no orders
```

The INNER JOIN requires `oi.order_id = o.id` to be TRUE. For a null-extended user, `o.id` is NULL, so no `oi` matches, and INNER JOIN drops the row. One INNER JOIN in a LEFT JOIN chain poisons everything upstream of it. Keep the whole optional chain LEFT.

### 6.6 LEFT JOIN and Aggregation — NULL-Aware Counting

After a LEFT JOIN, the choice of aggregate function determines whether unmatched rows count as "zero" or get lost.

```sql
SELECT
  u.id,
  u.name,
  COUNT(*)          AS count_star,     -- counts the null-extended row too → min 1 for every user
  COUNT(o.id)       AS count_orders,   -- ignores NULL o.id → correct 0 for users with no orders
  COALESCE(SUM(o.total_amount), 0) AS revenue  -- SUM skips NULLs; COALESCE turns all-NULL into 0
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;
```

The three behaviors you must know cold:

| Expression | For a user with zero orders | Why |
|---|---|---|
| `COUNT(*)` | `1` | Counts rows, and the null-extended row is a row. **Almost always wrong** for "how many orders." |
| `COUNT(o.id)` | `0` | `COUNT(col)` ignores NULLs; the single null-extended row contributes nothing. **Correct.** |
| `SUM(o.total_amount)` | `NULL` | `SUM` over an all-NULL group is `NULL`, not `0`. Wrap in `COALESCE(..., 0)`. |
| `AVG(o.total_amount)` | `NULL` | Same — `AVG` of no non-NULL values is `NULL`. |
| `MAX(o.created_at)` | `NULL` | No non-NULL inputs → `NULL`. Often acceptable ("no last order date"). |

The single most common LEFT-JOIN aggregation bug is using `COUNT(*)` where `COUNT(o.id)` is meant — it reports `1` for every user who has never ordered, making "users with no orders" indistinguishable from "users with one order." Always count a non-NULL column of the optional table.

### 6.7 The Fan-Out-Then-Aggregate Trap Still Applies

Everything from Topic 11, §6.8 about row multiplication carries over, *plus* the NULL wrinkle. Joining a user to two one-to-many tables multiplies as before:

```sql
-- Users, their order count, and their review count — WRONG
SELECT u.id,
       COUNT(DISTINCT o.id)  AS orders,
       COUNT(DISTINCT r.id)  AS reviews
FROM users u
LEFT JOIN orders  o ON o.user_id = u.id
LEFT JOIN reviews r ON r.user_id = u.id
GROUP BY u.id;
```

Here two independent one-to-many LEFT JOINs form a *cross product per user*: a user with 3 orders and 4 reviews yields 12 rows. `COUNT(DISTINCT o.id)` and `COUNT(DISTINCT r.id)` are correct (they de-duplicate), but `SUM(o.total_amount)` would be quadruple-counted. The robust fix, as with INNER JOIN, is to pre-aggregate each optional table separately, then LEFT JOIN the summaries (see §8, Example 3).

### 6.8 Self-LEFT-JOIN — Preserving Rows With No Related Row

Remember from Topic 11, §6.9 that a self-INNER-JOIN drops top-level rows. The self-LEFT-JOIN keeps them:

```sql
-- Every employee with their manager's name; top-level employees (no manager) still appear
SELECT e.name AS employee, m.name AS manager
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id;
-- e with manager_id IS NULL → m.* is NULL → row survives with manager = NULL
```

The INNER version silently dropped the CEO (no manager). The LEFT version keeps every employee, with `manager = NULL` for those at the top. This is the correct way to build an org chart that includes the root.

### 6.9 LEFT JOIN with a Filtered / Correlated Optional Side

You can combine multiple conditions in `ON`, including constants and correlated predicates, all evaluated during match-making:

```sql
-- Every product, plus its price from the currently-active price list (if any)
SELECT p.id, p.name, pl.price
FROM products p
LEFT JOIN price_list pl
  ON pl.product_id = p.id
  AND pl.effective_from <= CURRENT_DATE
  AND (pl.effective_to IS NULL OR pl.effective_to > CURRENT_DATE)
  AND pl.currency = 'USD';
-- Products with no active USD price → pl.price = NULL, product still listed
```

Every one of those `AND` conditions in the `ON` refines *what counts as a match*. A product with no matching active price row is null-extended — it stays in the result with `price = NULL`, which is exactly what "show all products, price if we have one" requires. Moving any of these into `WHERE` would drop the priceless products.

### 6.10 Interaction with DISTINCT, ORDER BY, and LIMIT

- **DISTINCT** after a LEFT JOIN collapses duplicate preserved rows but keeps null-extended rows (a null-extended row is still distinct from a matched one). `SELECT DISTINCT u.id FROM users u LEFT JOIN orders o …` returns every user id exactly once — the LEFT JOIN guarantees each appears, DISTINCT removes the fan-out duplicates.
- **ORDER BY** on an optional column sorts NULLs to the end by default (`NULLS LAST` for `ASC` is *not* the default in Postgres — Postgres puts NULLs *last* for `ASC` and *first* for `DESC`; use explicit `NULLS FIRST/LAST` when it matters). A LEFT JOIN's null-extended rows therefore cluster at one end.
- **LIMIT** interacts subtly: because null-extended rows are real rows, `LIMIT 10` on a LEFT JOIN can return rows that are entirely NULL on the optional side. Don't assume `LIMIT` gives you "10 matched rows" — it gives you 10 result rows, matched or not.

### 6.11 NULLs in the Join Key Itself

If the *preserved* side has a NULL join key, it can never match (NULL = anything is UNKNOWN), so it's always null-extended — it survives, but never joins to anything. If the *optional* side has NULL join keys, those optional rows simply never participate in any match (and, being on the droppable side, contribute nothing). This mirrors Topic 07's three-valued logic: `NULL` join keys are inert in equality matching regardless of join type; the difference is only whether the *carrying* row is preserved.

---

## 7. EXPLAIN — LEFT JOIN in the Plan

### Hash Left Join (the common case)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.id, u.name, o.id AS order_id, o.total_amount
FROM users u
LEFT JOIN orders o ON o.user_id = u.id;
```

```
Hash Left Join  (cost=38.00..3120.00 rows=42000 width=60)
                (actual time=0.9..34.7 rows=42000 loops=1)
  Hash Cond: (u.id = o.user_id)
  ->  Seq Scan on users u  (cost=0.00..320.00 rows=12000 width=40)
                           (actual time=0.01..3.1 rows=12000 loops=1)
  ->  Hash  (cost=1730.00..1730.00 rows=30000 width=20)
             (actual time=0.8..0.8 rows=30000 loops=1)
        Buckets: 32768  Batches: 1  Memory Usage: 1690kB
        ->  Seq Scan on orders o  (cost=0.00..1730.00 rows=30000 width=20)
                                   (actual time=0.01..7.9 rows=30000 loops=1)
Buffers: shared hit=2050
Planning Time: 0.4 ms
Execution Time: 36.2 ms
```

**Reading it:**
- The node is `Hash Left Join`, not `Hash Join` — the planner has preserved the outer (`users`) side. Every user appears in the output regardless of whether the hash probe finds a matching order.
- `Hash Cond: (u.id = o.user_id)` — the equi-join predicate. The **optional** table (`orders`) is hashed here; the **preserved** table (`users`) is scanned and probes the hash, so unmatched users flow straight through null-extended.
- `rows=42000` output vs `rows=12000` users: there's fan-out (users with multiple orders) *plus* null-extension (users with none), netting more output rows than users but every user is represented.
- `Batches: 1` — hash fits in `work_mem`, no disk spill.

### Nested Loop Left Join (small preserved side, indexed optional side)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.id, u.name, o.id, o.total_amount
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.id = ANY(ARRAY[10, 20, 30]);
```

```
Nested Loop Left Join  (cost=0.43..48.6 rows=14 width=60)
                       (actual time=0.05..0.19 rows=11 loops=1)
  ->  Index Scan using users_pkey on users u  (cost=0.29..12.9 rows=3 width=40)
        Index Cond: (id = ANY ('{10,20,30}'::integer[]))
        (actual rows=3 loops=1)
  ->  Index Scan using orders_user_id_idx on orders o
        Index Cond: (user_id = u.id)
        (actual rows=4 loops=3)
Buffers: shared hit=27
Planning Time: 0.3 ms
Execution Time: 0.24 ms
```

**Reading it:**
- `Nested Loop Left Join` — for each of the 3 users, probe `orders` by the `user_id` index (`loops=3`).
- Crucially, even if a user's probe returned **zero** rows, the Left Join node still emits that user once, null-extended. That's why `actual rows=11` for 3 users can include a user with 0 orders contributing 1 row.
- All index access, tiny buffers — the ideal shape for a small preserved set with an indexed optional side.

### Merge Left Join (both sides sorted on the key)

```sql
EXPLAIN (ANALYZE)
SELECT e.name, m.name AS manager
FROM employees e
LEFT JOIN employees m ON m.id = e.manager_id
ORDER BY e.manager_id;
```

```
Merge Left Join  (cost=0.57..640.00 rows=5000 width=64)
                 (actual time=0.06..5.8 rows=5000 loops=1)
  Merge Cond: (e.manager_id = m.id)
  ->  Index Scan using employees_manager_id_idx on employees e
        (actual rows=5000 loops=1)
  ->  Index Scan using employees_pkey on employees m
        (actual rows=4998 loops=1)
```

**Reading it:**
- `Merge Left Join` walks both sorted index streams. When an `e.manager_id` is NULL (top-level employee), it has no counterpart in the `m` stream, and the node emits that employee null-extended.
- No Sort nodes — both inputs arrive pre-sorted from indexes. Efficient for large preserved inputs.

### The Silent-INNER-JOIN Bug, Seen in EXPLAIN

When you write the buggy `WHERE o.status = 'completed'` on the optional table, the planner is smart enough to *notice* the join can no longer produce NULLs and **rewrites your Left Join into an Inner Join**:

```sql
EXPLAIN
SELECT u.id, COUNT(o.id)
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.status = 'completed'
GROUP BY u.id;
```

```
HashAggregate  (cost=...)
  ->  Hash Join  (cost=...)          ← NOTE: "Hash Join", NOT "Hash Left Join"
        Hash Cond: (o.user_id = u.id)
        ->  Seq Scan on orders o
              Filter: (status = 'completed')
        ->  Seq Scan on users u
```

The node reads `Hash Join`, not `Hash Left Join` — Postgres has formally proven your outer join is really an inner join and planned it as such. **This is your smoking gun in EXPLAIN:** if you wrote `LEFT JOIN` but the plan says plain `Hash Join` / `Nested Loop` / `Merge Join` (no "Left"), a `WHERE` predicate has collapsed your outer join. Diff the plan node type against your SQL to catch the bug.

### What to Look For

| Symptom | Likely Problem | Fix |
|---|---|---|
| Plan says `Hash Join` but SQL says `LEFT JOIN` | `WHERE` on optional table collapsed it to INNER | Move predicate to `ON` |
| `Nested Loop Left Join` + `Seq Scan` on optional side | No index on optional table's join column | Add index on optional FK |
| Output rows ≫ preserved-table rows | Fan-out from one-to-many optional side | Pre-aggregate before joining |
| `Merge Left Join` with Sort nodes | Inputs not pre-sorted | Add index on join keys |
| `COUNT(*)` shows min 1 for "empty" groups | Counting the null-extended row | Use `COUNT(optional.id)` |

---

## 8. Query Examples

### Example 1 — Basic: Every User With Their Order Count

```sql
-- Show all users, including those who have never ordered (count = 0)
SELECT
  u.id,
  u.name,
  u.email,
  COUNT(o.id)                       AS order_count,     -- COUNT(col) → 0 for no-order users
  COALESCE(SUM(o.total_amount), 0)  AS lifetime_value   -- COALESCE turns all-NULL SUM into 0
FROM users u
LEFT JOIN orders o ON o.user_id = u.id                  -- preserve every user
GROUP BY u.id, u.name, u.email
ORDER BY lifetime_value DESC;
```

### Example 2 — Intermediate: Anti-Join for "Products Never Sold"

```sql
-- Products that have never appeared in any order item
SELECT
  p.id,
  p.name,
  p.category_id,
  p.created_at
FROM products p
LEFT JOIN order_items oi ON oi.product_id = p.id
WHERE oi.id IS NULL                    -- oi.id is the PK → non-NULL only for real matches
ORDER BY p.created_at DESC;
-- Every product with zero order_items rows survives; all others are filtered out
```

### Example 3 — Production Grade: User Dashboard With Pre-Aggregated Optional Sides

```sql
-- User engagement dashboard: every user, with order + payment + session metrics.
-- Table sizes: users 2M rows, orders 15M rows, payments 14M rows, sessions 40M rows.
-- Indexes assumed: orders(user_id), payments(user_id), sessions(user_id, started_at).
-- Strategy: pre-aggregate each optional table to ONE row per user BEFORE joining, so the
--   LEFT JOINs are strictly 1:1 (no fan-out, no cross-product explosion).
-- Expectation: three index-driven aggregations feeding a 1:1 join fan-in; ~300-600ms on
--   warm cache for the full 2M users, far cheaper with a WHERE on users to scope it.
WITH order_stats AS (
  SELECT user_id,
         COUNT(*)                       AS order_count,
         COALESCE(SUM(total_amount), 0) AS total_spend,
         MAX(created_at)                AS last_order_at
  FROM orders
  WHERE status = 'completed'
  GROUP BY user_id
),
payment_stats AS (
  SELECT user_id,
         COUNT(*)                       AS payment_count,
         COALESCE(SUM(amount), 0)       AS total_paid
  FROM payments
  WHERE state = 'captured'
  GROUP BY user_id
),
session_stats AS (
  SELECT user_id,
         COUNT(*)                       AS session_count,
         MAX(started_at)                AS last_seen_at
  FROM sessions
  WHERE started_at >= CURRENT_DATE - INTERVAL '90 days'
  GROUP BY user_id
)
SELECT
  u.id,
  u.name,
  u.email,
  COALESCE(os.order_count, 0)    AS orders,
  COALESCE(os.total_spend, 0)    AS spend,
  os.last_order_at,
  COALESCE(ps.payment_count, 0)  AS payments,
  COALESCE(ps.total_paid, 0)     AS paid,
  COALESCE(ss.session_count, 0)  AS sessions_90d,
  ss.last_seen_at,
  CASE
    WHEN ss.last_seen_at IS NULL                       THEN 'dormant'
    WHEN os.order_count IS NULL OR os.order_count = 0  THEN 'browsing'
    ELSE 'active'
  END AS segment
FROM users u
LEFT JOIN order_stats   os ON os.user_id = u.id        -- 1:1, no fan-out
LEFT JOIN payment_stats ps ON ps.user_id = u.id        -- 1:1
LEFT JOIN session_stats ss ON ss.user_id = u.id        -- 1:1
ORDER BY spend DESC
LIMIT 100;
```

```
Limit  (cost=210500.00..210500.25 rows=100 width=120)
  ->  Sort  (cost=210500.00..215500.00 rows=2000000 width=120)
        Sort Key: (COALESCE(os.total_spend, '0')) DESC
        ->  Hash Left Join  (cost=88000.00..150000.00 rows=2000000 width=120)
              Hash Cond: (u.id = ss.user_id)
              ->  Hash Left Join  (cost=60000.00..100000.00 rows=2000000 width=96)
                    Hash Cond: (u.id = ps.user_id)
                    ->  Hash Left Join  (cost=32000.00..60000.00 rows=2000000 width=72)
                          Hash Cond: (u.id = os.user_id)
                          ->  Seq Scan on users u  (rows=2000000)
                          ->  Hash  (rows=1200000)
                                ->  Subquery Scan on order_stats  (rows=1200000)
                                      ->  HashAggregate  (rows=1200000)
                                            ->  Index Scan ... on orders
                    ->  Hash  (rows=1100000)  ->  HashAggregate on payments ...
              ->  Hash  (rows=900000)  ->  HashAggregate on sessions ...
```

Every join is `Hash Left Join` (LEFT preserved throughout), each optional side is a pre-aggregated 1:1 subquery, and there is no fan-out multiplication anywhere in the plan.

---

## 9. Wrong → Right Patterns

### Wrong 1: Optional-Table Filter in WHERE (the signature bug)

```sql
-- WRONG: intends "all users + their completed-order count", silently drops users with none
SELECT u.id, u.name, COUNT(o.id) AS completed_orders
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.status = 'completed'          -- collapses LEFT JOIN → INNER JOIN
GROUP BY u.id, u.name;
-- WRONG RESULT: only users with ≥1 completed order appear
```

Why it's wrong at the execution level: the LEFT JOIN null-extends order-less users (`o.status = NULL`); the WHERE then evaluates `NULL = 'completed'` → `UNKNOWN` → discards them. EXPLAIN confirms it by showing `Hash Join` instead of `Hash Left Join`.

```sql
-- RIGHT: filter on the optional table goes in ON
SELECT u.id, u.name, COUNT(o.id) AS completed_orders
FROM users u
LEFT JOIN orders o
  ON o.user_id = u.id
  AND o.status = 'completed'
GROUP BY u.id, u.name;
-- Users with no completed orders survive with completed_orders = 0
```

### Wrong 2: COUNT(*) Instead of COUNT(optional_column)

```sql
-- WRONG: reports 1 for users who have never ordered
SELECT u.id, u.name, COUNT(*) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;
-- WRONG RESULT: a user with zero orders shows order_count = 1 (the null-extended row)
```

Why it's wrong: `COUNT(*)` counts rows, and the null-extended row *is* a row. The order-less user's single NULL row is counted as 1.

```sql
-- RIGHT: count a non-NULL column of the optional table
SELECT u.id, u.name, COUNT(o.id) AS order_count
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.name;
-- COUNT(o.id) ignores the NULL → order-less users correctly show 0
```

### Wrong 3: Anti-Join on a Nullable Column

```sql
-- WRONG: shipped_at is NULL for un-shipped (but existing) orders too
SELECT u.id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.shipped_at IS NULL;
-- WRONG RESULT: returns users-with-no-orders AND users-whose-orders-are-unshipped
```

Why it's wrong: the `IS NULL` is meant to detect "no matching order," but `shipped_at` is legitimately NULL on matched-but-unshipped orders, so those get swept in too.

```sql
-- RIGHT: test the PK (or any NOT NULL column) of the optional table
SELECT u.id
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.id IS NULL;   -- o.id is non-NULL for every real match → clean anti-join
```

### Wrong 4: INNER JOIN Poisoning a LEFT JOIN Chain

```sql
-- WRONG: wanted all users; the second join is INNER and prunes order-less users
SELECT u.id, u.name, oi.product_id
FROM users u
LEFT JOIN  orders o       ON o.user_id = u.id
INNER JOIN order_items oi ON oi.order_id = o.id;   -- drops every user with no orders
```

Why it's wrong: for an order-less user, `o.id` is NULL; the INNER JOIN's `oi.order_id = o.id` is `UNKNOWN`, so no `oi` matches and INNER JOIN removes the row entirely — undoing the LEFT JOIN above it.

```sql
-- RIGHT: keep the whole optional chain LEFT
SELECT u.id, u.name, oi.product_id
FROM users u
LEFT JOIN orders o       ON o.user_id = u.id
LEFT JOIN order_items oi ON oi.order_id = o.id;   -- order-less users survive with NULLs
```

### Wrong 5: Confusing RIGHT JOIN Direction

```sql
-- WRONG: developer wanted "all users, orders optional" but wrote RIGHT JOIN
SELECT u.id, u.name, o.id
FROM orders o
RIGHT JOIN users u ON u.id = o.user_id
WHERE o.total_amount > 100;           -- and then filtered the optional (orders) side in WHERE
-- Double problem: the mental model is inverted AND the WHERE collapses it to INNER
```

Why it's wrong: RIGHT JOIN here preserves `users` (the right table), which happens to be the intent — but it's confusing to read, and the `WHERE o.total_amount > 100` on the optional `orders` side collapses it back to INNER JOIN anyway.

```sql
-- RIGHT: write it as LEFT with the preserved table first, filter optional side in ON
SELECT u.id, u.name, o.id
FROM users u
LEFT JOIN orders o
  ON o.user_id = u.id
  AND o.total_amount > 100;
-- Clear reading order; users with no >100 order survive with o.id = NULL
```

---

## 10. Performance Profile

LEFT/RIGHT JOIN uses the same three physical algorithms as INNER JOIN, with modest overhead for tracking unmatched rows. The cost differences worth knowing:

| Algorithm | LEFT-JOIN behavior | Extra cost vs INNER |
|---|---|---|
| Nested Loop Left Join | Emit outer row even on empty inner probe | Negligible — one branch per outer row |
| Hash Left Join | Track "matched" flag per outer row; emit unmatched null-extended | Small — a bitmap/flag per probe |
| Merge Left Join | Emit outer row when inner stream has no counterpart | Negligible |

### Scaling at 1M / 10M / 100M Rows

| Scenario | Behavior |
|---|---|
| 1M preserved LEFT JOIN 1M optional, indexed FK, Hash Left Join | ~200–500ms; hash the optional side, one pass over preserved |
| 10M preserved, optional side too big to hash in `work_mem` | `Batches > 1` disk spill → 3–8× slower; raise `work_mem` or pre-aggregate |
| 100M preserved, LEFT JOIN with fan-out one-to-many | Output can be *larger* than 100M; often the real cost is the downstream Sort/Aggregate, not the join |
| Anti-join (LEFT … WHERE opt.id IS NULL) at 10M×10M | Hash Left (Anti) Join ~ same cost as the equi-join; planner may pick `Hash Anti Join` explicitly |

### The Join-Order Constraint (LEFT JOIN's Hidden Tax)

Because OUTER JOIN is not commutative, the planner has **fewer legal join orders** than for the equivalent all-INNER query. A long chain of LEFT JOINs can force a suboptimal order that an all-INNER query would have avoided. Consequences and mitigations:

- If you *know* a relationship is mandatory (FK is `NOT NULL` and always present), use INNER JOIN — it gives the planner more reordering freedom and can be faster.
- `join_collapse_limit` (default 8) bounds how many joins the planner reorders; beyond it, join order follows your written order. For big LEFT-JOIN chains, the *order you write* increasingly becomes the order that runs — put the most selective / smallest preserved anchor first.
- Pre-aggregating optional one-to-many tables into 1:1 subqueries (Example 3) both eliminates fan-out and simplifies the join graph, usually the biggest single win.

### Index Rules Specific to LEFT JOIN

- The **optional** (null-extended) table almost always wants an index on its join column — it's the side being probed, and a Seq Scan there under a Nested Loop Left Join is the classic slow plan.
- For the anti-join pattern, the same optional-side FK index lets the planner use a `Hash Anti Join` or index-driven `Nested Loop Anti Join` efficiently.
- The preserved side benefits from an index only when a `WHERE` on the *preserved* table scopes it (e.g. `WHERE u.created_at > …`) — index that predicate, not the join column.

---

## 11. Node.js Integration

### 11.1 Basic LEFT JOIN With Zero-Safe Aggregates

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Every user with order stats; users with no orders come back with 0s, not missing
async function getUsersWithOrderStats(limit = 100) {
  const { rows } = await pool.query(
    `SELECT
       u.id,
       u.name,
       u.email,
       COUNT(o.id)                      AS order_count,
       COALESCE(SUM(o.total_amount), 0) AS lifetime_value,
       MAX(o.created_at)                AS last_order_at
     FROM users u
     LEFT JOIN orders o ON o.user_id = u.id
     GROUP BY u.id, u.name, u.email
     ORDER BY lifetime_value DESC
     LIMIT $1`,
    [limit]
  );
  // NB: last_order_at is null for users who never ordered — handle in the app layer
  return rows;
}
```

### 11.2 Optional-Table Filter Belongs in ON (parameterized)

```javascript
// Every user + count of their COMPLETED orders (0 if none). The status filter is in ON,
// so users with no completed orders are preserved.
async function completedOrderCounts(status = 'completed') {
  const { rows } = await pool.query(
    `SELECT u.id, u.name, COUNT(o.id) AS completed_orders
     FROM users u
     LEFT JOIN orders o
       ON o.user_id = u.id
       AND o.status = $1          -- $1 in ON, NOT in WHERE — preserves LEFT semantics
     GROUP BY u.id, u.name
     ORDER BY completed_orders DESC`,
    [status]
  );
  return rows;
}
```

### 11.3 Anti-Join — Users Who Never Ordered

```javascript
async function usersWithNoOrders() {
  const { rows } = await pool.query(
    `SELECT u.id, u.name, u.email, u.created_at
     FROM users u
     LEFT JOIN orders o ON o.user_id = u.id
     WHERE o.id IS NULL           -- PK IS NULL → clean anti-join
     ORDER BY u.created_at DESC`
  );
  return rows;
}
```

### 11.4 Reading NULLs From the Optional Side

```javascript
// The pg driver maps SQL NULL → JavaScript null. Guard every optional-side column.
async function userProfileCard(userId) {
  const { rows } = await pool.query(
    `SELECT
       u.id,
       u.name,
       p.avatar_url,          -- profiles is optional: avatar_url may be null
       p.bio,
       o.id           AS last_order_id,
       o.total_amount AS last_order_amount
     FROM users u
     LEFT JOIN profiles p ON p.user_id = u.id
     LEFT JOIN LATERAL (
       SELECT id, total_amount FROM orders
       WHERE user_id = u.id ORDER BY created_at DESC LIMIT 1
     ) o ON true                -- LATERAL LEFT JOIN: "the latest order, if any"
     WHERE u.id = $1`,
    [userId]
  );
  const r = rows[0];
  if (!r) return null;
  return {
    id: r.id,
    name: r.name,
    avatarUrl: r.avatar_url ?? '/default-avatar.png',   // null → default
    bio: r.bio ?? '',
    lastOrder: r.last_order_id
      ? { id: r.last_order_id, amount: Number(r.last_order_amount) }
      : null,                                            // no order → null, not undefined
  };
}
```

Note the `LEFT JOIN LATERAL … ON true` idiom — a common, powerful pattern for "attach the single most-recent related row, or NULL if none." The `ON true` makes it a pure LATERAL correlation; the LEFT keeps users with no orders.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do LEFT JOIN?** — Yes, implicitly. Prisma generates a LEFT JOIN (or separate queries, per its relation-load strategy) for `include` on a **nullable/optional** relation, and for any `include` where you want the parent kept regardless of children.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// include on an optional relation → LEFT JOIN semantics: user kept even with no profile
const users = await prisma.user.findMany({
  include: {
    profile: true,        // optional relation → LEFT JOIN
    orders: {             // to-many: user kept even with zero orders (empty array)
      where: { status: 'completed' },   // filter lives on the relation → ON-like semantics
    },
  },
});
// users with no completed orders come back with orders: []  (not dropped)

// Anti-join ("users with no orders") — Prisma expresses it via `none`
const noOrderUsers = await prisma.user.findMany({
  where: { orders: { none: {} } },   // compiles to a NOT EXISTS / anti-join
});
```

**Where it breaks:** Prisma's `include … where` on a to-many relation correctly behaves like an `ON` filter (parent preserved) — but it does **not** let you express aggregate-with-LEFT-JOIN in one query with zero-fill; you use `_count` or `$queryRaw`. There's no way to say "LEFT JOIN and COUNT(o.id) as a scalar with COALESCE" without raw SQL or a `_count` selector.

**Verdict:** Excellent for object-shaped optional relations (`include`) and for anti-joins (`none`). Drop to `$queryRaw` for LEFT-JOIN aggregations with zero-fill.

---

### Drizzle ORM

**Can Drizzle do LEFT JOIN?** — Yes, explicitly and typed. `leftJoin` makes the joined columns nullable in the TypeScript result type — a genuinely nice safety feature.

```typescript
import { db } from './db';
import { users, orders } from './schema';
import { eq, and, isNull, sql } from 'drizzle-orm';

// Every user + completed-order count; status filter in the ON (via and())
const rows = await db
  .select({
    userId: users.id,
    name: users.name,
    completedOrders: sql<number>`COUNT(${orders.id})`,   // COUNT(col) → 0 for no match
  })
  .from(users)
  .leftJoin(
    orders,
    and(eq(orders.userId, users.id), eq(orders.status, 'completed')),  // filter in ON
  )
  .groupBy(users.id, users.name);

// Anti-join: users with no orders
const noOrders = await db
  .select({ id: users.id, name: users.name })
  .from(users)
  .leftJoin(orders, eq(orders.userId, users.id))
  .where(isNull(orders.id));   // PK IS NULL → clean anti-join
```

Drizzle's type system marks `orders.*` fields as `T | null` in the result of a `leftJoin`, forcing you to handle the NULL case in TypeScript — it encodes the outer-join semantics in the type.

**Where it breaks:** nothing significant. Passing the optional-table filter to `.where()` instead of into the `and(...)` inside `leftJoin` reproduces the classic collapse-to-INNER bug — Drizzle won't stop you.

**Verdict:** Best-in-class. Explicit `leftJoin`, null-typed columns, filter-in-ON expressible via `and()`. Use confidently.

---

### Sequelize

**Can Sequelize do LEFT JOIN?** — Yes; LEFT OUTER JOIN is Sequelize's **default** for `include` (the inverse of the INNER-JOIN default trap from Topic 11).

```javascript
const { Op } = require('sequelize');
const { User, Order } = require('./models');

// Default include → LEFT OUTER JOIN: users kept even with no orders
const users = await User.findAll({
  include: [
    {
      model: Order,
      required: false,                    // false (default) = LEFT JOIN; true = INNER JOIN
      where: { status: 'completed' },     // ← DANGER: see below
    },
  ],
});
```

**Where it breaks — the big one:** putting a `where` on an `include` with `required: false` does **not** behave like an `ON` filter in the way you'd hope. Sequelize moves relation `where` conditions in a way that, in many versions, effectively forces `required: true` (INNER JOIN) *or* injects the condition into the outer WHERE — either way collapsing the LEFT JOIN and dropping unmatched parents. To keep true LEFT semantics with a filter on the joined table, put the condition in the `on` option or use `required: false` with the filter expressed via `Sequelize.literal` in the join's `on`. This is a notorious, version-dependent footgun.

```javascript
// Safer: express the optional-side filter in the ON, and pre-aggregate with raw SQL
const rows = await sequelize.query(
  `SELECT u.id, u.name, COUNT(o.id) AS completed_orders
   FROM users u
   LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'completed'
   GROUP BY u.id, u.name`,
  { type: QueryTypes.SELECT }
);
```

**Verdict:** LEFT JOIN by default is convenient, but `where` on an included model is a landmine that silently reintroduces INNER-JOIN behavior. For any filtered LEFT JOIN or aggregation, use `sequelize.query()`.

---

### TypeORM

**Can TypeORM do LEFT JOIN?** — Yes, via `leftJoinAndSelect` / `leftJoin`. Filters on the joined table must go in the join condition argument, not `.where()`, to preserve LEFT semantics.

```typescript
import { DataSource } from 'typeorm';
import { User } from './entities/User';

const ds = new DataSource({ /* config */ });

// LEFT JOIN + load orders; optional-side filter in the JOIN condition (4th arg)
const users = await ds
  .getRepository(User)
  .createQueryBuilder('u')
  .leftJoinAndSelect('u.orders', 'o', 'o.status = :status', { status: 'completed' })
  .getMany();
// users with no completed orders come back with orders: [] (preserved)

// Anti-join: users with no orders
const noOrders = await ds
  .getRepository(User)
  .createQueryBuilder('u')
  .leftJoin('u.orders', 'o')
  .where('o.id IS NULL')
  .getMany();

// Aggregate LEFT JOIN with zero-fill → raw
const stats = await ds
  .createQueryBuilder()
  .select('u.id', 'userId')
  .addSelect('COUNT(o.id)', 'orderCount')
  .from('users', 'u')
  .leftJoin('orders', 'o', 'o.user_id = u.id AND o.status = :s', { s: 'completed' })
  .groupBy('u.id')
  .getRawMany();
```

**Where it breaks:** the classic trap — writing `.andWhere('o.status = :s')` instead of putting the condition in the `leftJoin`'s condition argument collapses the join to INNER. TypeORM gives you the right tool (the join-condition argument) but doesn't prevent the wrong one.

**Verdict:** Solid. Use the join-condition argument for optional-side filters; `getRawMany()` for aggregations.

---

### Knex.js

**Can Knex do LEFT JOIN?** — Yes, fully, via `.leftJoin()` (and `.rightJoin()`, though you'll rarely want it). Knex is SQL-transparent, so the ON-vs-WHERE discipline is entirely in your hands.

```javascript
const knex = require('knex')({ client: 'pg', connection: process.env.DATABASE_URL });

// Every user + completed-order count; filter in the ON via the function form
const rows = await knex('users as u')
  .leftJoin('orders as o', function () {
    this.on('o.user_id', '=', 'u.id')
        .andOn('o.status', '=', knex.raw('?', ['completed']));  // filter in ON
  })
  .groupBy('u.id', 'u.name')
  .select('u.id', 'u.name', knex.raw('COUNT(o.id) AS completed_orders'));

// Anti-join
const noOrders = await knex('users as u')
  .leftJoin('orders as o', 'o.user_id', 'u.id')
  .whereNull('o.id')                    // PK IS NULL
  .select('u.id', 'u.name');
```

**Where it breaks:** `.where('o.status', 'completed')` after a `.leftJoin()` collapses it to INNER — Knex offers `.andOn(...)` in the join's function form precisely so you can keep the filter in the ON. Non-obvious to newcomers.

**Verdict:** Most transparent. `.leftJoin()` with the function form (`.on().andOn()`) is the correct way to filter the optional side. Prefer LEFT over `.rightJoin()`.

---

### ORM Summary Table

| ORM | LEFT JOIN Method | Default for optional relation | Optional-side filter | Anti-join | Verdict |
|---|---|---|---|---|---|
| Prisma | `include` (nullable relation) | LEFT | `include … where` (parent kept) | `where: { rel: { none: {} } }` | Great for objects; raw for agg |
| Drizzle | `.leftJoin()` | Must specify | `and()` inside `leftJoin` | `.where(isNull(opt.id))` | Best typed support |
| Sequelize | `include` (`required:false`) | LEFT (default) | `on` option / raw (footgun via `where`) | raw / `require:false`+`IS NULL` | `where` on include is a trap |
| TypeORM | `leftJoinAndSelect()` | N/A (explicit) | join-condition arg (4th param) | `.where('o.id IS NULL')` | Solid; use join-cond arg |
| Knex | `.leftJoin()` | INNER unless leftJoin | `.on().andOn()` fn form | `.whereNull('o.id')` | Most transparent |

---

## 13. Practice Exercises

### Exercise 1 — Easy

Given:
- `users(id, name, email, created_at)`
- `orders(id, user_id, total_amount, status, created_at)`

Write a query returning **every** user with their total number of orders and total spend, including users who have never ordered (they should show `0` orders and `0` spend, not be missing and not show NULL). Order by total spend descending.

```sql
-- Write your query here
```

Then: write a second query that returns only the users who have **never** placed an order.

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines Topics 09, 11, 12)

Given:
- `users(id, name, created_at)`
- `orders(id, user_id, total_amount, status, created_at)`
- `payments(id, order_id, amount, state)`  -- state ∈ {pending, captured, refunded}

Write a query returning, for **every** user (including inactive ones):
- `user_id`, `name`
- `completed_orders` — count of distinct orders with `status = 'completed'` (0 if none)
- `captured_revenue` — sum of `payments.amount` where `state = 'captured'`, but only for that user's completed orders (0 if none)

Requirements: users with no qualifying orders/payments must still appear with zeros. Be careful where each filter goes (ON vs WHERE) and watch for fan-out between orders and payments.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

A teammate wrote this to build a "product catalog with sales stats" report and complains that products that never sold are missing, and that revenue looks inflated:

```sql
SELECT p.id, p.name, c.name AS category,
       COUNT(oi.id) AS times_ordered,
       SUM(oi.quantity * oi.unit_price) AS revenue
FROM products p
INNER JOIN order_items oi ON oi.product_id = p.id
INNER JOIN orders o       ON o.id = oi.order_id
LEFT  JOIN categories c   ON c.id = p.category_id
WHERE o.status = 'completed'
GROUP BY p.id, p.name, c.name
ORDER BY revenue DESC;
```

1. Explain why products that never sold are missing.
2. Explain why filtering `o.status = 'completed'` in `WHERE` is a problem *if* the intent is to keep never-sold products, and where it must go instead.
3. Rewrite the query so that: **every** product appears (never-sold products show `times_ordered = 0`, `revenue = 0`); revenue counts only completed orders; and the category is included even if `category_id` is NULL.
4. There are 40M rows in `order_items`. What single indexing + query-structure change makes this fast, and what does the resulting EXPLAIN top node look like?

```sql
-- Write your corrected query here
```

---

### Exercise 4 — Interview Simulation

You're given `employees(id, name, manager_id, department_id, salary)` and `departments(id, name)`.

Write a single query that returns, for **every** employee:
- the employee's name and salary,
- their manager's name (`NULL` if they have no manager — e.g. the CEO),
- their department name (`NULL` if unassigned),
- a boolean `earns_more_than_manager` that is `TRUE` when the employee's salary exceeds their manager's, `FALSE` when it doesn't, and `NULL` when they have no manager.

No employee may be dropped, including the CEO and anyone with no department. Explain aloud which joins must be LEFT and why.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What is the difference between INNER JOIN and LEFT JOIN?**

A: INNER JOIN returns only rows that have a match in both tables. LEFT JOIN returns all rows from the left table, and for each left row with no match in the right table, it fills the right table's columns with NULL. So LEFT JOIN never drops a left-table row; INNER JOIN drops any row without a match on either side. RIGHT JOIN is the mirror — it preserves the right table instead.

**Follow-up the interviewer asks:** *"So when would `A LEFT JOIN B` and `A INNER JOIN B` return the exact same rows?"* — When every row in A has at least one match in B (no A row would ever be null-extended). If the FK from A to B is `NOT NULL` and enforced, LEFT and INNER produce identical results, and INNER is preferable because it gives the planner more join-reordering freedom.

---

**Q: I wrote a LEFT JOIN but I'm getting fewer rows than the left table has. What happened?**

A: Almost certainly a `WHERE` clause that references the optional (right) table. The LEFT JOIN correctly null-extends the unmatched left rows, but then a predicate like `WHERE right.column = 'x'` evaluates against those NULLs, produces `UNKNOWN`, and discards them — collapsing the LEFT JOIN into an INNER JOIN. The fix is to move any filter on the optional table into the `ON` clause. You can confirm the diagnosis in `EXPLAIN`: the plan node will say `Hash Join` instead of `Hash Left Join`.

**Follow-up:** *"What if the WHERE filter is on the LEFT table instead?"* — Then it's fine; filtering the preserved table just reduces which left rows you start with, and doesn't affect the null-extension of the ones that remain.

---

**Q: How do you find rows in one table that have no matching row in another?**

A: A LEFT JOIN with an `IS NULL` filter on the optional side's primary key — an anti-join. `SELECT a.* FROM a LEFT JOIN b ON b.a_id = a.id WHERE b.id IS NULL`. The LEFT JOIN null-extends unmatched `a` rows, and `WHERE b.id IS NULL` keeps exactly those. Critically, test a `NOT NULL` column (usually the PK) — testing a nullable column would also catch matched rows that happen to have a NULL there. `NOT EXISTS` expresses the same thing and sidesteps that trap.

---

### Principal Level

**Q: Walk me through, at the execution-plan level, exactly why `WHERE optional.col = 'x'` collapses a LEFT JOIN into an INNER JOIN — and why the *same* predicate in `ON` does not.**

A: Logical query processing runs the JOIN phase before the WHERE phase. In the JOIN phase, the LEFT JOIN produces matched pairs plus null-extended rows for unmatched left rows — for those, every optional-side column, including `col`, is NULL. When the WHERE phase then evaluates `optional.col = 'x'`, the null-extended rows compute `NULL = 'x'` → `UNKNOWN`, and WHERE retains only `TRUE`, so every null-extended row is discarded. The net effect is that only rows with a real match survive — the definition of an INNER JOIN. Postgres's planner recognizes this algebraically: it proves the outer join "cannot produce nulls that survive" and rewrites it to an inner join during planning, which is why the plan node changes from `Hash Left Join` to `Hash Join`. Placing the predicate in `ON` instead makes it part of match-making within the JOIN phase: a left row whose only candidate matches fail `col = 'x'` simply gets zero matches and is null-extended — it survives. So `ON` filters *what matches*; `WHERE` filters *what remains after matching*, and only the latter can eliminate the preserved rows. The corollary: a filter on the *preserved* table is safe in `WHERE` (it never produces surviving NULLs to be tested), which is why the rule is specifically "optional-table filters go in ON."

**Follow-up:** *"Is there ever a reason to deliberately rely on the WHERE-collapse instead of writing INNER JOIN?"* — No. It's strictly worse for readability and invites the exact bug we're discussing. If you want INNER semantics, write INNER JOIN. The only WHERE-on-optional-table that's legitimate is `IS NULL` for an anti-join, which is a fundamentally different operation.

---

**Q: You have `A LEFT JOIN B LEFT JOIN C LEFT JOIN D` where the FKs are all `NOT NULL` and always present. A colleague says the query is slow and blames the LEFT JOINs. Are they right?**

A: Possibly, but not for the reason they think. LEFT JOIN's per-row overhead vs INNER JOIN is negligible. The real cost is that OUTER JOIN is not commutative, so the planner has fewer legal join orderings — it can't freely reorder the chain to join the most selective tables first the way it could with all-INNER. If the FKs are genuinely `NOT NULL` and always satisfied, the LEFT JOINs are semantically identical to INNER JOINs (no left row is ever null-extended), so I'd rewrite them as INNER JOINs. That hands the planner full reordering freedom and often flips it to a materially better plan. I'd verify with `EXPLAIN (ANALYZE)` before and after, and also check `join_collapse_limit` — beyond 8 joins the planner stops reordering and just runs them in written order, so for large chains the *written* order matters too. Net: the fix is "use INNER where the relationship is truly mandatory," not "LEFT JOINs are inherently slow."

**Follow-up:** *"What if one of them is genuinely optional?"* — Keep that one LEFT and make the mandatory ones INNER. Mixed is fine; the point is to only pay the non-commutativity tax where the semantics actually require it.

---

**Q: A LEFT JOIN feeding a `GROUP BY` reports `COUNT(*) = 1` for entities that should have zero children. What's happening and what are all the ways to fix it?**

A: The null-extended row is a real row, and `COUNT(*)` counts rows, so an entity with no children still contributes its single null-extended row as a count of 1. Fixes, in order of preference: (1) `COUNT(child.id)` — `COUNT(column)` ignores NULLs, so the null-extended row contributes 0; this is the idiomatic fix. (2) `COUNT(child.pk)` specifically, using a non-nullable column so you never accidentally undercount matched-but-null rows. (3) Restructure by pre-aggregating the child table into a per-parent count subquery and LEFT JOIN that — this also eliminates fan-out if there are multiple one-to-many children. For sums, the parallel gotcha is `SUM(child.amount)` returning NULL (not 0) for empty groups, fixed with `COALESCE(SUM(...), 0)`. The root cause in every case is forgetting that LEFT JOIN's "no match" is represented as a present row full of NULLs, and choosing aggregate functions that are NULL-aware accordingly.

---

## 15. Mental Model Checkpoint

1. **`users` has 10,000 rows. 6,200 of them have at least one order; 3,800 have none. How many rows does `SELECT u.id FROM users u LEFT JOIN orders o ON o.user_id = u.id` return — exactly, more, or fewer than 10,000 — and why?**

2. **You change `LEFT JOIN orders o ON o.user_id = u.id` by adding `WHERE o.created_at > '2025-01-01'`. What happens to users whose only orders are from 2024? What about users with no orders at all? How would you rewrite it to keep every user while still only counting 2025 orders?**

3. **Explain in one sentence why `A RIGHT JOIN B` is exactly `B LEFT JOIN A`, and why that makes RIGHT JOIN avoidable in hand-written SQL.**

4. **After a LEFT JOIN, `COUNT(*)`, `COUNT(o.id)`, and `SUM(o.amount)` each return a different thing for an entity with no matches. State what each returns and why.**

5. **In a chain `users LEFT JOIN orders LEFT JOIN order_items`, a user has no orders. Trace what happens at the second join and explain why the user still appears in the output.**

6. **You want "employees who have no manager assigned." Is that a filter on the preserved side or the optional side, and does the `IS NULL` go in `ON` or `WHERE`? What column must it test?**

7. **When are `A LEFT JOIN B` and `A INNER JOIN B` guaranteed to return identical rows, and given that guarantee, which should you write and why?**

---

## 16. Quick Reference Card

```sql
-- Syntax (OUTER is a noise word; direction is mandatory)
SELECT ...
FROM preserved_table p
LEFT [OUTER] JOIN optional_table o ON p.key = o.key
-- every p row appears; o.* is NULL where no match

-- RIGHT JOIN == LEFT JOIN with tables swapped. Prefer LEFT. Always.
A RIGHT JOIN B  ≡  B LEFT JOIN A

-- THE RULE: filter on the OPTIONAL table → ON.  filter on PRESERVED table → WHERE.
LEFT JOIN orders o ON o.user_id = u.id AND o.status = 'completed'   -- ✅ preserves users
LEFT JOIN orders o ON o.user_id = u.id WHERE o.status = 'completed' -- ❌ collapses to INNER

-- Diagnose the collapse in EXPLAIN:
--   plan says "Hash Left Join"  → LEFT semantics intact
--   plan says "Hash Join"       → a WHERE predicate collapsed it to INNER

-- Anti-join: "rows with NO match" — test the optional PK (non-NULL)
SELECT a.* FROM a LEFT JOIN b ON b.a_id = a.id WHERE b.id IS NULL

-- NULL-aware aggregation after LEFT JOIN:
COUNT(*)                       -- counts null-extended row too → min 1 (usually WRONG)
COUNT(o.id)                    -- ignores NULL → correct 0 for no-match
COALESCE(SUM(o.amount), 0)     -- SUM of empty group is NULL → COALESCE to 0
COALESCE(AVG(o.amount), 0)     -- AVG of empty group is NULL too

-- Keep the whole optional chain LEFT — one INNER JOIN poisons everything upstream
FROM users u
LEFT JOIN orders o       ON o.user_id = u.id
LEFT JOIN order_items oi ON oi.order_id = o.id   -- NOT inner here

-- "latest related row, or NULL" idiom
LEFT JOIN LATERAL (
  SELECT ... FROM orders WHERE user_id = u.id ORDER BY created_at DESC LIMIT 1
) o ON true

-- Pre-aggregate optional one-to-many to 1:1 before joining (kills fan-out)
FROM users u
LEFT JOIN (SELECT user_id, COUNT(*) n FROM orders GROUP BY user_id) s ON s.user_id = u.id

-- Perf: LEFT JOIN constrains planner join order (not commutative).
--   If FK is NOT NULL and always present → use INNER (more reordering freedom).
--   Index the OPTIONAL side's join column (it's the probed side).

-- ORM pitfalls:
--   Sequelize: `where` on an include silently collapses LEFT → INNER (footgun)
--   TypeORM/Knex: optional-side filter goes in the JOIN condition arg, not .where()
--   Drizzle: put filter in and() inside .leftJoin(); leftJoin marks columns T | null
--   Prisma: include on optional relation = LEFT; anti-join via `{ rel: { none: {} } }`
```

---

## Connected Topics

- **Topic 07 — NULL in Depth**: The three-valued logic (`NULL = x → UNKNOWN`) that makes `WHERE` on the optional table collapse a LEFT JOIN. The single most important prerequisite for this topic.
- **Topic 09 — WHERE Fundamentals**: WHERE keeps only `TRUE`, discarding `FALSE` and `UNKNOWN` — the exact mechanism that eliminates null-extended rows.
- **Topic 11 — INNER JOIN in Depth**: The matched-pairs core that LEFT JOIN extends with null-extension; fan-out and aggregation behavior carry over unchanged.
- **Topic 13 — FULL OUTER JOIN**: Preserve *both* sides' unmatched rows — the natural next step once one-sided outer joins are mastered.
- **Topic 17 — JOIN Performance Deep Dive**: Why OUTER JOIN's non-commutativity constrains the planner, and how `join_collapse_limit` interacts with written join order.
- **Topic 20 — GROUP BY Fundamentals**: NULL-aware aggregation (`COUNT(col)` vs `COUNT(*)`, `COALESCE(SUM())`) after a LEFT JOIN.
- **Topic 27 — EXISTS and NOT EXISTS**: `NOT EXISTS` as the cleaner alternative to the `LEFT JOIN … IS NULL` anti-join, immune to the nullable-column trap.
