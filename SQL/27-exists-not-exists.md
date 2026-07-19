# Topic 27 — EXISTS and NOT EXISTS
### SQL Mastery Curriculum — Phase 5: Subqueries and CTEs

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you're a bouncer at a club with a guest list. A line of people is waiting outside. Your job: let in every person whose name appears **somewhere** on the guest list.

Here's the key detail about how you work. When a person walks up, you don't count how many times their name appears on the list. You don't care if "Alice" is written once or seventeen times. You start scanning the list top to bottom, and the **moment** you find "Alice," you stop reading, wave her in, and call the next person. First match — done. You never read the rest of the list.

That's `EXISTS`. It asks one question: "Is there **at least one** match?" The answer is yes or no. It stops the instant the answer becomes yes.

Now flip it. Suppose instead you're running a "newcomers only" event — you let people in **only if** their name is **not** on the list of past attendees. A person walks up, you scan the past-attendee list, and if you get all the way to the bottom without finding their name, you let them in. If you find their name even once, you turn them away.

That's `NOT EXISTS`. It asks: "Is there **no** match at all?" To answer "no match," you unfortunately have to check the whole list before you can be sure — there's no early exit for a "not found" answer.

And here's the trap that ruins careers: what if the past-attendee list has a smudged entry — a name you literally cannot read? A NULL. With the naïve `NOT IN` version of this check, that one unreadable smudge makes you turn **everyone** away, because you can never prove their name *isn't* the smudge. `NOT EXISTS` doesn't have this problem. That difference is the single most important thing in this entire topic.

The whole topic is those three ideas: stop at the first match (`EXISTS`), scan to the end to prove absence (`NOT EXISTS`), and the smudged-entry catastrophe (`NOT IN` + NULL).

---

## 2. Connection to SQL Internals

`EXISTS` and `NOT EXISTS` are the SQL surface syntax for two relational-algebra operators the planner reasons about directly:

- **Semi-join** (`⋉`): return rows from the left relation that have **at least one** match in the right relation. Each left row appears **at most once**, regardless of how many right-side matches exist. `EXISTS` compiles to this.
- **Anti-join** (`▷`): return rows from the left relation that have **no** match in the right relation. `NOT EXISTS` compiles to this.

The critical internal property is that neither operator projects any columns from the right relation. The right side is used purely as a **membership test**, never as a source of output data. This is what distinguishes a semi-join from an INNER JOIN (Topic 11): the INNER JOIN materialises the matched pairs and can multiply rows; the semi-join returns a boolean per left row and never multiplies.

At the physical layer, PostgreSQL implements these with the same three join algorithms you already know, specialised for early exit:

1. **Nested Loop Semi Join / Nested Loop Anti Join**: for each outer row, probe the inner relation. A **Semi Join stops probing at the first match** (short-circuit). An **Anti Join stops probing at the first match too — but to *reject* the row**; it only *emits* the outer row if the probe exhausts with zero matches.
2. **Hash Semi Join / Hash Anti Join**: build a hash table on the inner join key, then probe once per outer row. The hash lookup is itself the membership test — the presence of a bucket entry answers the EXISTS question in O(1) expected time.
3. **Merge Semi Join / Merge Anti Join**: both inputs sorted on the join key, walked in lockstep, emitting (semi) or skipping (anti) the first time keys align.

The planner's most important trick here is the **subquery-to-semi-join transformation**: a correlated `EXISTS (SELECT ...)` in the `WHERE` clause is *not* executed as a per-row subplan re-run millions of times. The planner **flattens** it into a semi-join node and then costs Nested Loop / Hash / Merge alternatives just like any other join. This is why "EXISTS is slow because it runs the subquery per row" is a junior misconception — it's usually pulled up into a set-based join.

The MVCC and buffer-pool layers matter too: because a semi-join can stop at the first visible matching tuple, it often touches **far fewer heap pages** than the equivalent INNER JOIN, which must find and return every matching tuple. Fewer page visits means fewer buffer-pool lookups and less WAL-irrelevant read I/O.

---

## 3. Logical Execution Order Context

```
FROM <outer table>
WHERE EXISTS (correlated subquery)   ← evaluated during WHERE phase
GROUP BY ...
HAVING  ...
SELECT  ...
ORDER BY ...
LIMIT   ...
```

`EXISTS` and `NOT EXISTS` are predicates. Logically they live in the `WHERE` clause (or `HAVING`, or a `CASE`, or a `SELECT`-list boolean expression), and therefore run in the **row-filtering phase**, after `FROM`/`JOIN` have produced the candidate rows and before `GROUP BY`.

Several consequences flow from this placement:

