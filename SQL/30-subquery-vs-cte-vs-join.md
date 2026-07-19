# Topic 30 — Subquery vs CTE vs JOIN
### SQL Mastery Curriculum — Phase 5: Subqueries and CTEs

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a busy kitchen and a customer orders a "chicken sandwich with the house sauce, plated next to the soup of the day." There are three ways your line cooks can pull this off.

**The JOIN way** — everyone works the same cutting board at once. The person on the grill, the person on the sauce, and the person on the soup all reach across each other, combining ingredients in a single coordinated motion. It's fast and there's no wasted movement, but the board can get crowded and confusing when six people are reaching over it.

**The subquery way** — you shout an instruction *inside* another instruction: "grab me the sauce that goes with whatever the grill guy is making right now." The sauce cook has to keep glancing at the grill to know what to make. If the grill changes, the sauce changes. It reads naturally ("the thing that matches this other thing"), but the sauce cook is tied to the grill cook's every move.

**The CTE way** — you write the steps on a whiteboard first: "Step 1: make the sauce batch. Step 2: make the soup. Step 3: assemble." Each step has a *name*. Anyone can read the whiteboard top to bottom and understand the whole dish. It's the clearest to read — but here's the trick a great head chef knows: **writing it on the whiteboard doesn't necessarily mean each step is cooked separately and set aside.** A smart kitchen still combines steps on the fly when it's faster.

The punchline of this entire topic: **all three are usually the same dish.** The grill, the sauce, and the soup end up on the same plate no matter which way you organized the work. The question is never "which one gives the right answer" — they almost always give the *same* answer. The question is "which one reads best, and which one does the kitchen actually cook the fastest?" And since PostgreSQL 12, the answer to the second question is far more often "they're identical" than most developers believe.

---

## 2. Connection to SQL Internals

At the engine level, `subquery`, `CTE`, and `JOIN` are not three different machines. They are three different **surface syntaxes** that get compiled down into the same internal representation: a **query tree** of relational operators (scans, joins, aggregates, sorts) that the planner then rearranges freely.

The key internal concepts:

1. **The parser / rewriter / planner pipeline.** SQL text is parsed into a raw parse tree, rewritten (views expanded, some subqueries pulled up), then handed to the planner/optimizer, which produces a physical plan of executor nodes. By the time the executor runs, the words "CTE", "subquery", and "JOIN" no longer exist — only `Hash Join`, `Nested Loop`, `Seq Scan`, `Aggregate`, and `CTE Scan` nodes.

2. **Subquery flattening (a.k.a. "pull-up").** The planner routinely *flattens* subqueries in the `FROM` clause and `IN`/`EXISTS` subqueries into joins. A `WHERE x IN (SELECT ...)` becomes a **semi-join**; a `WHERE NOT EXISTS (...)` becomes an **anti-join**. A derived table in `FROM` gets merged into the parent query's join tree. This is why a subquery and a JOIN so often produce identical plans — the planner literally turns one into the other.

3. **CTE inlining (PG12+).** This is the headline fact of this topic. Before PostgreSQL 12, a `WITH` clause was an **optimization fence** — always materialized into a temporary in-memory work table (a `CTE Scan` node), executed exactly once, with no predicate push-down across the boundary. Since **PostgreSQL 12**, a CTE that is (a) referenced exactly once, (b) not recursive, and (c) side-effect-free (no `INSERT`/`UPDATE`/`DELETE`, no volatile functions) is **inlined** by default — spliced directly into the outer query tree as if it were a subquery, and then optimized as one unit. The fence is gone unless you ask for it with `MATERIALIZED`.

4. **The optimization fence.** `WITH x AS MATERIALIZED (...)` forces the old behavior: the CTE is computed once into a work table, and the planner may **not** push predicates in or pull columns out. `WITH x AS NOT MATERIALIZED (...)` forces inlining even when referenced multiple times. This single keyword is the crux of "when they differ."

5. **`work_mem` and materialization.** A materialized CTE or a subquery that must be sorted/hashed lands in `work_mem`. If it exceeds `work_mem`, it spills to a temp file on disk — the same spill behavior you saw for Hash Joins in Topic 11. The three syntaxes only differ in performance when one of them forces a materialization the others avoid.

The mental model to carry through this entire document: **the planner sees a graph of relations, not your syntax.** Your job is to know when your syntax *constrains* the planner (materialized CTE, `LIMIT` inside a subquery, volatile functions) versus when it leaves the planner free (inlinable CTE, flattenable subquery, plain JOIN).

---

## 3. Logical Execution Order Context

Recall the logical clause order from Topic 02 and the JOIN topics:

```
WITH        ← named subqueries defined here (but NOT necessarily executed first!)
FROM        ← base tables, derived tables (FROM-subqueries), CTE references resolved
JOIN ... ON ← joins evaluated
WHERE       ← row filter (may contain scalar/IN/EXISTS subqueries)
GROUP BY
HAVING      ← may contain subqueries
SELECT      ← may contain scalar/correlated subqueries in the select list
DISTINCT
window
ORDER BY
LIMIT
```

The critical, counter-intuitive point: **the position of `WITH` at the top of the text does NOT mean the CTE runs first.** That is the single most common misconception in this entire phase.

- With **inlining** (PG12+ default, single reference): the CTE's logic is dissolved into wherever it is referenced. It runs *interleaved* with the outer query, exactly like a `FROM`-clause subquery. There is no "first."
- With **`MATERIALIZED`**: the CTE genuinely does compute first, into a work table, before the outer query touches it. Here, and only here, does the textual top-to-bottom order map to execution order.

For **subqueries**, logical position depends on *where* they sit:

- A **`FROM`-clause subquery** (derived table) is resolved in the FROM phase, then flattened or materialized.
- A **`WHERE` `IN`/`EXISTS` subquery** is evaluated as part of the WHERE phase — turned into a semi-join.
- A **scalar subquery in `SELECT`** is evaluated per output row (unless it's uncorrelated, in which case it's computed once and cached — an `InitPlan`).
- A **correlated subquery** re-evaluates conceptually once per outer row, though the planner often rewrites it into a join to avoid that.

For **JOINs**, everything happens in the `FROM`/`JOIN` phase — the earliest stage — which is why a JOIN gives the planner the most freedom to reorder, filter early, and pick algorithms.

The practical takeaway: if you write `WITH recent AS (SELECT ... WHERE created_at > X)` and reference it once, PG12+ does *not* build `recent` first and then scan it. It pushes your outer predicates *into* `recent` and plans the whole thing as one query. The clause order in your text is a reading convenience, not an execution contract.

---

## 4. What Is Subquery vs CTE vs JOIN?

These are three syntaxes for combining or filtering data from multiple relations. A **JOIN** matches rows from two relations on a condition (Topic 10–17). A **subquery** is a `SELECT` nested inside another statement — in `FROM`, `WHERE`, `SELECT`, or `HAVING`. A **CTE** (Common Table Expression, the `WITH` clause) is a named subquery defined before the main query and referenced by name. This topic is about **choosing between them** and understanding **when the planner treats them identically.**

```sql
-- ┌─ JOIN form: match orders to customers directly in the FROM clause
SELECT c.name, o.total_amount
FROM customers c                          -- │ base relation
INNER JOIN orders o                       -- │ combined in the FROM/JOIN phase
  ON o.customer_id = c.id                 -- │ └── join predicate
WHERE o.status = 'completed';             -- │ outer filter (pushed into the join)


-- ┌─ Subquery form (derived table in FROM): pre-shape a relation, then join it
SELECT c.name, o.total_amount
FROM customers c
INNER JOIN (
  SELECT customer_id, total_amount        -- │ └── nested SELECT = "subquery"
  FROM orders                             -- │     produces a virtual relation
  WHERE status = 'completed'              -- │     filter lives inside
) o ON o.customer_id = c.id;              -- │ join the derived table by alias


-- ┌─ Subquery form (IN / semi-join in WHERE): filter by existence
SELECT c.name
FROM customers c
WHERE c.id IN (                           -- │ └── subquery in WHERE
  SELECT customer_id FROM orders          -- │     becomes a SEMI-JOIN internally
  WHERE status = 'completed'              -- │     no duplicates, stops at first match
);


-- ┌─ CTE form: name the subquery up front, reference it by name
WITH completed_orders AS (                -- │ └── named subquery (the CTE)
  SELECT customer_id, total_amount        -- │     PG12+: INLINED if referenced once
  FROM orders
  WHERE status = 'completed'
)                                         -- │
SELECT c.name, co.total_amount            -- │
FROM customers c
INNER JOIN completed_orders co            -- │ reference the CTE by its name
  ON co.customer_id = c.id;


-- ┌─ CTE with an explicit fence (forces materialize-once behavior)
WITH completed_orders AS MATERIALIZED (   -- │ └── MATERIALIZED = optimization fence
  SELECT customer_id, total_amount        -- │     computed once into a work table
  FROM orders WHERE status = 'completed'  -- │     predicates CANNOT be pushed in
)
SELECT c.name, co.total_amount
FROM customers c
INNER JOIN completed_orders co ON co.customer_id = c.id;
```

