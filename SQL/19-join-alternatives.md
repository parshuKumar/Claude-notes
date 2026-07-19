# Topic 19 — JOIN Alternatives
### SQL Mastery Curriculum — Phase 3: JOINs — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you're a bouncer at a club with a guest list problem to solve, and there are three subtly different jobs you might be asked to do.

**Job 1 — "Let in every guest who has at least one friend already inside."**
You don't care *how many* friends are inside. You don't care *who* they are. The moment you confirm one friend is in there, you wave the guest through and move to the next person in line. You stop checking the instant you find the first match. This is **EXISTS** — a semi-join. It answers a yes/no question and stops early.

**Job 2 — "Let in every guest, and for each friend they have inside, stamp their hand once."**
Now you care about every match. A guest with five friends inside gets five stamps — they effectively pass through the door five times. This is a **JOIN** — it produces one output row per matching pair, and it multiplies.

**Job 3 — "Let in every guest who has NO friends inside."**
You have to check the *entire* inside before you can be sure nobody matches. And here's the cruel twist: if someone hands you a guest list where one entry is smudged and unreadable — you genuinely cannot say "this person is definitely not their friend," because you can't read the name. That single unreadable entry poisons your ability to say "no match exists for anyone." This is **NOT IN** with a NULL in the list — the infamous trap where one NULL makes the whole query return *nothing*.

The entire topic is this: three jobs that look almost identical in English — "has a match," "has this many matches," "has no match" — map to three different SQL constructs with three different performance profiles and one catastrophic NULL landmine. Getting them confused is one of the most common sources of both wrong results and slow queries in production.

---

## 2. Connection to SQL Internals

The deep insight of this topic is that **`EXISTS`, `IN`, correlated subqueries, and `INNER JOIN` are frequently the same operation to the planner** — and sometimes they are decisively not. Understanding *which* case you're in requires naming what the engine does underneath.

PostgreSQL's planner has two internal join "shapes" that don't have dedicated SQL keywords but drive everything in this topic:

1. **Semi-join** (`JOIN_SEMI` internally): "return each outer row that has *at least one* match, without duplicating it and without stopping to enumerate all matches." This is what `EXISTS` and `IN` both normally get rewritten into. A semi-join short-circuits — it stops probing the inner side the moment the first match is found for a given outer row.

