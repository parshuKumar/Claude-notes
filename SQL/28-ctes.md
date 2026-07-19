# Topic 28 — CTEs (Common Table Expressions)
### SQL Mastery Curriculum — Phase 5: Subqueries and CTEs

---

## 1. ELI5 — The Analogy That Makes It Click

You are cooking a complicated dinner. The recipe has three stages: first you make a stock, then you use that stock to make a sauce, then you use the sauce to finish the main dish.

Imagine writing that whole recipe as one giant run-on sentence: "Take the pot with the thing you boiled the bones in for the sauce that you pour on the meat that you seared in the pan that…" — nobody can read it. Instead, a good recipe gives each stage a name and does them in order:

- **Step 1 — "the stock":** boil bones, strain. *Set it aside, call it "stock".*
- **Step 2 — "the sauce":** take *the stock*, reduce, add cream. *Call it "sauce".*
- **Step 3 — the plate:** sear the meat, spoon over *the sauce*.

A **CTE (Common Table Expression)** is exactly this: you give a name to an intermediate query result, then you refer to that name later — in the next CTE or in the final query. The `WITH` keyword is "first, prepare these named things." Each named thing is a query. The final `SELECT` is "now plate the dish using the things I prepared."

```sql
WITH stock AS (   -- Step 1: prepare and name a result
  SELECT ...
),
sauce AS (        -- Step 2: use "stock", produce and name "sauce"
  SELECT ... FROM stock ...
)
SELECT ... FROM sauce ...;   -- Step 3: the final plate
```

The whole point is readability: instead of one unreadable nested blob (a subquery inside a subquery inside a subquery), you write named, ordered stages that a human reads top to bottom. That is 90% of what a CTE is for.

The other 10% — and where senior engineers earn their pay — is that in PostgreSQL a CTE is *not just cosmetic*. It can change how the planner executes your query. That subtlety (the "optimisation fence") is the heart of this topic.

---

## 2. Connection to SQL Internals

A CTE is a **named subquery** that lives for the duration of a single statement. Internally, PostgreSQL parses the `WITH` list into a set of `CommonTableExpr` nodes hanging off the top-level query. Each one is a fully-formed subquery plan. The crucial internal question is: **does the planner treat that subquery as an opaque, pre-computed block, or does it fold it into the main query and optimise across the boundary?**

This is the concept of the **optimisation fence** (also called a *materialisation boundary*).

- **Pre-PostgreSQL 12:** every CTE was an unconditional optimisation fence. The planner computed each CTE **exactly once**, top to bottom, stored the entire result set in a work area (an in-memory `tuplestore` that spills to a temp file if it exceeds `work_mem`), and the outer query read from that stored tuplestore. Predicates from the outer query were **never** pushed down into the CTE. Indexes on base tables inside the CTE were used only for the CTE's own internal scan — never to satisfy an outer filter. This was a documented, relied-upon behaviour.

- **PostgreSQL 12 and later:** the planner may **inline** a CTE — copy its definition into the referencing location and optimise the combined query as a whole, exactly as it would a plain subquery or a view. This lets it push predicates down, use indexes end-to-end, and reorder joins across the old boundary. Inlining is applied when the CTE is (a) referenced exactly once, (b) not recursive, and (c) side-effect-free (no volatile functions, no data-modifying statement). Otherwise it falls back to materialising.

The internal machinery involved:

1. **`tuplestore`** — the buffer that holds a materialised CTE's rows. Backed by `work_mem`; spills to a temporary file on disk when it overflows. This is the same structure used for `Materialize` nodes and some cursors.
2. **The planner's `subquery_planner` / pull-up logic** — when a CTE is inlined, it goes through the same "subquery pull-up" path that turns a `FROM (SELECT …)` into part of the parent query, enabling predicate pushdown and join reordering.
3. **`CTEScan` vs the inlined plan** — in `EXPLAIN`, a materialised CTE shows up as a `CTE` init-node plus a `CTE Scan` node reading from it; an inlined CTE shows *no* `CTE Scan` at all — its tables appear directly in the main plan tree.
4. **MVCC snapshot semantics** — all CTEs in a statement, including data-modifying ones, see the **same snapshot** taken at the start of the statement. A writable CTE's changes are *not* visible to sibling CTEs reading the same table. This is a direct consequence of statement-level snapshot isolation and is the single biggest gotcha in writable CTEs.

So a CTE sits at the intersection of the **planner** (fence vs inline), the **executor** (tuplestore materialisation), and **MVCC** (snapshot visibility for writable CTEs). Understanding it means understanding all three.

---

## 3. Logical Execution Order Context

A CTE does not have its own slot in the classic single-block clause order (`FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT`). Instead, the `WITH` clause sits *outside* and *before* that pipeline for the statement it decorates:

```
WITH cte1 AS (...),          ← each CTE is its own full SELECT pipeline
     cte2 AS (... FROM cte1) ← may reference earlier CTEs
------------------------------------------------
FROM ...                     ← the PRIMARY statement's pipeline begins here
JOIN ...
WHERE ...
GROUP BY ...
HAVING ...
SELECT ...
ORDER BY ...
LIMIT ...
```

Key rules of ordering and scope:

1. **Definition order matters for reference, not for execution.** A CTE may reference any CTE defined *earlier* in the same `WITH` list (`cte2` can use `cte1`). It cannot reference a *later* one — except in the special case of a recursive CTE referencing itself (Topic 29). This is lexical scoping, top to bottom.

2. **Logical vs physical execution diverge.** *Logically*, you can read a `WITH` clause as "materialise cte1, then materialise cte2, then run the main query." That is the correct **mental model** and it is exactly what happens in the materialised case. But *physically* (PG12+), an inlined CTE is not computed as a discrete step at all — it is fused into the outer plan and may be evaluated lazily, partially, or interleaved. Never assume a CTE "runs first" for performance reasoning unless you know it is materialised.

3. **Each CTE is a complete pipeline.** Inside a CTE you have the full `FROM…LIMIT` machinery. A CTE can have its own `GROUP BY`, `ORDER BY`, `LIMIT`, window functions — everything. `ORDER BY` inside a materialised CTE *is* preserved into the tuplestore (unlike a plain subquery, where ordering is not guaranteed to survive) — though relying on this is discouraged; put `ORDER BY` in the final query.

4. **The `WITH` clause attaches to the primary statement, which need not be `SELECT`.** You can write `WITH … INSERT`, `WITH … UPDATE`, `WITH … DELETE`, and CTEs can themselves be `INSERT/UPDATE/DELETE … RETURNING`. That is the writable-CTE feature covered in 6.7.

5. **Relative to the outer `WHERE`:** in the materialised (fenced) case, the outer `WHERE` runs *after* the CTE has fully produced its rows — it filters the tuplestore output, and cannot reach back inside. In the inlined case, the outer `WHERE` may be pushed *down* to the CTE's base tables. This difference is the whole performance story of this topic.

---

## 4. What Is a CTE?

A Common Table Expression is a temporary, named result set defined with the `WITH` keyword that exists only for the duration of the single statement that immediately follows it. It behaves like a disposable, statement-scoped view: you name a query once and reference that name one or more times in the statement, improving readability and enabling reuse, recursion, and data-modifying pipelines.

```sql
WITH                                   -- ┐ begins the CTE list; one per statement
  recent_orders AS (                   -- │ ┌── CTE name, then AS (
    SELECT customer_id, total_amount   -- │ │   a complete SELECT — the CTE body
    FROM orders                        -- │ │
    WHERE created_at >= NOW()          -- │ │   its own WHERE/GROUP BY/etc. allowed
             - INTERVAL '30 days'      -- │ │
  ),                                   -- │ └── comma separates multiple CTEs
  customer_totals AS (                 -- │ ┌── second CTE
    SELECT customer_id,                -- │ │   MAY reference earlier CTE by name:
           SUM(total_amount) AS spend  -- │ │
    FROM recent_orders                 -- │ │   ← reads from the CTE above
    GROUP BY customer_id               -- │ │
  )                                    -- │ └── NO comma after the last CTE
SELECT c.name, ct.spend               -- ┘ the PRIMARY query begins here
FROM customer_totals ct                --   references a CTE like a table
INNER JOIN users c ON c.id = ct.customer_id
WHERE ct.spend > 1000                  --   outer filter (may or may not push down)
ORDER BY ct.spend DESC;
```

Annotated grammar of the full feature:

```sql
WITH [ RECURSIVE ]                     -- RECURSIVE enables self-reference (Topic 29)
  cte_name [ (col1, col2, ...) ] AS    -- optional explicit output column-name list
    [ [ NOT ] MATERIALIZED ]           -- ┐ PG12+ planner hint:
    (                                  -- │  MATERIALIZED     → force the fence
       <SELECT | INSERT | UPDATE       -- │  NOT MATERIALIZED → force inlining
        | DELETE ... RETURNING>        -- │  (omitted)        → planner decides
    )                                  -- ┘
  [, cte_name2 AS ( ... ) ]            -- additional CTEs, comma-separated
<primary statement>;                   -- SELECT / INSERT / UPDATE / DELETE
                                       -- that references the CTE name(s)
```