- **The subquery is correlated to the outer row.** By the time `WHERE EXISTS (...)` runs for a given outer row, that outer row's columns are bound, so the subquery can reference `outer.id`, `outer.customer_id`, etc.
- **It runs before aggregation.** You filter the row set *before* `GROUP BY` collapses it. If you need existence checks *after* aggregation, that goes in `HAVING` or a subquery over the grouped result.
- **It runs before `SELECT` projection.** So an `EXISTS` in the `WHERE` cannot reference a column alias defined in the `SELECT` list (aliases don't exist yet at `WHERE` time — same rule as any `WHERE` predicate).
- **Logically per-row, physically set-based.** Although the mental model is "evaluate this subquery once per outer row," the planner is free to (and usually does) rewrite it into a single semi-/anti-join executed once. The *logical* result must be identical to the per-row evaluation; the *physical* execution is optimised.

Contrast with a scalar subquery in the `SELECT` list (Topic 26): that also correlates per row, but it must return exactly one value, whereas `EXISTS` returns a boolean and tolerates zero-or-many matching rows by design.

---

## 4. What Is EXISTS / NOT EXISTS?

`EXISTS (subquery)` is a boolean predicate that evaluates to `TRUE` if the subquery returns **one or more rows**, and `FALSE` if it returns **zero rows**. `NOT EXISTS (subquery)` is its negation: `TRUE` when the subquery returns zero rows, `FALSE` otherwise. Crucially, **`EXISTS` never returns `UNKNOWN`/NULL** — it is strictly two-valued, which is the root of why it is NULL-safe where `IN`/`NOT IN` are not.

```sql
SELECT c.id, c.name
FROM customers c
WHERE EXISTS (
  SELECT 1
  FROM orders o
  WHERE o.customer_id = c.id
    AND o.status = 'completed'
);
```

### Annotated syntax breakdown

```sql
SELECT c.id, c.name              -- outer query projection (only outer columns available)
FROM customers c                 -- outer relation; each row is a candidate
WHERE EXISTS (                   -- ┐ boolean predicate: TRUE iff subquery yields ≥1 row
                                 -- │  └── returns TRUE/FALSE only, NEVER NULL
  SELECT 1                       -- │ ┐ projection is IRRELEVANT — EXISTS ignores it
                                 -- │ │  └── use "1" (or *) by convention; the planner
                                 -- │ │      discards the target list entirely
  FROM orders o                  -- │ │ inner relation being tested for membership
  WHERE o.customer_id = c.id     -- │ │ ┐ CORRELATION: binds inner to the current outer row
                                 -- │ │ │  └── makes this a correlated (semi-join) subquery
    AND o.status = 'completed'   -- │ │ └── additional membership constraints
);                               -- ┘ ┘ subquery closes; predicate resolves per outer row
```

Key facts encoded above:

- **`SELECT 1` vs `SELECT *` vs `SELECT column`** — makes **no difference**. `EXISTS` only checks *whether* rows exist, not what they contain. The planner strips the target list. `SELECT 1` is idiomatic and signals intent ("I don't care about the columns"). There is **no performance difference** between `SELECT 1` and `SELECT *` inside `EXISTS`; benchmarks confirm identical plans. (The old folklore that `SELECT 1` is faster than `SELECT *` is false in PostgreSQL.)
- **The correlation predicate** (`o.customer_id = c.id`) is what makes this useful. Without it, the subquery is *uncorrelated* — it returns the same boolean for every outer row (see §6.7).
- **No `LIMIT 1` is needed.** People add `... AND EXISTS (SELECT 1 ... LIMIT 1)` thinking it forces early exit. The semi-join already stops at the first match; `LIMIT 1` is redundant and the planner ignores it for correctness (it may keep it as a no-op).

`NOT EXISTS` is identical in shape, negated:

```sql
SELECT c.id, c.name
FROM customers c
WHERE NOT EXISTS (               -- TRUE iff subquery yields ZERO rows (anti-join)
  SELECT 1
  FROM orders o
  WHERE o.customer_id = c.id
);
-- customers who have never placed any order
```

---

## 5. Why EXISTS / NOT EXISTS Mastery Matters in Production

1. **The `NOT IN` + NULL data-loss bug is silent and catastrophic.** `WHERE x NOT IN (SELECT y FROM t)` returns **zero rows** the instant any `y` is NULL — no error, no warning. This has shipped to production countless times as "the report is mysteriously empty." Knowing to reach for `NOT EXISTS` instead is a career-defining habit. This is the single highest-value item in this topic.

2. **Avoiding accidental row multiplication.** Developers reach for INNER JOIN to answer "which customers have orders?" and get duplicate customers (one per order), then paper over it with `DISTINCT`. `EXISTS` answers the existence question *without* fan-out and *without* `DISTINCT` — cleaner and often faster.

3. **Correct anti-join semantics.** "Find products never ordered," "users with no sessions in 30 days," "orders with no payment" — these are anti-joins. Getting them right (and NULL-safe) is bread-and-butter analytics and billing-correctness work. A wrong anti-join can under-bill, over-refund, or hide fraud.

4. **Performance predictability.** Understanding that `EXISTS` short-circuits at the first match — and that the planner turns it into a semi-join — lets you reason about why `EXISTS` beats `COUNT(*) > 0` and why it interacts with indexes the way it does. On large tables the difference is seconds vs milliseconds.

5. **Query intent clarity.** `EXISTS` reads as "does a related row exist?" That semantic clarity prevents a whole class of bugs where a JOIN's side effects (multiplication, NULL widening) leak into results the author never intended.

---

## 6. Deep Technical Content

### 6.1 EXISTS Short-Circuits at the First Match

The defining behaviour: `EXISTS` stops evaluating the subquery the moment one matching row is found. It does not enumerate all matches, does not count them, does not sort them.

```sql
-- Customer 42 has 10,000 completed orders.
SELECT c.id
FROM customers c
WHERE c.id = 42
  AND EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status = 'completed');
```

Even though customer 42 has 10,000 matching orders, the semi-join finds the **first** one — ideally via an index on `orders(customer_id, status)` — and immediately returns `TRUE`. The other 9,999 are never touched. This is why `EXISTS` cost is bounded by "cost to find one match," not "cost to find all matches."

Compare with `COUNT`:

```sql
-- ANTI-PATTERN: counts ALL 10,000 rows just to check "> 0"
WHERE (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id AND o.status='completed') > 0
```

`COUNT(*)` must visit every matching row to produce the total, then compare to zero. `EXISTS` stops at one. On a customer with millions of orders, `COUNT(*) > 0` reads millions of rows to answer a question that one row settles.

### 6.2 NOT EXISTS Cannot Short-Circuit the "Absent" Answer

Symmetry breaks for `NOT EXISTS`. To prove `TRUE` (no match exists), the engine must confirm that **not a single** matching row exists — which in the worst case means scanning all candidate inner rows for that outer row. There is no early "yes" for absence.

However, it **does** short-circuit the *`FALSE`* answer: the moment any matching row is found, `NOT EXISTS` resolves to `FALSE` and stops. So:

- Outer rows that **have** a match: resolved fast (first match ⇒ FALSE ⇒ stop). Cheap.
- Outer rows that **have no** match: must exhaust the inner search to confirm. This is where an index on the inner join key is essential — it turns "scan everything" into "probe the index, find nothing, done."

This asymmetry is why a well-indexed `NOT EXISTS` is fast even though "proving absence" sounds expensive: the index lets the engine confirm absence with a single ranged lookup rather than a full scan.

### 6.3 The NOT IN + NULL Trap (The Most Important Section in This Topic)

`NOT IN` with a subquery that can produce NULL is a **correctness landmine**. Consider:

```sql
-- Intent: products that have never been ordered
SELECT p.id, p.name
FROM products p
WHERE p.id NOT IN (SELECT oi.product_id FROM order_items oi);
```

If **any** row in `order_items` has `product_id IS NULL`, this query returns **zero rows** — always, regardless of your data. Here's the exact mechanism, rooted in three-valued logic (Topic 07):

`x NOT IN (a, b, NULL)` expands to:
```
x <> a  AND  x <> b  AND  x <> NULL
```
`x <> NULL` evaluates to `UNKNOWN` (never `TRUE`). So the whole conjunction becomes:
```
(x <> a) AND (x <> b) AND UNKNOWN
```
The best this can ever be is `UNKNOWN` (when `x <> a` and `x <> b` are both TRUE, `TRUE AND TRUE AND UNKNOWN = UNKNOWN`). And `WHERE UNKNOWN` filters the row **out**. So no row can ever satisfy the predicate. Empty result. Silently.

The fix is `NOT EXISTS`, which uses `=` internally and is driven by two-valued existence logic:

```sql
-- CORRECT and NULL-safe
SELECT p.id, p.name
FROM products p
WHERE NOT EXISTS (
  SELECT 1 FROM order_items oi WHERE oi.product_id = p.id
);
```

Why is this NULL-safe? `NOT EXISTS` asks "does a row with `oi.product_id = p.id` exist?" A row where `oi.product_id IS NULL` does **not** satisfy `oi.product_id = p.id` (that comparison is `UNKNOWN`, not `TRUE`), so it simply isn't a match — it doesn't poison the whole result. The NULL row is quietly ignored, exactly as intended.

**Side-by-side truth table** for checking whether `p.id = 5` is "not in" a set containing NULL:

| Set from subquery | `5 NOT IN (set)` | `NOT EXISTS (... = 5)` |
|-------------------|------------------|------------------------|
| `{1, 2, 3}`       | TRUE (5 absent)  | TRUE (5 absent)        |
| `{1, 2, 5}`       | FALSE (5 present)| FALSE (5 present)      |
| `{1, 2, NULL}`    | **UNKNOWN → excluded** | TRUE (5 absent) |
| `{5, NULL}`       | FALSE            | FALSE                  |
| `{NULL}`          | **UNKNOWN → excluded** | TRUE            |

Only `NOT EXISTS` gives the intuitively correct answer in the NULL rows.

**If you must use `NOT IN`**, you have to guarantee no NULLs reach it:
```sql
WHERE p.id NOT IN (SELECT oi.product_id FROM order_items oi WHERE oi.product_id IS NOT NULL);
```
This works but is fragile — one future refactor that drops the `IS NOT NULL` filter reintroduces the bug. `NOT EXISTS` is the robust default. Make it a rule: **never write `NOT IN (subquery)` unless the subquery column is `NOT NULL` at the schema level.**

### 6.4 EXISTS vs IN — Semantics and When They Diverge

For the *positive* case, `IN` does **not** have the NULL catastrophe (a NULL in the list just never matches, which for `IN` means "not this one" — harmless). So `IN` and `EXISTS` are usually interchangeable for existence checks:

```sql
-- These are logically equivalent when product_id has no NULL semantics issue
WHERE c.id IN (SELECT o.customer_id FROM orders o WHERE o.status = 'completed')
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status = 'completed')
```

Differences that matter:

- **NULLs in the `IN` list**: harmless for `IN` (positive), catastrophic for `NOT IN` (negative). This is the asymmetry to burn into memory.
- **Correlation direction**: `IN` uses an uncorrelated subquery (the subquery is evaluated independently and produces a value list); `EXISTS` is typically correlated. Modern PostgreSQL planners transform both into semi-joins, so **performance is usually identical** — the old "EXISTS is always faster than IN" rule is largely obsolete in PostgreSQL. Both become `Hash Semi Join` / `Nested Loop Semi Join` under the same conditions.
- **Multi-column checks**: `EXISTS` handles composite keys naturally (`WHERE a.x = b.x AND a.y = b.y`). `IN` needs row-constructor syntax: `WHERE (a.x, a.y) IN (SELECT b.x, b.y FROM b)` which is less common and has its own NULL subtleties.

Rule of thumb: **use `EXISTS` for correlated existence checks and always for the negative case; `IN` is fine for the positive case with a small, known, NULL-free value set.**

### 6.5 EXISTS vs INNER JOIN — The Semi-Join Advantage

Both can answer "which customers have at least one completed order?" but they differ in fan-out:

```sql
-- INNER JOIN: multiplies — one row per matching order, needs DISTINCT
SELECT DISTINCT c.id, c.name
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id AND o.status = 'completed';

-- EXISTS: no multiplication — each customer appears at most once, no DISTINCT
SELECT c.id, c.name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status = 'completed');
```

The INNER JOIN version:
- Produces a row per matching order (a customer with 500 orders appears 500 times).
- Requires `DISTINCT` to collapse them, which forces a sort or hash-aggregate — extra cost.
- Reads **all** matching orders before de-duplicating.

The `EXISTS` version:
- Produces each customer at most once by construction (semi-join).
- Needs no `DISTINCT`.
- Stops at the **first** matching order per customer.

**When to prefer INNER JOIN instead:** when you actually need **columns from the right table** in the output. `EXISTS` cannot project inner columns. If you need the order date/amount, you must JOIN. If you only need "yes/no it exists," `EXISTS` is cleaner and avoids the DISTINCT tax.

A subtle correctness point: `DISTINCT` + JOIN and `EXISTS` are equivalent **only** for the existence question. The moment you add a second joined table or an aggregate, the `DISTINCT`-JOIN form gets tricky (which columns to DISTINCT on?) while `EXISTS` stays clean.

### 6.6 Correlated vs Uncorrelated EXISTS

An `EXISTS` subquery that references the outer row is **correlated**:
```sql
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)  -- references c.id
```

One that does **not** reference the outer row is **uncorrelated** — it evaluates to a single constant boolean for the entire query:
```sql
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.status = 'completed')
-- TRUE for every outer row if ANY completed order exists anywhere; else FALSE for all
```

The uncorrelated form is almost always a bug when used as a per-row filter — it's a global "does any completed order exist?" gate that either passes all outer rows or none. It's occasionally intentional as a feature-flag-style guard, but if you meant to correlate and forgot the `WHERE o.customer_id = c.id`, you get wildly wrong results (all rows or no rows). **Always double-check the correlation predicate is present.**

### 6.7 EXISTS in SELECT, CASE, and HAVING

`EXISTS` is a boolean expression, so it works anywhere a boolean is valid — not just `WHERE`.

**In the `SELECT` list** (as a computed flag):
```sql
SELECT
  c.id,
  c.name,
  EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id) AS has_ordered,
  EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id
          AND o.created_at > NOW() - INTERVAL '30 days') AS active_recently
FROM customers c;
-- Returns boolean columns; each EXISTS still short-circuits
```

**In `CASE`**:
```sql
SELECT c.id,
  CASE
    WHEN EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status='completed')
      THEN 'buyer'
    WHEN EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id)
      THEN 'browser'   -- has orders but none completed
    ELSE 'prospect'
  END AS segment
FROM customers c;
```

**In `HAVING`** (existence check after grouping):
```sql
SELECT o.customer_id, COUNT(*) AS order_count
FROM orders o
GROUP BY o.customer_id
HAVING COUNT(*) > 5
   AND EXISTS (SELECT 1 FROM customers c WHERE c.id = o.customer_id AND c.tier = 'premium');
```

Note that an `EXISTS` in the `SELECT` list is evaluated once per **output** row and generally cannot be flattened into a semi-join (it's producing a value, not filtering), so it may run as a correlated subplan. If you need `has_ordered` for many rows and it's slow, consider a `LEFT JOIN ... IS NOT NULL` or a pre-aggregated flag.

### 6.8 Multi-Column and Composite-Key EXISTS

`EXISTS` shines with composite correlation because it's just a normal `WHERE` inside:

```sql
-- Orders that have at least one item matching a currently-active promotion
-- promotions keyed on (product_id, region)
SELECT o.id
FROM orders o
WHERE EXISTS (
  SELECT 1
  FROM order_items oi
  JOIN promotions pr
    ON pr.product_id = oi.product_id
   AND pr.region     = o.ship_region       -- correlate on region from OUTER order
  WHERE oi.order_id = o.id                  -- correlate on order
    AND pr.active_from <= o.created_at
    AND pr.active_to   >  o.created_at
);
```

The subquery can join multiple tables, correlate on several outer columns, and use ranges — all while still resolving to a single boolean per outer order. Expressing this with `IN` would require an awkward multi-column row constructor and lose clarity.

### 6.9 NOT EXISTS for the Classic Anti-Join Patterns

The canonical uses, each NULL-safe:

```sql
-- 1. Products never ordered
SELECT p.* FROM products p
WHERE NOT EXISTS (SELECT 1 FROM order_items oi WHERE oi.product_id = p.id);

-- 2. Customers with no orders in the last 90 days (churned)
SELECT c.* FROM customers c
WHERE NOT EXISTS (
  SELECT 1 FROM orders o
  WHERE o.customer_id = c.id
    AND o.created_at >= NOW() - INTERVAL '90 days'
);

-- 3. Orders that have no successful payment (billing gap)
SELECT o.* FROM orders o
WHERE o.status = 'completed'
  AND NOT EXISTS (
    SELECT 1 FROM payments pay
    WHERE pay.order_id = o.id AND pay.status = 'succeeded'
  );

-- 4. Employees who manage nobody (leaf nodes in the org tree)
SELECT e.* FROM employees e
WHERE NOT EXISTS (SELECT 1 FROM employees sub WHERE sub.manager_id = e.id);
```

Pattern 3 is a common correctness query: it surfaces revenue leakage (fulfilled orders with no captured payment). Getting the anti-join NULL-safe here directly protects money.

### 6.10 The "Relational Division" Pattern — Double NOT EXISTS

A famous advanced use: "find X that relate to **all** Y." For example, "customers who have ordered **every** product in category 'Premium'." This is relational division, classically expressed with nested `NOT EXISTS`:

```sql
-- Customers who have ordered EVERY premium product
SELECT c.id, c.name
FROM customers c
WHERE NOT EXISTS (                                   -- there is no premium product...
  SELECT 1
  FROM products p
  WHERE p.category = 'Premium'
    AND NOT EXISTS (                                 -- ...that this customer has NOT ordered
      SELECT 1
      FROM order_items oi
      JOIN orders o ON o.id = oi.order_id
      WHERE oi.product_id = p.id
        AND o.customer_id = c.id
    )
);
```

Read it as the double negative it is: "no premium product exists that the customer hasn't ordered" ≡ "the customer has ordered all premium products." This is the textbook division idiom and a frequent senior-interview question. The alternative — `GROUP BY ... HAVING COUNT(DISTINCT ...) = (total premium count)` — is often more readable and sometimes faster, but the `NOT EXISTS` form handles edge cases (empty premium set) with clean logic: if there are **no** premium products, the inner query returns nothing, the outer `NOT EXISTS` is `TRUE`, and **every** customer qualifies (vacuously true — "ordered all zero products"). Know both forms and their empty-set behaviour.

### 6.11 EXISTS and the Empty Set

- `EXISTS (<query returning 0 rows>)` → `FALSE`.
- `NOT EXISTS (<query returning 0 rows>)` → `TRUE`.
- If the **outer** table is empty, the whole query returns 0 rows regardless of the predicate.
- An `EXISTS` whose subquery is `SELECT 1 WHERE FALSE` is a constant `FALSE`; the planner may fold it away entirely (`One-Time Filter: false`, inner never executed).

### 6.12 Why `SELECT 1`, `SELECT *`, `SELECT NULL`, `SELECT 1/0` All Behave the Same

Inside `EXISTS`, the target list is never evaluated for its values — only row existence matters. So `SELECT 1`, `SELECT *`, `SELECT NULL`, even `SELECT 1/0` produce identical results and identical plans; the division-by-zero is **never** triggered because the expression is discarded. (Don't write `SELECT 1/0`; the point is illustrative — it proves the target list is inert.) Idiom: use `SELECT 1`.

---

## 7. EXPLAIN — Semi-Join and Anti-Join in the Plan

### Hash Semi Join (EXISTS, large tables, equality correlation)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.id, c.name
FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status = 'completed'
);
```

```
Hash Semi Join  (cost=4200.00..9850.00 rows=6200 width=40)
                (actual time=18.2..71.4 rows=6187 loops=1)
  Hash Cond: (c.id = o.customer_id)
  ->  Seq Scan on customers c  (cost=0.00..2100.00 rows=100000 width=40)
                                (actual time=0.01..12.3 rows=100000 loops=1)
  ->  Hash  (cost=3800.00..3800.00 rows=210000 width=8)
             (actual time=17.9..17.9 rows=214300 loops=1)
        Buckets: 262144  Batches: 1  Memory Usage: 10420kB
        ->  Seq Scan on orders o  (cost=0.00..3800.00 rows=210000 width=8)
              Filter: (status = 'completed'::text)
              (actual time=0.02..9.8 rows=214300 loops=1)
Buffers: shared hit=5120
Planning Time: 0.4 ms
Execution Time: 72.1 ms
```

**Reading it**:
- **`Hash Semi Join`** — not a plain Hash Join. The `Semi` keyword confirms the planner flattened `EXISTS` into a semi-join: each customer emitted **at most once**, no duplicates, no `DISTINCT` needed.
- The inner `orders` side is hashed on `customer_id`. Each customer probes the hash; the **first** matching bucket entry satisfies EXISTS and the probe stops.
- `rows=6187` outer rows survive — these are customers with ≥1 completed order.
- `Batches: 1` — hash fits in `work_mem`; no disk spill.

### Nested Loop Semi Join (EXISTS, indexed inner, selective outer)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.id, c.name
FROM customers c
WHERE c.tier = 'premium'
  AND EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status='completed');
```

```
Nested Loop Semi Join  (cost=0.43..820.50 rows=95 width=40)
                       (actual time=0.05..3.10 rows=91 loops=1)
  ->  Index Scan using customers_tier_idx on customers c
        Index Cond: (tier = 'premium'::text)
        (actual time=0.02..0.9 rows=1200 loops=1)
  ->  Index Only Scan using orders_customer_status_idx on orders o
        Index Cond: (customer_id = c.id AND status = 'completed')
        (actual time=0.001..0.001 rows=1 loops=1200)   ← rows=1: STOPS at first match
Buffers: shared hit=3840
Planning Time: 0.3 ms
Execution Time: 3.2 ms
```

**Reading it**:
- Outer: 1,200 premium customers via index.
- Inner: for each, an **Index Only Scan** probes `orders(customer_id, status)`. Note **`actual rows=1`** on the inner despite many customers having hundreds of completed orders — this is the semi-join short-circuit in the plan. It found one and stopped.
- `loops=1200` = one probe per premium customer; each probe is O(log n) index descent.

### Hash Anti Join (NOT EXISTS)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT p.id, p.name
FROM products p
WHERE NOT EXISTS (SELECT 1 FROM order_items oi WHERE oi.product_id = p.id);
```

```
Hash Anti Join  (cost=6400.00..9900.00 rows=430 width=40)
                (actual time=41.0..58.7 rows=418 loops=1)
  Hash Cond: (p.id = oi.product_id)
  ->  Seq Scan on products p  (cost=0.00..900.00 rows=50000 width=40)
                               (actual time=0.01..5.2 rows=50000 loops=1)
  ->  Hash  (cost=4200.00..4200.00 rows=2000000 width=8)
             (actual time=40.5..40.5 rows=2000000 loops=1)
        Buckets: 262144  Batches: 8  Memory Usage: 3840kB
        ->  Seq Scan on order_items oi  (cost=0.00..4200.00 rows=2000000 width=8)
              (actual time=0.02..18.0 rows=2000000 loops=1)
Buffers: shared hit=9600 read=2100
Planning Time: 0.5 ms
Execution Time: 59.4 ms
```

**Reading it**:
- **`Hash Anti Join`** — the physical operator for `NOT EXISTS`. It emits a product **only if** no `order_items` row hashes to a matching `product_id`.
- `rows=418` products survive — the never-ordered ones.
- `Batches: 8` — the 2M-row hash **spilled to disk** (work_mem too small). The `read=2100` buffers confirm disk I/O. Fix: raise `work_mem` so `Batches: 1`, or add an index enabling a Nested Loop Anti Join instead.
- NULL safety is inherent: an `order_items` row with `product_id IS NULL` hashes to nothing useful and never causes a product to be wrongly excluded — unlike `NOT IN`.

### What to Look For

| Symptom | Meaning | Action |
|---------|---------|--------|
| `Hash Semi Join` / `Nested Loop Semi Join` | `EXISTS` was flattened to a semi-join (good) | Nothing — this is optimal |
| `Hash Anti Join` / `Nested Loop Anti Join` | `NOT EXISTS`/`NOT IN` flattened to anti-join (good) | Nothing |
| Inner `actual rows=1` under a Semi Join | Short-circuit working | Confirms EXISTS efficiency |
| `SubPlan` node with high `loops` | Subquery **not** flattened — running per row | Rewrite; check for volatile funcs / LATERAL blockers |
| `Batches > 1` on the hash | Spilling to disk | Increase `work_mem` |
| `One-Time Filter: false` | Uncorrelated EXISTS folded to constant | Usually a logic bug (missing correlation) |

The worst outcome is seeing `SubPlan` with `loops=1000000` instead of a `Semi Join` node — that means the planner could not flatten the subquery (often due to a volatile function, a `LIMIT`, or a correlation it can't pull up) and is re-running it per row.

---

## 8. Query Examples

### Example 1 — Basic: Customers Who Have Ordered

```sql
-- "Which customers have placed at least one order?"
-- EXISTS avoids duplicates and needs no DISTINCT.
SELECT
  c.id,
  c.name,
  c.email
FROM customers c
WHERE EXISTS (
  SELECT 1                        -- value irrelevant; existence is the question
  FROM orders o
  WHERE o.customer_id = c.id      -- correlation to the current customer
)
ORDER BY c.name;
```

### Example 2 — Intermediate: Anti-Join for Churn Detection

```sql
-- "Active customers who have NOT ordered in the last 90 days."
-- NULL-safe anti-join; NOT IN would be dangerous here.
SELECT
  c.id,
  c.name,
  c.email,
  c.last_seen_at
FROM customers c
WHERE c.status = 'active'
  AND EXISTS (                                   -- has ordered at some point (a real customer)
        SELECT 1 FROM orders o WHERE o.customer_id = c.id
      )
  AND NOT EXISTS (                               -- ...but not recently (churn signal)
        SELECT 1
        FROM orders o
        WHERE o.customer_id = c.id
          AND o.created_at >= NOW() - INTERVAL '90 days'
      )
ORDER BY c.last_seen_at ASC;                     -- longest-dormant first
```

### Example 3 — Production Grade: Orders Missing a Successful Payment

```sql
-- CONTEXT:
--   orders:   80M rows, ~2M with status='completed' in the target window
--   payments: 120M rows
-- INDEXES REQUIRED:
--   CREATE INDEX ON orders (status, created_at);
--   CREATE INDEX ON payments (order_id, status);   -- enables index-only anti-join probe
-- PERF EXPECTATION: ~30–90ms via Nested Loop Anti Join with index-only inner probe;
--   the outer set is pre-filtered small (~a few thousand rows/day), each probe is O(log n).
--
-- GOAL: fulfilled orders with no captured payment — direct revenue-leakage report.
SELECT
  o.id                AS order_id,
  o.customer_id,
  o.total_amount,
  o.created_at
FROM orders o
WHERE o.status = 'completed'
  AND o.created_at >= DATE_TRUNC('day', NOW()) - INTERVAL '7 days'
  AND NOT EXISTS (
    SELECT 1
    FROM payments pay
    WHERE pay.order_id = o.id
      AND pay.status   = 'succeeded'
  )
ORDER BY o.created_at DESC;
```

```
EXPLAIN (ANALYZE, BUFFERS) output:

Nested Loop Anti Join  (cost=1.00..48200.00 rows=140 width=32)
                       (actual time=0.20..44.6 rows=137 loops=1)
  ->  Index Scan using orders_status_created_idx on orders o
        Index Cond: (status = 'completed' AND created_at >= (date_trunc('day', now()) - '7 days'))
        (actual time=0.03..21.2 rows=41500 loops=1)
  ->  Index Only Scan using payments_order_status_idx on payments pay
        Index Cond: (order_id = o.id AND status = 'succeeded')
        Heap Fetches: 0
        (actual time=0.0005..0.0005 rows=0 loops=41500)   ← rows=0 ⇒ order qualifies
Buffers: shared hit=166000
Planning Time: 0.6 ms
Execution Time: 45.0 ms
```

Reading: 41,500 completed orders in the window are probed; for each, an index-only scan on `payments(order_id, status)` checks for a succeeded payment. `rows=0` on the inner means "no successful payment found" → the anti-join emits the order. Only 137 leaked orders survive. `Heap Fetches: 0` confirms the covering index answered every probe without touching the heap.

---

## 9. Wrong → Right Patterns

### Wrong 1: `NOT IN` with a nullable subquery column (the empty-result bug)

```sql
-- WRONG: order_items.product_id is nullable; one NULL makes this return ZERO rows
SELECT p.id, p.name
FROM products p
WHERE p.id NOT IN (SELECT oi.product_id FROM order_items oi);
-- ACTUAL RESULT: 0 rows (always), because `x <> NULL` is UNKNOWN and poisons the AND-chain.
-- The report silently comes back empty. No error is raised.
```

Why at the execution level: `NOT IN` expands to `p.id <> v1 AND p.id <> v2 AND ... AND p.id <> NULL`. The `<> NULL` term is `UNKNOWN`, so the conjunction can never be `TRUE`, so `WHERE` rejects every row.

```sql
-- RIGHT: NOT EXISTS is NULL-safe — a NULL product_id simply doesn't match and is ignored
SELECT p.id, p.name
FROM products p
WHERE NOT EXISTS (
  SELECT 1 FROM order_items oi WHERE oi.product_id = p.id
);
```

### Wrong 2: INNER JOIN for an existence check → duplicate rows

```sql
-- WRONG: "customers who have completed orders" but each appears once PER order
SELECT c.id, c.name
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id AND o.status = 'completed';
-- BUG: a customer with 300 completed orders appears 300 times.
-- Downstream COUNT(*) is inflated; a UI list shows the same customer 300 times.
```

Why: INNER JOIN materialises one output row per matching pair (Topic 11 fan-out).

```sql
-- RIGHT: semi-join semantics — each customer at most once, no DISTINCT, stops at first match
SELECT c.id, c.name
FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status = 'completed'
);
```

### Wrong 3: `COUNT(*) > 0` instead of EXISTS

```sql
-- WRONG: counts every matching order just to check "at least one"
SELECT c.id, c.name
FROM customers c
WHERE (SELECT COUNT(*) FROM orders o WHERE o.customer_id = c.id) > 0;
-- BUG (performance): for a customer with 1M orders, this reads all 1M rows to compute a
-- count, then compares to 0. EXISTS would stop after the first row.
```

```sql
-- RIGHT: short-circuits at the first matching row
SELECT c.id, c.name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```

### Wrong 4: Forgetting the correlation predicate (uncorrelated EXISTS)

```sql
-- WRONG: no reference to c — this is a GLOBAL check
SELECT c.id, c.name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.status = 'completed');
-- BUG: if ANY completed order exists anywhere, EVERY customer is returned.
-- If none exists, NO customer is returned. The result ignores each customer entirely.
```

```sql
-- RIGHT: correlate the subquery to the current customer
SELECT c.id, c.name
FROM customers c
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status = 'completed'
);
```

### Wrong 5: `NOT EXISTS` with a correlation that also filters out the NULLs you care about

```sql
-- WRONG: intent is "orders with no *failed* payment", but the correlation is on order_id only,
-- and the status filter is INSIDE, so an order whose only payment failed still counts as
-- "has a payment" incorrectly if you misplace the condition.
-- Subtle bug: putting the status filter in the OUTER query instead of the subquery.
SELECT o.id
FROM orders o
JOIN payments pay ON pay.order_id = o.id      -- forces at least one payment row (INNER)
WHERE pay.status <> 'failed';
-- BUG: an order with two payments (one failed, one succeeded) appears; an order with a single
-- failed payment is dropped by the JOIN — but so is an order with NO payments at all, which
-- you may or may not want. The JOIN conflates "no payment" with "no non-failed payment".
```

```sql
-- RIGHT: state the anti-condition precisely with NOT EXISTS
-- "orders that have NO successful payment" (whether or not other payments exist)
SELECT o.id
FROM orders o
WHERE NOT EXISTS (
  SELECT 1 FROM payments pay
  WHERE pay.order_id = o.id AND pay.status = 'succeeded'
);
```

---

## 10. Performance Profile

### Cost Model by Pattern

| Pattern | Physical operator | Cost driver | Memory |
|---------|-------------------|-------------|--------|
| `EXISTS`, selective outer, indexed inner | Nested Loop Semi Join | O(outer × log inner), stops at 1st match | O(1) |
| `EXISTS`, large both, equality | Hash Semi Join | O(outer + inner) build+probe | O(inner distinct keys) |
| `NOT EXISTS`, indexed inner | Nested Loop Anti Join | O(outer × log inner) | O(1) |
| `NOT EXISTS`, large both | Hash Anti Join | O(outer + inner), builds full inner hash | O(inner) |
| `COUNT(*)>0` (anti-pattern) | SubPlan / Aggregate per row | O(outer × all matches) | varies |

### Scaling at 1M / 10M / 100M Rows

- **`EXISTS` with indexed inner join key** scales beautifully. The inner probe is O(log n) and stops at the first hit, so going from 1M to 100M inner rows adds only a few index levels (~log₁₀₀ growth). A `Nested Loop Semi Join` over 10K selective outer rows against a 100M-row indexed inner is typically single-digit to low-tens of milliseconds.
- **`Hash Semi/Anti Join`** builds a hash on the inner. At 100M inner rows this hash may not fit `work_mem` → `Batches > 1` → disk spill → 5–10× slowdown. Mitigations: raise `work_mem` for the session, ensure the inner is pre-filtered small, or steer toward a Nested Loop by providing the right index.
- **`NOT EXISTS` absence-proof cost**: for outer rows that have **no** match, the engine must confirm absence. With an index this is one ranged probe returning nothing (fast). **Without** an index it degrades to a full inner scan per outer row — catastrophic at scale. The index on the inner correlation key is non-negotiable for large `NOT EXISTS`.

### Index Strategy

The single most important index is on the **inner subquery's correlation column(s)**, ideally as a composite covering the inner filter:

```sql
-- For: EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status='completed')
CREATE INDEX ON orders (customer_id, status);
-- covers both the correlation (customer_id) and the inner filter (status);
-- enables an Index Only Scan that answers EXISTS without heap fetches.