2. **Anti-join** (`JOIN_ANTI` internally): "return each outer row that has *zero* matches." This is what `NOT EXISTS` gets rewritten into. It also short-circuits — it stops the moment it finds *any* match (because that's enough to *exclude* the row).

At the physical layer, both semi-joins and anti-joins are executed by the same three algorithms you already know from Topic 11 and Topic 17 — **Nested Loop**, **Hash Join** (as a `Hash Semi Join` / `Hash Anti Join` node), and **Merge Join** — but with the short-circuit modification baked in. When you see `Hash Semi Join` or `Nested Loop Anti Join` in an `EXPLAIN` plan, you are looking directly at this internal machinery.

The reason `NOT IN` is dangerous and `NOT EXISTS` is safe comes down to **three-valued logic** (Topic 07) at the planner level. `NOT IN (subquery)` cannot be safely rewritten into an anti-join when the subquery column is nullable, because SQL's `NOT IN` semantics require that a NULL in the list produce `UNKNOWN` for *every* comparison — which the anti-join short-circuit would violate. So the planner is *forced* to keep `NOT IN` as a slow, NULL-aware `Filter` unless it can prove the column is `NOT NULL`. `NOT EXISTS` carries no such baggage and rewrites cleanly to an anti-join.

Everything in this document is downstream of these two facts: (1) semi/anti-join is the internal currency, and (2) `NOT IN` refuses to become an anti-join when NULLs are possible.

---

## 3. Logical Execution Order Context

```
FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id)  ← subquery in WHERE phase
GROUP BY ...
HAVING ...
SELECT ...
ORDER BY ...
LIMIT ...
```

Where these constructs sit in the logical pipeline matters:

- **`EXISTS` / `IN` / `NOT IN` / `NOT EXISTS` live in the `WHERE` clause** (or `HAVING`). They are *filters* — they decide whether an outer row survives. They do **not** add columns to the output. You cannot `SELECT` a column from an `EXISTS` subquery; the subquery is a boolean predicate, not a data source.

- **`JOIN` lives in the `FROM` phase**, before `WHERE`. It is a *data source combiner* — it adds columns and can multiply rows. Its output feeds into `WHERE`, `GROUP BY`, and the rest.

This placement difference is the crux of choosing between them. **If you only need to filter the outer rows (a yes/no existence test) and you do not need any columns from the other table, `EXISTS` is the semantically correct tool** — it filters without multiplying. **If you need columns from the other table in your output, you need a `JOIN`** — because a `WHERE`-clause subquery cannot contribute columns.

The key logical-order consequence: because `EXISTS` is a `WHERE` filter that runs *conceptually* per-surviving-outer-row, and because the planner short-circuits it, its cost is bounded by "one probe per outer row," not "all matching pairs." A `JOIN` in the `FROM` phase must materialize every matching pair *before* a later `DISTINCT` or `GROUP BY` can collapse them — which is why `SELECT DISTINCT ... JOIN` can be slower than `EXISTS` for existence checks: the DISTINCT has to de-duplicate work the JOIN already exploded.

---

## 4. What Is a JOIN Alternative?

A **JOIN alternative** is any construct that achieves the *result* of a join — combining information across tables, or filtering one table by its relationship to another — *without* using explicit `JOIN` syntax. The three primary alternatives are the `EXISTS`/`NOT EXISTS` predicate, the `IN`/`NOT IN` predicate, and the correlated (or uncorrelated) subquery. The central skill is knowing when they are equivalent to a join, when they are strictly better, and when they are a trap.

### 4.1 EXISTS — annotated

```sql
SELECT u.id, u.name
FROM users u
WHERE EXISTS (                    -- ┐ predicate: TRUE if the subquery returns ≥ 1 row
  SELECT 1                        -- │ └── projection is irrelevant — 1, *, NULL all identical
  FROM orders o                   -- │     the inner table being probed
  WHERE o.user_id = u.id          -- │ └── CORRELATION: references outer u — this is what makes it a semi-join
    AND o.status = 'completed'    -- │ └── additional inner predicate, ANDed into the existence test
);                                -- ┘ subquery stops at first matching row (short-circuit)
```

- `EXISTS (subquery)` → evaluates to `TRUE` the instant the subquery yields one row, `FALSE` if it yields none. Never `UNKNOWN`. Never `NULL`.
- `SELECT 1` → the select list is **never evaluated for values**; existence only checks row presence. `SELECT 1`, `SELECT *`, `SELECT NULL`, `SELECT 1/0` (in the projection) are all equivalent — the planner ignores the projection.
- The **correlation** (`o.user_id = u.id`) is what ties the subquery to the outer row. Without it, the subquery is *uncorrelated* and returns the same answer for every outer row.

### 4.2 NOT EXISTS — annotated

```sql
SELECT u.id, u.name
FROM users u
WHERE NOT EXISTS (               -- ┐ predicate: TRUE if the subquery returns ZERO rows
  SELECT 1                       -- │
  FROM orders o                  -- │
  WHERE o.user_id = u.id         -- │ └── correlation → anti-join
);                               -- ┘ TRUE only when NO matching order exists
```

- `NOT EXISTS` is `NULL`-safe. A matching row with NULLs in non-correlated columns still counts as "a row exists." The anti-join short-circuits on the first match to *exclude* the outer row.

### 4.3 IN — annotated

```sql
SELECT u.id, u.name
FROM users u
WHERE u.id IN (                  -- ┐ TRUE if u.id equals ANY value the subquery returns
  SELECT o.user_id              -- │ └── the subquery must return exactly ONE column
  FROM orders o                 -- │
  WHERE o.status = 'completed'  -- │
);                              -- ┘ typically rewritten to a semi-join, like EXISTS
```

- `x IN (subquery)` → `TRUE` if `x = v` for some value `v` in the subquery's single-column output. Equivalent to `x = ANY(subquery)`.
- **Three-valued twist**: `IN` returns `UNKNOWN` (not `FALSE`) when `x` matches nothing *and* the list contains a NULL. This is harmless for `IN` in a `WHERE` clause (UNKNOWN is filtered out just like FALSE), but it is the seed of the `NOT IN` disaster.

### 4.4 NOT IN — annotated (the dangerous one)

```sql
SELECT u.id, u.name
FROM users u
WHERE u.id NOT IN (              -- ┐ TRUE only if u.id ≠ EVERY value returned
  SELECT o.user_id              -- │ └── ⚠ if ANY returned value is NULL, whole predicate → UNKNOWN
  FROM orders o                 -- │     → outer row is dropped → query may return ZERO rows
);                              -- ┘ semantically: NOT (u.id = ANY(...)) with SQL NULL rules
```

- `x NOT IN (list)` is defined as `x <> v1 AND x <> v2 AND ...`. If any `vi` is NULL, that conjunct is `UNKNOWN`, and `TRUE AND UNKNOWN = UNKNOWN`, so the whole thing collapses to `UNKNOWN` → the row is dropped. **A single NULL in the subquery makes `NOT IN` return no rows at all.** This is *the* trap.

---

## 5. Why JOIN Alternatives Mastery Matters in Production

1. **The `NOT IN` + NULL bug is a silent, total data-loss bug.** A query that worked for months suddenly returns *zero rows* the day someone inserts a row with a NULL in the referenced column. There is no error, no warning — just an empty result set feeding into a report, a billing job, or a cleanup script. This has caused real incidents (dunning emails not sent, records not purged, dashboards showing zero). Knowing to use `NOT EXISTS` instead is the single most valuable takeaway in this topic.

2. **Choosing `EXISTS` over `JOIN + DISTINCT` avoids row multiplication.** The classic junior mistake is `SELECT DISTINCT u.* FROM users u JOIN orders o ON o.user_id = u.id`. This explodes users by their order count, then pays to de-duplicate. `EXISTS` never multiplies — it's semantically cleaner and often faster because the semi-join short-circuits.

3. **The planner treats them identically *sometimes* — and knowing when it doesn't is a senior skill.** `IN` and `EXISTS` and `JOIN`(semi) are usually interchangeable to PostgreSQL's optimizer. But `NOT IN`, correlated aggregate subqueries, and subqueries with `LIMIT` or `volatile functions` are *not* freely rewritten. Being able to read an `EXPLAIN` and see `Hash Semi Join` vs `Hash Anti Join` vs a slow `SubPlan` filter tells you whether the planner optimized your query or was forced into a naive execution.

4. **Correlated subqueries in the SELECT list are an N+1 problem in disguise** (Topic 18). A scalar subquery per output row can execute millions of times. Recognizing when to rewrite it as a join or a `LATERAL` avoids a whole class of slow queries.

5. **`EXISTS` enables early-exit semantics that JOINs cannot.** For "does this user have *any* order," `EXISTS` can stop at the first index hit — O(1) per user — whereas a naive join might scan all of a user's orders. On hot paths (auth checks, permission gates, "has unread notifications?"), this difference is real latency.

---

## 6. Deep Technical Content

### 6.1 Semi-Join Semantics: EXISTS ≡ IN ≡ JOIN-with-DISTINCT

These three queries return the **same rows** — every user who has at least one completed order:

```sql
-- (A) EXISTS — the semi-join, expressed directly
SELECT u.id, u.name
FROM users u
WHERE EXISTS (
  SELECT 1 FROM orders o
  WHERE o.user_id = u.id AND o.status = 'completed'
);

-- (B) IN — semi-join via membership test
SELECT u.id, u.name
FROM users u
WHERE u.id IN (
  SELECT o.user_id FROM orders o WHERE o.status = 'completed'
);

-- (C) JOIN + DISTINCT — semi-join emulated by exploding then de-duplicating
SELECT DISTINCT u.id, u.name
FROM users u
INNER JOIN orders o ON o.user_id = u.id
WHERE o.status = 'completed';
```

**The critical difference is (C).** Forms (A) and (B) are semi-joins — each user appears **once**, guaranteed, and the engine short-circuits at the first matching order. Form (C) is a full inner join that produces one row *per matching order* (a user with 40 completed orders produces 40 identical rows), and the `DISTINCT` then collapses them back to one. (C) does strictly more work: it materializes the fan-out, then sorts or hashes to de-duplicate.

PostgreSQL's planner will usually rewrite (A) and (B) into an identical `Semi Join` plan node. It will **not** generally rewrite (C) into a semi-join — the `DISTINCT` is applied *after* the join in the logical order, so the planner honors the explicit "join then de-dup" you wrote (though modern versions can sometimes prove equivalence, you should not rely on it). **Rule: for pure existence filtering, prefer `EXISTS`; never reach for `JOIN + DISTINCT`.**

### 6.2 When You Need Columns — JOIN Wins by Definition

`EXISTS` filters but cannot project. The moment you need a *value* from the other table, `EXISTS` is the wrong tool:

```sql
-- You need the order total → EXISTS cannot give it to you.
-- This is a JOIN's job:
SELECT u.id, u.name, o.id AS order_id, o.total_amount
FROM users u
INNER JOIN orders o ON o.user_id = u.id
WHERE o.status = 'completed';
-- Note: this INTENTIONALLY returns one row per order. That's correct here —
-- you WANT per-order data, so there's no fan-out "bug," it's the desired shape.
```

The decision tree:
- **Need only "does a related row exist?"** → `EXISTS` (semi-join, no multiplication).
- **Need "does NO related row exist?"** → `NOT EXISTS` (anti-join).
- **Need columns/values from the related table?** → `JOIN`.
- **Need an aggregate of the related table (count, sum) as one value?** → correlated scalar subquery *or* pre-aggregated join (see 6.7).

### 6.3 Anti-Join Semantics: NOT EXISTS vs NOT IN vs LEFT JOIN ... IS NULL

Three ways to express "users with no orders." They are *supposed* to be equivalent, but they diverge on NULLs and performance:

```sql
-- (A) NOT EXISTS — the correct, NULL-safe anti-join
SELECT u.id, u.name
FROM users u
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.id
);

-- (B) NOT IN — ⚠ DANGEROUS if orders.user_id can be NULL
SELECT u.id, u.name
FROM users u
WHERE u.id NOT IN (
  SELECT o.user_id FROM orders o
);

-- (C) LEFT JOIN ... IS NULL — the "anti-join by hand" idiom
SELECT u.id, u.name
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE o.user_id IS NULL;   -- keep only rows where the join found no match
```

**(A) `NOT EXISTS`** is the gold standard. It's NULL-safe: a NULL `user_id` in `orders` simply never satisfies `o.user_id = u.id` (that comparison is UNKNOWN, so the orphan order doesn't "count as a match" for any real user), and correctly, users still get evaluated for existence. The planner turns it into a clean `Anti Join`.

**(B) `NOT IN`** is a landmine. If *any* row in `orders` has `user_id = NULL`, the subquery's output contains a NULL, and — as shown in §4.4 — the entire predicate becomes UNKNOWN for every outer row, returning **zero rows**. Even when there are no NULLs, PostgreSQL often can't *prove* that at plan time (unless the column has a `NOT NULL` constraint), so it emits a slower NULL-aware plan (a `SubPlan` with per-row `hashed` or materialized checks) rather than a hash anti-join.

**(C) `LEFT JOIN ... IS NULL`** works and is NULL-safe *provided you test a NOT-NULL column of the right table for IS NULL* (here `o.user_id`, which after a failed left join is NULL because *the whole right row is NULL*). It's a valid idiom and the planner does recognize it and can convert it to an anti-join in many cases. But it's more error-prone: if you accidentally test a genuinely-nullable column, you can get wrong results. `NOT EXISTS` is clearer and just as fast.

**Verdict: use `NOT EXISTS` for anti-joins. Avoid `NOT IN` with subqueries unless the column is `NOT NULL`-constrained. `LEFT JOIN ... IS NULL` is acceptable but less self-documenting.**

### 6.4 The NOT IN + NULL Trap — In Full Detail

This deserves its own dissection because it's the most-tested interview gotcha and the most damaging production bug.

Setup:

```sql
-- orders.user_id is NULLABLE (e.g., guest checkouts store NULL)
-- users: id = 1, 2, 3
-- orders: user_id = 1, user_id = NULL   (one order has a NULL user_id)

SELECT u.id FROM users u
WHERE u.id NOT IN (SELECT o.user_id FROM orders o);
-- The subquery returns the set {1, NULL}
```

Now expand `NOT IN` per its definition for user id = 3:

```
3 NOT IN (1, NULL)
≡ NOT (3 = 1 OR 3 = NULL)
≡ NOT (FALSE OR UNKNOWN)
≡ NOT (UNKNOWN)
≡ UNKNOWN
```

`UNKNOWN` is not `TRUE`, so user 3 is **dropped** — even though intuitively user 3 clearly isn't in `{1, NULL}`. The same happens for user 2. The result is **empty**. The presence of that one NULL made SQL say "I cannot guarantee id is not in this list, because one list entry is unknown."