- **`WITH`** — introduces the CTE list. One `WITH` per statement; it holds any number of comma-separated CTEs.
- **`RECURSIVE`** — allows a CTE to reference itself for iterative/graph traversal. Covered fully in Topic 29. Note: `RECURSIVE` is written once after `WITH` and applies to the whole list.
- **`cte_name (col_list)`** — the name you reference later, with an optional renamed column list. The column list is required if the body's output names are ambiguous or you want to rename them (mandatory for recursive CTEs' UNION shape).
- **`MATERIALIZED` / `NOT MATERIALIZED`** — the explicit control over the optimisation fence. This is the senior-level lever and the reason this topic is HIGH interview priority.
- **The body** — any `SELECT`, or a data-modifying `INSERT`/`UPDATE`/`DELETE` (optionally with `RETURNING`) — the *writable CTE* case.

---

## 5. Why CTE Mastery Matters in Production

1. **Readability is a correctness feature, not a nicety.** A five-level nested subquery is where bugs hide — a misplaced filter, a wrong join, an aggregate at the wrong level. Refactoring it into named CTEs makes each stage independently reviewable and testable. On a team, the query that the on-call engineer can read at 3am is the query that gets fixed correctly. CTEs are the primary tool for this.

2. **The pre-PG12 fence caused (and still causes) silent performance cliffs.** Enormous amounts of production SQL was written before PG12 relying on CTEs as an optimisation fence — deliberately, to stop the planner from making a bad choice. On PG12+, those same queries may now inline and change plans, sometimes for the better, sometimes catastrophically worse. Knowing which version you are on, and what changed, is the difference between a 5ms query and a 5-minute one after a major-version upgrade.

3. **`MATERIALIZED` is a scalpel for planner misbehaviour.** When the planner inlines a CTE and then re-evaluates an expensive computation multiple times (because the CTE is referenced several times, or a predicate pushes a volatile call down), forcing `MATERIALIZED` computes it once. Conversely, `NOT MATERIALIZED` rescues a query where the old fence blocks index usage. Reaching for the right one is a routine part of senior performance tuning.

4. **Writable CTEs enable atomic multi-table pipelines.** "Move rows from `orders` to `orders_archive` and delete them, in one statement, one snapshot, all-or-nothing" is a data-modifying CTE. Done wrong (or done as separate statements) you get partial writes, race conditions, or double-processing. This is a genuinely load-bearing production pattern for archival, moderation queues, and outbox processing.

5. **Snapshot semantics of writable CTEs are a landmine.** Because all sub-statements see the same snapshot, a CTE that `DELETE`s rows does not hide them from a sibling CTE `SELECT`ing the same table. Engineers who don't know this write "delete and then count what's left" pipelines that return the wrong count. This is a classic principal-level interview trap.

6. **CTEs are the gateway to recursion and window-heavy analytics.** Sessionisation, funnel analysis, hierarchy walking, running totals over cleaned intermediate sets — all are dramatically clearer when built as a chain of CTEs. This topic is the foundation for Topic 29 (Recursive CTEs) and pairs constantly with window functions.

---

## 6. Deep Technical Content

### 6.1 The Old Optimisation Fence (Pre-PostgreSQL 12)

Before version 12, the rule was simple and absolute: **a CTE is always materialised, always computed exactly once, and always an optimisation fence.**

```sql
-- PostgreSQL 11 and earlier
WITH big AS (
  SELECT * FROM order_items   -- 20M rows
)
SELECT * FROM big WHERE product_id = 42;
```

On PG11, this **materialises all 20M rows of `order_items` into a tuplestore**, then scans that tuplestore applying `product_id = 42`. Even if `order_items(product_id)` has a perfect index, it is **not used** to satisfy the outer filter — the fence stops the predicate from being pushed into the CTE. The equivalent subquery/view form *would* use the index:

```sql
-- Same logic without a CTE — PG11 pushes the predicate, uses the index
SELECT * FROM (SELECT * FROM order_items) big WHERE product_id = 42;
```

This asymmetry surprised people constantly. But it was also *deliberately exploited*:

```sql
-- Intentional fence: force the small filtered set to be computed first,
-- so the planner cannot make a bad join-order or predicate-pushdown choice.
WITH filtered AS (
  SELECT id FROM users WHERE country = 'JP' AND status = 'active'
)
SELECT * FROM events e JOIN filtered f ON f.id = e.user_id;
```

On PG11 the `filtered` set is guaranteed to be computed once, up front. People wrote this precisely because the planner, left to inline, sometimes chose a disastrous plan. The CTE was a **planner hint disguised as syntax**.

The costs of the fence:
- **Always pays materialisation cost** — every row written to and read back from the tuplestore, even when the outer query needs only a handful.
- **Blocks predicate pushdown** — outer `WHERE` never reaches base-table indexes inside the CTE.
- **Blocks join reordering across the boundary** — the CTE result is an opaque relation.
- **Spills to disk** when the result exceeds `work_mem`, adding temp-file I/O.

### 6.2 The New Inlining Default (PostgreSQL 12+)

PostgreSQL 12 changed the default. Now a CTE is **inlined** (folded into the parent query, optimised across the boundary) when **all** of these hold:

1. It is **not** `RECURSIVE`.
2. It is referenced **exactly once** in the query.
3. It has **no side effects** — it is a plain `SELECT` (not `INSERT`/`UPDATE`/`DELETE`) and contains no **volatile** functions (e.g. `random()`, `nextval()`, `clock_timestamp()`, `setval()`).

If any condition fails, the CTE is **materialised** (the old behaviour). Specifically: a CTE referenced **two or more times** is materialised by default (so the shared work is computed once), and any data-modifying or volatile CTE is always materialised.

```sql
-- PG12+: this CTE is referenced once, no side effects → INLINED
WITH big AS (
  SELECT * FROM order_items
)
SELECT * FROM big WHERE product_id = 42;
-- Now equivalent to: SELECT * FROM order_items WHERE product_id = 42;
-- → uses order_items(product_id) index, no 20M-row materialisation
```

This is almost always a performance *win* for the single-reference case. But it is a **behaviour change on upgrade**: a PG11 query that leaned on the fence for a good plan may regress on PG12+. The migration checklist for a major upgrade should include "re-EXPLAIN every heavy CTE query."

Decision table for PG12+ defaults:

| CTE property | Default behaviour |
|---|---|
| Referenced once, plain SELECT, no volatile fn | **Inlined** |
| Referenced 2+ times | **Materialised** (compute shared work once) |
| `RECURSIVE` | **Materialised** (cannot inline) |
| Contains `INSERT`/`UPDATE`/`DELETE` | **Materialised** (side effects) |
| Contains a volatile function | **Materialised** |
| Explicit `MATERIALIZED` | **Materialised** (forced) |
| Explicit `NOT MATERIALIZED` | **Inlined** (forced, if legal) |

### 6.3 MATERIALIZED — Forcing the Fence

`MATERIALIZED` (PG12+) restores the old guarantee: compute this CTE once, into a tuplestore, do not push predicates in, do not inline.

```sql
WITH expensive AS MATERIALIZED (
  SELECT user_id, compute_score(activity_blob) AS score   -- costly per row
  FROM raw_activity
  WHERE captured_on = CURRENT_DATE
)
SELECT * FROM expensive WHERE score > 0.9
UNION ALL
SELECT * FROM expensive WHERE score BETWEEN 0.5 AND 0.9;
```

Here `expensive` is referenced twice, so it would be materialised anyway — but writing `MATERIALIZED` documents intent. The real value of *explicit* `MATERIALIZED` is when a CTE is referenced **once** but you still want the fence:

**When to force `MATERIALIZED`:**

1. **The body contains an expensive computation you don't want re-evaluated.** If inlining would push a costly function or a heavy aggregate into a context where it runs per output row or gets duplicated, materialising computes it once.

2. **You are deliberately shaping the plan.** You have measured that the inlined plan is worse (bad join order, a predicate that pushes down and *disables* an index-only scan, an estimate blow-up). `MATERIALIZED` pins the good plan. This is the modern, explicit replacement for the old "abuse the fence" trick — now it says what it means.

3. **Stable row-set semantics with volatile-ish reads.** You want every reference to see the identical set of rows (e.g. you sample rows and reference the sample multiple times). Materialising once guarantees one consistent snapshot of that computed set.

4. **Optimisation-fence for planner estimate errors.** When base-table statistics are so skewed that inlining leads the planner to a wildly wrong row estimate and a bad plan, materialising isolates the CTE so its (measured, single-pass) output is treated as a fixed relation.