-- For: NOT EXISTS (SELECT 1 FROM payments pay WHERE pay.order_id = o.id AND pay.status='succeeded')
CREATE INDEX ON payments (order_id, status);
```

Put the **correlation/equality column first**, then the inner filter columns, so a single index range serves both. A **partial index** can help when the inner filter is a fixed constant:

```sql
-- Only index successful payments — smaller, faster for this specific NOT EXISTS
CREATE INDEX ON payments (order_id) WHERE status = 'succeeded';
```

### Optimization Techniques Specific to EXISTS

1. **Let the planner flatten.** Avoid constructs that block semi-join pull-up: `LIMIT` inside the subquery, volatile functions (e.g., `random()`), and certain `ORDER BY`. Check `EXPLAIN` shows `Semi Join`, not `SubPlan`.
2. **Covering index → Index Only Scan** so the probe never touches the heap (`Heap Fetches: 0`).
3. **Pre-filter the outer** to shrink the number of probes for Nested Loop plans.
4. **Raise `work_mem`** for Hash Semi/Anti Joins on large inputs to keep `Batches: 1`.
5. **Prefer `NOT EXISTS` over `LEFT JOIN ... IS NULL`** for anti-joins in modern PostgreSQL — they usually produce the same `Anti Join` plan, but `NOT EXISTS` is clearer and can't be broken by selecting a nullable inner column that happens to be legitimately NULL.

---

## 11. Node.js Integration

### 11.1 Basic EXISTS check

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Does this customer have any completed order? Returns a boolean.
async function hasCompletedOrder(customerId) {
  const { rows } = await pool.query(
    `SELECT EXISTS (
       SELECT 1 FROM orders o
       WHERE o.customer_id = $1 AND o.status = 'completed'
     ) AS has_order`,
    [customerId]
  );
  return rows[0].has_order;   // pg maps SQL boolean → JS boolean
}
```