Why it's so insidious:
- It's **data-dependent**: the query is correct until the day a NULL appears. Passes every test with clean data.
- It's **silent**: no error, empty result. Downstream code sees "no rows" and often does nothing.
- It's **timing-sensitive**: often the NULL arrives months after deploy via some new code path (guest checkout, a soft-deleted record, an import).

Fixes, in order of preference:

```sql
-- BEST: NOT EXISTS (immune to the problem, correct anti-join semantics)
SELECT u.id FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);

-- ALSO FINE: filter NULLs out of the subquery explicitly
SELECT u.id FROM users u
WHERE u.id NOT IN (SELECT o.user_id FROM orders o WHERE o.user_id IS NOT NULL);

-- STRUCTURAL FIX: make the column NOT NULL so the planner can prove safety
ALTER TABLE orders ALTER COLUMN user_id SET NOT NULL;  -- if business rules allow
```

Note: `NOT IN` with a **literal list** (`WHERE status NOT IN ('a', 'b')`) is far less dangerous because you control the list and rarely put a NULL literal in it — but the same rule applies: `x NOT IN ('a', NULL)` still returns UNKNOWN. The trap is overwhelmingly a *subquery* problem.

### 6.5 Correlated vs Uncorrelated Subqueries

A subquery is **correlated** if it references a column from the outer query; **uncorrelated** if it's self-contained.

```sql
-- CORRELATED: inner query references u.id — logically re-evaluated per outer row
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id)

-- UNCORRELATED: inner query is constant across all outer rows
WHERE u.id IN (SELECT user_id FROM orders WHERE status = 'completed')
```

Key facts:
- **Uncorrelated subqueries** can be executed **once** and their result cached/materialized (you'll see an `InitPlan` or a hashed `SubPlan` in EXPLAIN). Cheap.
- **Correlated subqueries** are *logically* executed per outer row, but the planner usually **decorrelates** them — rewriting `EXISTS (correlated)` into a semi-join so the "per row" execution becomes a single join operation. When it *can't* decorrelate (see 6.6), the correlated subquery really does run per row → potential N+1-style blowup.
- Correlated subqueries in the **SELECT list** (scalar subqueries) are the most dangerous form for performance — see 6.7.

### 6.6 When the Planner Can and Cannot Decorrelate

PostgreSQL will convert `EXISTS`/`IN` subqueries into semi/anti-joins (decorrelate) in most cases. It is **blocked** from doing so when:

1. **The subquery has `LIMIT`/`OFFSET`** — `EXISTS (SELECT 1 ... LIMIT 1)` is fine (LIMIT 1 is redundant for EXISTS), but a subquery whose semantics depend on LIMIT can't be flattened.
2. **The subquery uses aggregates or `GROUP BY` correlated to the outer row** — e.g. `WHERE u.total > (SELECT AVG(amount) FROM orders o WHERE o.user_id = u.id)` cannot become a simple semi-join.
3. **Volatile functions** (`random()`, `now()` in some contexts, non-immutable UDFs) in the subquery prevent flattening because caching/reordering would change results.
4. **`NOT IN` with a nullable column** — as covered, can't become an anti-join safely.
5. **The subquery is inside an `OR`** with other predicates — `WHERE x = 5 OR EXISTS(...)` often blocks the semi-join rewrite; the planner falls back to a per-row `SubPlan`.

When decorrelation fails, EXPLAIN shows a `SubPlan` node with `loops=` equal to the outer row count — that's your signal the subquery is running per-row. The fix is usually to manually rewrite (e.g., move the correlated aggregate into a pre-aggregated CTE and join it).

### 6.7 Correlated Scalar Subquery in SELECT — The Hidden N+1

```sql
-- One scalar subquery per output row — logically N executions
SELECT
  u.id,
  u.name,
  (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS order_count,
  (SELECT MAX(o.created_at) FROM orders o WHERE o.user_id = u.id) AS last_order
FROM users u
WHERE u.status = 'active';
```

For 100K active users, this *may* execute each subquery 100K times (200K subquery executions total) if the planner can't optimize it — an in-database N+1 (Topic 18). The planner *sometimes* rewrites these into a single grouped join, but it's not guaranteed, especially with multiple different subqueries.

The robust rewrite is a **pre-aggregated LEFT JOIN**:

```sql
SELECT u.id, u.name,
       COALESCE(oa.order_count, 0) AS order_count,
       oa.last_order
FROM users u
LEFT JOIN (
  SELECT user_id, COUNT(*) AS order_count, MAX(created_at) AS last_order
  FROM orders
  GROUP BY user_id
) oa ON oa.user_id = u.id
WHERE u.status = 'active';
```

This computes all aggregates in **one** pass over `orders` (a single hash aggregate), then joins once. On large tables this is the difference between one 200ms query and a 30-second per-row grind. `LATERAL` (Topic 26) is the other tool when you need top-N-per-group rather than a plain aggregate.

### 6.8 IN with a Literal List vs IN with a Subquery

These are different beasts internally:

```sql
-- IN with literal list → becomes = ANY('{...}') → scalar array op, uses index if available
WHERE u.id IN (1, 2, 3, 42, 100);

-- IN with subquery → semi-join
WHERE u.id IN (SELECT user_id FROM premium_subscriptions);
```

The literal-list form compiles to `id = ANY(ARRAY[...])` and can use an index on `id`. For **very large** lists (thousands of values), the array form can be slow to plan/execute, and it's often faster to put the values in a `VALUES` list or temp table and join. For Node.js, this matters when you pass an array of IDs — see §11.

### 6.9 EXISTS vs IN on Performance — Historically and Now

Old Oracle-era advice said "`IN` is faster for small inner tables, `EXISTS` for large." **On modern PostgreSQL this is largely obsolete** — the planner rewrites both `IN` and `EXISTS` into the same `Semi Join` and picks the same physical algorithm based on cost. In the vast majority of cases `IN (subquery)` and `EXISTS (correlated)` produce *identical plans and identical performance*.

The remaining differences:
- **`IN` requires a single-column subquery**; `EXISTS` can correlate on multiple columns (`WHERE o.user_id = u.id AND o.region = u.region`). For multi-column matching, `EXISTS` is cleaner and `IN` needs the row-constructor form `WHERE (u.id, u.region) IN (SELECT user_id, region FROM ...)`.
- **`IN` with duplicate-heavy subquery output** may materialize duplicates before de-duping; `EXISTS` short-circuits. Usually the planner handles both well, but `EXISTS` is marginally safer as a default.
- **NULL semantics**: `IN` and `NOT IN` carry NULL baggage; `EXISTS`/`NOT EXISTS` never do.

**Practical rule: default to `EXISTS`/`NOT EXISTS`. Use `IN` for readability with small literal lists or clearly non-nullable single-column subqueries. Never use `NOT IN` with a nullable subquery column.**

### 6.10 Semi/Anti-Join and Row Estimation

The planner estimates semi-join output as "number of outer rows with ≥1 match" and anti-join as "outer rows with 0 matches." These estimates depend on statistics about the *correlation* selectivity. When stats are stale, the planner may badly mis-estimate — e.g., assume most users have orders when few do — and pick a Nested Loop where a Hash Anti Join would win. `ANALYZE` on both tables keeps these estimates honest. This is why a `NOT EXISTS` query can suddenly get slow after a bulk load until `ANALYZE` runs.

---

## 7. EXPLAIN — Semi-Join and Anti-Join in the Plan

### Hash Semi Join (EXISTS / IN, large tables)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.id, u.name
FROM users u
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.status = 'completed'
);
```

```
Hash Semi Join  (cost=4210.00..9800.00 rows=41000 width=36)
                (actual time=22.1..70.4 rows=38921 loops=1)
  Hash Cond: (u.id = o.user_id)
  ->  Seq Scan on users u  (cost=0.00..1840.00 rows=100000 width=36)
                           (actual time=0.01..8.0 rows=100000 loops=1)
  ->  Hash  (cost=3200.00..3200.00 rows=210000 width=8)
             (actual time=21.8..21.8 rows=205314 loops=1)
        Buckets: 262144  Batches: 1  Memory Usage: 10048kB
        ->  Seq Scan on orders o  (cost=0.00..3200.00 rows=210000 width=8)
                                   (actual time=0.02..12.1 rows=205314 loops=1)
              Filter: (status = 'completed'::text)