Annotation of the decisive keywords:

- `WITH name AS (...)` — defines a CTE. Since PG12, inlined by default when referenced once and non-recursive and non-volatile.
- `AS MATERIALIZED (...)` — force the CTE to compute once into a work table. Optimization fence.
- `AS NOT MATERIALIZED (...)` — force inlining even with multiple references.
- `WITH RECURSIVE` — recursive CTE (Topic 29). **Never** inlined; always materialized by nature.
- `IN (SELECT ...)` / `EXISTS (SELECT ...)` — subquery predicate → semi-join.
- `NOT IN` / `NOT EXISTS` — → anti-join (with a critical NULL trap, see §6.6).
- A derived table `FROM (SELECT ...) alias` — flattened into the parent when possible.

The equivalence claim, stated precisely: **for a single-reference, non-recursive, side-effect-free CTE, the three forms above compile to the same plan on PostgreSQL 12 and later.** The divergences are the interesting part, and §6 is where they live.

---

## 5. Why Subquery vs CTE vs JOIN Mastery Matters in Production

1. **The "CTEs are always slow" myth costs real money.** A whole generation of developers learned (correctly, pre-2019) that "CTEs are an optimization fence — never use them in hot paths." That advice is **obsolete since PostgreSQL 12**. Engineers still refactor readable CTEs into unreadable nested subqueries "for performance," gaining nothing and losing maintainability. Knowing exactly when the fence exists lets you keep the readable form without the penalty.

2. **The `MATERIALIZED` fence can silently 100× a query.** The flip side: a CTE referenced once but written before PG12 semantics were understood, or copied from Stack Overflow with `MATERIALIZED`, blocks predicate push-down. A `WHERE id = 42` that would have used an index scan instead triggers a full materialization of millions of rows first. This is a real, common, catastrophic slowdown.

3. **`COUNT(DISTINCT)` and fan-out bugs hide in the wrong choice.** As you saw in Topic 11, joining to a one-to-many table multiplies rows. Whether you fix that with a pre-aggregating subquery/CTE or with `COUNT(DISTINCT)` changes both correctness *and* performance. Choosing the wrong structure produces wrong numbers that pass code review because they "look reasonable."

4. **`NOT IN` with a nullable column returns zero rows.** The single most dangerous divergence: `NOT IN (SELECT col ...)` where `col` can be NULL returns an empty set — silently, no error. `NOT EXISTS` and anti-join do not. Knowing which subquery form is NULL-safe prevents a class of production data-integrity bugs.

5. **Readability is a production concern, not a luxury.** A 4-level nested subquery that a senior engineer can't parse at 2am during an incident is a liability. A CTE chain that reads top-to-bottom is debuggable. Since PG12 removed the performance penalty for single-reference CTEs, the readability win is often free — you should reach for it deliberately.

6. **`EXPLAIN` is the only source of truth.** The entire topic reduces to one professional habit: never *assume* which form is faster — run `EXPLAIN (ANALYZE, BUFFERS)` and read whether the plans are identical. Principal engineers prove equivalence; juniors argue about it.

---

## 6. Deep Technical Content

### 6.1 The Three Forms Are Usually the Same Plan (PG12+)

Start from the strongest claim and then find its exceptions. Consider "customers who have a completed order." Here are three spellings:

```sql
-- Form A: JOIN + DISTINCT
SELECT DISTINCT c.id, c.name
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'completed';

-- Form B: IN subquery
SELECT c.id, c.name
FROM customers c
WHERE c.id IN (SELECT customer_id FROM orders WHERE status = 'completed');

-- Form C: EXISTS subquery
SELECT c.id, c.name
FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o
              WHERE o.customer_id = c.id AND o.status = 'completed');

-- Form D: CTE
WITH completed AS (
  SELECT DISTINCT customer_id FROM orders WHERE status = 'completed'
)
SELECT c.id, c.name
FROM customers c
INNER JOIN completed cp ON cp.customer_id = c.id;
```

Forms B, C, and D almost always produce an **identical plan**: a **semi-join** (`Hash Semi Join` or `Nested Loop Semi Join`). Form A produces a full inner join *plus* a `Unique`/`HashAggregate` to implement `DISTINCT` — which is *sometimes* the same cost and sometimes slightly worse because it materializes duplicates before removing them. The semi-join forms are theoretically cleaner because a semi-join **stops at the first match** per outer row — no duplicates ever generated. This is the one place where `EXISTS`/`IN` can genuinely beat `JOIN + DISTINCT`.

The point stands: **B, C, D are the same plan.** The choice among them is pure readability.

### 6.2 CTE Inlining — The Exact Rules (PG12+)

A CTE is **inlined** (treated exactly like a `FROM`-clause subquery, then optimized as one unit) if and only if **all** of these hold:

1. It is **not** `RECURSIVE`.
2. It is referenced **exactly once** in the query.
3. It does **not** contain volatile functions (`random()`, `nextval()`, `now()` is stable so it's fine, but `clock_timestamp()` is volatile) in a way that would change results if evaluated more than once.
4. It is **side-effect-free** — it is a plain `SELECT`, not a data-modifying CTE (`INSERT`/`UPDATE`/`DELETE ... RETURNING`).
5. You did not force `MATERIALIZED`.

If it is referenced **more than once**, PostgreSQL defaults to **materializing** it (compute once, reuse) — because recomputing an expensive subquery N times is usually worse than materializing once. You can override this with `NOT MATERIALIZED`.

```sql
-- Inlined (single reference, no fence): plan identical to a subquery
WITH big AS (SELECT * FROM orders WHERE created_at > '2025-01-01')
SELECT * FROM big WHERE customer_id = 42;
-- The customer_id = 42 predicate is PUSHED INTO the CTE.
-- If there's an index on (customer_id, created_at), it gets used. Fast.

-- Materialized by default (referenced twice): computed once, reused
WITH big AS (SELECT * FROM orders WHERE created_at > '2025-01-01')
SELECT (SELECT count(*) FROM big) AS total,
       (SELECT count(*) FROM big WHERE status = 'completed') AS done;
-- 'big' is built once; both references scan the same work table.
```

### 6.3 The Optimization Fence — `MATERIALIZED`

`MATERIALIZED` restores pre-PG12 behavior on demand. It is a **fence**: the planner treats the CTE as an opaque, pre-computed relation. Two consequences:

1. **No predicate push-down.** Filters in the outer query cannot move inside the CTE.
2. **Computed exactly once**, into `work_mem` (spilling to disk if too large).

When the fence *helps*:

```sql
-- A deliberately expensive computation you reference multiple times,
-- or want to compute exactly once to avoid re-execution of a volatile step.
WITH expensive AS MATERIALIZED (
  SELECT product_id, my_expensive_scoring_function(features) AS score
  FROM products
)
SELECT * FROM expensive WHERE score > 0.9
UNION ALL
SELECT * FROM expensive WHERE score BETWEEN 0.5 AND 0.9;
-- Without MATERIALIZED and with two references, PG materializes anyway,
-- but MATERIALIZED makes the intent explicit and guarantees single evaluation.
```

When the fence *hurts* (the classic catastrophe):

```sql
-- BAD: MATERIALIZED blocks the customer_id predicate from reaching the index
WITH all_orders AS MATERIALIZED (
  SELECT * FROM orders   -- 50M rows
)
SELECT * FROM all_orders WHERE customer_id = 42;
-- Plan: materialize ALL 50M rows into work_mem/disk, THEN filter to ~5 rows.
-- Removing MATERIALIZED (or using NOT MATERIALIZED) → Index Scan, <1ms.
```

**Rule of thumb:** never use `MATERIALIZED` on a CTE that the outer query filters, unless you have measured that materialization is genuinely cheaper (rare — usually only when the CTE is referenced many times and is expensive to recompute).

### 6.4 `NOT MATERIALIZED` — Forcing Inlining

The inverse fence-breaker. Use it when a CTE is referenced multiple times (so PG would materialize by default) but you *know* inlining each reference is cheaper — typically because each reference has a different, highly selective predicate that would use an index.

```sql
WITH active_users AS NOT MATERIALIZED (
  SELECT * FROM users WHERE deleted_at IS NULL
)
SELECT (SELECT count(*) FROM active_users WHERE country = 'US') AS us,
       (SELECT count(*) FROM active_users WHERE country = 'IN') AS in_;
-- NOT MATERIALIZED: each reference inlines, so country predicates hit
-- an index on (deleted_at, country) independently. Two cheap index scans
-- beat one giant materialization if users is huge and each slice is small.
```

### 6.5 Subquery Flattening (Pull-Up) — What the Planner Does For Free

The planner flattens subqueries into the parent query whenever it is safe. Cases:

- **`FROM`-clause derived table** without `LIMIT`, `DISTINCT`, `GROUP BY`, or window functions → merged into the parent's join tree.
- **`IN (SELECT ...)`** → semi-join.
- **`EXISTS (SELECT ...)`** → semi-join.
- **`NOT EXISTS (SELECT ...)`** → anti-join.
- **Uncorrelated scalar subquery** in `SELECT`/`WHERE` → computed once as an `InitPlan`, result cached.
- **Correlated scalar subquery** → sometimes rewritten as a join, sometimes left as a `SubPlan` re-executed per row.

What **blocks** flattening (forces materialization / a subquery boundary):

- `LIMIT` inside the subquery (changes semantics — you can't push a filter past a LIMIT).
- `DISTINCT`, `GROUP BY`, `HAVING`, window functions, `ORDER BY` with `LIMIT`.
- Set operations (`UNION`, `INTERSECT`, `EXCEPT`).
- Volatile functions.

```sql
-- Flattened: identical to a plain JOIN
SELECT * FROM customers c
JOIN (SELECT * FROM orders WHERE status='completed') o ON o.customer_id = c.id;

-- NOT flattened: LIMIT forces the subquery to run as its own unit first
SELECT * FROM customers c
JOIN (SELECT * FROM orders WHERE status='completed'
      ORDER BY created_at DESC LIMIT 100) o ON o.customer_id = c.id;
-- The LIMIT 100 must be computed before the join — it changes the result set.
```

### 6.6 The `NOT IN` NULL Trap — The Most Dangerous Divergence

This is the one divergence that produces **silently wrong results**, so it gets its own section.

```sql
-- Goal: customers who have NEVER placed an order.

-- DANGEROUS: NOT IN with a nullable column
SELECT c.id, c.name
FROM customers c
WHERE c.id NOT IN (SELECT customer_id FROM orders);
-- If ANY row in orders has customer_id = NULL, this returns ZERO ROWS.
```

Why: `x NOT IN (a, b, NULL)` expands to `x <> a AND x <> b AND x <> NULL`. The last term is `UNKNOWN`, so the whole `AND` can never be `TRUE` — it's `UNKNOWN` or `FALSE`. Every row is filtered out. (Three-valued logic from Topic 07.)

```sql
-- SAFE: NOT EXISTS (anti-join, NULL-immune)
SELECT c.id, c.name
FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
-- NULL customer_id rows simply never match c.id = o.customer_id → ignored. Correct.

-- SAFE: LEFT JOIN / anti-join spelling
SELECT c.id, c.name
FROM customers c
LEFT JOIN orders o ON o.customer_id = c.id
WHERE o.customer_id IS NULL;
```

**Principal-level rule:** prefer `NOT EXISTS` over `NOT IN` for "absence" queries, always, unless you can guarantee the subquery column is `NOT NULL`. The three forms diverge here, and only two of them are correct.

### 6.7 Correlated vs Uncorrelated Subqueries

An **uncorrelated** subquery does not reference the outer query — it can be computed once:

```sql
-- Uncorrelated: computed once (InitPlan), cached, compared to every row
SELECT * FROM products
WHERE price > (SELECT avg(price) FROM products);
```

A **correlated** subquery references the outer row — conceptually re-evaluated per outer row:

```sql
-- Correlated: references o.customer_id from the outer query
SELECT o.id,
  (SELECT c.name FROM customers c WHERE c.id = o.customer_id) AS customer_name
FROM orders o;
-- Better as a JOIN: the planner may rewrite it, but writing the JOIN is clearer
SELECT o.id, c.name AS customer_name
FROM orders o JOIN customers c ON c.id = o.customer_id;
```

A correlated scalar subquery in `SELECT` is a common N+1-in-SQL anti-pattern (Topic 18). For a per-row lookup it usually plans as a nested-loop index probe (fine if indexed), but a JOIN is more honest and lets the planner pick Hash Join if better.

### 6.8 Pre-Aggregation: The Fan-Out Fix (Subquery/CTE Beats Bare JOIN)

The one scenario where restructuring genuinely changes correctness *and* performance, first seen in Topic 11 §6.8. When you join a "one" table to a "many" table and then aggregate, the "one" side's own aggregates get multiplied.

```sql
-- WRONG: joining orders to order_items multiplies orders by item count,
-- so SUM(o.total_amount) double-counts.
SELECT c.id, sum(o.total_amount) AS spend, count(oi.id) AS items
FROM customers c
JOIN orders o ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
GROUP BY c.id;
-- BUG: an order with 5 items adds its total_amount 5 times.

-- RIGHT: pre-aggregate each grain separately in CTEs, then join
WITH order_totals AS (
  SELECT customer_id, sum(total_amount) AS spend, count(*) AS order_count
  FROM orders GROUP BY customer_id
),
item_counts AS (
  SELECT o.customer_id, count(*) AS items
  FROM orders o JOIN order_items oi ON oi.order_id = o.id
  GROUP BY o.customer_id
)
SELECT c.id, ot.spend, ot.order_count, ic.items
FROM customers c
JOIN order_totals ot ON ot.customer_id = c.id
JOIN item_counts ic ON ic.customer_id = c.id;
```

Here CTEs are not merely a readability choice — they express the correct grain of each aggregate. This is the canonical "use a CTE/subquery, not a bare JOIN" case.

### 6.9 The Decision Framework

A concrete rubric for choosing:

| Intent | Prefer | Why |
|--------|--------|-----|
| Combine columns from related tables (1:1 or driving side) | **JOIN** | Most direct; planner has maximum freedom |
| "Does a related row exist?" (existence) | **EXISTS** | Semi-join, no fan-out, NULL-safe |
| "Does a related row NOT exist?" (absence) | **NOT EXISTS** | Anti-join, NULL-safe (never `NOT IN` on nullable) |
| Filter by a set of values | **IN** (or `= ANY`) | Semi-join; readable |
| Aggregate at a different grain than the main query | **subquery / CTE (pre-aggregate)** | Avoids fan-out double-counting |
| A multi-step pipeline read top-to-bottom | **CTE** | Readability; free since PG12 if single-ref |
| Same expensive sub-result used many times | **CTE (default materialize)** or `MATERIALIZED` | Compute once |
| Recursion / hierarchy | **WITH RECURSIVE** | Only option (Topic 29) |
| Per-row scalar lookup | **JOIN** (not correlated subquery in SELECT) | Avoids N+1-shaped plan |

**Default posture:** write it the way it reads clearest — usually a JOIN for combining and a CTE for a pipeline — then run `EXPLAIN` to confirm the plan is what you expect. Refactor to a different form *only* if `EXPLAIN` shows a materialization or fan-out you need to remove.

### 6.10 Readability vs Performance — The Real Trade-off Table

| Concern | JOIN | Subquery (FROM) | Subquery (IN/EXISTS) | CTE (inlined) | CTE (MATERIALIZED) |
|---------|------|-----------------|----------------------|---------------|--------------------|
| Reads top-to-bottom | No | No | Partially | **Yes** | **Yes** |
| Planner reorders freely | **Yes** | Yes (if flattened) | Yes (semi/anti-join) | **Yes** | No (fence) |
| Predicate push-down | **Yes** | Yes (if flattened) | Yes | **Yes** | **No** |
| Avoids fan-out naturally | No | Yes (pre-agg) | **Yes** (semi-join) | Yes (pre-agg) | Yes |
| NULL-safe absence check | via LEFT/anti | n/a | **only NOT EXISTS** | via anti | via anti |
| Guaranteed single evaluation | No | No | No | No (single-ref inlines) | **Yes** |
| Debuggable in pieces | Hard | Medium | Medium | **Easy** | **Easy** |

The through-line: since PG12, the **inlined CTE column matches the JOIN column** on every performance row while winning every readability row. That is why modern PostgreSQL style favors CTEs for pipelines — the historical penalty is gone.

---

## 7. EXPLAIN — Proving Equivalence in the Plan

The professional skill this topic teaches is **using `EXPLAIN` to prove two syntaxes are the same plan.** Let's do it.

### 7.1 Inlined CTE == Subquery == JOIN

```sql
EXPLAIN (ANALYZE, BUFFERS, COSTS)
WITH completed AS (
  SELECT customer_id FROM orders WHERE status = 'completed'
)
SELECT c.id, c.name
FROM customers c
JOIN completed cp ON cp.customer_id = c.id;
```

```
Hash Join  (cost=1234.00..8900.00 rows=24000 width=40)
           (actual time=6.2..71.4 rows=23811 loops=1)
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o  (cost=0.00..6100.00 rows=24000 width=8)
        Filter: (status = 'completed')
        Rows Removed by Filter: 176000
        (actual time=0.02..40.1 rows=24000 loops=1)
  ->  Hash  (cost=734.00..734.00 rows=40000 width=40)
        Buckets: 65536  Batches: 1  Memory Usage: 3585kB
        ->  Seq Scan on customers c  (cost=0.00..734.00 rows=40000 width=40)
              (actual time=0.01..2.9 rows=40000 loops=1)
Buffers: shared hit=4102
Planning Time: 0.4 ms
Execution Time: 72.9 ms
```

Now the subquery form:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT c.id, c.name
FROM customers c
JOIN (SELECT customer_id FROM orders WHERE status='completed') cp
  ON cp.customer_id = c.id;
```

```
Hash Join  (cost=1234.00..8900.00 rows=24000 width=40)
           (actual time=6.1..71.0 rows=23811 loops=1)
  Hash Cond: (o.customer_id = c.id)
  ->  Seq Scan on orders o  ...
  ->  Hash  ...
        ->  Seq Scan on customers c  ...
Execution Time: 72.6 ms
```

**Reading it:** the two plans are **byte-for-byte identical** — same `Hash Join`, same cost estimates, same row estimates, same node tree. There is **no `CTE Scan` node** in the CTE version. That absence is the proof of inlining: pre-PG12 you would see a `CTE Scan on completed` node and a separate materialization step. Its disappearance is how you *prove* the fence is gone.

### 7.2 The `MATERIALIZED` Fence Shows a `CTE Scan`

```sql
EXPLAIN (ANALYZE, BUFFERS)
WITH completed AS MATERIALIZED (
  SELECT customer_id FROM orders WHERE status='completed'
)
SELECT c.id, c.name FROM customers c JOIN completed cp ON cp.customer_id = c.id;
```

```
Hash Join  (cost=1740.00..9950.00 rows=24000 width=40)
           (actual time=48.5..119.3 rows=23811 loops=1)
  Hash Cond: (cp.customer_id = c.id)
  CTE completed
    ->  Seq Scan on orders o  (cost=0.00..6100.00 rows=24000 width=8)
          Filter: (status = 'completed')
          (actual time=0.02..41.0 rows=24000 loops=1)
  ->  CTE Scan on completed cp  (cost=0.00..480.00 rows=24000 width=8)
        (actual time=0.03..6.1 rows=24000 loops=1)     ← the fence node
  ->  Hash  (cost=734.00..734.00 rows=40000 width=40)
        ->  Seq Scan on customers c  ...
Buffers: shared hit=4102, temp written=94
Planning Time: 0.4 ms
Execution Time: 121.0 ms
```

**Reading it:**
- The `CTE completed` sub-node and the `CTE Scan on completed cp` node are the tell-tale signs of materialization.
- `temp written=94` — the work table spilled buffers; this is extra I/O the inlined form avoided.
- Execution time rose from ~73ms to ~121ms **for the same result** — the fence's cost, here modest, but on a filter-pushable query it can be 100×.

### 7.3 Where the Fence Is Catastrophic

```sql
-- Inlinable: predicate reaches the index
EXPLAIN
WITH o AS (SELECT * FROM orders)
SELECT * FROM o WHERE customer_id = 42;
```
```
Index Scan using orders_customer_id_idx on orders  (cost=0.43..18.7 rows=5 width=64)
  Index Cond: (customer_id = 42)
```
```sql
-- Fenced: full materialization, then filter
EXPLAIN
WITH o AS MATERIALIZED (SELECT * FROM orders)
SELECT * FROM o WHERE customer_id = 42;
```
```
CTE Scan on o  (cost=0.00..5000.00 rows=5 width=64)
  Filter: (customer_id = 42)
  CTE o
    ->  Seq Scan on orders  (cost=0.00..4000.00 rows=200000 width=64)   ← all 200k rows
```
The fenced plan reads all 200,000 rows to return 5. Same query, 40,000× more rows scanned.

### 7.4 `EXISTS` Semi-Join vs `JOIN + DISTINCT`

```sql
EXPLAIN (ANALYZE)
SELECT c.id FROM customers c
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```
```
Hash Semi Join  (cost=1234.00..7800.00 rows=32000 width=8)
                (actual time=5.9..48.2 rows=31200 loops=1)
  Hash Cond: (c.id = o.customer_id)
  ...
```
Note `Hash Semi Join`, not `Hash Join`. The semi-join stops at the first match per customer — no duplicates generated, no `Unique` node needed. The `JOIN + DISTINCT` version would show a `Hash Join` feeding a `HashAggregate` (for the DISTINCT), doing strictly more work.

### 7.5 What to Look For

| In the plan | Meaning |
|-------------|---------|
| No `CTE Scan` node | CTE was inlined (good — planner had full freedom) |
| `CTE <name>` + `CTE Scan` nodes | CTE materialized (fence: check if intended) |
| `temp written` / `temp read` | A materialization/sort spilled to disk (raise `work_mem` or remove fence) |
| `Hash Semi Join` / `Nested Loop Semi Join` | `IN`/`EXISTS` correctly became a semi-join |
| `Hash Anti Join` / `Nested Loop Anti Join` | `NOT EXISTS`/`NOT IN` became an anti-join |
| `SubPlan` re-executed with high `loops` | Correlated subquery not flattened — consider a JOIN |
| `InitPlan` | Uncorrelated subquery computed once and cached (good) |
| Identical node trees across two forms | Proof the syntaxes are equivalent |

---

## 8. Query Examples

### Example 1 — Basic: The Same Question, Three Ways

```sql
-- "List customers who have at least one completed order."
-- All three return the same rows on PG12+ with (near-)identical plans.

-- (a) JOIN + DISTINCT
SELECT DISTINCT c.id, c.name
FROM customers c
INNER JOIN orders o ON o.customer_id = c.id
WHERE o.status = 'completed';

-- (b) IN subquery (semi-join)
SELECT c.id, c.name
FROM customers c
WHERE c.id IN (SELECT customer_id FROM orders WHERE status = 'completed');

-- (c) CTE (inlined on PG12+)
WITH completed AS (
  SELECT DISTINCT customer_id FROM orders WHERE status = 'completed'
)
SELECT c.id, c.name
FROM customers c
INNER JOIN completed cp ON cp.customer_id = c.id;
```

### Example 2 — Intermediate: Pre-Aggregation to Avoid Fan-Out

```sql
-- Per customer: number of completed orders, total spend, and number of
-- distinct products purchased. Bare multi-join would double-count spend.
WITH order_agg AS (                       -- grain: one row per customer
  SELECT customer_id,
         count(*)          AS order_count,
         sum(total_amount) AS total_spend
  FROM orders
  WHERE status = 'completed'
  GROUP BY customer_id
),
product_agg AS (                          -- grain: one row per customer
  SELECT o.customer_id,
         count(DISTINCT oi.product_id) AS distinct_products
  FROM orders o
  JOIN order_items oi ON oi.order_id = o.id
  WHERE o.status = 'completed'
  GROUP BY o.customer_id
)
SELECT c.id, c.name,
       oa.order_count,
       oa.total_spend,
       pa.distinct_products
FROM customers c
JOIN order_agg   oa ON oa.customer_id = c.id
JOIN product_agg pa ON pa.customer_id = c.id
ORDER BY oa.total_spend DESC
LIMIT 100;
```

### Example 3 — Production Grade: Decision Framework Applied

```sql
-- Monthly cohort revenue report with churn flagging.
-- Tables:
--   users(id, signup_date, country)                        ~2M rows
--   orders(id, user_id, total_amount, status, created_at)  ~50M rows
--   payments(id, order_id, amount, captured_at, status)    ~55M rows
-- Indexes:
--   orders(user_id, created_at), orders(status),
--   payments(order_id), users(signup_date)
-- Expectation: 3 pre-aggregating CTEs (each inlined or cheaply grouped),
--   one anti-join for churn. Target < 800ms on the above.

WITH last_90d_revenue AS (               -- grain: user; pre-agg avoids fan-out
  SELECT o.user_id,
         sum(p.amount) AS revenue_90d,
         count(DISTINCT o.id) AS orders_90d
  FROM orders o
  JOIN payments p ON p.order_id = o.id AND p.status = 'captured'
  WHERE o.created_at >= now() - INTERVAL '90 days'
  GROUP BY o.user_id
),
lifetime AS (                            -- grain: user
  SELECT user_id, sum(total_amount) AS lifetime_value
  FROM orders WHERE status = 'completed'
  GROUP BY user_id
)
SELECT
  date_trunc('month', u.signup_date)      AS cohort_month,
  u.country,
  count(*)                                AS cohort_size,
  count(r.user_id)                        AS active_last_90d,
  sum(COALESCE(r.revenue_90d, 0))         AS revenue_90d,
  sum(l.lifetime_value)                   AS cohort_ltv,
  -- churned = no order in 90d (anti-join via LEFT JOIN + IS NULL)
  count(*) FILTER (WHERE r.user_id IS NULL) AS churned
FROM users u
LEFT JOIN last_90d_revenue r ON r.user_id = u.id   -- keep churned users
LEFT JOIN lifetime         l ON l.user_id = u.id
GROUP BY 1, 2
ORDER BY cohort_month DESC, cohort_ltv DESC;
```

```
-- EXPLAIN (ANALYZE) shape (abridged):
GroupAggregate  (cost=...)  (actual time=... rows=1440 loops=1)
  ->  Sort  (Sort Key: (date_trunc('month', u.signup_date)), u.country)
        ->  Hash Left Join  (Hash Cond: (u.id = r.user_id))
              ->  Hash Left Join  (Hash Cond: (u.id = l.user_id))
                    ->  Seq Scan on users u
                    ->  Hash  -> Subquery Scan (lifetime GroupAggregate)
              ->  Hash  -> Subquery Scan (last_90d_revenue HashAggregate)
                        ->  Hash Join (orders o x payments p)   ← pre-agg here
Execution Time: 690 ms
```
Note: **no `CTE Scan` nodes** — both CTEs inlined into subquery scans that group at user grain, so the fan-out from the `payments` join is contained inside each CTE and never multiplies the outer counts.

---

## 9. Wrong → Right Patterns

### Wrong 1: `NOT IN` with a nullable column (silent empty result)

```sql
-- WRONG: returns ZERO rows if any orders.customer_id is NULL
SELECT c.id, c.name
FROM customers c
WHERE c.id NOT IN (SELECT customer_id FROM orders);
-- Wrong result: empty set. No error. Three-valued logic kills every row.

-- RIGHT: NOT EXISTS is NULL-immune
SELECT c.id, c.name
FROM customers c
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.customer_id = c.id);
```
Why at execution level: `NOT IN` expands to a chain of `<>` ANDed together; a NULL member makes the chain `UNKNOWN`, which is never `TRUE`, so the anti-filter admits nothing. `NOT EXISTS` becomes a `Hash Anti Join` where non-matching (including NULL-key) rows simply don't join.

### Wrong 2: `MATERIALIZED` blocking predicate push-down

```sql
-- WRONG: fence forces full scan+materialize before the filter
WITH recent AS MATERIALIZED (SELECT * FROM orders)
SELECT * FROM recent WHERE customer_id = 42 AND created_at > '2025-01-01';
-- Wrong at execution level: 50M rows materialized to work_mem/disk,
-- THEN filtered to a handful. Index on (customer_id, created_at) unused.

-- RIGHT: let it inline so predicates reach the index
WITH recent AS (SELECT * FROM orders)   -- single reference → inlined
SELECT * FROM recent WHERE customer_id = 42 AND created_at > '2025-01-01';
-- Plan: Index Scan using orders_customer_id_created_at_idx. Sub-millisecond.
```

### Wrong 3: Fan-out double-counting with a bare multi-JOIN

```sql
-- WRONG: joining orders to order_items multiplies total_amount
SELECT c.id, sum(o.total_amount) AS spend
FROM customers c
JOIN orders o       ON o.customer_id = c.id
JOIN order_items oi ON oi.order_id = o.id
GROUP BY c.id;
-- Wrong result: an order with 5 items adds total_amount 5 times → 5× inflation.

-- RIGHT: pre-aggregate orders at its own grain in a CTE, then join
WITH per_customer AS (
  SELECT customer_id, sum(total_amount) AS spend
  FROM orders GROUP BY customer_id
)
SELECT c.id, pc.spend
FROM customers c JOIN per_customer pc ON pc.customer_id = c.id;
```

### Wrong 4: Correlated scalar subquery in SELECT (N+1-shaped plan)

```sql
-- WRONG shape: a scalar subquery per output row
SELECT o.id, o.total_amount,
  (SELECT c.name  FROM customers c WHERE c.id = o.customer_id) AS name,
  (SELECT c.email FROM customers c WHERE c.id = o.customer_id) AS email
FROM orders o
WHERE o.created_at > '2025-01-01';
-- Two SubPlans, each re-probed per row. Doubles the lookups; awkward plan.

-- RIGHT: a single JOIN fetches both columns in one pass
SELECT o.id, o.total_amount, c.name, c.email
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.created_at > '2025-01-01';
```

### Wrong 5: Assuming the CTE runs first (stale mental model)

```sql
-- WRONG assumption: "the CTE builds the whole recent set first, so it's slow"
WITH recent AS (SELECT * FROM orders WHERE created_at > now() - INTERVAL '1 day')
SELECT * FROM recent WHERE customer_id = 42;
-- Developer "fixes" it by inlining manually into a subquery for "performance."
-- Reality on PG12+: identical plan. The refactor gains nothing, loses clarity.

-- RIGHT: trust inlining; verify with EXPLAIN that there's no CTE Scan node.
EXPLAIN WITH recent AS (SELECT * FROM orders WHERE created_at > now() - INTERVAL '1 day')
        SELECT * FROM recent WHERE customer_id = 42;
-- If you see "Index Scan ... Index Cond: (customer_id = 42)" and no CTE Scan,
-- the CTE was inlined and the predicate pushed down. Keep the readable CTE.
```

---

## 10. Performance Profile

### 10.1 The Core Numbers

| Scenario | Inlined CTE / Subquery / JOIN | MATERIALIZED CTE |
|----------|------------------------------|------------------|
| Filter pushable to an index, 50M-row base | Index Scan, <1ms | Full materialize (50M rows), seconds |
| CTE referenced once, no outer filter | Same as subquery | Slightly slower (work table build) |
| CTE referenced 5×, cheap to recompute | 5× recompute (may be fine) | 1× compute, 5× cheap scans (often wins) |
| CTE referenced 5×, expensive to recompute | 5× expensive (bad) | 1× compute (much better) |

### 10.2 Scaling at 1M / 10M / 100M Rows

- **1M rows:** the choice rarely matters; even a materialized CTE fits in `work_mem`. Differences are sub-10ms. Optimize for readability.
- **10M rows:** predicate push-down starts to dominate. An accidental `MATERIALIZED` on a filterable CTE turns a 2ms index scan into a multi-hundred-ms full scan. Fan-out double-counting produces visibly wrong aggregates. Structure matters.
- **100M rows:** the fence is catastrophic — materializing 100M rows spills gigabytes to disk. Semi-joins vs `JOIN + DISTINCT` differ meaningfully because `DISTINCT` must hash/sort the full multiplied set. Pre-aggregation in CTEs is often the only way to keep intermediate results bounded.

### 10.3 `work_mem` Interactions

A materialized CTE, a `DISTINCT`, a hash for `IN`/`EXISTS`, and a Hash Join all draw from `work_mem` (per node, per connection). Symptoms of exhaustion in `EXPLAIN (ANALYZE, BUFFERS)`:

- `temp written` / `temp read` on a `CTE Scan`, `HashAggregate`, or `Sort` → spilled to disk.
- `Batches > 1` on a `Hash` node → hash spilled (Topic 11).

Fixes: remove an unnecessary `MATERIALIZED`; pre-aggregate to shrink the intermediate; raise `work_mem` for the session (`SET work_mem = '256MB'`) for a heavy analytical query.

### 10.4 Optimization Techniques Specific to This Topic

1. **Prove equivalence before refactoring.** Run `EXPLAIN` on both forms; if the node trees match, choose readability.
2. **Strip stray `MATERIALIZED`.** Grep your codebase for `AS MATERIALIZED` and confirm each one is intentional (referenced many times, or genuinely needs single evaluation).
3. **Pre-aggregate at the correct grain** in CTEs to bound intermediate row counts and prevent fan-out.
4. **Prefer `EXISTS`/`NOT EXISTS`** over `IN`/`NOT IN` for existence — semi/anti-joins avoid duplicate generation and the NULL trap.
5. **Turn correlated `SELECT` subqueries into JOINs** when they fetch multiple columns or run per-row.
6. **Use `NOT MATERIALIZED`** deliberately when a multi-referenced CTE would benefit from per-reference predicate push-down.
7. **Watch for `LIMIT` inside subqueries/CTEs** — it blocks flattening and push-down by design; that's usually intentional but be aware of the cost.

---

## 11. Node.js Integration

### 11.1 Same Query, Choosing the Form at the Query Layer

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Existence check — EXISTS semi-join, NULL-safe, no fan-out
async function customersWithCompletedOrders() {
  const { rows } = await pool.query(
    `SELECT c.id, c.name
       FROM customers c
      WHERE EXISTS (
        SELECT 1 FROM orders o
         WHERE o.customer_id = c.id AND o.status = 'completed'
      )
      ORDER BY c.name`
  );
  return rows;
}
```

### 11.2 Pre-Aggregating CTE Pipeline with Params

```javascript
// The readable CTE form; PG12+ inlines the single-reference CTEs.
async function customerSpendReport(sinceDays = 90, minSpend = 0) {
  const { rows } = await pool.query(
    `WITH order_agg AS (
       SELECT customer_id,
              count(*)          AS order_count,
              sum(total_amount) AS total_spend
         FROM orders
        WHERE status = 'completed'
          AND created_at >= now() - ($1 || ' days')::INTERVAL
        GROUP BY customer_id
     )
     SELECT c.id, c.name, oa.order_count, oa.total_spend
       FROM customers c
       JOIN order_agg oa ON oa.customer_id = c.id
      WHERE oa.total_spend >= $2
      ORDER BY oa.total_spend DESC`,
    [sinceDays, minSpend]
  );
  return rows;
}
```

### 11.3 The NULL-Safe Absence Query (never build `NOT IN` from JS arrays)

```javascript
// Find customers with no orders. Use NOT EXISTS, not NOT IN.
async function customersWithNoOrders() {
  const { rows } = await pool.query(
    `SELECT c.id, c.name
       FROM customers c
      WHERE NOT EXISTS (
        SELECT 1 FROM orders o WHERE o.customer_id = c.id
      )`
  );
  return rows;
}

// If you must filter by a JS array, use = ANY($1) — NOT "IN (...)" string-built.
async function customersInList(ids /* number[] */) {
  const { rows } = await pool.query(
    `SELECT id, name FROM customers WHERE id = ANY($1::int[])`,
    [ids]              // parameterized array — no SQL injection, no NULL trap
  );
  return rows;
}
```

### 11.4 Proving Equivalence from Node (a diagnostic helper)

```javascript
// Dump the plan for two forms and compare — useful in a perf test/CI check.
async function explain(sql, params = []) {
  const { rows } = await pool.query(`EXPLAIN (FORMAT JSON) ${sql}`, params);
  return rows[0]['QUERY PLAN'][0].Plan['Node Type']; // top node type
}
// e.g. assert both spellings yield the same top node ("Hash Join").
```

A note on ORMs: none of the mainstream Node ORMs let you control CTE inlining or `MATERIALIZED` — that lives in raw SQL. ORMs *do* generate subqueries and joins for you (relation loading, `where exists`-style filters), but the moment you need pre-aggregation at a specific grain, a NULL-safe anti-join, or an explicit fence, drop to the raw escape hatch.

---

## 12. ORM Comparison

### Prisma

**Can Prisma express these?** Partially. Prisma models relation existence and combination well, but has no concept of CTEs, `MATERIALIZED`, or manual pre-aggregation.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// "Customers with at least one completed order" — Prisma compiles `some`
// into an EXISTS semi-join. NULL-safe and fan-out-free. Good.
const customers = await prisma.customer.findMany({
  where: { orders: { some: { status: 'completed' } } },
  select: { id: true, name: true },
});

// "Customers with NO orders" — `none` compiles to NOT EXISTS (NULL-safe). Good.
const inactive = await prisma.customer.findMany({
  where: { orders: { none: {} } },
});

// Pre-aggregation at a custom grain (sum spend, distinct products per customer):
// NOT expressible in the query API. Use raw SQL.
const report = await prisma.$queryRaw`
  WITH order_agg AS (
    SELECT customer_id, sum(total_amount) AS spend, count(*) AS n
    FROM orders WHERE status = 'completed' GROUP BY customer_id
  )
  SELECT c.id, c.name, oa.spend, oa.n
  FROM customers c JOIN order_agg oa ON oa.customer_id = c.id
  ORDER BY oa.spend DESC`;
```

**Where it breaks:** no CTEs, no `MATERIALIZED`/`NOT MATERIALIZED`, no `COUNT(DISTINCT)` at arbitrary grain, no window functions in the query API. **Verdict:** `some`/`none` correctly map to NULL-safe semi/anti-joins — use them. For pipelines and pre-aggregation, `$queryRaw`.

### Drizzle ORM

**Can Drizzle express these?** Yes — Drizzle has first-class `with()` for CTEs, `exists()`/`notExists()`, and subqueries as values.

```typescript
import { db } from './db';
import { customers, orders } from './schema';
import { eq, sql, exists, and } from 'drizzle-orm';

// EXISTS semi-join
const rows = await db.select({ id: customers.id, name: customers.name })
  .from(customers)
  .where(exists(
    db.select({ x: sql`1` }).from(orders)
      .where(and(eq(orders.customerId, customers.id),
                 eq(orders.status, 'completed')))
  ));

// CTE via .with()
const orderAgg = db.$with('order_agg').as(
  db.select({
      customerId: orders.customerId,
      spend: sql<number>`sum(${orders.totalAmount})`.as('spend'),
    })
    .from(orders).where(eq(orders.status, 'completed'))
    .groupBy(orders.customerId)
);
const report = await db.with(orderAgg)
  .select({ id: customers.id, name: customers.name,
            spend: orderAgg.spend })
  .from(customers)
  .innerJoin(orderAgg, eq(orderAgg.customerId, customers.id));
```

**Where it breaks:** Drizzle's `.with()` emits a plain `WITH` (inlined on PG12+); it does not expose `MATERIALIZED`/`NOT MATERIALIZED` keywords — use `sql` raw if you must force a fence. **Verdict:** the best typed fit for this topic — real CTEs, real `exists`.

### Sequelize

**Can Sequelize express these?** Partially. It has `where: { [Op.or]: ... }`, literal subqueries, and `include` (joins), but no native CTE builder.

```javascript
const { Op, literal } = require('sequelize');

// EXISTS via a literal subquery (NULL-safe because it's EXISTS, not IN)
const customers = await Customer.findAll({
  where: literal(`EXISTS (SELECT 1 FROM orders o
                          WHERE o.customer_id = "Customer"."id"
                            AND o.status = 'completed')`),
});

// Anything CTE-shaped or pre-aggregated → raw query
const report = await sequelize.query(
  `WITH order_agg AS (
     SELECT customer_id, sum(total_amount) AS spend
     FROM orders WHERE status = 'completed' GROUP BY customer_id
   )
   SELECT c.id, c.name, oa.spend
   FROM customers c JOIN order_agg oa ON oa.customer_id = c.id`,
  { type: QueryTypes.SELECT }
);
```

**Where it breaks:** `literal()` subqueries are unparameterized strings — injection risk if you interpolate. No CTE/fence control. **Verdict:** fine for `include`-based joins; use `sequelize.query()` for CTEs and pre-aggregation.

### TypeORM

**Can TypeORM express these?** Yes for joins and subqueries; CTEs via `addCommonTableExpression`.

```typescript
// EXISTS-style filter with a QueryBuilder subquery
const qb = dataSource.getRepository(Customer).createQueryBuilder('c')
  .where(q => 'EXISTS ' + q.subQuery()
    .select('1').from('orders', 'o')
    .where('o.customer_id = c.id AND o.status = :s')
    .getQuery())
  .setParameter('s', 'completed');
const customers = await qb.getMany();

// CTE
const report = await dataSource.createQueryBuilder()
  .addCommonTableExpression(
    `SELECT customer_id, sum(total_amount) AS spend
       FROM orders WHERE status='completed' GROUP BY customer_id`,
    'order_agg')
  .select('c.id', 'id').addSelect('oa.spend', 'spend')
  .from('customers', 'c')
  .innerJoin('order_agg', 'oa', 'oa.customer_id = c.id')
  .getRawMany();
```

**Where it breaks:** `addCommonTableExpression` takes raw SQL strings (limited type safety); no `MATERIALIZED` control. **Verdict:** capable, verbose; raw string CTE bodies undercut type safety.

### Knex.js

**Can Knex express these?** Yes — `.with()` for CTEs, `.whereExists()`/`.whereNotExists()`, `.whereIn()` with subqueries.

```javascript
// NOT EXISTS anti-join (NULL-safe)
const noOrders = await knex('customers as c')
  .whereNotExists(qb => qb.select('*').from('orders as o')
                          .whereRaw('o.customer_id = c.id'))
  .select('c.id', 'c.name');

// CTE + pre-aggregation
const report = await knex
  .with('order_agg', qb =>
    qb.from('orders').where('status', 'completed')
      .groupBy('customer_id')
      .select('customer_id', knex.raw('sum(total_amount) as spend')))
  .from('customers as c')
  .join('order_agg as oa', 'oa.customer_id', 'c.id')
  .select('c.id', 'c.name', 'oa.spend')
  .orderBy('oa.spend', 'desc');
```

**Where it breaks:** `.with()` emits plain `WITH` (no `MATERIALIZED` keyword — use `knex.raw` to force it). `.whereNotExists` correctly avoids the `NOT IN` trap. **Verdict:** the most SQL-transparent; `whereExists`/`whereNotExists`/`with` map directly to the right constructs.

### ORM Summary Table

| ORM | EXISTS / NOT EXISTS | CTE | MATERIALIZED control | Pre-aggregation | Verdict |
|-----|---------------------|-----|----------------------|-----------------|---------|
| Prisma | `some` / `none` (semi/anti-join) | No | No | Raw only | Great for existence; raw for pipelines |
| Drizzle | `exists()` / `notExists()` | `.with()` | No (raw) | Yes | Best typed fit |
| Sequelize | `literal('EXISTS ...')` | Raw only | No | Raw only | Joins yes; raw for the rest |
| TypeORM | subquery in `where` | `addCommonTableExpression` | No | Verbose | Capable, low type safety on CTE body |
| Knex | `.whereExists`/`.whereNotExists` | `.with()` | No (raw) | Yes | Most SQL-transparent |

The universal pattern: every ORM maps `EXISTS`/`NOT EXISTS` to the correct NULL-safe semi/anti-join, but **none expose the `MATERIALIZED` fence** — that is a raw-SQL-only lever, and it's the one you most need to control on large tables.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given:
- `customers(id, name, country)`
- `orders(id, customer_id, total_amount, status, created_at)`

Write the query "list all customers who have placed at least one order" three ways: (a) as `JOIN + DISTINCT`, (b) as an `IN` subquery, (c) as an `EXISTS` subquery. Then run `EXPLAIN` on each (conceptually) and state which node type each produces. Which two are guaranteed to be identical plans?

```sql
-- (a) Write your JOIN + DISTINCT query here

-- (b) Write your IN subquery here

-- (c) Write your EXISTS subquery here
```

### Exercise 2 — Medium (combines Topic 11 fan-out + this topic)

Given `customers`, `orders`, and `order_items(id, order_id, product_id, quantity, unit_price)`, write a single statement that returns, per customer: `total_spend` (sum of `orders.total_amount` for completed orders) and `total_units` (sum of `order_items.quantity` for those orders). Do it **without** double-counting `total_spend`. Use CTEs to keep each aggregate at its correct grain, then join.

```sql
-- Write your query here (hint: two CTEs at customer grain, then join)
```

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

You must find "customers who have **never** had a *failed* payment." A colleague writes:

```sql
SELECT c.id, c.name
FROM customers c
WHERE c.id NOT IN (
  SELECT o.customer_id
  FROM orders o JOIN payments p ON p.order_id = o.id
  WHERE p.status = 'failed'
);
```

`payments.status` is `NOT NULL`, but `orders.customer_id` **is nullable** (guest checkouts). 

1. Explain precisely why this query can return zero rows.
2. Rewrite it correctly using `NOT EXISTS`.
3. Add a second condition: among those customers, keep only ones whose lifetime completed-order spend exceeds 1000. Use a CTE for the spend.

```sql
-- Write your corrected, extended query here
```

### Exercise 4 — Interview Simulation

A CTE-based report runs in 40ms in staging but 6 seconds in production on the same query. The only difference a teammate can find is that a previous engineer added `MATERIALIZED` to one CTE "to make it deterministic." The CTE is referenced once and the outer query filters it by an indexed column.

1. Explain the mechanism causing the 150× slowdown.
2. What single change fixes it, and why is the result still correct?
3. What would you look for in `EXPLAIN (ANALYZE, BUFFERS)` to confirm your diagnosis before and after?

```sql
-- Write the before/after queries and the EXPLAIN signals you'd check
```

---

## 14. Interview Questions

### Q1 — "Are CTEs slower than subqueries in PostgreSQL?"

**Junior answer:** "Yes, CTEs are always materialized, so avoid them in performance-critical code."

**Principal answer:** "That was true before PostgreSQL 12, when a `WITH` clause was an unconditional optimization fence — always materialized into a work table, no predicate push-down. Since PostgreSQL 12, a non-recursive, side-effect-free CTE referenced exactly once is **inlined** by default and optimized identically to a `FROM`-clause subquery — same plan, same cost. You only get materialization when the CTE is referenced multiple times, is recursive, or you explicitly write `MATERIALIZED`. So the blanket 'CTEs are slow' rule is obsolete. I verify by running `EXPLAIN` and checking whether a `CTE Scan` node appears — its absence proves inlining."

**Interviewer follow-up:** "How would you force the old materialized behavior, and when would you actually want it?" → `AS MATERIALIZED`; want it when the CTE is expensive and referenced many times, or when you need a stable single evaluation of a volatile expression.

### Q2 — "What's the difference between `NOT IN` and `NOT EXISTS`?"

**Junior answer:** "They're the same — both find rows that aren't in the other set."

**Principal answer:** "Semantically they diverge on NULLs, and it's a production hazard. `x NOT IN (SELECT col ...)` expands to `x <> v1 AND x <> v2 AND ...`; if the subquery yields even one NULL, the corresponding `x <> NULL` is `UNKNOWN`, the whole `AND` can never be `TRUE`, and the query returns **zero rows** — silently, no error. `NOT EXISTS` compiles to an anti-join where non-matching rows, including NULL-keyed ones, simply don't join, giving the correct result. So I default to `NOT EXISTS` for absence queries unless the subquery column is provably `NOT NULL`. Performance-wise they're usually similar (both anti-joins) once the NULL issue is off the table, but `NOT IN` can also disable some anti-join optimizations."

**Interviewer follow-up:** "If the column is `NOT NULL`, are they identical?" → Semantically yes; the planner can turn both into a `Hash Anti Join`, though `NOT EXISTS` is more reliably optimized.

### Q3 — "You have a `WITH` query that filters by an indexed column but it's doing a full sequential scan. Why?"

**Junior answer:** "Maybe the index is missing or the stats are stale — run `ANALYZE`."

**Principal answer:** "Those are worth checking, but the classic cause is an **optimization fence**: the CTE is declared `AS MATERIALIZED`, or it's referenced multiple times so PostgreSQL materializes it by default. A fence blocks predicate push-down, so your outer `WHERE indexed_col = X` can't reach the index inside the CTE — PostgreSQL materializes the entire relation first, then filters. `EXPLAIN` shows a `CTE Scan` with a `Filter` instead of an `Index Scan` with an `Index Cond`. The fix is to remove `MATERIALIZED` (or add `NOT MATERIALIZED` if multi-referenced) so the CTE inlines and the predicate pushes down to the index. I'd confirm with `EXPLAIN (ANALYZE, BUFFERS)` — the fenced version shows huge `Rows Removed by Filter` and possibly `temp written`; the inlined version shows a tight `Index Cond`."

**Interviewer follow-up:** "It's referenced three times, though — I can't just inline." → Use `NOT MATERIALIZED` if each reference has a selective predicate that benefits from the index; or restructure so the shared, filtered subset is computed once and each branch reads from it. Measure both.

### Q4 — "When would you deliberately restructure a JOIN into a CTE or subquery?"

**Junior answer:** "When the query gets too long to read."

**Principal answer:** "Two concrete reasons beyond readability. First, **fan-out correctness**: when I aggregate across a one-to-many join, joining first and summing later double-counts the 'one' side. Pre-aggregating each grain in its own CTE/subquery, then joining the results, produces correct numbers and bounds the intermediate row count. Second, **existence semantics**: replacing a `JOIN + DISTINCT` with `EXISTS` yields a semi-join that stops at the first match and never generates duplicates — cheaper on wide fan-outs. Readability is real too — a single-reference CTE inlines for free on PG12+, so I lose nothing by writing a clear top-to-bottom pipeline. The discipline is: pick the clearest form, then `EXPLAIN` to confirm the plan, and only trade clarity for a different structure when the plan shows a materialization or fan-out I need to remove."

**Interviewer follow-up:** "Show me the fan-out bug." → `SUM(o.total_amount)` over `orders JOIN order_items` multiplies each order's amount by its item count; fix with pre-aggregation or `SUM(...)` over a de-duplicated grain.

---

## 15. Mental Model Checkpoint

1. On PostgreSQL 12+, you write a non-recursive CTE referenced exactly once and filter it in the outer query by an indexed column. Does the index get used? What single keyword would break that, and why?

2. `WHERE id NOT IN (SELECT customer_id FROM orders)` returns zero rows in production but many rows in your local test data. What is the difference between the two datasets, and which form should you have used?

3. You see `CTE Scan on foo` in an `EXPLAIN` output. What does its presence tell you about how the planner treated the CTE? What would its *absence* have told you?

4. A `FROM`-clause subquery contains `ORDER BY created_at DESC LIMIT 100`. Can the planner flatten it into the parent join and push outer predicates into it? Why or why not?

5. You aggregate `SUM(orders.total_amount)` across a join to `order_items`. Without running it, predict whether the sum is too high, too low, or correct — and explain the mechanism.

6. A CTE is referenced five times and is expensive to compute. PostgreSQL's default behavior helps you here — what is it, and when would you override it with `NOT MATERIALIZED`?

7. You must prove to a skeptical teammate that a CTE version and a JOIN version of a query are the *same* plan. What exact command do you run, and what specific feature of the two outputs constitutes the proof?

---

## 16. Quick Reference Card

```sql
-- ── INLINING (PG12+) ─────────────────────────────────────────────
-- A CTE is INLINED (== subquery == JOIN, same plan) when it is:
--   • non-recursive, • referenced exactly once,
--   • side-effect-free (plain SELECT, no volatile funcs),
--   • not marked MATERIALIZED.
WITH x AS (SELECT ...) SELECT ... FROM x WHERE indexed_col = $1;  -- push-down OK

-- ── FORCING / REMOVING THE FENCE ─────────────────────────────────
WITH x AS MATERIALIZED (...)      -- fence: compute once, NO push-down
WITH x AS NOT MATERIALIZED (...)  -- inline even if referenced many times

-- ── EQUIVALENT EXISTENCE FORMS (usually same semi-join plan) ─────
WHERE EXISTS (SELECT 1 FROM t WHERE t.k = outer.k)   -- preferred
WHERE outer.k IN (SELECT k FROM t)                    -- semi-join
JOIN (SELECT DISTINCT k FROM t) d ON d.k = outer.k    -- + Unique node

-- ── ABSENCE: NULL-SAFE vs TRAP ──────────────────────────────────
WHERE NOT EXISTS (SELECT 1 FROM t WHERE t.k = outer.k)  -- SAFE (anti-join)
WHERE outer.k NOT IN (SELECT k FROM t)                  -- TRAP if k nullable!
-- NOT IN + any NULL in the subquery  ⇒  returns ZERO rows, silently.

-- ── PRE-AGGREGATE TO AVOID FAN-OUT ──────────────────────────────
WITH a AS (SELECT fk, sum(x) s FROM one GROUP BY fk),
     b AS (SELECT fk, count(*) c FROM many GROUP BY fk)
SELECT ... FROM base JOIN a USING(fk) JOIN b USING(fk);  -- correct grains

-- ── PROVE EQUIVALENCE ───────────────────────────────────────────
EXPLAIN (ANALYZE, BUFFERS) <query>;
--  No "CTE Scan" node          → CTE was inlined (planner had full freedom)
--  "CTE <name>" + "CTE Scan"   → materialized (fence — intended?)
--  "temp written/read"         → spilled to disk (raise work_mem / drop fence)
--  "Hash Semi Join"            → IN/EXISTS became a semi-join
--  "Hash Anti Join"            → NOT EXISTS/NOT IN became an anti-join
--  "SubPlan" high loops        → correlated subquery not flattened → try JOIN
--  Identical node trees        → the two syntaxes ARE the same plan
```

**Perf rules of thumb**
- Since PG12, single-reference CTE == subquery == JOIN. Write for readability, verify with `EXPLAIN`.
- Never `MATERIALIZED` a CTE the outer query filters — it blocks index push-down.
- `NOT EXISTS`, not `NOT IN`, for absence — unless the column is provably `NOT NULL`.
- Pre-aggregate at the correct grain to avoid fan-out double-counting.
- `EXISTS` beats `JOIN + DISTINCT` on wide fan-outs (semi-join stops at first match).

**Interview one-liners**
- "CTEs stopped being an optimization fence in PostgreSQL 12 — they inline unless materialized, recursive, or multiply referenced."
- "`NOT IN` with a nullable subquery column returns zero rows; use `NOT EXISTS`."
- "The absence of a `CTE Scan` node in `EXPLAIN` is the proof that the CTE was inlined."
- "Same plan, different words — pick the clearest, then read the plan."

---

## Connected Topics

- **Topic 07 — NULL in Depth**: The three-valued logic behind the `NOT IN` trap — why one NULL in the subquery empties the result.
- **Topic 11 — INNER JOIN in Depth**: Fan-out and row multiplication; the pre-aggregation fix that motivates choosing a CTE/subquery over a bare JOIN.
- **Topic 12 — LEFT JOIN / Anti-Join**: The `LEFT JOIN ... IS NULL` spelling of an anti-join, equivalent to `NOT EXISTS`.
- **Topic 18 — The N+1 Query Problem**: Correlated scalar subqueries in `SELECT` as an in-SQL N+1 shape, and why a JOIN is better.
- **Topic 19 / 27 — EXISTS and Semi-Joins**: Full treatment of the semi-join and anti-join transformations this topic relies on.
- **Topic 28 — CTE Fundamentals**: The `WITH` syntax, scoping, and multi-CTE chaining.
- **Topic 29 — Recursive CTEs**: `WITH RECURSIVE` — the one CTE form that is *never* inlined and always materializes by nature.
- **Topic 31 — Lateral Joins**: When a subquery must reference the outer row per iteration — the case a plain flattened subquery cannot express.
- **Internals — The Planner/Optimizer**: Subquery pull-up, semi/anti-join transformation, CTE inlining, and predicate push-down are all planner behaviors; `EXPLAIN` is the window into them.
```