```sql
-- Force the fence on a single-reference CTE to stop a bad pushdown
WITH candidate AS MATERIALIZED (
  SELECT id FROM products
  WHERE search_vector @@ to_tsquery('english', 'wireless & headphones')
)
SELECT p.*, oi.units
FROM candidate c
JOIN products p ON p.id = c.id
JOIN LATERAL (SELECT SUM(quantity) units FROM order_items WHERE product_id = c.id) oi ON true;
-- Materialising 'candidate' pins the full-text scan to run once, up front,
-- rather than letting the planner interleave it unfavourably.
```

### 6.4 NOT MATERIALIZED — Forcing Inlining

`NOT MATERIALIZED` forces the planner to inline even a CTE it would otherwise materialise — most usefully a CTE referenced **multiple times** where each reference has a *different, selective* outer filter that could use an index.

```sql
WITH lookup AS NOT MATERIALIZED (
  SELECT * FROM products        -- 500K rows, indexed on id and on category_id
)
SELECT * FROM lookup WHERE id = 42
UNION ALL
SELECT * FROM lookup WHERE category_id = 7;
```

With the default (materialise-because-referenced-twice), PG would scan all 500K products into a tuplestore, then filter it twice. With `NOT MATERIALIZED`, each reference is inlined and can use its own index (`products_pkey` for the first, `products_category_id_idx` for the second) — far cheaper than a full materialise if both filters are selective.

**When to force `NOT MATERIALIZED`:**
- A multi-reference CTE where each reference is highly selective and index-served, so inlining-per-reference beats one big materialise.
- You want the readability of a CTE with none of the fence cost, and you know the base scan is cheap or index-served.

**When NOT to:** if the CTE body is genuinely expensive (heavy aggregate, big join) and referenced many times, inlining re-runs that expensive body per reference — the fence (materialise once) is what you want. `NOT MATERIALIZED` on an expensive multi-reference CTE is a classic self-inflicted regression.

The planner will **refuse** `NOT MATERIALIZED` where inlining is illegal (recursive, or data-modifying) — it silently keeps materialisation in those cases rather than erroring.

### 6.5 Multiple CTEs and Chaining

A `WITH` list can hold many CTEs, each referencing earlier ones, forming a readable pipeline. This is the everyday use of CTEs.

```sql
WITH
paid AS (                                   -- stage 1: successful payments
  SELECT order_id, amount
  FROM payments
  WHERE status = 'captured'
),
order_totals AS (                           -- stage 2: roll up per order
  SELECT o.id, o.customer_id, SUM(p.amount) AS paid_amount
  FROM orders o
  JOIN paid p ON p.order_id = o.id
  GROUP BY o.id, o.customer_id
),
customer_rollup AS (                        -- stage 3: roll up per customer
  SELECT customer_id,
         COUNT(*)            AS order_count,
         SUM(paid_amount)    AS lifetime_value
  FROM order_totals
  GROUP BY customer_id
)
SELECT u.name, cr.order_count, cr.lifetime_value
FROM customer_rollup cr
JOIN users u ON u.id = cr.customer_id
ORDER BY cr.lifetime_value DESC
LIMIT 100;
```

Each stage is independently readable. **Caveat:** each of these single-reference CTEs is inlined on PG12+, so this is *not* "compute stage 1 fully, then stage 2" physically — the planner fuses the whole thing into one plan. That is usually good. If a specific stage must be pinned (measured better materialised), tag just that one `AS MATERIALIZED`.

### 6.6 A CTE Referenced Multiple Times — The Reuse Case

The canonical reason to *want* materialisation is genuine reuse: an expensive result consumed several times.

```sql
WITH monthly AS (      -- referenced 3× below → materialised by default on PG12+
  SELECT date_trunc('month', created_at) AS month,
         SUM(total_amount)               AS revenue,
         COUNT(*)                        AS orders
  FROM orders
  WHERE status = 'completed'
  GROUP BY 1
)
SELECT
  this.month,
  this.revenue,
  this.revenue - prev.revenue                                   AS mom_delta,
  ROUND(100.0*(this.revenue-prev.revenue)/NULLIF(prev.revenue,0),1) AS mom_pct
FROM monthly this
LEFT JOIN monthly prev ON prev.month = this.month - INTERVAL '1 month'
ORDER BY this.month;
```

`monthly` is referenced twice (`this`, `prev`). PG12+ materialises it once — the heavy `GROUP BY` over `orders` runs a single time, and both references read the small monthly tuplestore. This is exactly the behaviour you want; the default gets it right. (A window function could also solve this, but the self-join form shows the multi-reference materialisation clearly.)

### 6.7 Writable CTEs (Data-Modifying WITH)

A CTE body can be `INSERT`, `UPDATE`, or `DELETE`, optionally with `RETURNING`. This lets you build multi-table, multi-step mutations in a single atomic statement.

```sql
-- Archive-and-delete: move fulfilled old orders to an archive table, atomically
WITH moved AS (
  DELETE FROM orders
  WHERE status = 'fulfilled'
    AND created_at < NOW() - INTERVAL '2 years'
  RETURNING *                    -- the deleted rows flow out of the CTE
)
INSERT INTO audit_logs (entity, entity_id, action, payload, logged_at)
SELECT 'order', id, 'archived', row_to_json(moved), NOW()
FROM moved;
```

Rules and mechanics:

1. **All sub-statements execute within one statement, one transaction, one snapshot.** Either the whole thing commits or it all rolls back. The `DELETE` and the `INSERT` above cannot partially apply.

2. **A data-modifying CTE runs to completion regardless of whether its output is referenced.** Even a `DELETE … RETURNING` that nobody selects from *still deletes*. The write is the point; `RETURNING` is optional plumbing.

3. **`RETURNING` is the only way to pass modified rows onward.** Without it, the CTE has no output columns to reference.

4. **Snapshot visibility — THE big gotcha.** Every sub-statement sees the database as of the snapshot taken at statement start. A sibling CTE that reads a table another CTE is modifying sees the **pre-modification** state. Order of definition does *not* create data-flow ordering for the underlying tables (only the explicit `RETURNING` → reference chain does).

```sql
-- DANGEROUS misconception:
WITH del AS (
  DELETE FROM sessions WHERE last_seen < NOW() - INTERVAL '30 days' RETURNING id
)
SELECT COUNT(*) FROM sessions;   -- ← still counts the just-deleted rows!
-- Because SELECT sees the snapshot from statement start, before the DELETE applied.
```

5. **Execution order of multiple data-modifying CTEs is not guaranteed** and they all see the same snapshot. You must not write two CTEs that modify the *same* rows and depend on one seeing the other's effect — that behaviour is undefined. If A must see B's writes, they cannot be siblings in one statement.

6. **No `ON CONFLICT` target confusion, but watch constraint timing.** Constraints (unique, FK) are checked as each sub-statement runs. A writable CTE that inserts rows violating a unique constraint fails the whole statement.

7. **Data-modifying CTEs are always materialised** — they can never be inlined (they have side effects), so they are always a fence and always run once.

Canonical safe patterns:

```sql
-- Move-and-log (delete → insert into two different tables): SAFE
WITH removed AS (
  DELETE FROM cart_items WHERE cart_id = $1 RETURNING product_id, quantity
)
INSERT INTO order_items (order_id, product_id, quantity)
SELECT $2, product_id, quantity FROM removed;

-- Upsert-then-collect: insert new, return everything touched
WITH ins AS (
  INSERT INTO products (sku, name, price)
  VALUES ($1, $2, $3)
  ON CONFLICT (sku) DO UPDATE SET price = EXCLUDED.price
  RETURNING id, sku
)
SELECT * FROM ins;

-- Multi-table cascade in one atomic statement
WITH closed AS (
  UPDATE orders SET status = 'cancelled' WHERE id = $1 RETURNING id
),
refunded AS (
  UPDATE payments SET status = 'refunded'
  WHERE order_id IN (SELECT id FROM closed) RETURNING id
)
INSERT INTO audit_logs (entity, entity_id, action)
SELECT 'order', id, 'cancelled_with_refund' FROM closed;
```

### 6.8 CTEs vs Subqueries vs Views vs Temp Tables

| Construct | Scope / lifetime | Fence (default) | Reuse across statements | Indexable |
|---|---|---|---|---|
| **CTE** | one statement | inlined PG12+ / fenced ≤PG11 | no (single statement) | no |
| **Derived subquery** `FROM (…)` | one statement | always inlined | no | no |
| **View** | schema object | inlined (like a subquery) | yes (across statements) | no (materialized view: yes) |
| **Materialized view** | schema object, on-disk | pre-computed, refreshed | yes, persistent | yes |
| **Temp table** | session/txn | independent statements | yes (within session) | yes (you can `CREATE INDEX`) |

Rules of thumb:
- Need it **once, within a statement, readable** → CTE.
- Need to **reuse a heavy result across several statements**, or need an **index on the intermediate** → temp table.
- Need to **share a definition across the app** → view; if pre-computation is worth the staleness → materialized view.

### 6.9 CTEs and NULL / Empty-Set Behaviour