Buffers: shared hit=5040
Planning Time: 0.4 ms
Execution Time: 72.9 ms
```

**Reading it**:
- **`Hash Semi Join`** — the key node. "Semi" means each `users` row is emitted **at most once**, regardless of how many matching orders exist. The planner turned `EXISTS` into a semi-join. Notice `IN` would produce the identical node.
- `Hash Cond` — the equi-correlation used to probe.
- `orders` is hashed (once), then each `users` row probes and is emitted on first match — no duplication of users despite 205K orders.
- `rows=38921` out of 100K users have at least one completed order.

### Hash Anti Join (NOT EXISTS)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.id, u.name
FROM users u
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.id
);
```

```
Hash Anti Join  (cost=4210.00..9950.00 rows=59000 width=36)
                (actual time=20.4..66.2 rows=61079 loops=1)
  Hash Cond: (u.id = o.user_id)
  ->  Seq Scan on users u  (cost=0.00..1840.00 rows=100000 width=36)
                           (actual time=0.01..7.9 rows=100000 loops=1)
  ->  Hash  (cost=3200.00..3200.00 rows=240000 width=8)
             (actual time=19.9..19.9 rows=240000 loops=1)
        Buckets: 262144  Batches: 1  Memory Usage: 11720kB
        ->  Seq Scan on orders o  (cost=0.00..3200.00 rows=240000 width=8)
                                   (actual time=0.02..9.0 rows=240000 loops=1)
Buffers: shared hit=5040
Planning Time: 0.3 ms
Execution Time: 68.1 ms
```

**Reading it**:
- **`Hash Anti Join`** — emits each `users` row **only if** the probe finds **no** matching order. This is the physical anti-join.
- `rows=61079` users have zero orders.
- Same cost shape as the semi-join; anti-join is not inherently more expensive.

### The NOT IN Anti-Pattern in EXPLAIN (nullable column)

```sql
EXPLAIN (ANALYZE)
SELECT u.id FROM users u
WHERE u.id NOT IN (SELECT o.user_id FROM orders o);   -- orders.user_id is NULLABLE
```

```
Seq Scan on users u  (cost=3200.00..7100.00 rows=50000 width=4)
                     (actual time=41.0..980.5 rows=0 loops=1)
  Filter: (NOT (hashed SubPlan 1))
  Rows Removed by Filter: 100000
  SubPlan 1
    ->  Seq Scan on orders o  (cost=0.00..3200.00 rows=240000 width=8)
                              (actual time=0.02..18.0 rows=240000 loops=1)
Planning Time: 0.5 ms
Execution Time: 984.2 ms
```

**Reading it — the tells**:
- **No `Anti Join` node.** Instead a `Filter: (NOT (hashed SubPlan 1))`. The planner was **forced** to keep a NULL-aware subplan because `orders.user_id` is nullable — it *cannot* prove the anti-join is safe.
- **`rows=0`** — the smoking gun of the NULL trap. Because the subquery output contains a NULL, `NOT IN` returned nothing. `Rows Removed by Filter: 100000` — every user was wrongly discarded.
- Even without a NULL, this NULL-aware filter form is slower than a hash anti-join. `NOT EXISTS` would show `Hash Anti Join` and correct rows.

### Correlated SubPlan Running Per-Row (decorrelation failure)

```sql
EXPLAIN (ANALYZE)
SELECT u.id,
  (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS n
FROM users u
WHERE u.status = 'active';
```

```
Seq Scan on users u  (cost=0.00..820500.00 rows=50000 width=12)
                     (actual time=0.3..8402.1 rows=50000 loops=1)
  Filter: (status = 'active'::text)
  SubPlan 1
    ->  Aggregate  (cost=16.3..16.4 rows=1 width=8)
                   (actual time=0.16..0.16 rows=1 loops=50000)   ← loops = per outer row!
          ->  Index Only Scan using orders_user_id_idx on orders o
                Index Cond: (user_id = u.id)
Planning Time: 0.4 ms
Execution Time: 8420.0 ms
```

**Reading it**:
- **`loops=50000`** on the SubPlan — the scalar subquery ran once per active user. This is an in-database N+1.
- 8.4 seconds. The pre-aggregated `LEFT JOIN` rewrite from §6.7 would collapse this to a single `HashAggregate` + `Hash Left Join`, dropping it to a few hundred ms.

### What to Look For in EXPLAIN

| Symptom | Meaning | Action |
|---------|---------|--------|
| `Hash Semi Join` / `Nested Loop Semi Join` | `EXISTS`/`IN` correctly decorrelated | Good — no dedup needed |
| `Hash Anti Join` / `Nested Loop Anti Join` | `NOT EXISTS` correctly decorrelated | Good — NULL-safe |
| `Filter: (NOT (hashed SubPlan))` + `rows=0` | `NOT IN` NULL trap fired | Switch to `NOT EXISTS` |
| `SubPlan` with high `loops=` | Correlated subquery running per-row | Rewrite as pre-aggregated join |
| `InitPlan` (loops=1) | Uncorrelated subquery run once | Fine |
| `Unique`/`HashAggregate` above a `Hash Join` | `JOIN + DISTINCT` emulating a semi-join | Consider `EXISTS` to skip the dedup |

---

## 8. Query Examples

### Example 1 — Basic: EXISTS existence check

```sql
-- "Which products have ever been ordered?" — pure existence, no columns from order_items needed.
-- EXISTS is the semantically exact tool: one row per product, no multiplication.
SELECT
  p.id,
  p.name,
  p.category_id
FROM products p
WHERE EXISTS (
  SELECT 1
  FROM order_items oi
  WHERE oi.product_id = p.id        -- correlation → semi-join
)
ORDER BY p.name;
-- Compare to: SELECT DISTINCT p.* FROM products p JOIN order_items oi ...
-- which would explode each product by its order-line count, then de-dup. EXISTS avoids that.
```

### Example 2 — Intermediate: NOT EXISTS anti-join vs the NOT IN trap

```sql
-- "Find active users who have never placed a completed order" — a classic anti-join.
-- We use NOT EXISTS because orders.user_id could contain NULLs (guest checkouts),
-- which would make NOT IN silently return zero rows.
SELECT
  u.id,
  u.name,
  u.email,
  u.created_at
FROM users u
WHERE u.status = 'active'
  AND NOT EXISTS (
    SELECT 1
    FROM orders o
    WHERE o.user_id = u.id
      AND o.status = 'completed'    -- inner predicate ANDed into the existence test
  )
ORDER BY u.created_at DESC;

-- The DANGEROUS equivalent you must NOT ship:
--   WHERE u.id NOT IN (SELECT user_id FROM orders WHERE status = 'completed')
-- If any completed order has a NULL user_id, this returns ZERO rows.
```

### Example 3 — Production Grade: replacing per-row scalar subqueries with a pre-aggregated join

```sql
-- CONTEXT:
--   users: 2,000,000 rows (500K active)
--   orders: 40,000,000 rows, index on orders(user_id, status, created_at)
--   Dashboard needs: per active user — completed order count, lifetime spend, last order date,
--   AND a flag for "has an open support ticket" (existence check).
-- NAIVE version uses 3 correlated scalar subqueries in SELECT → each runs ~500K times → minutes.
-- PRODUCTION version: pre-aggregate orders once, EXISTS for the ticket flag.
-- Expectation: single hash-aggregate pass over relevant orders + one semi-join. ~1–2s warm.

SELECT
  u.id,
  u.name,
  u.email,
  COALESCE(oa.completed_orders, 0)          AS completed_orders,
  COALESCE(oa.lifetime_spend, 0)            AS lifetime_spend,
  oa.last_order_at,
  EXISTS (                                   -- existence flag: cheap semi-join, short-circuits
    SELECT 1 FROM support_tickets st          -- (support_tickets stands in for audit_logs-like table)
    WHERE st.user_id = u.id AND st.status = 'open'
  )                                          AS has_open_ticket
FROM users u
LEFT JOIN (                                   -- pre-aggregate: ONE pass over orders, not per-user
  SELECT
    o.user_id,
    COUNT(*)              AS completed_orders,
    SUM(o.total_amount)  AS lifetime_spend,
    MAX(o.created_at)    AS last_order_at
  FROM orders o
  WHERE o.status = 'completed'
  GROUP BY o.user_id
) oa ON oa.user_id = u.id
WHERE u.status = 'active'
ORDER BY oa.lifetime_spend DESC NULLS LAST
LIMIT 100;
```

EXPLAIN sketch for Example 3:

```
Limit  (actual time=1180.2..1180.4 rows=100 loops=1)
  ->  Sort  (actual rows=500000 loops=1)
        Sort Key: (COALESCE(oa.lifetime_spend, 0)) DESC
        ->  Hash Right Join  (actual rows=500000 loops=1)
              Hash Cond: (oa.user_id = u.id)
              ->  Subquery Scan on oa  (actual rows=1200000 loops=1)
                    ->  HashAggregate  (actual rows=1200000 loops=1)   ← single aggregate pass
                          Group Key: o.user_id
                          ->  Index Scan on orders o (status='completed') (actual rows=28000000)
              ->  Hash  (actual rows=500000 loops=1)
                    ->  Seq Scan on users u  Filter: (status='active')
              SubPlan 1 (has_open_ticket)
                ->  Index Only Scan on support_tickets  (loops=... but cheap, indexed & short-circuits)
Execution Time: 1183.0 ms
```

The key win: `HashAggregate` runs **once** over the completed orders, versus the naive form's per-user re-scan.

---

## 9. Wrong → Right Patterns

### Wrong 1: NOT IN with a nullable subquery column

```sql
-- WRONG: returns ZERO rows the moment any order has NULL user_id
SELECT u.id, u.name
FROM users u
WHERE u.id NOT IN (SELECT user_id FROM orders);
-- EXECUTION-LEVEL WHY: subquery yields {..., NULL}; per §4.4, every comparison
-- collapses to UNKNOWN → all outer rows dropped → empty result, no error.

-- RIGHT: NOT EXISTS — a true NULL-safe anti-join
SELECT u.id, u.name
FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);
```

### Wrong 2: JOIN + DISTINCT for an existence check

```sql
-- WRONG (works but wasteful): explode by order count, then de-duplicate
SELECT DISTINCT u.id, u.name
FROM users u
INNER JOIN orders o ON o.user_id = u.id
WHERE o.status = 'completed';
-- EXECUTION-LEVEL WHY: a user with 200 completed orders produces 200 identical rows;
-- the engine materializes all 200, then a Sort/HashAggregate collapses them. Pure waste
-- when you only wanted "does at least one exist?".

-- RIGHT: EXISTS — semi-join, short-circuits at first match, one row per user
SELECT u.id, u.name
FROM users u
WHERE EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.status = 'completed'
);
```

### Wrong 3: Correlated scalar subqueries per row (in-database N+1)

```sql
-- WRONG: three subqueries, each executed once per output row
SELECT u.id, u.name,
  (SELECT COUNT(*) FROM orders o WHERE o.user_id = u.id) AS orders,
  (SELECT SUM(total_amount) FROM orders o WHERE o.user_id = u.id) AS spend,
  (SELECT MAX(created_at) FROM orders o WHERE o.user_id = u.id) AS last_at
FROM users u WHERE u.status = 'active';
-- EXECUTION-LEVEL WHY: EXPLAIN shows SubPlan loops = active-user count. On 500K users
-- that's ~1.5M subquery executions. Minutes.

-- RIGHT: one pre-aggregated join computes all three in a single pass
SELECT u.id, u.name,
  COALESCE(a.orders, 0) AS orders,
  COALESCE(a.spend, 0)  AS spend,
  a.last_at
FROM users u
LEFT JOIN (
  SELECT user_id, COUNT(*) AS orders, SUM(total_amount) AS spend, MAX(created_at) AS last_at
  FROM orders GROUP BY user_id
) a ON a.user_id = u.id
WHERE u.status = 'active';
```

### Wrong 4: Using IN when you actually needed columns from the other table

```sql
-- WRONG shape: IN filters, but the requirement was "show each user's latest order total".
-- IN can't project columns, so people bolt on a second correlated subquery — slow and clumsy.
SELECT u.id, u.name,
  (SELECT total_amount FROM orders o WHERE o.user_id = u.id ORDER BY created_at DESC LIMIT 1) AS latest
FROM users u
WHERE u.id IN (SELECT user_id FROM orders);

-- RIGHT: if you need per-related-row data, that's a JOIN's job. Use LATERAL for top-1-per-group.
SELECT u.id, u.name, lo.total_amount AS latest
FROM users u
JOIN LATERAL (
  SELECT o.total_amount FROM orders o
  WHERE o.user_id = u.id ORDER BY o.created_at DESC LIMIT 1
) lo ON TRUE;
-- (LATERAL is Topic 26; the point: don't force EXISTS/IN to do a JOIN's projection job.)
```

### Wrong 5: NOT EXISTS with a broken correlation (accidental cross-anti-join)

```sql
-- WRONG: forgot the correlation predicate. The subquery is now uncorrelated.
SELECT u.id FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.status = 'completed');
-- EXECUTION-LEVEL WHY: the inner query returns rows for ALL users' completed orders
-- (no o.user_id = u.id link). If ANY completed order exists anywhere, EXISTS is TRUE for
-- every user, so NOT EXISTS is FALSE for every user → empty result. It "works" only when
-- there are zero completed orders in the whole table. Silent logic bug.

-- RIGHT: include the correlation
SELECT u.id FROM users u
WHERE NOT EXISTS (
  SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.status = 'completed'
);
```

---

## 10. Performance Profile

### Cost model at scale

| Pattern | 1M outer rows | 10M outer rows | 100M outer rows | Notes |
|---------|--------------|----------------|-----------------|-------|
| `EXISTS` (Hash Semi Join) | ~150–400ms | ~1.5–4s | ~15–40s | Scales linearly; needs `work_mem` for hash of inner keys |
| `EXISTS` (Nested Loop Semi, indexed inner) | ~50–200ms | ~0.5–2s | ~5–20s | Best when inner has a selective index and outer is pre-filtered |
| `NOT EXISTS` (Hash Anti Join) | ~150–400ms | ~1.5–4s | ~15–40s | Same shape as semi-join |
| `NOT IN` (nullable → NULL-aware SubPlan) | ~400ms–2s | ~4–20s | catastrophic / wrong | Can't become anti-join; also risks empty result |
| `JOIN + DISTINCT` | 2–10× the EXISTS cost | worse | worse | Pays for fan-out + de-dup |
| Correlated scalar subquery in SELECT | risk of N×inner | minutes | unusable | In-DB N+1 if not decorrelated |
| Pre-aggregated LEFT JOIN | ~200–600ms | ~2–6s | ~20–60s | Single aggregate pass; the go-to for "count/sum per parent" |

### Memory

- **Hash Semi/Anti Join** builds a hash of the inner table's join keys (not full rows — only the keys and a match bit are needed), so memory is O(distinct inner keys). Cheaper than a full hash join that must store payload columns. If it exceeds `work_mem`, `Batches > 1` and it spills to disk (5–10× slower) — raise `work_mem` for these analytic queries.
- **Nested Loop Semi/Anti Join** is O(1) memory but needs a good inner index or it's quadratic.

### CPU / short-circuit advantage

Semi and anti joins **short-circuit**: for a given outer row, the semi-join stops probing at the *first* match; the anti-join stops (and excludes the row) at the *first* match. This is why, with a selective inner index, `EXISTS` on a hot path is close to O(1) per outer row regardless of how many matches actually exist. A plain `JOIN` has no such early exit — it enumerates all matches.

### Index interactions

- The **inner table's correlation column** should be indexed (`orders(user_id)` for `EXISTS ... WHERE o.user_id = u.id`). With it, the planner can use `Nested Loop Semi Join` + `Index Only Scan`, ideal when the outer side is already small.
- **Covering / composite indexes** help: `orders(user_id, status)` lets `EXISTS ... WHERE user_id = u.id AND status = 'completed'` be answered entirely from the index (`Index Only Scan`), no heap fetch.
- For the **pre-aggregated join**, an index matching the aggregation filter (`orders(status, user_id)` or a partial index `WHERE status = 'completed'`) lets the aggregate scan only relevant rows.

### Optimization techniques specific to this topic

1. **Replace `NOT IN` (nullable) with `NOT EXISTS`** — both correctness and speed.
2. **Replace `JOIN + DISTINCT` with `EXISTS`** when no columns are needed from the joined table.
3. **Replace per-row scalar subqueries with one pre-aggregated join** (or `LATERAL` for top-N).
4. **Add the inner correlation index** so semi/anti joins can nested-loop with index probes on small outer sets.
5. **Raise `work_mem`** for large hash semi/anti joins to avoid `Batches > 1` disk spills.
6. **`ANALYZE` after bulk loads** so semi/anti-join selectivity estimates stay accurate and the planner keeps choosing the right algorithm.

---

## 11. Node.js Integration