Returning `EXISTS (...)` as a column is the idiomatic fast existence probe — the DB stops at the first match and hands back a single boolean, far cheaper than fetching rows and checking `rowCount` in JS.

### 11.2 Anti-join report (NULL-safe) with parameters

```javascript
// Fulfilled orders with no successful payment — revenue leakage report.
async function ordersMissingPayment(sinceDays = 7) {
  const { rows } = await pool.query(
    `SELECT o.id, o.customer_id, o.total_amount, o.created_at
     FROM orders o
     WHERE o.status = 'completed'
       AND o.created_at >= NOW() - ($1 || ' days')::INTERVAL
       AND NOT EXISTS (
         SELECT 1 FROM payments pay
         WHERE pay.order_id = o.id AND pay.status = 'succeeded'
       )
     ORDER BY o.created_at DESC`,
    [sinceDays]
  );
  return rows;
}
```

### 11.3 The NOT IN trap — and why you should never build one from a JS array without filtering

```javascript
// DANGER: if excludedIds contains null/undefined, NOT IN silently returns nothing.
// pg turns JS null into SQL NULL inside the array → NOT IN poison.
async function productsExcept_BAD(excludedIds) {
  // excludedIds might be [1, 2, null] due to an upstream bug
  const { rows } = await pool.query(
    `SELECT * FROM products WHERE id <> ALL($1::int[])`,  // <> ALL behaves like NOT IN
    [excludedIds]
  );
  return rows; // may be unexpectedly empty if excludedIds has a null
}

// SAFE: strip nulls, or better, use NOT EXISTS against a real table.
async function productsExcept_GOOD(excludedIds) {
  const clean = excludedIds.filter((x) => x != null);
  const { rows } = await pool.query(
    `SELECT * FROM products WHERE id <> ALL($1::int[])`,
    [clean]
  );
  return rows;
}
```