- A CTE that produces **zero rows** is legal; downstream references simply see an empty relation. Joins to it yield nothing (inner) or NULLs (outer), exactly as with any empty table.
- NULLs pass through a CTE unchanged; a CTE does not add "NOT NULL" semantics. `GROUP BY` inside a CTE follows normal NULL-grouping rules (all NULLs group together).
- A CTE column's type is inferred from its body. An all-NULL literal column may come out as `unknown`/`text` — cast it explicitly (`NULL::int`) if a later reference needs a specific type, especially in recursive CTEs where both UNION arms must have matching types.

### 6.10 Common Misuses

1. **Using a CTE purely to "make it run first" on PG12+** — no longer works by default; it inlines. Use `MATERIALIZED` if you truly need the fence.
2. **Materialising a huge base table then filtering** — the pre-PG12 fence anti-pattern; on PG12+ let it inline, or don't wrap the base table at all.
3. **Multi-reference expensive CTE forced `NOT MATERIALIZED`** — re-runs the expense per reference.
4. **Expecting a writable CTE's changes to be visible to a sibling** — snapshot semantics say no.
5. **Long chains where one stage explodes row counts** — a CTE doesn't magically bound intermediate sizes; a bad join in stage 2 fans out just as it would inline. Readability ≠ performance.

---

## 7. EXPLAIN — CTE Materialisation vs Inlining in the Plan

### Inlined CTE (PG12+ default, single reference)

```sql
EXPLAIN (ANALYZE, BUFFERS)
WITH big AS (
  SELECT * FROM order_items
)
SELECT * FROM big WHERE product_id = 42;
```

```
Index Scan using order_items_product_id_idx on order_items
        (cost=0.43..38.20 rows=9 width=40)
        (actual time=0.020..0.051 rows=11 loops=1)
  Index Cond: (product_id = 42)
Buffers: shared hit=5
Planning Time: 0.15 ms
Execution Time: 0.08 ms
```

**Reading it:** There is **no `CTE` node and no `CTE Scan`**. The CTE was inlined — the plan is exactly what a plain `SELECT … WHERE product_id = 42` would produce. The `order_items_product_id_idx` index is used because the predicate pushed all the way down. 5 buffers, sub-millisecond. This is the modern default and it is great.

### Materialised CTE (forced fence)

```sql
EXPLAIN (ANALYZE, BUFFERS)
WITH big AS MATERIALIZED (
  SELECT * FROM order_items
)
SELECT * FROM big WHERE product_id = 42;
```

```
CTE Scan on big  (cost=0.00..400000.00 rows=100 width=40)
                 (actual time=0.03..1180.4 rows=11 loops=1)
  Filter: (product_id = 42)
  Rows Removed by Filter: 19999989
  CTE big
    ->  Seq Scan on order_items
              (cost=0.00..350000.00 rows=20000000 width=40)
              (actual time=0.01..610.2 rows=20000000 loops=1)
Buffers: shared hit=192308, temp read=104000 written=104000
Planning Time: 0.12 ms
Execution Time: 1181.0 ms
```

**Reading it:**
- `CTE big` init-node with a `Seq Scan on order_items` — all **20,000,000** rows are read and written into the tuplestore.
- `temp read=104000 written=104000` — the tuplestore **spilled to disk** (exceeded `work_mem`), adding temp-file I/O.
- `CTE Scan on big` reads all 20M back, `Filter: (product_id = 42)`, `Rows Removed by Filter: 19999989` — the index is uselessly bypassed because the fence blocks pushdown.
- **1181 ms vs 0.08 ms** — the fence made this ~14,000× slower. This is precisely the pre-PG12 trap, now reproducible on demand with `MATERIALIZED`.

### Multi-reference CTE (materialised by default)

```sql
EXPLAIN (ANALYZE)
WITH monthly AS (
  SELECT date_trunc('month', created_at) AS m, SUM(total_amount) rev
  FROM orders WHERE status='completed' GROUP BY 1
)
SELECT a.m, a.rev, a.rev - b.rev delta
FROM monthly a LEFT JOIN monthly b ON b.m = a.m - INTERVAL '1 month';
```

```
Hash Left Join  (cost=... rows=24 width=...) (actual time=45.2..45.4 rows=24 loops=1)
  Hash Cond: (a.m = (b.m - '1 mon'::interval))
  CTE monthly
    ->  HashAggregate  (cost=... rows=24 ...) (actual time=44.9..45.0 rows=24 loops=1)
          Group Key: date_trunc('month', created_at)
          ->  Seq Scan on orders  (actual rows=480000 loops=1)
                Filter: (status = 'completed')
  ->  CTE Scan on monthly a   (actual rows=24 loops=1)
  ->  Hash
        ->  CTE Scan on monthly b   (actual rows=24 loops=1)
Execution Time: 45.6 ms
```

**Reading it:** One `CTE monthly` init-node computes the heavy `HashAggregate` over 480K orders **once**. Both references (`CTE Scan on monthly a`, `... b`) read the tiny 24-row result. The multi-reference default correctly avoids re-aggregating. Forcing `NOT MATERIALIZED` here would run the 480K-row aggregate twice.

### Writable CTE

```sql
EXPLAIN (ANALYZE)
WITH moved AS (
  DELETE FROM sessions WHERE last_seen < NOW() - INTERVAL '30 days' RETURNING id
)
INSERT INTO audit_logs(entity, entity_id, action)
SELECT 'session', id, 'expired' FROM moved;
```

```
Insert on audit_logs  (cost=... rows=0) (actual time=88.1..88.1 rows=0 loops=1)
  CTE moved
    ->  Delete on sessions  (actual time=61.0..79.9 rows=15230 loops=1)
          ->  Seq Scan on sessions
                Filter: (last_seen < (now() - '30 days'::interval))
                Rows Removed by Filter: 984770
  ->  CTE Scan on moved  (actual rows=15230 loops=1)
Execution Time: 88.4 ms
```

**Reading it:** The `Delete on sessions` node lives *inside* the `CTE moved` init-node — it is always materialised (side effects). 15,230 rows deleted and returned, then fed to `Insert on audit_logs`. The whole thing is one atomic node tree.

### What to look for

| Symptom in EXPLAIN | Meaning | Action |
|---|---|---|
| No `CTE Scan`, base tables in main tree | CTE was **inlined** | usually good; predicates pushed |
| `CTE <name>` + `CTE Scan` | CTE was **materialised** | check if fence is helping or hurting |
| `temp read/written` under a CTE | tuplestore spilled to disk | raise `work_mem`, or avoid materialising |
| `Rows Removed by Filter` ~= whole table under CTE Scan | fence blocked index pushdown | drop `MATERIALIZED` / use `NOT MATERIALIZED` |
| One `CTE` node, multiple `CTE Scan`s | shared reuse computed once | the multi-ref materialise default working |

---

## 8. Query Examples

### Example 1 — Basic: Readability Refactor

```sql
-- Nested subquery (hard to read) → named CTEs (clear stages)
-- Goal: customers whose 30-day spend exceeds their lifetime average order value.
WITH recent AS (                       -- last 30 days of completed orders
  SELECT customer_id, total_amount
  FROM orders
  WHERE status = 'completed'
    AND created_at >= NOW() - INTERVAL '30 days'
),
recent_spend AS (                      -- 30-day spend per customer
  SELECT customer_id, SUM(total_amount) AS spend_30d
  FROM recent
  GROUP BY customer_id
)
SELECT u.id, u.name, rs.spend_30d
FROM recent_spend rs
JOIN users u ON u.id = rs.customer_id
WHERE rs.spend_30d > 1000
ORDER BY rs.spend_30d DESC;
```

### Example 2 — Intermediate: Multi-Reference Materialisation for a MoM Report

```sql
-- Month-over-month revenue. 'monthly' referenced twice → materialised once.
WITH monthly AS (
  SELECT date_trunc('month', created_at)::date AS month,
         SUM(total_amount)                     AS revenue,
         COUNT(*)                              AS order_count
  FROM orders
  WHERE status = 'completed'
  GROUP BY 1
)
SELECT
  cur.month,
  cur.revenue,
  cur.order_count,
  prev.revenue                                              AS prev_revenue,
  ROUND(100.0 * (cur.revenue - prev.revenue)
        / NULLIF(prev.revenue, 0), 1)                       AS mom_growth_pct
FROM monthly cur
LEFT JOIN monthly prev ON prev.month = (cur.month - INTERVAL '1 month')::date
ORDER BY cur.month;
```

### Example 3 — Production-Grade: Atomic Archival Pipeline with Writable CTEs