### 11.1 EXISTS as a fast boolean gate

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Auth/permission style check: does this user have ANY open ticket?
// EXISTS short-circuits — ideal for a hot-path yes/no. Return a plain boolean.
async function userHasOpenTicket(userId) {
  const { rows } = await pool.query(
    `SELECT EXISTS (
       SELECT 1 FROM support_tickets
       WHERE user_id = $1 AND status = 'open'
     ) AS has_open`,
    [userId]
  );
  return rows[0].has_open;   // pg maps SQL boolean → JS boolean
}
```

### 11.2 NOT EXISTS anti-join (never NOT IN)

```javascript
// Users who have never placed a completed order — NULL-safe anti-join.
// Deliberately NOT using NOT IN, which would break if orders.user_id has NULLs.
async function usersWithoutCompletedOrders(limit = 100) {
  const { rows } = await pool.query(
    `SELECT u.id, u.name, u.email
     FROM users u
     WHERE u.status = 'active'
       AND NOT EXISTS (
         SELECT 1 FROM orders o
         WHERE o.user_id = u.id AND o.status = 'completed'
       )
     ORDER BY u.created_at DESC
     LIMIT $1`,
    [limit]
  );
  return rows;
}
```

### 11.3 IN with an array of IDs from the app — use ANY($1), not string interpolation

```javascript
// Common need: filter by a dynamic list of IDs passed from the app.
// DO NOT build "IN (1,2,3)" by string concatenation (injection + plan-cache churn).
// Use = ANY($1::int[]) — one bound array param, safe and index-usable.
async function usersByIds(ids /* number[] */) {
  const { rows } = await pool.query(
    `SELECT id, name, email
     FROM users
     WHERE id = ANY($1::int[])`,   // equivalent to IN (...) but a single bound param
    [ids]
  );
  return rows;
}

// For "users who are in the premium set" (subquery IN), keep it in SQL:
async function premiumActiveUsers() {
  const { rows } = await pool.query(
    `SELECT u.id, u.name
     FROM users u
     WHERE u.status = 'active'
       AND u.id IN (SELECT user_id FROM premium_subscriptions WHERE active)`
  );
  return rows;
}
```

### 11.4 Pre-aggregated join to avoid an in-app N+1

```javascript
// Instead of fetching users then looping to count each user's orders (app-level N+1, Topic 18),
// or writing per-row scalar subqueries (in-DB N+1), do it in ONE pre-aggregated query.
async function activeUserOrderStats(limit = 100) {
  const { rows } = await pool.query(
    `SELECT u.id, u.name,
            COALESCE(a.order_count, 0) AS order_count,
            COALESCE(a.spend, 0)       AS spend,
            a.last_order_at
     FROM users u
     LEFT JOIN (
       SELECT user_id,
              COUNT(*)          AS order_count,
              SUM(total_amount) AS spend,
              MAX(created_at)   AS last_order_at
       FROM orders
       WHERE status = 'completed'
       GROUP BY user_id
     ) a ON a.user_id = u.id
     WHERE u.status = 'active'
     ORDER BY spend DESC NULLS LAST
     LIMIT $1`,
    [limit]
  );
  return rows;
}
```

**Do ORMs handle this?** ORMs express `EXISTS`/`IN` well for simple existence filters, and most generate a subquery for `whereExists`-style helpers. But the *pre-aggregated join* and the *`NOT IN` → `NOT EXISTS`* safety rewrite are things you must drive deliberately — ORMs will happily generate a `NOT IN` if you ask for `notIn`, inheriting the NULL trap. See §12.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do these?** — Existence and anti-existence via relation filters `some` / `none`; `in` / `notIn` on scalar fields.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// EXISTS: users with at least one completed order → `some`
const withCompleted = await prisma.user.findMany({
  where: {
    status: 'active',
    orders: { some: { status: 'completed' } },   // → EXISTS (...) semi-join
  },
});

// NOT EXISTS: users with no completed order → `none`  (NULL-safe; Prisma emits NOT EXISTS)
const withoutCompleted = await prisma.user.findMany({
  where: {
    status: 'active',
    orders: { none: { status: 'completed' } },    // → NOT EXISTS (...) anti-join
  },
});

// notIn on a scalar → be careful: Prisma emits SQL NOT IN. If the list can contain NULL
// (e.g. from a nullable-column subquery you inline via $queryRaw), the NULL trap applies.
const exceptThese = await prisma.user.findMany({
  where: { id: { notIn: [1, 2, 3] } },            // literal list, safe (no NULLs)
});
```

**Where it breaks**: Prisma's `some`/`none` correctly generate `EXISTS`/`NOT EXISTS` (good — NULL-safe). But `notIn` maps to SQL `NOT IN`; with a literal array it's fine, but never feed it a nullable-derived set. Pre-aggregated joins and correlated aggregates require `$queryRaw`.

**Verdict**: Use `some` for EXISTS and `none` for NOT EXISTS — these are the safe, idiomatic tools. Drop to `$queryRaw` for aggregation-heavy alternatives.

---

### Drizzle ORM

**Can Drizzle do these?** — Yes, with explicit `exists`/`notExists`/`inArray`/`notInArray` helpers.

```typescript
import { db } from './db';
import { users, orders } from './schema';
import { and, eq, exists, notExists, inArray, sql } from 'drizzle-orm';

// EXISTS
const withCompleted = await db
  .select({ id: users.id, name: users.name })
  .from(users)
  .where(
    and(
      eq(users.status, 'active'),
      exists(
        db.select({ one: sql`1` }).from(orders)
          .where(and(eq(orders.userId, users.id), eq(orders.status, 'completed')))
      )
    )
  );

// NOT EXISTS (preferred anti-join)
const withoutCompleted = await db
  .select()
  .from(users)
  .where(
    notExists(
      db.select({ one: sql`1` }).from(orders)
        .where(and(eq(orders.userId, users.id), eq(orders.status, 'completed')))
    )
  );

// inArray = IN; notInArray = NOT IN (same NULL caveat — keep the array non-null)
const some = await db.select().from(users).where(inArray(users.id, [1, 2, 3]));
```

**Where it breaks**: `notInArray` generates `NOT IN` — safe for literal arrays, but Drizzle won't protect you from the NULL trap if you pass a subquery that can yield NULL. Prefer `notExists` for anti-joins.

**Verdict**: Drizzle exposes `exists`/`notExists` as first-class — use them. Its explicitness makes the semi/anti-join choice clear in code.

---

### Sequelize

**Can Sequelize do these?** — `Op.in` / `Op.notIn` for lists; existence typically via a required include or a literal subquery.

```javascript
const { Op, Sequelize } = require('sequelize');
const { User } = require('./models');

// IN (list)
const some = await User.findAll({ where: { id: { [Op.in]: [1, 2, 3] } } });

// NOT IN — ⚠ Sequelize emits raw SQL NOT IN; nullable subquery → the trap
const notThese = await User.findAll({ where: { id: { [Op.notIn]: [1, 2, 3] } } });

// EXISTS-style: Sequelize has no clean `whereExists`; use a literal correlated subquery
const withCompleted = await User.findAll({
  where: {
    status: 'active',
    [Op.and]: Sequelize.literal(`EXISTS (
      SELECT 1 FROM orders o WHERE o.user_id = "User".id AND o.status = 'completed'
    )`),
  },
});

// NOT EXISTS via literal (the safe anti-join)
const withoutCompleted = await User.findAll({
  where: Sequelize.literal(`NOT EXISTS (
    SELECT 1 FROM orders o WHERE o.user_id = "User".id AND o.status = 'completed'
  )`),
});
```

**Where it breaks**: No native `EXISTS` helper — you drop to `Sequelize.literal`, which loses type safety and requires knowing the generated table alias (`"User"`). `Op.notIn` with a nullable subquery inherits the NULL trap silently.

**Verdict**: Use `Sequelize.literal('NOT EXISTS (...)')` for anti-joins; don't trust `Op.notIn` for subquery-derived sets. Raw query for pre-aggregated joins.

---

### TypeORM

**Can TypeORM do these?** — `In()` / `Not(In())` operators; `Exists`/`whereExists` via QueryBuilder subqueries.

```typescript
import { DataSource, In, Not } from 'typeorm';
import { User } from './entities/User';

const ds = new DataSource({ /* ... */ });

// IN / NOT IN via find operators (literal lists — safe)
await ds.getRepository(User).find({ where: { id: In([1, 2, 3]) } });
await ds.getRepository(User).find({ where: { id: Not(In([1, 2, 3])) } });

// EXISTS via QueryBuilder subquery
const withCompleted = await ds.getRepository(User)
  .createQueryBuilder('u')
  .where('u.status = :s', { s: 'active' })
  .andWhere(qb => {
    const sub = qb.subQuery()
      .select('1').from('orders', 'o')
      .where('o.user_id = u.id')
      .andWhere("o.status = 'completed'")
      .getQuery();
    return `EXISTS ${sub}`;
  })
  .getMany();

// NOT EXISTS: same pattern with `NOT EXISTS ${sub}` (preferred anti-join)
```