Note: `x <> ALL(array)` is the array analogue of `NOT IN` and has the **same** NULL poisoning. In Node, sanitise arrays before binding, or drive the anti-check off a table with `NOT EXISTS`.

### 11.4 Computed EXISTS flags per row

```javascript
// Attach boolean flags to a customer list in one round trip.
async function customersWithFlags() {
  const { rows } = await pool.query(
    `SELECT
       c.id, c.name,
       EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id) AS has_ordered,
       NOT EXISTS (
         SELECT 1 FROM orders o
         WHERE o.customer_id = c.id AND o.created_at >= NOW() - INTERVAL '90 days'
       ) AS is_dormant
     FROM c_active c`
  );
  return rows; // each row: { id, name, has_ordered: boolean, is_dormant: boolean }
}
```

**Do ORMs handle EXISTS?** Basic relation-existence filters, yes (see §12). Correlated multi-condition `EXISTS`, the `NOT IN`/`NOT EXISTS` distinction, and relational-division patterns almost always require the raw-SQL escape hatch.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do EXISTS?** — Partially. Relation filters `some` / `none` compile to correlated `EXISTS` / `NOT EXISTS` semi-/anti-joins.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// EXISTS: customers with at least one completed order → uses `some`
const buyers = await prisma.customer.findMany({
  where: { orders: { some: { status: 'completed' } } },  // → EXISTS (...)
});