```sql
-- Context: `orders` ~ 40M rows, `order_items` ~ 200M rows.
--   Index: orders(status, created_at), order_items(order_id).
--   Goal: in ONE atomic statement, archive fully-shipped orders older than
--   18 months plus their items, and write an audit record — all-or-nothing.
--   Perf expectation: bounded by the number of archived rows (thousands),
--   NOT by table size, thanks to the indexed predicate. ~50-300ms typical.

WITH victims AS MATERIALIZED (          -- pin the selection set once, up front
  SELECT id, customer_id, total_amount, created_at
  FROM orders
  WHERE status = 'shipped'
    AND created_at < NOW() - INTERVAL '18 months'
  LIMIT 5000                            -- batch to keep the txn/ lock window small
),
archived_items AS (                     -- move the items first (FK child)
  DELETE FROM order_items
  WHERE order_id IN (SELECT id FROM victims)
  RETURNING order_id, product_id, quantity, unit_price
),
item_rollup AS (                        -- summarise moved items per order
  SELECT order_id,
         COUNT(*)                         AS item_count,
         SUM(quantity * unit_price)       AS items_value
  FROM archived_items
  GROUP BY order_id
),
archived_orders AS (                    -- then move the parent orders
  DELETE FROM orders
  WHERE id IN (SELECT id FROM victims)
  RETURNING id, customer_id, total_amount, created_at
)
INSERT INTO audit_logs (entity, entity_id, action, payload, logged_at)
SELECT 'order',
       ao.id,
       'archived',
       jsonb_build_object(
         'customer_id', ao.customer_id,
         'total_amount', ao.total_amount,
         'item_count',  COALESCE(ir.item_count, 0),
         'items_value', COALESCE(ir.items_value, 0)
       ),
       NOW()
FROM archived_orders ao
LEFT JOIN item_rollup ir ON ir.order_id = ao.id;
```

```sql
EXPLAIN (ANALYZE, BUFFERS)   -- shape of the plan
```
```
Insert on audit_logs  (actual time=142.6..142.6 rows=0 loops=1)
  CTE victims
    ->  Limit  (actual rows=5000 loops=1)
          ->  Index Scan using orders_status_created_at_idx on orders
                Index Cond: (status='shipped' AND created_at < ...)
  CTE archived_items
    ->  Delete on order_items  (actual rows=24831 loops=1)
          ->  Nested Loop
                ->  CTE Scan on victims  (actual rows=5000 loops=1)
                ->  Index Scan using order_items_order_id_idx on order_items
  CTE item_rollup
    ->  HashAggregate  (actual rows=4998 loops=1)
          ->  CTE Scan on archived_items (actual rows=24831 loops=1)
  CTE archived_orders
    ->  Delete on orders  (actual rows=5000 loops=1)
          ->  Nested Loop
                ->  CTE Scan on victims
                ->  Index Scan using orders_pkey on orders
  ->  Hash Left Join  (actual rows=5000 loops=1)
Buffers: shared hit=51204 read=980
Execution Time: 143.9 ms
```

The indexed `victims` selection means the whole atomic archival touches only the ~5,000 targeted orders and their items — never a full scan of the 40M/200M tables.

---

## 9. Wrong → Right Patterns

### Wrong 1: Wrapping a base table in a CTE expecting the old fence to help

```sql
-- WRONG (intent: "compute the filtered set first"), on PG12+
WITH all_items AS (
  SELECT * FROM order_items          -- 200M rows
)
SELECT * FROM all_items WHERE order_id = 998877;
```
On PG12+ this happens to be fine (it inlines and uses the index). But developers migrating **from** PG11 sometimes add `MATERIALIZED` "to be safe," reintroducing the fence:
```sql
-- WRONG on any version: forces a 200M-row materialise + spill
WITH all_items AS MATERIALIZED (
  SELECT * FROM order_items
)
SELECT * FROM all_items WHERE order_id = 998877;
-- Seq Scan 200M rows → tuplestore → filter. Seconds instead of microseconds.
```
```sql
-- RIGHT: don't wrap a base table; or let it inline (no MATERIALIZED)
SELECT * FROM order_items WHERE order_id = 998877;
-- Index Scan on order_items(order_id) → sub-millisecond.
```
**Why:** `MATERIALIZED` blocks predicate pushdown, so the `order_id` index is never used to satisfy the outer filter — the full table is materialised first.

### Wrong 2: Expecting a writable CTE's delete to be visible to a sibling count

```sql
-- WRONG: expecting COUNT to reflect the deletion
WITH purged AS (
  DELETE FROM sessions WHERE last_seen < NOW() - INTERVAL '90 days' RETURNING id
)
SELECT (SELECT COUNT(*) FROM sessions)      AS remaining,   -- WRONG value
       (SELECT COUNT(*) FROM purged)        AS deleted;
-- 'remaining' STILL includes the just-deleted rows: both statements share the
-- snapshot from statement start, so SELECT sees sessions before the DELETE.
```
```sql
-- RIGHT: derive "remaining" from arithmetic, or query in a separate statement
WITH purged AS (
  DELETE FROM sessions WHERE last_seen < NOW() - INTERVAL '90 days' RETURNING id
)
SELECT
  (SELECT COUNT(*) FROM purged)                                    AS deleted,
  (SELECT COUNT(*) FROM sessions) - (SELECT COUNT(*) FROM purged)  AS remaining;
-- 'sessions' here still reflects the pre-delete snapshot, so subtract the
-- deleted count. For a truly current count, run a second statement after commit.
```
**Why:** MVCC statement-level snapshot — all sub-statements of one command see the same pre-command view of every table.

### Wrong 3: Forcing `NOT MATERIALIZED` on an expensive, multi-referenced CTE

```sql
-- WRONG: heavy aggregate, referenced twice, forced to inline → runs twice
WITH heavy AS NOT MATERIALIZED (
  SELECT customer_id, SUM(total_amount) AS lifetime
  FROM orders GROUP BY customer_id           -- scans/aggregates 40M orders
)
SELECT * FROM heavy WHERE lifetime > 10000
UNION ALL
SELECT * FROM heavy WHERE lifetime BETWEEN 5000 AND 10000;
-- The 40M-row aggregate is executed TWICE (once per reference). ~2× the cost.
```
```sql
-- RIGHT: let the multi-reference default materialise it once (or say so)
WITH heavy AS MATERIALIZED (
  SELECT customer_id, SUM(total_amount) AS lifetime
  FROM orders GROUP BY customer_id
)
SELECT * FROM heavy WHERE lifetime > 10000
UNION ALL
SELECT * FROM heavy WHERE lifetime BETWEEN 5000 AND 10000;
-- Aggregate runs once into a tuplestore; both references read it.
```
**Why:** `NOT MATERIALIZED` copies the CTE body into each reference site; an expensive body is then re-executed per site. The fence (materialise once) is exactly right for shared expensive work.

### Wrong 4: Deleting child and parent in the wrong CTE order (FK violation)

```sql
-- WRONG: delete parent orders first while child order_items still reference them
WITH del_orders AS (
  DELETE FROM orders WHERE id = ANY($1) RETURNING id
)
DELETE FROM order_items WHERE order_id IN (SELECT id FROM del_orders);
-- FK order_items.order_id → orders.id is violated when parents go first
-- (unless ON DELETE CASCADE). Statement aborts.
```
```sql
-- RIGHT: delete the child rows first, then the parent
WITH del_items AS (
  DELETE FROM order_items WHERE order_id = ANY($1) RETURNING order_id
)
DELETE FROM orders
WHERE id = ANY($1);
-- Children removed before parents → FK satisfied. Both in one atomic statement.
```
**Why:** foreign-key constraints are enforced per sub-statement; parents cannot be removed while referencing children exist.

### Wrong 5: Relying on CTE definition order for data flow between writes

```sql
-- WRONG: assuming 'b' sees rows inserted by 'a'
WITH a AS (
  INSERT INTO staging(x) SELECT g FROM generate_series(1,100) g RETURNING x
),
b AS (
  SELECT COUNT(*) AS n FROM staging      -- reads pre-insert snapshot → sees 0 new
)
SELECT n FROM b;
```
```sql
-- RIGHT: pass data explicitly via RETURNING, not via the shared table
WITH a AS (
  INSERT INTO staging(x) SELECT g FROM generate_series(1,100) g RETURNING x
)
SELECT COUNT(*) AS n FROM a;             -- reads the RETURNING output = 100
```
**Why:** the only guaranteed data path between CTEs is the `RETURNING` → reference chain; reading the underlying table from a sibling gives the snapshot, not the sibling's uncommitted writes.

---

## 10. Performance Profile

### Cost model of the two modes

| Aspect | Inlined CTE | Materialised CTE |
|---|---|---|
| Base-table access | index-served if predicate pushes down | as written inside the CTE only |
| Extra memory | none beyond the fused plan | tuplestore ≈ full result size |
| Disk | none | temp file if result > `work_mem` |
| Recompute on N references | body evaluated per reference | body evaluated once, read N times |
| Planner freedom | full (join reorder, pushdown) | boundary is opaque |

### Scaling behaviour