**Where it breaks**: `Not(In([subquery]))` is awkward and can emit `NOT IN`. The QueryBuilder `EXISTS` subquery pattern is verbose and the correlation alias must match manually. No compile-time check that you avoided the `NOT IN` trap.

**Verdict**: Use the QueryBuilder `EXISTS`/`NOT EXISTS` subquery form for semi/anti-joins. Reserve `In()` for literal lists.

---

### Knex.js

**Can Knex do these?** — Yes: `.whereExists()`, `.whereNotExists()`, `.whereIn()`, `.whereNotIn()`.

```javascript
const knex = require('knex')({ client: 'pg', connection: process.env.DATABASE_URL });

// EXISTS
const withCompleted = await knex('users as u')
  .select('u.id', 'u.name')
  .where('u.status', 'active')
  .whereExists(function () {
    this.select(knex.raw('1')).from('orders as o')
      .whereRaw('o.user_id = u.id')
      .andWhere('o.status', 'completed');
  });

// NOT EXISTS (preferred anti-join)
const withoutCompleted = await knex('users as u')
  .select('u.id', 'u.name')
  .where('u.status', 'active')
  .whereNotExists(function () {
    this.select(knex.raw('1')).from('orders as o')
      .whereRaw('o.user_id = u.id')
      .andWhere('o.status', 'completed');
  });

// whereIn with subquery
const premium = await knex('users')
  .whereIn('id', knex('premium_subscriptions').select('user_id').where('active', true));

// ⚠ whereNotIn with a nullable subquery column → the NULL trap. Prefer whereNotExists.
```

**Where it breaks**: `.whereNotIn()` with a subquery whose column is nullable generates a `NOT IN` that can silently return nothing. Knex won't warn you.

**Verdict**: Knex has first-class `.whereExists()` / `.whereNotExists()` — the cleanest of the ORMs for semi/anti-joins. Use them and avoid `.whereNotIn()` for subqueries.

---

### ORM Summary Table

| ORM | EXISTS | NOT EXISTS | IN | NOT IN safety | Pre-agg join | Verdict |
|-----|--------|-----------|-----|---------------|--------------|---------|
| Prisma | `some` | `none` (safe) | `in` | `notIn` = SQL NOT IN (literal ok) | `$queryRaw` | `some`/`none` are ideal |
| Drizzle | `exists()` | `notExists()` | `inArray()` | `notInArray()` = NOT IN | raw `sql` subquery | Explicit, first-class helpers |
| Sequelize | `literal('EXISTS...')` | `literal('NOT EXISTS...')` | `Op.in` | `Op.notIn` risky | `sequelize.query()` | No native EXISTS; use literal |
| TypeORM | QB subquery | QB subquery | `In()` | `Not(In())` risky | `.query()` | Verbose but works |
| Knex | `.whereExists()` | `.whereNotExists()` | `.whereIn()` | `.whereNotIn()` risky | raw aggregates | Cleanest semi/anti helpers |

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given:
- `users(id, name, email, status, created_at)`
- `orders(id, user_id, total_amount, status, created_at)`

Write a query returning the `id`, `name`, and `email` of every **active** user who has placed **at least one** order — using `EXISTS`, not a JOIN. Then explain, in one sentence, why `EXISTS` returns one row per user while `INNER JOIN orders` would not.

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines Topic 12 LEFT JOIN and Topic 07 NULL)

Given the same tables, plus `payments(id, order_id, amount, method, created_at)`.

Write a query returning active users who have **never** made a payment. Provide **two** correct versions:
1. Using `NOT EXISTS`.
2. Using `LEFT JOIN ... IS NULL`.

Then state what would go wrong if you instead wrote `WHERE u.id NOT IN (SELECT o.user_id FROM orders o LEFT JOIN payments p ON p.order_id = o.id WHERE p.id IS NULL)` — specifically, which value in the subquery could poison the result.

```sql
-- Write your query here (both versions)
```

---

### Exercise 3 — Hard (production simulation — naive answer is wrong)

Tables:
- `products(id, name, category_id)`
- `order_items(id, order_id, product_id, quantity, unit_price)`
- `orders(id, user_id, status, created_at)`

A teammate wrote this to find "products never sold in a completed order":

```sql
SELECT p.id, p.name
FROM products p
WHERE p.id NOT IN (
  SELECT oi.product_id
  FROM order_items oi
  JOIN orders o ON o.id = oi.order_id
  WHERE o.status = 'completed'
);
```

1. Under what data condition does this return the **wrong** result (possibly zero rows)? Be specific about which column and value.
2. Rewrite it correctly using `NOT EXISTS`.
3. The `order_items` table has 80M rows. What index makes the `NOT EXISTS` fast, and what EXPLAIN node do you expect to see?

```sql
-- Write your corrected query and index here
```

---

### Exercise 4 — Interview Simulation

You're given `users` (5M rows) and `sessions(id, user_id, started_at, ended_at)` (900M rows, indexed on `sessions(user_id, started_at)`).

The product team wants: for each active user, (a) whether they have **any** session in the last 7 days (a boolean flag), and (b) their **total session count** in the last 30 days. A junior wrote two correlated scalar subqueries in the SELECT and it times out.

1. Explain, referencing EXPLAIN, why the junior's version is slow.
2. Rewrite it so (a) uses an existence check and (b) uses a pre-aggregated join, in a single query.
3. Would you compute (a) and (b) in one query or two? Justify based on the different access patterns (existence short-circuit vs full aggregate).

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What's the difference between `IN` and `EXISTS`?**

A: `IN` checks whether a value is a member of a set returned by a subquery (or a literal list); the subquery must return a single column. `EXISTS` checks whether a correlated subquery returns *any* row at all — it's a pure yes/no test and ignores the subquery's select list. On modern PostgreSQL they're usually optimized into the same semi-join and perform identically. The practical differences: `EXISTS` handles multi-column correlation and never has NULL surprises, while `IN`/`NOT IN` carry NULL semantics that can bite you. I default to `EXISTS`.

**Follow-up the interviewer asks**: *"So if they produce the same plan, why prefer one?"* → Because `NOT IN` diverges dangerously (NULL trap), `EXISTS` supports multi-column correlation, and `EXISTS` reads as an existence test which matches intent.

---

**Q: Why might `SELECT ... WHERE id NOT IN (subquery)` suddenly return zero rows?**

A: Because the subquery returned a NULL. `NOT IN` is defined as `id <> v1 AND id <> v2 AND ...`; if any `vi` is NULL, that comparison is UNKNOWN, and the whole conjunction collapses to UNKNOWN for every row, so every row is filtered out. It's data-dependent — it works until a NULL appears in the referenced column. The fix is `NOT EXISTS`, which is NULL-safe, or adding `WHERE col IS NOT NULL` to the subquery.

**Follow-up**: *"Does `IN` (not `NOT IN`) have the same problem?"* → No. With `IN`, a NULL in the list can make a non-match return UNKNOWN instead of FALSE, but in a `WHERE` clause UNKNOWN and FALSE both drop the row — so the result is still correct. Only `NOT IN` turns the UNKNOWN into wrong/empty output.

---

**Q: You want users who have at least one order. When would you use a JOIN instead of EXISTS?**

A: Use `EXISTS` if I only need to *filter* users by the existence of an order and don't need any order columns — it returns one row per user with no multiplication. I'd use a `JOIN` if I actually need columns from `orders` in the output (order id, amount, date), because a `WHERE`-clause `EXISTS` subquery can't project columns. If I use a JOIN just to filter, I'd have to add `DISTINCT` to undo the fan-out, which is wasteful — that's the sign I should've used `EXISTS`.

**Follow-up**: *"What if you need the order count per user?"* → That's neither a plain EXISTS nor a plain JOIN — it's an aggregate. I'd use a pre-aggregated `LEFT JOIN` (group orders by user_id, then join) rather than a correlated scalar subquery, to avoid per-row execution.

---

### Principal Level

**Q: A `NOT EXISTS` anti-join query that ran in 200ms started taking 40 seconds after a nightly bulk load, then recovered on its own the next morning. What happened?**

A: This is a statistics staleness problem. After the bulk load, the planner's estimates for the anti-join's selectivity (how many outer rows have zero matches) were based on pre-load stats. If it wrongly estimated that few rows would match, it may have chosen a `Nested Loop Anti Join` expecting a tiny inner side, when a `Hash Anti Join` was appropriate for the now-larger table — or vice versa. The recovery "on its own the next morning" is the tell: autovacuum's `ANALYZE` eventually ran and refreshed the statistics, and the planner switched back to the good plan. The fix is to run an explicit `ANALYZE` (or `VACUUM ANALYZE`) as the final step of the bulk load job, so the good plan is chosen immediately instead of hours later. I'd confirm by capturing `EXPLAIN (ANALYZE, BUFFERS)` in both states and comparing the join node and its estimated-vs-actual rows.