// NOT EXISTS: customers with NO orders in the last 90 days → uses `none`
const dormant = await prisma.customer.findMany({
  where: {
    orders: { none: { createdAt: { gte: new Date(Date.now() - 90*864e5) } } }, // → NOT EXISTS
  },
});

// Relational division / correlated multi-table EXISTS → raw SQL
const orderedAllPremium = await prisma.$queryRaw`
  SELECT c.id, c.name FROM customers c
  WHERE NOT EXISTS (
    SELECT 1 FROM products p WHERE p.category = 'Premium'
      AND NOT EXISTS (
        SELECT 1 FROM order_items oi JOIN orders o ON o.id = oi.order_id
        WHERE oi.product_id = p.id AND o.customer_id = c.id))`;
```

**Where it breaks**: `some`/`none` only correlate on the defined relation. Multi-column correlation across unrelated tables, `NOT EXISTS` with extra join conditions, and division patterns need `$queryRaw`. **Verdict**: excellent for the common existence filters; raw SQL for anything correlated beyond a single relation.

---

### Drizzle ORM

**Can Drizzle do EXISTS?** — Yes, first-class via the `exists()` / `notExists()` helpers with a correlated subquery.

```typescript
import { db } from './db';
import { customers, orders } from './schema';
import { exists, notExists, and, eq, gte, sql } from 'drizzle-orm';

// EXISTS
const buyers = await db.select().from(customers).where(
  exists(
    db.select({ one: sql`1` }).from(orders)
      .where(and(eq(orders.customerId, customers.id), eq(orders.status, 'completed')))
  )
);

// NOT EXISTS (anti-join, NULL-safe)
const dormant = await db.select().from(customers).where(
  notExists(
    db.select({ one: sql`1` }).from(orders)
      .where(and(
        eq(orders.customerId, customers.id),
        gte(orders.createdAt, sql`NOW() - INTERVAL '90 days'`)
      ))
  )
);
```

**Where it breaks**: nothing significant — the correlation reference (`customers.id` inside the subquery) is fully typed. Deeply nested division is verbose but expressible. **Verdict**: best-in-class; `exists`/`notExists` map straight to the SQL and stay type-safe.

---

### Sequelize

**Can Sequelize do EXISTS?** — Not directly. There is no native `EXISTS` operator; you emulate it with `Sequelize.literal` inside `where`, or use `include` with `required` (which is a JOIN, not a semi-join — beware fan-out).

```javascript
const { Op, literal } = require('sequelize');

// EXISTS via literal (correlated subquery hand-written)
const buyers = await Customer.findAll({
  where: literal(`EXISTS (
    SELECT 1 FROM orders o WHERE o.customer_id = "Customer"."id" AND o.status = 'completed'
  )`),
});