- **1M rows, single-reference, filtered:** inlined → index scan, a few pages, sub-ms to low-ms. Materialised → full 1M materialise + filter, tens to hundreds of ms, possible spill.
- **10M rows:** the gap widens to seconds. A materialised CTE over a base table with an outer selective predicate is the classic self-inflicted regression — the fence forfeits the index.
- **100M rows:** materialising a base table is effectively a full-table copy to a temp file; do not do it. Either inline (default) or don't wrap the table at all. Use `LIMIT`/indexed predicates *inside* the CTE if you must materialise a bounded set.
- **Multi-reference expensive aggregate:** materialise-once scales far better than inline-per-reference — cost is O(body) + O(N × small read) instead of O(N × body).

### `work_mem` and tuplestore spill

A materialised CTE holds its full result in a tuplestore sized by `work_mem`. Exceed it and it spills to a temp file (`temp read/written` in EXPLAIN), which can dominate runtime. For a deliberately materialised intermediate that is large but reused many times, consider raising `work_mem` for that session/transaction (`SET LOCAL work_mem = '256MB'`) — but prefer bounding the CTE's row count instead.

### Optimisation techniques specific to CTEs

1. **Push selective predicates *inside* the CTE.** If you must materialise, filter within the body so the tuplestore is small: `WITH x AS MATERIALIZED (SELECT … WHERE selective_cond)`.
2. **Bound with `LIMIT` inside a materialised CTE** for batch pipelines (archival), keeping lock and memory windows small.
3. **Prefer the default for single-reference CTEs** — it inlines and uses indexes; only override with evidence from EXPLAIN.
4. **Use `MATERIALIZED` to compute-once genuinely shared expensive work**; use `NOT MATERIALIZED` only when each reference is index-selective and the body is cheap.
5. **Add indexes on the base tables**, not the CTE (you cannot index a CTE). If you need an index on an intermediate, use a temp table instead.
6. **Re-EXPLAIN after a major-version upgrade** — the default fence behaviour changed at PG12; plans can flip.

---

## 11. Node.js Integration

### 11.1 Basic multi-stage CTE query

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function topCustomersByRecentSpend(minSpend = 1000, days = 30) {
  const { rows } = await pool.query(
    `WITH recent AS (
       SELECT customer_id, total_amount
       FROM orders
       WHERE status = 'completed'
         AND created_at >= NOW() - ($2 || ' days')::INTERVAL
     ),
     recent_spend AS (
       SELECT customer_id, SUM(total_amount) AS spend_30d
       FROM recent
       GROUP BY customer_id
     )
     SELECT u.id, u.name, rs.spend_30d
     FROM recent_spend rs
     JOIN users u ON u.id = rs.customer_id
     WHERE rs.spend_30d > $1
     ORDER BY rs.spend_30d DESC`,
    [minSpend, days]           // $1, $2 — always parameterise
  );
  return rows;
}
```

### 11.2 Writable CTE inside an explicit transaction

```javascript
// Atomic archival. The writable CTE is already atomic, but wrapping in a
// transaction lets you batch-loop and control commit boundaries.
async function archiveOldOrders(batchSize = 5000) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const { rows } = await client.query(
      `WITH victims AS MATERIALIZED (
         SELECT id FROM orders
         WHERE status = 'shipped'
           AND created_at < NOW() - INTERVAL '18 months'
         LIMIT $1
       ),
       del_items AS (
         DELETE FROM order_items
         WHERE order_id IN (SELECT id FROM victims)
         RETURNING order_id
       ),
       del_orders AS (
         DELETE FROM orders
         WHERE id IN (SELECT id FROM victims)
         RETURNING id
       )
       INSERT INTO audit_logs (entity, entity_id, action, logged_at)
       SELECT 'order', id, 'archived', NOW() FROM del_orders
       RETURNING entity_id`,
      [batchSize]
    );
    await client.query('COMMIT');
    return rows.length;        // number of orders archived this batch
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}
```

### 11.3 Reading RETURNING output from a mutation pipeline

```javascript
// Move cart items into an order and get back what was moved, in one round-trip.
async function checkoutCart(cartId, orderId) {
  const { rows } = await pool.query(
    `WITH moved AS (
       DELETE FROM cart_items WHERE cart_id = $1
       RETURNING product_id, quantity, unit_price
     )
     INSERT INTO order_items (order_id, product_id, quantity, unit_price)
     SELECT $2, product_id, quantity, unit_price FROM moved
     RETURNING id, product_id, quantity`,
    [cartId, orderId]
  );
  return rows;                 // the newly inserted order_items rows
}
```

### 11.4 Forcing MATERIALIZED from the app when the plan regresses

```javascript
// After profiling showed the inlined plan re-runs an expensive function,
// pin the fence. Keep the raw SQL — no ORM expresses MATERIALIZED.
async function riskScores(day) {
  const { rows } = await pool.query(
    `WITH scored AS MATERIALIZED (
       SELECT user_id, expensive_risk_score(activity) AS score
       FROM raw_activity WHERE captured_on = $1
     )
     SELECT * FROM scored WHERE score > 0.9
     UNION ALL
     SELECT * FROM scored WHERE score BETWEEN 0.5 AND 0.9`,
    [day]
  );
  return rows;
}
```

**Note on ORMs:** every mainstream Node ORM can *run* a CTE query, but essentially none model `MATERIALIZED`/`NOT MATERIALIZED` or writable CTEs in their query builders. Treat CTEs — especially materialised or data-modifying ones — as raw SQL escape-hatch territory.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do CTEs?** — Not in its type-safe query API. Prisma has no `WITH` builder, no materialisation control, no writable-CTE construct. You use `$queryRaw` / `$executeRaw`.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

// SELECT CTE via $queryRaw (tagged template parameterises safely)
const days = 30, minSpend = 1000;
const top = await prisma.$queryRaw<Array<{ id: number; name: string; spend_30d: number }>>(
  Prisma.sql`
    WITH recent AS (
      SELECT customer_id, total_amount FROM orders
      WHERE status = 'completed'
        AND created_at >= NOW() - (${days} || ' days')::INTERVAL
    ),
    recent_spend AS (
      SELECT customer_id, SUM(total_amount) AS spend_30d
      FROM recent GROUP BY customer_id
    )
    SELECT u.id, u.name, rs.spend_30d
    FROM recent_spend rs JOIN users u ON u.id = rs.customer_id
    WHERE rs.spend_30d > ${minSpend}
    ORDER BY rs.spend_30d DESC`
);

// Writable CTE MUST use $executeRaw (it mutates); returns affected-row count
const archived = await prisma.$executeRaw`
  WITH victims AS (
    SELECT id FROM orders WHERE status='shipped'
      AND created_at < NOW() - INTERVAL '18 months' LIMIT 5000
  ),
  di AS (DELETE FROM order_items WHERE order_id IN (SELECT id FROM victims) RETURNING 1)
  DELETE FROM orders WHERE id IN (SELECT id FROM victims)`;
```

**Where it breaks:** no compile-time typing of raw CTE columns beyond the generic you assert; no `MATERIALIZED` control except in the raw string. **Verdict:** raw SQL for anything beyond trivial. Use `Prisma.sql` interpolation, never string concatenation.

---

### Drizzle ORM

**Can Drizzle do CTEs?** — Yes, `SELECT` CTEs are first-class via `db.$with(...)` and `.with(...)`. Materialisation hints and writable CTEs are not modelled — use `sql` raw.

```typescript
import { db } from './db';
import { orders, users } from './schema';
import { sql, eq, gte, and } from 'drizzle-orm';

const recentSpend = db.$with('recent_spend').as(
  db.select({
      customerId: orders.customerId,
      spend: sql<number>`SUM(${orders.totalAmount})`.as('spend_30d'),
    })
    .from(orders)
    .where(and(eq(orders.status, 'completed'),
               gte(orders.createdAt, sql`NOW() - INTERVAL '30 days'`)))
    .groupBy(orders.customerId)
);

const rows = await db
  .with(recentSpend)
  .select({ id: users.id, name: users.name, spend: recentSpend.spend })
  .from(recentSpend)
  .innerJoin(users, eq(users.id, recentSpend.customerId))
  .where(sql`${recentSpend.spend} > 1000`)
  .orderBy(sql`${recentSpend.spend} DESC`);

// Writable CTE / MATERIALIZED → raw
await db.execute(sql`
  WITH moved AS (
    DELETE FROM cart_items WHERE cart_id = ${cartId}
    RETURNING product_id, quantity, unit_price
  )
  INSERT INTO order_items (order_id, product_id, quantity, unit_price)
  SELECT ${orderId}, product_id, quantity, unit_price FROM moved`);
```

**Where it breaks:** no `MATERIALIZED`/`NOT MATERIALIZED` builder; data-modifying CTEs need `sql` raw. **Verdict:** best-in-class for *read* CTEs — typed, composable. Drop to `sql` for materialisation control and writable CTEs.

---

### Sequelize

**Can Sequelize do CTEs?** — No builder support at all. `sequelize.query()` raw only.