**Follow-up**: *"How would you make the plan more stable regardless of stats?"* → Ensure the inner correlation column is indexed so even a nested loop stays cheap, consider increasing the statistics target on the join column (`ALTER TABLE ... ALTER COLUMN ... SET STATISTICS`), and for critical jobs run `ANALYZE` inline post-load.

---

**Q: Explain what the planner does internally with `WHERE x IN (SELECT ...)` versus `WHERE x NOT IN (SELECT ...)` when the subquery column is nullable, and why the performance and correctness differ.**

A: For `IN (subquery)`, the planner performs *subquery flattening / decorrelation* and turns it into a `Semi Join` (`Hash Semi Join`, `Nested Loop Semi Join`, or `Merge Semi Join`). A NULL in the subquery is harmless: a non-matching outer row yields UNKNOWN which, in a WHERE clause, is filtered exactly like FALSE — same result, and the semi-join is fast because it short-circuits on first match.

For `NOT IN (subquery)` with a nullable column, the planner *cannot* legally rewrite it into an `Anti Join`, because the SQL semantics require that a NULL in the list make the predicate UNKNOWN for all outer rows — an anti-join's "no match found → keep row" logic would violate that. So unless the column has a `NOT NULL` constraint (or the planner can otherwise prove non-nullability), it keeps a NULL-aware `SubPlan`/`Filter` — a materialized or hashed set with special NULL handling — which is both slower and, when a NULL is present, returns zero rows. That's why the recommendation is `NOT EXISTS`: it has no NULL baggage, so the planner always produces a clean `Anti Join`. In EXPLAIN, the difference is visible as `Hash Anti Join` (good, for NOT EXISTS) versus `Filter: (NOT (hashed SubPlan 1))` (the constrained NOT IN form).

**Follow-up**: *"How could you keep `NOT IN` and still be safe and fast?"* → Add a `NOT NULL` constraint on the column so the planner can prove safety and use an anti-join, or add `WHERE col IS NOT NULL` inside the subquery. But `NOT EXISTS` is the cleaner default.

---

**Q: When is a correlated subquery genuinely faster than the equivalent JOIN, and when is it a disaster?**

A: A correlated `EXISTS` subquery is *faster or equal* for existence tests because it short-circuits: with a selective index on the inner correlation column, each outer row does a single index probe and stops at the first match — the planner realizes this as a semi-join with an index scan inner, effectively O(1) per outer row. It never materializes the fan-out a JOIN would, and it never needs a downstream DISTINCT.

It becomes a disaster when the correlated subquery is a **scalar subquery in the SELECT list computing an aggregate**, and the planner can't decorrelate it — then it runs once per output row (EXPLAIN shows `SubPlan` with `loops` equal to the row count), turning into an in-database N+1. For 500K rows that's 500K aggregate executions. The fix is a pre-aggregated join or `LATERAL`: compute the aggregate once per group in a single hash-aggregate pass, then join. So the rule is: correlated subquery for *existence* (great), pre-aggregated join for *aggregation over the correlated set* (great); correlated *aggregate scalar subquery per row* (dangerous).

**Follow-up**: *"How do you tell from EXPLAIN which situation you're in?"* → Look for `Semi Join`/`Anti Join` nodes (decorrelated, good) versus a `SubPlan` with high `loops` (per-row execution, bad). The `loops` count is the direct signal.

---

## 15. Mental Model Checkpoint

1. **A user has 300 completed orders. `WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id AND o.status='completed')` — how many of those 300 order rows does the engine read before it decides the user qualifies, assuming an index on `orders(user_id, status)`?**

2. **You run `WHERE id NOT IN (SELECT parent_id FROM t)` and get zero rows, but you're certain some ids aren't referenced. What single fact about `t.parent_id` explains it, and what one-word SQL change fixes it?**

3. **`SELECT DISTINCT u.* FROM users u JOIN orders o ON o.user_id = u.id` vs `SELECT u.* FROM users u WHERE EXISTS (...)`. Both return the same rows. Which does more physical work and why?**

4. **Under what condition does `IN (subquery)` and `EXISTS (correlated subquery)` produce literally the same EXPLAIN plan? Under what condition do they diverge?**

5. **You see `SubPlan 1` with `loops=1200000` in an EXPLAIN. What kind of construct produces that, and what's the general rewrite to eliminate it?**

6. **Why is `LEFT JOIN orders o ON o.user_id = u.id WHERE o.user_id IS NULL` a valid anti-join, but the same idea would be wrong if you tested a genuinely-nullable column like `o.discount_code IS NULL` instead?**

7. **The planner converts `NOT EXISTS` into a `Hash Anti Join` but refuses to convert `NOT IN` (nullable column) into one. State the exact semantic reason in terms of three-valued logic.**

---

## 16. Quick Reference Card

```sql
-- SEMI-JOIN (has at least one match) — one output row per outer row, short-circuits
SELECT u.* FROM users u
WHERE EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);   -- preferred
-- equivalently:
WHERE u.id IN (SELECT user_id FROM orders);                     -- single-column only

-- ANTI-JOIN (has NO match) — NULL-safe form
SELECT u.* FROM users u
WHERE NOT EXISTS (SELECT 1 FROM orders o WHERE o.user_id = u.id);  -- ALWAYS use this

-- ⚠ NOT IN + nullable subquery column = returns ZERO rows if any NULL present
--   WHERE id NOT IN (SELECT user_id FROM orders)   -- DANGEROUS
--   Fixes: NOT EXISTS  |  add "WHERE user_id IS NOT NULL"  |  make column NOT NULL

-- DON'T fake a semi-join with JOIN + DISTINCT (explodes then de-dups):
--   SELECT DISTINCT u.* FROM users u JOIN orders o ON o.user_id=u.id   -- wasteful
--   → use EXISTS instead

-- NEED COLUMNS from the other table? That's a JOIN, not EXISTS/IN.
-- NEED AGGREGATES per parent? Pre-aggregate, don't use per-row scalar subqueries:
SELECT u.*, COALESCE(a.n,0) AS n
FROM users u
LEFT JOIN (SELECT user_id, COUNT(*) n FROM orders GROUP BY user_id) a
  ON a.user_id = u.id;

-- Node.js: array param instead of IN-list string building
--   WHERE id = ANY($1::int[])        -- one bound param, injection-safe

-- EXPLAIN signals:
--   Hash Semi Join / Anti Join   → EXISTS/NOT EXISTS decorrelated (good)
--   Filter: NOT (hashed SubPlan) + rows=0  → NOT IN NULL trap fired
--   SubPlan with high loops=     → correlated scalar subquery per row (N+1) → pre-aggregate

-- Rules of thumb:
--   Existence check      → EXISTS
--   Non-existence check  → NOT EXISTS  (never NOT IN with nullable subquery)
--   Need other table's columns → JOIN
--   Need counts/sums per parent → pre-aggregated LEFT JOIN
--   IN/EXISTS perform identically on modern PostgreSQL — pick for correctness & clarity
```

---

## Connected Topics

- **Topic 07 — NULL in Depth**: The three-valued logic that makes `NOT IN` collapse to UNKNOWN — the root cause of the trap in this topic.
- **Topic 11 — INNER JOIN in Depth**: The semi-join equivalence (`INNER JOIN + DISTINCT` ≈ `EXISTS`) and the fan-out that EXISTS avoids.
- **Topic 12 — LEFT JOIN and RIGHT JOIN**: The `LEFT JOIN ... IS NULL` anti-join idiom and why the tested column must be NOT-NULL-after-join.
- **Topic 17 — JOIN Performance Deep Dive**: Nested Loop / Hash / Merge — the same physical algorithms that execute Semi Join and Anti Join nodes.
- **Topic 18 — The N+1 Query Problem**: The application-level N+1 mirrored in-database by correlated scalar subqueries; the pre-aggregated join is the shared fix.
- **Topic 20 — GROUP BY Fundamentals**: The pre-aggregation technique used to replace per-row scalar subqueries relies on GROUP BY.
- **Topic 26 — LATERAL Joins**: The tool for "top-N per group" when a pre-aggregated join isn't enough and a per-row correlated subquery would be an N+1.
- **Topic 27 — EXISTS and NOT EXISTS**: The full dedicated treatment of semi-join and anti-join semantics that this topic introduces.