// NOT EXISTS via literal
const dormant = await Customer.findAll({
  where: literal(`NOT EXISTS (
    SELECT 1 FROM orders o
    WHERE o.customer_id = "Customer"."id" AND o.created_at >= NOW() - INTERVAL '90 days'
  )`),
});
```

**Where it breaks**: `include: { required: true }` looks like EXISTS but is an INNER JOIN → duplicates unless you add `group`/`distinct`. The `literal` approach works but hard-codes the quoted table alias (`"Customer"`) and loses type safety. **Verdict**: use `literal` for real semi-/anti-joins; avoid faking EXISTS with `required` JOINs.

---

### TypeORM

**Can TypeORM do EXISTS?** — Yes, via `.where('EXISTS (...)')` with a `QueryBuilder` subquery, which keeps it parameterised.

```typescript
const buyers = await dataSource
  .getRepository(Customer)
  .createQueryBuilder('c')
  .where((qb) => {
    const sub = qb.subQuery()
      .select('1')
      .from('orders', 'o')
      .where('o.customer_id = c.id')
      .andWhere("o.status = 'completed'")
      .getQuery();
    return `EXISTS ${sub}`;
  })
  .getMany();

// NOT EXISTS: same pattern, prefix with NOT
// return `NOT EXISTS ${sub}`;
```

**Where it breaks**: the correlation (`o.customer_id = c.id`) is a raw string — no compile-time check that alias `c` exists. Nested `NOT EXISTS` division gets unwieldy in the builder. **Verdict**: workable and parameter-safe; drop to `dataSource.query()` for complex correlated logic.

---

### Knex.js

**Can Knex do EXISTS?** — Yes, cleanly, via `whereExists` / `whereNotExists` taking a subquery builder.

```javascript
// EXISTS
const buyers = await knex('customers as c')
  .select('c.id', 'c.name')
  .whereExists(function () {
    this.select(knex.raw('1'))
      .from('orders as o')
      .whereRaw('o.customer_id = c.id')
      .andWhere('o.status', 'completed');
  });

// NOT EXISTS (NULL-safe anti-join)
const neverOrdered = await knex('products as p')
  .select('p.id', 'p.name')
  .whereNotExists(function () {
    this.select(knex.raw('1'))
      .from('order_items as oi')
      .whereRaw('oi.product_id = p.id');
  });
```

**Where it breaks**: the correlation predicate uses `whereRaw` (Knex can't type the cross-query column reference), but the structure is otherwise clean and composable. **Verdict**: most transparent — `whereExists`/`whereNotExists` are exactly the right primitives and map 1:1 to SQL.

---

### ORM Summary Table

| ORM | EXISTS support | NOT EXISTS | Correlated multi-table | NULL-safe by default | Verdict |
|-----|----------------|-----------|------------------------|----------------------|---------|
| Prisma | `some` filter | `none` filter | Raw SQL | Yes (`none` = anti-join) | Great for single-relation checks |
| Drizzle | `exists()` | `notExists()` | Typed subquery | Yes | Best typed support |
| Sequelize | `literal` only | `literal` only | `literal` | Only if you write NOT EXISTS | Avoid faking with required JOIN |
| TypeORM | subquery + `EXISTS` string | `NOT EXISTS` string | Raw string correlation | Yes if you use NOT EXISTS | Workable, param-safe |
| Knex | `whereExists` | `whereNotExists` | `whereRaw` correlation | Yes | Most transparent |

The cross-cutting lesson: every ORM that offers a real "none"/`whereNotExists` primitive gives you NULL-safe anti-joins for free. The danger is the ORMs where developers *fake* it with `NOT IN`-style array filters or `required` JOINs.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given:
- `users(id, name, email, created_at)`
- `sessions(id, user_id, started_at, ended_at)`

Write a query returning every user who has **at least one** session — `id`, `name`, `email` — without returning duplicates and without using `DISTINCT` or a `JOIN`. Then write a second query returning users who have **never** had a session, NULL-safe.

```sql
-- Write your query here
```

---

### Exercise 2 — Intermediate (combines Topic 11 JOIN + Topic 20 GROUP BY)

Given:
- `orders(id, customer_id, status, total_amount, created_at)`
- `order_items(id, order_id, product_id, quantity, unit_price)`
- `products(id, name, category_id)`
- `categories(id, name)`

For each **category**, return `category_name` and `products_never_ordered` — the count of products in that category that have **never** appeared in any order item. Only include categories that have at least one product. Order by `products_never_ordered` descending. Use `NOT EXISTS` for the never-ordered test (explain in a comment why `NOT IN` would be unsafe here).

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

A developer wrote this to find "products that have never been ordered" on a table where `order_items.product_id` is nullable (some legacy rows have NULL):

```sql
SELECT p.id, p.name
FROM products p
WHERE p.id NOT IN (SELECT oi.product_id FROM order_items oi);
```

1. Explain precisely why this returns zero rows.
2. Rewrite it correctly two ways: (a) with `NOT EXISTS`, (b) with `NOT IN` made safe.
3. The `order_items` table has 50M rows and `products` has 200K. What index makes the `NOT EXISTS` version fast, and what `EXPLAIN` node do you expect to see?
4. Bonus: extend the correct query to only consider orders with `status = 'completed'` (so a product ordered only in cancelled orders still counts as "never really ordered").

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

You maintain a subscriptions system:
- `users(id, name)`
- `subscriptions(id, user_id, plan, status, started_at, canceled_at)`
- `payments(id, user_id, subscription_id, status, amount, paid_at)`

Write a single query that returns users who are **at risk**: they have an `active` subscription but have **no** `succeeded` payment in the last 35 days. Make it NULL-safe. Then describe how you'd verify with `EXPLAIN` that the subquery was flattened into an anti-join rather than run per row, and which index you'd add.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What does `EXISTS` return, and does the `SELECT` list inside it matter?**

A: `EXISTS(subquery)` returns a boolean: `TRUE` if the subquery produces at least one row, `FALSE` if it produces none. The `SELECT` list inside is irrelevant — `SELECT 1`, `SELECT *`, and `SELECT column` all behave identically because `EXISTS` only checks whether rows exist, not what they contain. The planner discards the target list. Idiomatically we write `SELECT 1`. There's no performance difference between `SELECT 1` and `SELECT *` in PostgreSQL.

---

**Q: Why might `SELECT * FROM products WHERE id NOT IN (SELECT product_id FROM order_items)` return no rows even though many products were never ordered?**

A: Because `order_items.product_id` contains a NULL. `NOT IN` expands to a chain of `id <> value` comparisons AND-ed together, and `id <> NULL` evaluates to `UNKNOWN` (three-valued logic). `TRUE AND UNKNOWN` is `UNKNOWN`, and `WHERE UNKNOWN` filters the row out — so no row can ever qualify. The fix is `NOT EXISTS`, which uses equality inside a membership test and simply ignores the NULL row instead of being poisoned by it.

---

**Q: When would you use `EXISTS` instead of an `INNER JOIN`?**

A: When I only need to know *whether* a related row exists, not any columns from it. `EXISTS` is a semi-join: each outer row appears at most once, so I avoid the row multiplication (fan-out) an INNER JOIN produces and I don't need `DISTINCT`. It also short-circuits at the first match. I'd use an INNER JOIN instead when I actually need to *project columns* from the related table, since `EXISTS` can't return inner columns.

---

### Principal Level

**Q: Is `EXISTS` slower than `IN` because the subquery runs once per row? Walk me through what PostgreSQL actually does.**

A: No — that's the junior misconception. PostgreSQL's planner performs a **subquery-to-semi-join transformation**: a correlated `WHERE EXISTS (...)` is pulled up and executed as a single `Semi Join` node (Hash Semi Join, Nested Loop Semi Join, or Merge Semi Join), not re-run per outer row. In `EXPLAIN` you'll see the `Semi Join` operator rather than a `SubPlan` with high `loops`. `IN` with an uncorrelated subquery is *also* turned into a semi-join. So in modern PostgreSQL `EXISTS` and `IN` typically produce the **same plan and same performance** for the positive case. The real difference is on the negative side: `NOT IN` can't safely become an anti-join when NULLs are possible (it must preserve the NULL semantics), whereas `NOT EXISTS` always becomes a clean `Anti Join`. The times `EXISTS` won't flatten are when something blocks pull-up — a `LIMIT` in the subquery, a volatile function, or certain lateral correlations — then you *do* get a per-row `SubPlan`, which is when it's genuinely slower.

---

**Q: Design a NULL-safe "anti-join" report: fulfilled orders with no successful payment, over an 80M-row orders table and 120M-row payments table. What query, what indexes, what plan?**

A: Use `NOT EXISTS`, never `NOT IN` — `payments.order_id` or `status` could be NULL and `NOT IN` would silently empty the result.
```sql
SELECT o.id, o.total_amount
FROM orders o
WHERE o.status = 'completed'
  AND o.created_at >= NOW() - INTERVAL '7 days'
  AND NOT EXISTS (
    SELECT 1 FROM payments p WHERE p.order_id = o.id AND p.status = 'succeeded'
  );