```javascript
const { QueryTypes } = require('sequelize');

const top = await sequelize.query(
  `WITH recent_spend AS (
     SELECT customer_id, SUM(total_amount) AS spend_30d
     FROM orders
     WHERE status = 'completed'
       AND created_at >= NOW() - (:days || ' days')::INTERVAL
     GROUP BY customer_id
   )
   SELECT u.id, u.name, rs.spend_30d
   FROM recent_spend rs JOIN users u ON u.id = rs.customer_id
   WHERE rs.spend_30d > :minSpend
   ORDER BY rs.spend_30d DESC`,
  { replacements: { days: 30, minSpend: 1000 }, type: QueryTypes.SELECT }
);

// Writable CTE — no type mapping; use raw with no SELECT type
await sequelize.query(
  `WITH moved AS (
     DELETE FROM cart_items WHERE cart_id = :cartId
     RETURNING product_id, quantity, unit_price
   )
   INSERT INTO order_items (order_id, product_id, quantity, unit_price)
   SELECT :orderId, product_id, quantity, unit_price FROM moved`,
  { replacements: { cartId, orderId } }
);
```

**Where it breaks:** everything CTE-related is raw; use `replacements` (or `bind`) — never template-concatenate. **Verdict:** raw only; parameterise carefully.

---

### TypeORM

**Can TypeORM do CTEs?** — Partial. QueryBuilder has `.addCommonTableExpression()` for `SELECT` CTEs. No materialisation hint; writable CTEs need raw.

```typescript
const rows = await dataSource
  .createQueryBuilder()
  .addCommonTableExpression(
    `SELECT customer_id, SUM(total_amount) AS spend_30d
     FROM orders
     WHERE status = 'completed'
       AND created_at >= NOW() - INTERVAL '30 days'
     GROUP BY customer_id`,
    'recent_spend'
  )
  .select(['u.id AS id', 'u.name AS name', 'rs.spend_30d AS spend'])
  .from('recent_spend', 'rs')
  .innerJoin('users', 'u', 'u.id = rs.customer_id')
  .where('rs.spend_30d > :min', { min: 1000 })
  .orderBy('rs.spend_30d', 'DESC')
  .getRawMany();

// Writable CTE → raw
await dataSource.query(
  `WITH moved AS (
     DELETE FROM cart_items WHERE cart_id = $1
     RETURNING product_id, quantity, unit_price
   )
   INSERT INTO order_items (order_id, product_id, quantity, unit_price)
   SELECT $2, product_id, quantity, unit_price FROM moved`,
  [cartId, orderId]
);
```

**Where it breaks:** `addCommonTableExpression` covers read CTEs only; no `MATERIALIZED`; recursive/writable CTEs are raw. **Verdict:** OK for read CTEs via QueryBuilder; raw for the rest.

---

### Knex.js

**Can Knex do CTEs?** — Yes, the best builder coverage: `.with()`, `.withRecursive()`, `.withMaterialized()` / `.withNotMaterialized()` (recent versions). Writable CTEs still need `knex.raw`.

```javascript
const rows = await knex
  .with('recent_spend', (qb) =>
    qb.select('customer_id')
      .sum({ spend_30d: 'total_amount' })
      .from('orders')
      .where('status', 'completed')
      .andWhereRaw(`created_at >= NOW() - INTERVAL '30 days'`)
      .groupBy('customer_id')
  )
  .select('u.id', 'u.name', 'rs.spend_30d')
  .from('recent_spend as rs')
  .join('users as u', 'u.id', 'rs.customer_id')
  .where('rs.spend_30d', '>', 1000)
  .orderBy('rs.spend_30d', 'desc');

// Materialisation hint (Knex >= 2)
knex.withMaterialized('heavy', (qb) => qb.select('*').from('big')).select('*').from('heavy');

// Writable CTE → raw
await knex.raw(
  `WITH moved AS (
     DELETE FROM cart_items WHERE cart_id = ? RETURNING product_id, quantity, unit_price
   )
   INSERT INTO order_items (order_id, product_id, quantity, unit_price)
   SELECT ?, product_id, quantity, unit_price FROM moved`,
  [cartId, orderId]
);
```

**Where it breaks:** data-modifying CTEs aren't in the builder. **Verdict:** most complete CTE builder — even materialisation hints — but writable CTEs remain raw.

---

### ORM Summary Table

| ORM | Read CTE builder | Materialisation hint | Writable CTE | Verdict |
|---|---|---|---|---|
| Prisma | No (`$queryRaw`) | No | `$executeRaw` | Raw for all CTEs |
| Drizzle | Yes (`$with`/`.with`) | No | `sql` raw | Great reads; raw otherwise |
| Sequelize | No | No | Raw | Raw only |
| TypeORM | Partial (`addCommonTableExpression`) | No | Raw | Reads via QB; raw rest |
| Knex | Yes (`.with*`) | Yes (`.withMaterialized`) | `knex.raw` | Most complete; raw writes |

---

## 13. Practice Exercises

### Exercise 1 — Easy

Given:
- `orders(id, customer_id, total_amount, status, created_at)`
- `users(id, name, email)`

Rewrite this nested query as a two-stage CTE pipeline that is easy to read, returning each customer's name and their number of completed orders, only for customers with 5+ completed orders, ordered by that count descending:

```sql
SELECT u.name, t.n
FROM users u
JOIN (SELECT customer_id, COUNT(*) AS n FROM orders
      WHERE status='completed' GROUP BY customer_id) t ON t.customer_id = u.id
WHERE t.n >= 5
ORDER BY t.n DESC;
```

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines prior topics)

Given `orders`, `order_items(id, order_id, product_id, quantity, unit_price)`, `products(id, name, category_id)`, `categories(id, name)`.

Using CTEs, build a report of the top 3 products **by revenue within each category** for completed orders in the last 90 days. Return `category_name`, `product_name`, `revenue`, and the product's `rank_in_category`. (Combines CTEs with window functions and the fan-out awareness from Topic 11 — count/sum at the right grain.)

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation, naive answer is wrong)

You must, in a **single atomic statement**, expire and archive stale shopping carts:
- Move every `cart_items` row whose cart's `updated_at` is older than 7 days into an `abandoned_cart_items` table.
- Delete those `cart_items`.
- Insert one `audit_logs` row per distinct cart with the count of items abandoned.
- Return, from the whole statement, the total number of items abandoned.

The naive approach (separate `SELECT`, then `DELETE`, then `INSERT`) races with concurrent checkouts and can double-count or miss rows. Do it as one writable-CTE statement. Explain why a sibling `SELECT COUNT(*) FROM cart_items` after the delete would give the wrong "remaining" number.

Tables: `cart_items(id, cart_id, product_id, quantity, unit_price)`, `carts(id, updated_at)`, `abandoned_cart_items(cart_id, product_id, quantity, unit_price, abandoned_at)`, `audit_logs(entity, entity_id, action, payload, logged_at)`.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

On PostgreSQL 14, this query is slow — EXPLAIN shows a `Seq Scan` reading all 50M `events` into a CTE tuplestore that spills to disk, then filters:

```sql
WITH e AS MATERIALIZED (
  SELECT * FROM events
)
SELECT * FROM e WHERE user_id = $1 AND created_at >= $2;
```

1. Explain, at the executor level, exactly why it is slow.
2. Give the one-word change that fixes it and explain what the planner then does differently.
3. Under what circumstance would you *keep* `MATERIALIZED` here?

```sql
-- Write your answer / fixed query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What is a CTE and why would you use one?**

A junior answer: "A CTE is a `WITH` clause that defines a temporary named result set you can use in the rest of the query. You use it to make complex queries more readable by breaking them into named steps instead of nested subqueries."

Principal answer: All of that, plus — a CTE is statement-scoped (it exists only for the single statement), it can be referenced multiple times, it enables recursion with `WITH RECURSIVE`, and it can be data-modifying (`INSERT`/`UPDATE`/`DELETE … RETURNING`). Critically, in PostgreSQL its *readability* framing hides a performance dimension: whether it is inlined or materialised changes the plan, so a CTE is not always "free" the way a plain subquery is.

Follow-up the interviewer asks: *"Is a CTE the same as a subquery for the optimiser?"* → Not historically. Pre-PG12 it was always an optimisation fence; PG12+ inlines single-reference non-recursive side-effect-free CTEs, making them equivalent to subqueries by default — but you can still force either behaviour.

---

**Q: Can a CTE modify data?**

Junior answer: "I think CTEs are just for SELECT."

Principal answer: Yes — a CTE body can be `INSERT`, `UPDATE`, or `DELETE`, typically with `RETURNING` to pass modified rows down the pipeline. This enables atomic multi-table mutations in one statement, e.g. delete-and-archive. All sub-statements run in one transaction and share one MVCC snapshot; a data-modifying CTE always runs (even if unreferenced) and is always materialised (never inlined, because it has side effects).

Follow-up: *"If one CTE deletes rows and a sibling selects the same table, what does the sibling see?"* → The pre-delete snapshot — the sibling does not see the deletion.

---

### Principal Level

**Q: A query that was fast on PostgreSQL 11 became slow after upgrading to 14. It uses a CTE. What likely happened and how do you confirm it?**

Junior answer: "Maybe the statistics are stale, run ANALYZE."

Principal answer: The most likely cause is the PG12 CTE-inlining change. On PG11 the CTE was an optimisation fence — computed once, predicates not pushed in. The developer may have *relied* on that fence to pin a good plan (e.g. to stop a bad join order or a pushdown that disables an index-only scan). On PG12+, if the CTE is single-reference, non-recursive, and side-effect-free, it now inlines, and the planner may pick a different, worse plan. Confirm by running `EXPLAIN (ANALYZE, BUFFERS)` on both the original and a version with `AS MATERIALIZED` added: if `MATERIALIZED` restores the old plan and speed, the inlining change is the cause. Fix: add `MATERIALIZED` to pin the fence (now explicit and self-documenting), or better, rewrite so the good plan is chosen naturally. Also re-run `ANALYZE` — the reverse is common too, where inlining *helps* and the old fence was the problem.

Follow-up: *"When is inlining the thing that made it slow rather than faster?"* → When the CTE is referenced multiple times and inlining re-executes an expensive body per reference, or when a pushed-down predicate disables an index-only scan or triggers a bad estimate. Then force `MATERIALIZED`.

---

**Q: Explain the difference between `MATERIALIZED` and `NOT MATERIALIZED`, and give a concrete case for each.**

Junior answer: "MATERIALIZED saves the result and NOT MATERIALIZED doesn't."

Principal answer: `MATERIALIZED` forces the optimisation fence: the CTE is computed exactly once into a tuplestore, no predicate pushdown, no inlining. Use it when (a) the body is expensive and referenced multiple times (compute-once beats recompute), or (b) you have measured that inlining produces a worse plan and you want to pin the good one, or (c) you need a stable single-evaluation of a volatile/sampled set. `NOT MATERIALIZED` forces inlining even when the default would materialise (e.g. a multi-reference CTE): use it when each reference has a selective, index-served predicate so per-reference inlining is cheaper than one big materialise of a cheap body. Danger: `NOT MATERIALIZED` on an expensive multi-reference CTE re-runs the expense per reference; `MATERIALIZED` on a single-reference base-table wrapper forfeits index pushdown and forces a full materialise.

Follow-up: *"What happens if you write `NOT MATERIALIZED` on a recursive or data-modifying CTE?"* → It is ignored/illegal — those cannot be inlined, so PostgreSQL keeps materialisation without error.

---

**Q: You need to move 10,000 rows from a live table to an archive and delete them, exactly once, with no chance of a partial result under concurrency. How?**

Junior answer: "SELECT them, INSERT into archive, then DELETE — in a transaction."

Principal answer: The three-statement version, even in a transaction, re-reads the table between statements and can pick up or miss rows changed by concurrent writers (the `SELECT` set and the `DELETE` set can differ). The clean solution is a single data-modifying CTE: `WITH moved AS (DELETE FROM src WHERE … RETURNING *) INSERT INTO archive SELECT * FROM moved`. The `DELETE … RETURNING` atomically removes exactly the rows it returns, and those exact rows feed the `INSERT`, all in one statement, one snapshot, all-or-nothing. Add a `LIMIT` inside a materialised sub-select to batch large volumes and keep lock windows short. This eliminates the read-then-write race entirely.

Follow-up: *"Why can't a sibling CTE observe the delete, and why is that actually fine here?"* → Statement-level snapshot isolation; it's fine because data flows via `RETURNING`, not via re-reading the table.

---

## 15. Mental Model Checkpoint

1. On PostgreSQL 14, you wrap a 100M-row base table in a plain CTE and select from it with a selective indexed predicate in the outer query. Does the index get used? What if you add `MATERIALIZED`?

2. A CTE is referenced three times and its body is an expensive aggregate. What does PostgreSQL do by default, and why is that the right choice? What would `NOT MATERIALIZED` do to the cost?

3. A writable CTE deletes rows from `sessions`; the same statement's final `SELECT COUNT(*) FROM sessions` returns the count *before* the delete. Explain why in terms of MVCC.

4. You have `WITH a AS (INSERT … RETURNING x), b AS (SELECT COUNT(*) FROM the_same_table) SELECT …`. Does `b` count the rows `a` inserted? Why or why not?

5. What are the three conditions under which PostgreSQL 12+ will inline a CTE by default? If any one fails, what happens?

6. When would you deliberately choose a temp table over a CTE for an intermediate result? Name two capabilities a temp table has that a CTE does not.

7. You add `MATERIALIZED` to a single-reference CTE and the query gets 1000× slower. What almost certainly happened at the plan level, and how would `EXPLAIN` reveal it?

---

## 16. Quick Reference Card

```sql
-- ─── BASIC CTE ───────────────────────────────────────────────
WITH name AS (
  SELECT ...
)
SELECT ... FROM name ...;

-- ─── CHAINED CTEs (each may reference earlier ones) ──────────
WITH a AS (...),
     b AS (SELECT ... FROM a ...),   -- b sees a; a cannot see b
     c AS (SELECT ... FROM b ...)
SELECT ... FROM c;

-- ─── MATERIALISATION CONTROL (PG12+) ─────────────────────────
WITH x AS MATERIALIZED     (...)  -- force fence: compute once, no pushdown
WITH x AS NOT MATERIALIZED (...)  -- force inline (ignored if recursive/DML)
WITH x AS                  (...)  -- planner decides

-- Default on PG12+:
--   referenced once + plain SELECT + no volatile fn  → INLINED
--   referenced 2+ times / RECURSIVE / DML / volatile → MATERIALISED

-- ─── WRITABLE (DATA-MODIFYING) CTE ───────────────────────────
WITH moved AS (
  DELETE FROM src WHERE cond RETURNING *   -- always runs, always materialised
)
INSERT INTO dst SELECT * FROM moved;       -- atomic, one snapshot

-- Child-before-parent for FK safety:
WITH di AS (DELETE FROM order_items WHERE order_id=$1 RETURNING 1)
DELETE FROM orders WHERE id=$1;

-- ─── SNAPSHOT RULE (memorise) ────────────────────────────────
-- All sub-statements share ONE snapshot from statement start.
-- A CTE's writes are NOT visible to sibling CTEs reading the table.
-- Pass data ONLY via RETURNING → reference, never via re-reading.

-- ─── EXPLAIN SIGNALS ─────────────────────────────────────────
-- No CTE Scan, base tables in main tree        → inlined (good)
-- "CTE <name>" + "CTE Scan"                     → materialised
-- temp read/written under a CTE                 → tuplestore spilled to disk
-- Rows Removed by Filter ≈ whole table          → fence killed index pushdown

-- ─── PERF RULES OF THUMB ─────────────────────────────────────
-- Single-ref CTE → let it inline (default); don't add MATERIALIZED.
-- Expensive + multi-ref → MATERIALIZED (compute once).
-- Never MATERIALIZE a bare base table you then filter selectively.
-- Need an index on the intermediate → use a TEMP TABLE, not a CTE.
-- Re-EXPLAIN heavy CTE queries after a major-version upgrade.

-- ─── INTERVIEW ONE-LINERS ────────────────────────────────────
-- "CTE = named, statement-scoped subquery; readability first,
--  fence-vs-inline second."
-- "Pre-PG12: always a fence. PG12+: inlined unless multi-ref,
--  recursive, or side-effecting."
-- "MATERIALIZED = compute once, block pushdown. NOT MATERIALIZED =
--  inline anyway."
-- "Writable CTEs are atomic and share one snapshot — siblings don't
--  see each other's writes."
```

---

## Connected Topics

- **Topic 27 — EXISTS and NOT EXISTS** (previous): the semi-join existence check that CTEs often wrap or feed; both are tools for structuring subquery logic, and a CTE can name the set that an `EXISTS` probes.
- **Topic 29 — Recursive CTEs** (next): `WITH RECURSIVE` builds directly on this topic's grammar and materialisation model to walk hierarchies and graphs — a recursive CTE is always materialised and can never inline.
- **Topic 11 — INNER JOIN in Depth**: the fan-out / grain reasoning that governs what you aggregate inside a CTE stage.
- **Topic 19 — JOIN Alternatives** and **Topic 20 — GROUP BY**: CTEs are the standard way to pre-aggregate before joining to avoid fan-out count bugs.
- **Internals — MVCC & Snapshots**: statement-level snapshot isolation is *the* mechanism behind writable-CTE visibility semantics.
- **Internals — The Planner (subquery pull-up, predicate pushdown)**: inlining is the same pull-up machinery that folds subqueries and views into the parent query.
- **Internals — `work_mem` & tuplestore**: the executor structure that holds a materialised CTE and spills to temp files under memory pressure.
- **Topic 17 — JOIN Performance Deep Dive**: reading `EXPLAIN` for CTE Scan vs inlined plans continues the plan-reading skills from the join chapters.