```
Indexes: `orders(status, created_at)` to make the outer set small and selective, and `payments(order_id, status)` — correlation column first, filter column second — to enable an **Index Only Scan** on the inner probe. I'd expect a `Nested Loop Anti Join`: the outer produces a few thousand recent completed orders, and each probes the payments index; `actual rows=0` on the inner means "no successful payment," so the anti-join emits the order. With a covering index, `Heap Fetches: 0`. If the planner instead chose a `Hash Anti Join` that spilled (`Batches > 1`), I'd raise `work_mem` or tighten the outer filter. A partial index `payments(order_id) WHERE status='succeeded'` shrinks the probe further.

---

**Q: Explain the double-`NOT EXISTS` relational-division query and its behaviour on the empty set.**

A: "Customers who ordered every premium product" is relational division. You express it as "there is no premium product that this customer has not ordered":
```sql
WHERE NOT EXISTS (
  SELECT 1 FROM products p WHERE p.category='Premium'
    AND NOT EXISTS (SELECT 1 FROM order_items oi JOIN orders o ON o.id=oi.order_id
                    WHERE oi.product_id=p.id AND o.customer_id=c.id))
```
The inner `NOT EXISTS` finds premium products the customer hasn't ordered; the outer `NOT EXISTS` requires that set to be empty. Empty-set behaviour: if there are **no** premium products at all, the inner query yields nothing, the outer `NOT EXISTS` is vacuously `TRUE`, and **every** customer qualifies — the mathematically correct "everyone has ordered all zero premium products." An alternative is `GROUP BY customer HAVING COUNT(DISTINCT premium_product) = (total premium count)`, which is often more readable but handles the empty-set case differently (the equality can degenerate), so I'd choose based on the exact semantics required and confirm with tests.

---

## 15. Mental Model Checkpoint

1. A customer has 2 million completed orders. How many `orders` rows does `WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id AND o.status='completed')` read for that customer, assuming a good index? Why?

2. `WHERE id NOT IN (SELECT product_id FROM order_items)` returns zero rows in production but worked in your test database. What's different about production data, and what's the exact three-valued-logic mechanism?

3. You replace `INNER JOIN orders ... ` + `SELECT DISTINCT` with `WHERE EXISTS (...)`. The result set is identical. What did you gain, and in what situation would the two forms *not* be interchangeable?

4. `EXPLAIN` shows a `SubPlan` with `loops=1000000` instead of a `Semi Join`. What are three things that could have prevented the planner from flattening your `EXISTS`?

5. For `NOT EXISTS`, which outer rows are cheap to evaluate and which are expensive, and why does an index on the inner correlation key fix the expensive case?

6. You have an uncorrelated `EXISTS` in a `WHERE` clause (no reference to the outer row). What does the result look like, and why is this almost always a bug?

7. Someone claims `SELECT 1` inside `EXISTS` is faster than `SELECT *`. Are they right in PostgreSQL? Explain what the planner does with the target list.

---

## 16. Quick Reference Card

```sql
-- EXISTS: TRUE iff subquery returns ≥1 row; short-circuits at first match (semi-join)
WHERE EXISTS (SELECT 1 FROM child c WHERE c.parent_id = p.id)

-- NOT EXISTS: TRUE iff subquery returns 0 rows (anti-join); NULL-SAFE
WHERE NOT EXISTS (SELECT 1 FROM child c WHERE c.parent_id = p.id)

-- SELECT 1 / SELECT * / SELECT col inside EXISTS are IDENTICAL (target list ignored)

-- ⚠️ THE TRAP: NOT IN + NULL ⇒ ZERO ROWS, silently
WHERE x NOT IN (SELECT nullable_col FROM t)     -- BROKEN if any NULL present
WHERE NOT EXISTS (SELECT 1 FROM t WHERE t.col = x)  -- CORRECT replacement
-- If you MUST use NOT IN:
WHERE x NOT IN (SELECT col FROM t WHERE col IS NOT NULL)

-- EXISTS vs alternatives:
--   EXISTS      → existence check, no fan-out, no DISTINCT, stops at 1st match
--   INNER JOIN  → use when you need COLUMNS from the child (multiplies rows)
--   COUNT(*)>0  → anti-pattern; reads ALL matches to answer "≥1?"
--   IN          → fine for POSITIVE checks; unsafe only in the NOT form

-- Existence as a boolean column / flag:
SELECT p.id, EXISTS (SELECT 1 FROM child c WHERE c.parent_id=p.id) AS has_child FROM p;

-- Relational division (ordered EVERY premium product) — double NOT EXISTS:
WHERE NOT EXISTS (SELECT 1 FROM products p WHERE p.category='Premium'
  AND NOT EXISTS (SELECT 1 FROM order_items oi JOIN orders o ON o.id=oi.order_id
                  WHERE oi.product_id=p.id AND o.customer_id=c.id))

-- Index rule: correlation column FIRST, inner filter SECOND (covering → Index Only Scan)
CREATE INDEX ON orders (customer_id, status);       -- for EXISTS(... customer_id AND status)
CREATE INDEX ON payments (order_id) WHERE status='succeeded';  -- partial, for NOT EXISTS

-- EXPLAIN signals:
--   Hash/Nested Loop Semi Join   → EXISTS flattened (good)
--   Hash/Nested Loop Anti Join   → NOT EXISTS flattened (good)
--   inner actual rows=1 (semi)   → short-circuit working
--   SubPlan + high loops         → NOT flattened; per-row execution (bad)
--   Batches > 1                  → hash spill; raise work_mem

-- ORM one-liners:
--   Prisma:  { orders: { some: {...} } } / { orders: { none: {...} } }
--   Drizzle: exists(sub) / notExists(sub)
--   Knex:    .whereExists(fn) / .whereNotExists(fn)
--   Sequelize: literal('EXISTS (...)')  — no native operator
--   TypeORM: .where(`EXISTS ${qb.subQuery()...getQuery()}`)
```

---

## Connected Topics

- **Topic 07 — NULL in Depth**: The three-valued logic (`x <> NULL = UNKNOWN`) that makes `NOT IN` fail and `NOT EXISTS` safe. Read this to fully understand the trap in §6.3.
- **Topic 11 — INNER JOIN in Depth**: The fan-out that `EXISTS` avoids; the semi-join is INNER JOIN without row multiplication or projection.
- **Topic 12 — LEFT JOIN and RIGHT JOIN**: `LEFT JOIN ... WHERE right IS NULL` is the alternative anti-join form; usually the same `Anti Join` plan as `NOT EXISTS`.
- **Topic 19 — JOIN Alternatives**: Where `EXISTS` was first introduced as an existence-check alternative to joining.
- **Topic 20 — GROUP BY Fundamentals**: `HAVING COUNT(...) = N` as the alternative to double-`NOT EXISTS` relational division.
- **Topic 26 — Subqueries in Depth**: Correlated vs uncorrelated subqueries, scalar subqueries, and the pull-up transformations the planner applies — the immediate predecessor to this topic.
- **Topic 28 — Common Table Expressions (CTEs)**: How to name and reuse the subqueries an `EXISTS`/`NOT EXISTS` correlates against, and when a CTE materialisation boundary changes the plan.
- **Internals — Semi-Join / Anti-Join operators**: The relational-algebra `⋉` and `▷` that `EXISTS`/`NOT EXISTS` compile to, and the physical Hash/Nested-Loop/Merge implementations that execute them.
```
