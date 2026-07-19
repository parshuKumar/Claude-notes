# Topic 16 — Multiple JOINs

### SQL Mastery Curriculum — Phase 3: JOINs — Complete Mastery

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you're planning a dinner party and you have five index-card boxes on your desk:

- **Box 1 — Guests**: each card has a guest's name and their guest ID.
- **Box 2 — RSVPs**: each card links a guest ID to the dish they signed up to bring.
- **Box 3 — Dishes**: each card describes a dish and which cuisine it belongs to.
- **Box 4 — Cuisines**: each card names a cuisine and the recommended wine.
- **Box 5 — Wines**: each card gives a wine's name and its price.

You want one final sheet: **guest name → dish → cuisine → wine → wine price**.

You cannot do this in one leap. You pull a guest card, walk to the RSVP box to find their dish ID, walk to the Dishes box to find the cuisine, walk to Cuisines to find the wine, then walk to Wines to find the price. Each "walk" is one join. Five boxes require **four walks** — four join operations chained together.

Now here's the subtle part: **the order you visit the boxes matters enormously.** If you happen to know that only 3 guests RSVP'd out of 500, you should start from the RSVP box (3 cards) and expand outward, not start from the Guests box (500 cards) and check each one. Same final sheet, wildly different amount of walking.

That's multiple joins. **The result is a chain of two-table joins, evaluated pairwise, and the order in which you chain them determines how much work you do.** The SQL planner is the party host who tries to figure out the cheapest walking route for you — and Topic 16 is about understanding, and occasionally overriding, that host's decision.

---

## 2. Connection to SQL Internals

A multi-table join is **not** a single N-way operation inside PostgreSQL. The executor only knows how to join **two** relations at a time. A query joining 5 tables is physically executed as a **tree of binary joins**:

```
              Join
             /    \
          Join    products
         /    \
      Join    order_items
     /    \
 orders  customers
```

Each internal node is one of the three physical join algorithms you learned in Topic 11 — **Nested Loop**, **Hash Join**, or **Merge Join**. The output of one join becomes the input (a relation) to the next join up the tree.

The internal machinery that decides the **shape** of this tree is the **query planner/optimizer**, specifically the **join order search**. Key internal concepts at play:

1. **Join tree enumeration** — the planner enumerates possible join orders. For N tables there are up to `(2(N-1))! / (N-1)!` possible join trees (the number of binary trees times leaf orderings). This grows factorially, so the planner cannot brute-force large joins.

2. **`join_collapse_limit`** — a planner GUC that controls how many `FROM`/explicit-`JOIN` items the planner is allowed to reorder as a single flat list. Below the limit, the planner uses **dynamic programming** (the System-R style bottom-up optimizer) to find the optimal order. Above it, it stops flattening and honors your written order for the excess.

3. **`from_collapse_limit`** — the sibling GUC that controls how far subqueries in the `FROM` list get flattened (pulled up) into the parent query before join planning.

4. **GEQO (Genetic Query Optimizer)** — when the number of join items exceeds `geqo_threshold` (default 12), the exhaustive dynamic-programming search becomes too expensive, so PostgreSQL switches to a **genetic algorithm** that evolves candidate join orders heuristically. It finds a *good* plan, not necessarily the *optimal* one, and is **non-deterministic** by default.

5. **Selectivity estimation** — the planner uses `pg_statistic` (histograms, most-common-values lists, `n_distinct`) to estimate how many rows each join produces. Bad statistics → bad row estimates → bad join order → slow query. This is the single most common cause of a multi-join query going wrong.

6. **The buffer pool and work_mem** — each Hash Join in the tree builds a hash table in `work_mem`. A 5-way join can allocate **up to 4 separate hash tables**, each up to `work_mem`. This is why a complex query can use far more memory than a single `work_mem` setting suggests.

The critical mental shift from Topic 11: with two tables the planner picks an *algorithm*; with many tables it picks an *algorithm AND a shape*, and the shape (join order) usually dominates performance.

---

## 3. Logical Execution Order Context

```
FROM t1
JOIN t2 ON ...       ← all joins evaluated in the FROM phase,
JOIN t3 ON ...          left-to-right AS WRITTEN, logically
JOIN t4 ON ...          producing one combined relation
WHERE ...            ← applied to the fully-joined relation
GROUP BY ...
HAVING ...
SELECT ...
DISTINCT
window functions
ORDER BY ...
LIMIT ...
```

Two things that matter specifically for multiple joins:

**1. Logical order vs. physical order are different things.** Logically, SQL defines that joins are evaluated left-to-right and produce a single wide relation *before* `WHERE` runs. But **physically**, the planner is free to reorder INNER joins (they are associative and commutative — Topic 11 §6.5) and to push `WHERE` predicates down into individual joins. So the *written* order in your query is a logical specification, not an execution instruction — **unless** you hit `join_collapse_limit`, disable `geqo`, or use an OUTER join that constrains reordering.

**2. Join evaluation is left-associative by syntax.** When you write:

```sql
FROM a JOIN b ON ... JOIN c ON ... JOIN d ON ...
```

this parses as `((a JOIN b) JOIN c) JOIN d`. Each successive `ON` clause can reference any table introduced **to its left**, but not tables introduced to its right. This left-to-right visibility rule is why `a JOIN b ON b.x = c.x JOIN c` is a syntax error — `c` isn't in scope yet when its `ON` runs.

The key production consequence: **for pure INNER-join queries, the planner ignores your written order** and finds its own. **The moment you introduce a LEFT/RIGHT/FULL join, you constrain what the planner may reorder** (covered in §6.6), because outer joins are *not* freely commutative.

---

## 4. What Is a Multiple JOIN?

A multiple join (or multi-way join) is any query whose `FROM` clause combines **three or more tables** through chained `JOIN` operations. It is not a distinct join *type* — it is the composition of the join types from Topics 11–15 applied repeatedly, evaluated as a tree of pairwise joins.

```sql
SELECT
  u.name,                              -- from the 1st table
  o.total_amount,                      -- from the 2nd table
  p.name AS product,                   -- from the 4th table
  cat.name AS category                 -- from the 5th table
FROM users u                           -- ① base relation (the driving table, logically)
INNER JOIN orders o                    -- ② join #1 target
       ON o.user_id = u.id             --    │ join condition references u (to the left) ✓
INNER JOIN order_items oi              -- ③ join #2 target
       ON oi.order_id = o.id           --    │ references o (already in scope) ✓
INNER JOIN products p                  -- ④ join #3 target
       ON p.id = oi.product_id         --    │ references oi (already in scope) ✓
INNER JOIN categories cat              -- ⑤ join #4 target
       ON cat.id = p.category_id       --    │ references p (already in scope) ✓
WHERE o.status = 'completed'           -- ⑥ filter applied after the joined relation is formed
ORDER BY o.total_amount DESC;          -- ⑦
```

Annotated breakdown of the mechanics each line introduces:

```
FROM users u
  │ └── the leftmost table; the planner may still choose NOT to start here physically
INNER JOIN orders o ON o.user_id = u.id
  │        │     │  └── ON condition: must be TRUE; NULL user_id in orders → row dropped
  │        │     └──── alias 'o' — required when the same column name exists in >1 table
  │        └────────── join TARGET; every JOIN adds exactly one new relation to the tree
  └───────────────── INNER: unmatched rows dropped (default if 'INNER' omitted)
INNER JOIN order_items oi ON oi.order_id = o.id
  │ └── this ON may reference u, o (everything to its left) but NOT p or cat (to its right)
INNER JOIN products p ON p.id = oi.product_id
  │ └── join #3: at this point the intermediate relation is (users×orders×order_items)
INNER JOIN categories cat ON cat.id = p.category_id
  │ └── join #4: N tables require N-1 JOIN operations. 5 tables → 4 joins.
WHERE o.status = 'completed'
  │ └── logically applied AFTER all joins; physically pushed down by the planner
```

The essential facts:
- **N tables require N−1 join operations.**
- Each `ON` clause sees only tables to its **left**.
- For all-INNER queries, written order is advisory; the planner reorders.
- For queries mixing INNER and OUTER joins, written order **constrains** reordering.

---

## 5. Why Multiple JOIN Mastery Matters in Production

1. **This is what real queries look like.** A toy query joins two tables. A production reporting query, an API endpoint hydrating a nested resource, or an analytics rollup routinely joins 4–8 tables. If you can only reason about two-table joins, you cannot debug the queries that actually run in production.

2. **Join order is the #1 cause of "it was fast yesterday" incidents.** A query that ran in 20ms can suddenly take 40 seconds after a data-distribution shift, a bulk load, or a missed `ANALYZE`, because the planner flipped to a bad join order based on stale statistics. Understanding join order is understanding this failure mode.

3. **The `join_collapse_limit` cliff.** Add one more join to a query that already has 8, cross the collapse limit, and the planner silently stops optimizing your join order — honoring your (possibly terrible) written order instead. The query gets 100× slower with no error and no obvious cause. Only someone who knows this GUC will find it.

4. **GEQO non-determinism.** Past 12 join items, the plan becomes the output of a *randomized* genetic algorithm. The same query can get different plans (and different runtimes) on different runs or different servers. Explaining and taming this requires knowing GEQO exists.

5. **Memory blowups.** A 6-way hash join can allocate 5 hash tables. On a busy server with `work_mem = 256MB` and 40 concurrent connections running the same report, you can OOM the box. Multi-join memory accounting is a real capacity-planning concern.

6. **Silent correctness bugs when mixing INNER and OUTER.** Putting a filter on an outer-joined table's column in the `WHERE` clause silently converts your LEFT JOIN into an INNER JOIN (Topic 12's trap), and in a 5-way query this is trivially easy to do by accident. The row count is wrong and no error is raised.

The cost is real: at 10M+ rows, the difference between the optimal and a naive join order is frequently the difference between a **sub-second query and a query that never finishes** before the statement timeout kills it.

---

## 6. Deep Technical Content

### 6.1 Joins Are a Left-Deep (or Bushy) Tree

Every multi-join is a binary tree. PostgreSQL's planner considers two families of shapes:

**Left-deep tree** (the classic, and what you usually get):

```
        ⋈
       / \
      ⋈   D
     / \
    ⋈   C
   / \
  A   B
```

Each join's **left** input is the accumulated result so far; the **right** input is always a base table. This is friendly to Nested Loop pipelining — you can stream the left side and probe the right.

**Bushy tree** (the planner will consider these too, below the collapse limit):

```
      ⋈
     / \
    ⋈   ⋈
   / \ / \
  A  B C  D
```

Here `(A⋈B)` and `(C⋈D)` are built independently, then joined. Bushy trees can be much cheaper when two independent pairs are each small, but they need more memory (two intermediate results materialized at once). The planner's dynamic-programming search evaluates both families.

You cannot directly force a shape in standard PostgreSQL (no query hints), but you influence it through `join_collapse_limit`, explicit `JOIN` nesting with parentheses, subqueries/CTEs, and the `pg_hint_plan` extension.

### 6.2 The Planner's Join-Order Search (System-R Dynamic Programming)

Below `geqo_threshold`, PostgreSQL uses a **bottom-up dynamic programming** algorithm derived from IBM's System R optimizer:

1. Compute the cost of accessing each single relation (seq scan vs. each usable index).
2. Compute the best plan for every **pair** of relations that have a join condition.
3. Build best plans for every **triple**, reusing the memoized best pair plans.
4. Continue up to the full set of N relations.

At each level it keeps the cheapest plan for each *set* of relations (and, importantly, for each useful *sort order* of that set — an interesting-order optimization that lets a later Merge Join reuse an existing sort). The result is the provably cheapest join order **according to the cost model** — which is only as good as the row estimates feeding it.

The complexity is roughly `O(2^N)` in the number of relations, which is why the planner caps N via the collapse limits and falls back to GEQO for large N.

### 6.3 `join_collapse_limit` — The Reordering Budget

```sql
SHOW join_collapse_limit;   -- default: 8
```

Meaning: when the planner flattens explicit `JOIN` syntax into an internal list of items it may freely reorder, it will only do so **up to this many items**. If your query has more explicit join items than `join_collapse_limit`, the planner stops collapsing and treats the leftover joins in **the order you wrote them**, planning them as a fixed nested structure.

```sql
-- With 10 joined tables and join_collapse_limit = 8:
-- The planner freely reorders the first 8 items it collapses,
-- then honors your written order for the rest. If your written
-- order is bad, you get a bad plan — silently.
```

**Why the limit exists:** the DP search is `O(2^N)`. Raising the limit to 20 could make *planning itself* take longer than executing the query. It's a tradeoff between planning time and plan quality.

**Tuning it:**

```sql
-- Raise it per-session for a known-complex analytical query:
SET join_collapse_limit = 12;
SET from_collapse_limit = 12;
-- Now run the 10-table report; the planner explores all orderings.
-- Watch planning time in EXPLAIN — if it balloons, back off.
```

A related trick: setting `join_collapse_limit = 1` **disables all join reordering**, forcing the planner to honor your exact written order. This is a deliberate tool — if *you* know the optimal order and the planner keeps getting it wrong, you can hand-write the order and pin it.

```sql
SET join_collapse_limit = 1;   -- "trust my written join order exactly"
```

### 6.4 `from_collapse_limit` — Subquery Flattening

```sql
SHOW from_collapse_limit;   -- default: 8
```

This controls how aggressively the planner **pulls up** subqueries in the `FROM` list into the parent, merging their tables into the parent's join-order search. If flattening a subquery would push the parent's item count over `from_collapse_limit`, the planner leaves the subquery as its own planning unit (an "optimization fence" of sorts).

Practical consequence: wrapping part of a query in a subquery can *prevent* the planner from reordering across the boundary once the limit is hit — sometimes helpful (you fence off a well-understood subplan), sometimes harmful (you block a beneficial reorder). Both GUCs typically move together.

### 6.5 GEQO — The Genetic Query Optimizer

```sql
SHOW geqo;              -- on
SHOW geqo_threshold;    -- 12
```

When the number of join items in a single planning problem **reaches `geqo_threshold`** (default 12), PostgreSQL abandons exhaustive DP and switches to GEQO — a **genetic algorithm**:

1. Encode each candidate join order as a "chromosome" (an ordering of relations).
2. Generate a random initial **population** of candidate orders (`geqo_pool_size`).
3. Evaluate each candidate's cost (the "fitness").
4. **Crossover**: combine good candidates to breed new orderings.
5. **Mutate** and iterate for `geqo_generations` rounds.
6. Return the best ordering found.

GEQO trades optimality for planning speed. Two important properties:

- **It is heuristic** — it may return a plan 10–20% worse than the true optimum, occasionally much worse if unlucky.
- **It is non-deterministic** — driven by a random seed (`geqo_seed`, default 0.0). By default the seed is fixed so plans are *reproducible per session*, but changing data or the seed changes the outcome. If you set a random seed, the same query can yield different plans run-to-run.

**Controlling GEQO:**

```sql
-- Disable GEQO to force exhaustive search (accept longer planning):
SET geqo = off;
-- Or raise the threshold so more tables use DP:
SET geqo_threshold = 20;
-- Now a 15-table join uses exact DP — planning is slower but plan is optimal.
```

For a critical 14-table nightly report where planning time is irrelevant compared to execution time, turning GEQO off (or raising the threshold) is a standard tuning move. For an OLTP query that must plan in microseconds, you leave GEQO on.

### 6.6 Mixing INNER and OUTER Joins — Reordering Constraints

INNER joins are commutative and associative, so the planner reorders them freely. **OUTER joins are not fully commutative or associative**, so they constrain the search space.

```sql
-- All INNER: planner may execute in ANY order
SELECT ...
FROM a
INNER JOIN b ON b.a_id = a.id
INNER JOIN c ON c.b_id = b.id;
-- Equivalent to any permutation; planner picks cheapest.

-- Mixed: the LEFT JOIN pins b to the left of c relative to a
SELECT ...
FROM a
LEFT JOIN b ON b.a_id = a.id
INNER JOIN c ON c.b_id = b.id;
-- The planner CANNOT freely float 'c' above the outer join without
-- proving it preserves semantics. Fewer orderings are legal.
```

**The classic correctness trap** (from Topic 12, now amplified in multi-join): a predicate on an outer-joined table placed in `WHERE` nullifies the outerness.

```sql
-- WRONG: intends "all users, with their orders if any, plus product info"
SELECT u.name, o.id, p.name
FROM users u
LEFT JOIN orders o    ON o.user_id = u.id
LEFT JOIN order_items oi ON oi.order_id = o.id
LEFT JOIN products p  ON p.id = oi.product_id
WHERE p.category_id = 5;      -- ← BUG: users with no orders have p.* = NULL,
                              --   NULL = 5 is not TRUE, so they're DROPPED.
                              --   The whole chain of LEFT JOINs collapses to INNER.

-- RIGHT: put the filter in the ON clause of the relevant join
SELECT u.name, o.id, p.name
FROM users u
LEFT JOIN orders o    ON o.user_id = u.id
LEFT JOIN order_items oi ON oi.order_id = o.id
LEFT JOIN products p  ON p.id = oi.product_id AND p.category_id = 5;
-- Now the category filter is part of the join match; unmatched rows
-- keep NULLs on the right and users without orders survive.
```

**Rule:** in a mixed-join query, a filter on the *nullable* (right) side of an OUTER join belongs in that join's `ON` clause, not in `WHERE` — unless you *intend* to convert it to an inner semantics.

### 6.7 Join Order and Row Multiplication (Fan-Out Compounds)

Every one-to-many join multiplies rows (Topic 11 §6.2). In a multi-join, **fan-outs compound multiplicatively.**

```sql
-- users → orders (1:many) → order_items (1:many) → products (many:1)
SELECT u.id, COUNT(*)
FROM users u
JOIN orders o        ON o.user_id = u.id     -- avg 5 orders/user
JOIN order_items oi  ON oi.order_id = o.id   -- avg 4 items/order
JOIN products p      ON p.id = oi.product_id -- 1 product/item (no fanout)
GROUP BY u.id;
-- One user with 5 orders × 4 items = 20 rows collapsed by GROUP BY.
-- Across 1M users → ~20M intermediate rows before aggregation.
```

If you then add a *second* many-branch off `orders` (say `payments`), the two branches **cross-multiply**:

```sql
-- DANGER: two independent one-to-many branches off orders
SELECT o.id, SUM(oi.quantity) AS units, SUM(pay.amount) AS paid
FROM orders o
JOIN order_items oi ON oi.order_id = o.id   -- 4 items
JOIN payments pay   ON pay.order_id = o.id  -- 2 payment rows (e.g., split pay)
GROUP BY o.id;
-- Each order now produces 4 × 2 = 8 rows.
-- SUM(oi.quantity) is inflated 2× (each item counted per payment)
-- SUM(pay.amount) is inflated 4× (each payment counted per item)
-- BOTH aggregates are WRONG.
```

The fix is to **pre-aggregate each branch separately** before joining, so neither fan-out touches the other:

```sql
SELECT o.id,
       items.units,
       pays.paid
FROM orders o
JOIN (
  SELECT order_id, SUM(quantity) AS units
  FROM order_items GROUP BY order_id
) items ON items.order_id = o.id
JOIN (
  SELECT order_id, SUM(amount) AS paid
  FROM payments GROUP BY order_id
) pays ON pays.order_id = o.id;
-- Each subquery yields exactly ONE row per order → no cross-multiplication.
```

This "aggregate-then-join" pattern is the single most important structural technique for correct multi-join reporting queries.

### 6.8 Star-Schema Joins (One Fact, Many Dimensions)

A very common shape is the **star join**: one large fact table joined to several small dimension tables.

```sql
SELECT
  d.name       AS department,
  cat.name     AS category,
  DATE_TRUNC('month', o.created_at) AS month,
  SUM(oi.quantity * oi.unit_price)  AS revenue
FROM order_items oi                             -- FACT: 50M rows
JOIN orders o        ON o.id = oi.order_id       -- dimension-ish: 10M
JOIN products p      ON p.id = oi.product_id     -- dimension: 50K
JOIN categories cat  ON cat.id = p.category_id   -- dimension: 200
JOIN users u         ON u.id = o.user_id         -- dimension: 1M
JOIN departments d   ON d.id = u.department_id   -- dimension: 50
GROUP BY d.name, cat.name, DATE_TRUNC('month', o.created_at);
```

For star joins the optimal strategy is almost always: **hash each small dimension into memory once, then scan the fact table a single time, probing each hash.** The planner recognizes this and produces a left-deep tree of Hash Joins with the fact table as the outermost scan. Because dimensions are tiny, all their hash tables fit in `work_mem` simultaneously. This is why star schemas perform well even at massive fact-table sizes — one sequential pass over the fact, cheap probes into RAM-resident dimensions.

### 6.9 Join Chains vs. Join Stars — Different Cost Profiles

- **Chain** (`a→b→c→d`, each linked only to its neighbor): the planner has limited freedom; row estimates propagate along the chain and estimation errors compound.
- **Star** (all dimensions link to one central fact): highly parallel-friendly, cheap, predictable.
- **Clique** (every table joins every other): rare, expensive, the DP search space is largest.

Recognizing which shape you have tells you whether to worry about join order (chains and cliques: yes; stars: rarely).

### 6.10 When the Planner Gets the Order Wrong

The planner picks join order from **estimated** row counts. It gets it wrong when estimates are wrong. The usual culprits:

1. **Stale statistics** — you bulk-loaded 10M rows and never ran `ANALYZE`. The planner thinks the table has 1,000 rows and picks a Nested Loop that becomes catastrophic. *Fix: `ANALYZE table;` or `VACUUM ANALYZE`.*

2. **Correlated columns** — the planner assumes `WHERE status='completed' AND country='US'` selectivities are independent and multiplies them. If they're correlated, the real result is far larger/smaller than estimated. *Fix: `CREATE STATISTICS` (extended statistics) on the correlated columns.*

3. **Cross-table correlation the planner can't see** — the planner has per-table stats but limited cross-join correlation info; a join whose result is much smaller/larger than the product of selectivities.

4. **Skewed data** — one `product_id` accounts for 40% of `order_items`. Average-based estimates mislead. *Fix: raise `default_statistics_target` for that column so the MCV list captures the skew.*

5. **Type mismatches / functions on join keys** — `ON a.id = b.id::text` defeats index usage and skews estimates.

The diagnostic signature is always the same in `EXPLAIN ANALYZE`: a **large gap between `rows=` (estimated) and `actual rows=`**, usually on a join node deep in the tree, and a Nested Loop chosen where a Hash Join was warranted (or vice versa). §7 walks through reading this.

### 6.11 Diagnosing With EXPLAIN

The workflow for a slow multi-join:

1. `EXPLAIN (ANALYZE, BUFFERS)` the query.
2. Find the node where **estimated vs. actual rows diverge most** — that's where the planner's model broke.
3. Check the **join order** (read the tree inside-out; the innermost/most-indented node runs first).
4. Look for **Nested Loop with a high `loops=`** over a non-trivial inner scan — the sign of a bad order (planner thought the outer side was tiny).
5. Check for **`Rows Removed by Filter`** — rows produced then thrown away, meaning a filter that should have been pushed earlier or an index that's missing.
6. Confirm statistics freshness (`pg_stat_user_tables.last_analyze`).

### 6.12 Controlling the Planner When It's Wrong

In order of preference:

1. **`ANALYZE`** — fix the statistics first. 80% of "wrong order" bugs vanish here.
2. **Extended statistics** (`CREATE STATISTICS ... (dependencies, ndistinct) ON ...`) — for correlated columns.
3. **Raise `default_statistics_target`** on skewed columns and re-`ANALYZE`.
4. **Restructure the query** — pre-aggregate branches (§6.7), or wrap a subquery to fence a subplan.
5. **`SET join_collapse_limit = 1`** and hand-write the order (last resort, but deterministic).
6. **`pg_hint_plan` extension** — real hints (`/*+ Leading(...) HashJoin(...) */`) if you must pin a plan in production. Not core PostgreSQL.
7. **Session GUCs** like `SET enable_nestloop = off` as a *diagnostic* to see if the planner has a better alternative it's mis-costing (never leave this in prod globally).

---

## 7. EXPLAIN — Multi-Join Order in the Plan

### 7.1 A Good 4-Way Star Join

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT u.name, o.total_amount, p.name AS product, cat.name AS category
FROM order_items oi
JOIN orders o       ON o.id = oi.order_id
JOIN users u        ON u.id = o.user_id
JOIN products p     ON p.id = oi.product_id
JOIN categories cat ON cat.id = p.category_id
WHERE o.status = 'completed'
  AND o.created_at >= CURRENT_DATE - INTERVAL '7 days';
```

```
Hash Join  (cost=4210.00..98230.00 rows=41000 width=96)
           (actual time=18.2..212.4 rows=39820 loops=1)
  Hash Cond: (p.category_id = cat.id)
  ->  Hash Join  (cost=4180.00..92110.00 rows=41000 width=72)
                 (actual time=17.9..198.1 rows=39820 loops=1)
        Hash Cond: (oi.product_id = p.id)
        ->  Hash Join  (cost=1980.00..84120.00 rows=41000 width=48)
                       (actual time=9.1..170.3 rows=39820 loops=1)
              Hash Cond: (o.user_id = u.id)
              ->  Hash Join  (cost=210.00..80100.00 rows=41000 width=32)
                             (actual time=1.2..150.7 rows=39820 loops=1)
                    Hash Cond: (oi.order_id = o.id)
                    ->  Seq Scan on order_items oi
                          (cost=0.00..70000.00 rows=5000000 width=16)
                          (actual time=0.01..90.2 rows=5000000 loops=1)
                    ->  Hash  (cost=190.00..190.00 rows=8200 width=24)
                          (actual time=1.1..1.1 rows=8200 loops=1)
                          Buckets: 16384  Batches: 1  Memory Usage: 520kB
                          ->  Index Scan using orders_status_created_idx on orders o
                                (cost=0.43..190.00 rows=8200 width=24)
                                Index Cond: (created_at >= CURRENT_DATE - '7 days')
                                Filter: (status = 'completed')
                                (actual rows=8200 loops=1)
              ->  Hash  (cost=1200.00..1200.00 rows=1000000 width=24)
                    (actual time=7.5..7.5 rows=1000000 loops=1)
                    Buckets: 131072  Batches: 1  Memory Usage: 58MB
                    ->  Seq Scan on users u  (actual rows=1000000 loops=1)
        ->  Hash  (cost=1400.00..1400.00 rows=50000 width=28)
              Buckets: 65536  Batches: 1  Memory Usage: 3400kB
              ->  Seq Scan on products p  (actual rows=50000 loops=1)
  ->  Hash  (cost=15.00..15.00 rows=200 width=28)
        Buckets: 1024  Batches: 1  Memory Usage: 20kB
        ->  Seq Scan on categories cat  (actual rows=200 loops=1)
Planning Time: 1.8 ms
Execution Time: 216.9 ms
```

**Reading it (inside-out = execution order):**

- The **innermost** join is `order_items ⋈ orders`. The planner used the `orders_status_created_idx` to pre-filter orders to 8,200 rows (last 7 days, completed), hashed those, and probed with the 5M-row `order_items` seq scan. Good: the *selective* side (orders) is the hash build side.
- Estimated `rows=41000` vs actual `rows=39820` at every join level — **estimates are excellent**, so the planner's order is trustworthy.
- **Four Hash Joins stacked left-deep**, each adding one dimension. Each `Hash` node shows `Batches: 1` → no disk spill; every hash table fit in `work_mem`.
- Total memory across hash tables: 58MB (users) + 3.4MB (products) + 520kB + 20kB ≈ **62MB in this one query**. Multiply by concurrency for capacity planning.
- `Execution Time: 216.9 ms` scanning 5M order_items once — this is a healthy star join.

### 7.2 The Same Query Gone Wrong (Stale Stats)

Suppose `order_items` was just bulk-loaded and never analyzed; the planner thinks it has 1,000 rows:

```
Nested Loop  (cost=0.85..15200.00 rows=1000 width=96)
             (actual time=0.3..41200.5 rows=39820 loops=1)   ← 41 SECONDS
  ->  Nested Loop  (cost=0.85..9100.00 rows=1000 width=72)
                   (actual time=0.2..38500.1 rows=39820 loops=1)
        ->  Seq Scan on order_items oi
              (cost=0.00..70.00 rows=1000 width=16)     ← estimate 1000
              (actual rows=5000000 loops=1)             ← ACTUAL 5,000,000
        ->  Index Scan using orders_pkey on orders o
              Index Cond: (id = oi.order_id)
              (actual rows=1 loops=5000000)             ← 5M index probes!
  ...
Execution Time: 41205.7 ms
```

**Diagnostic reading:**
- `order_items` estimate `rows=1000` vs actual `5,000,000` — **5000× underestimate**, the smoking gun.
- Because it thought `order_items` was tiny, the planner chose **Nested Loop**: for each order_items row, probe `orders` by PK. `loops=5000000` — five million index probes. Each is fast individually, but five million of them = 41 seconds.
- The fix is not a hint. It's `ANALYZE order_items;` — after which the estimate corrects to 5M, the planner flips to the Hash Join plan from §7.1, and runtime drops from 41s to 217ms.

This is the textbook "planner got the order wrong" signature: **massive estimate/actual divergence on a scan → wrong algorithm → catastrophic loop count.**

### 7.3 What to Look For

| Symptom in EXPLAIN | Likely Cause | Fix |
|---|---|---|
| Estimated rows ≪ actual rows on a scan | Stale statistics | `ANALYZE` |
| Nested Loop with `loops` in the millions | Planner underestimated outer side | `ANALYZE`; check index; verify order |
| Estimated ≪ actual on a join (not scan) | Correlated columns / cross-table correlation | Extended statistics |
| `Batches: N` where N > 1 on a Hash node | Hash table spilled to disk | Raise `work_mem` |
| `Rows Removed by Filter` very high | Filter not pushed down / missing index | Add index; move filter to ON |
| Long `Planning Time` on a big join | Collapse limit raised too high / GEQO off | Lower limit or re-enable GEQO |
| Different plan each run for a 12+ table join | GEQO non-determinism | `SET geqo=off` or raise `geqo_threshold` |

---

## 8. Query Examples

### Example 1 — Basic: A Straightforward 4-Table Chain

```sql
-- Hydrate an order-line view: who ordered what, in which category.
SELECT
  u.name          AS customer,
  o.id            AS order_id,
  o.created_at    AS order_date,
  p.name          AS product,
  cat.name        AS category,
  oi.quantity,
  oi.unit_price
FROM users u
INNER JOIN orders o       ON o.user_id = u.id
INNER JOIN order_items oi ON oi.order_id = o.id
INNER JOIN products p     ON p.id = oi.product_id
INNER JOIN categories cat ON cat.id = p.category_id
WHERE o.status = 'completed'
ORDER BY o.created_at DESC
LIMIT 100;
-- 5 tables, 4 joins. All INNER → planner picks its own order freely.
```

### Example 2 — Intermediate: Mixing INNER and LEFT Correctly

```sql
-- Every completed order, with its payment info IF a payment exists,
-- and the handling employee IF one was assigned.
-- Orders without payments or employees must STILL appear.
SELECT
  o.id                         AS order_id,
  u.name                       AS customer,
  o.total_amount,
  pay.method                   AS payment_method,   -- may be NULL
  pay.amount                   AS paid_amount,       -- may be NULL
  e.name                       AS handled_by         -- may be NULL
FROM orders o
INNER JOIN users u        ON u.id = o.user_id           -- every order HAS a user
LEFT  JOIN payments pay   ON pay.order_id = o.id
                          AND pay.status = 'captured'    -- filter in ON (preserves LEFT)
LEFT  JOIN employees e    ON e.id = o.handled_by
WHERE o.status = 'completed'                             -- filter on INNER side: safe in WHERE
ORDER BY o.id;
-- Note: pay.status filter is in ON, not WHERE — so orders with no
-- captured payment keep NULLs instead of being dropped.
```

### Example 3 — Production Grade: Multi-Branch Report with Pre-Aggregation

```sql
-- Monthly departmental sales report.
-- Tables (sizes at time of writing):
--   order_items  50M rows
--   orders       10M rows
--   users         1M rows, indexed on department_id
--   departments  50 rows
--   payments     12M rows  (independent one-to-many off orders)
-- Indexes assumed:
--   orders(status, created_at), orders(user_id)
--   order_items(order_id), payments(order_id, status)
--   users(department_id)
-- Perf expectation: single seq pass over order_items, ~1.5s cold, ~400ms warm.
--
-- We have TWO one-to-many branches off orders (order_items and payments).
-- Joining both directly would cross-multiply and corrupt both SUMs (§6.7).
-- So we pre-aggregate each branch to exactly one row per order.

WITH item_rollup AS (
  SELECT
    oi.order_id,
    SUM(oi.quantity)                    AS units,
    SUM(oi.quantity * oi.unit_price)    AS gross_revenue
  FROM order_items oi
  GROUP BY oi.order_id
),
payment_rollup AS (
  SELECT
    p.order_id,
    SUM(p.amount) FILTER (WHERE p.status = 'captured') AS captured_amount
  FROM payments p
  GROUP BY p.order_id
)
SELECT
  d.name                                        AS department,
  DATE_TRUNC('month', o.created_at)             AS month,
  COUNT(DISTINCT o.id)                          AS orders,
  SUM(ir.units)                                 AS units_sold,
  SUM(ir.gross_revenue)                         AS gross_revenue,
  SUM(pr.captured_amount)                       AS collected,
  ROUND(SUM(pr.captured_amount)
        / NULLIF(SUM(ir.gross_revenue), 0) * 100, 2) AS collection_pct
FROM orders o
INNER JOIN users u          ON u.id = o.user_id
INNER JOIN departments d    ON d.id = u.department_id
INNER JOIN item_rollup ir   ON ir.order_id = o.id     -- one row per order: no fanout
LEFT  JOIN payment_rollup pr ON pr.order_id = o.id     -- LEFT: unpaid orders still count
WHERE o.status = 'completed'
  AND o.created_at >= DATE_TRUNC('month', CURRENT_DATE - INTERVAL '12 months')
GROUP BY d.name, DATE_TRUNC('month', o.created_at)
ORDER BY department, month;
```

```
-- EXPLAIN (ANALYZE, BUFFERS) abridged:
GroupAggregate  (cost=... rows=600 width=...) (actual time=1380..1420 rows=600 loops=1)
  ->  Sort (Sort Key: d.name, date_trunc(...))
        ->  Hash Left Join  (Hash Cond: (o.id = pr.order_id))
              ->  Hash Join  (Hash Cond: (o.id = ir.order_id))
                    ->  Hash Join  (Hash Cond: (u.department_id = d.id))
                          ->  Hash Join  (Hash Cond: (o.user_id = u.id))
                                ->  Index Scan on orders  (actual rows=920000 loops=1)
                                      Index Cond: (created_at >= ...) Filter: status='completed'
                                ->  Hash  ->  Seq Scan on users (rows=1000000)
                          ->  Hash  ->  Seq Scan on departments (rows=50)
                    ->  Hash  ->  Subquery Scan item_rollup
                                    ->  HashAggregate (rows=10000000)
                                          ->  Seq Scan on order_items (rows=50000000)
              ->  Hash  ->  Subquery Scan payment_rollup
                              ->  HashAggregate (rows=10000000)
                                    ->  Index Scan on payments
Planning Time: 3.1 ms
Execution Time: 1421.6 ms
```

**Why it's fast:** each rollup CTE collapses its branch to one row per order *before* the main join, so no cross-multiplication occurs; the fact-heavy scans (`order_items`, `payments`) happen exactly once each inside their aggregates; and the top-level joins are all against small or pre-reduced relations.

---

## 9. Wrong → Right Patterns

### Wrong 1: Filter on a LEFT-joined table in WHERE (silent INNER-ification)

```sql
-- WRONG: "all users and their orders, but only electronics orders"
SELECT u.name, o.id, p.name
FROM users u
LEFT JOIN orders o        ON o.user_id = u.id
LEFT JOIN order_items oi  ON oi.order_id = o.id
LEFT JOIN products p      ON p.id = oi.product_id
WHERE p.category_id = 5;
-- WRONG RESULT: users with no orders (or no electronics) VANISH.
-- WHY: for a user with no orders, p.category_id is NULL after the LEFT JOINs.
--      NULL = 5 → UNKNOWN → row filtered out. The LEFT JOINs collapse to INNER.

-- RIGHT: move the filter into the ON clause of the products join
SELECT u.name, o.id, p.name
FROM users u
LEFT JOIN orders o        ON o.user_id = u.id
LEFT JOIN order_items oi  ON oi.order_id = o.id
LEFT JOIN products p      ON p.id = oi.product_id AND p.category_id = 5;
-- Now every user survives; non-electronics rows just carry NULLs on the right.
```

### Wrong 2: Two one-to-many branches cross-multiplying aggregates

```sql
-- WRONG: sum items and payments in one flat join
SELECT o.id,
       SUM(oi.quantity)  AS units,
       SUM(pay.amount)   AS paid
FROM orders o
JOIN order_items oi ON oi.order_id = o.id   -- 4 rows/order
JOIN payments pay   ON pay.order_id = o.id  -- 2 rows/order
GROUP BY o.id;
-- WRONG RESULT: each order yields 4×2=8 rows.
--   SUM(oi.quantity) counted 2× too high, SUM(pay.amount) counted 4× too high.
-- WHY: the two independent fan-outs form a Cartesian product per order.

-- RIGHT: pre-aggregate each branch to one row per order, then join
SELECT o.id, i.units, p.paid
FROM orders o
JOIN (SELECT order_id, SUM(quantity) AS units
      FROM order_items GROUP BY order_id) i ON i.order_id = o.id
JOIN (SELECT order_id, SUM(amount) AS paid
      FROM payments   GROUP BY order_id) p ON p.order_id = o.id;
-- Each subquery is 1 row/order → no cross-multiplication, both sums correct.
```

### Wrong 3: Crossing `join_collapse_limit` with a bad written order

```sql
-- WRONG: 10 explicit joins, written worst-first (two huge tables joined first),
-- with join_collapse_limit at its default of 8.
SET join_collapse_limit = 8;  -- (the default)
SELECT ...
FROM order_items oi                        -- 50M
JOIN orders o        ON o.id = oi.order_id -- 10M   ← written as the first join: huge×huge
JOIN payments pay    ON pay.order_id = o.id
JOIN refunds r       ON r.payment_id = pay.id
JOIN users u         ON u.id = o.user_id
JOIN departments d   ON d.id = u.department_id
JOIN products p      ON p.id = oi.product_id
JOIN categories cat  ON cat.id = p.category_id
JOIN suppliers s     ON s.id = p.supplier_id
JOIN warehouses w    ON w.id = s.warehouse_id
WHERE d.name = 'Retail';
-- With 10 items > limit 8, the planner stops collapsing and partially honors
-- the written order — starting from the 50M×10M join. Intermediate result is enormous.
-- EXPLAIN shows a giant early Hash Join and multi-second runtime.

-- RIGHT: let the planner see the whole problem so it can start from the
-- selective WHERE (d.name='Retail' → tiny) and expand outward.
SET join_collapse_limit = 12;
SET from_collapse_limit = 12;
-- Same query text; now the planner explores all orderings, filters
-- departments first, and the plan is orders-of-magnitude cheaper.
-- (Verify Planning Time stays small; if it balloons, cap the limit.)
```

### Wrong 4: GEQO non-determinism blamed on "random slowness"

```sql
-- WRONG diagnosis: "the 14-table nightly report is randomly slow, must be I/O."
-- Reality: 14 join items > geqo_threshold(12) → GEQO evolves a DIFFERENT
-- (sometimes worse) join order some nights. Runtime swings 4× with no data change.

-- RIGHT: for a batch report where planning time is irrelevant, force exact search:
SET geqo = off;                 -- or: SET geqo_threshold = 20;
-- Now the planner uses exhaustive DP every run → deterministic, optimal order.
-- Planning takes ~50ms more, execution is stable and often faster.
```

### Wrong 5: Function/type mismatch on a join key defeating the index and the estimate

```sql
-- WRONG: joining on a cast, killing index usage and skewing selectivity
SELECT u.name, o.id
FROM users u
JOIN orders o ON o.user_id::text = u.id::text   -- casts both sides to text
JOIN payments p ON p.order_id = o.id;
-- WHY WRONG: the ::text cast prevents use of the integer index on orders(user_id),
-- forces a Seq Scan, and gives the planner a poor row estimate for the join,
-- which then cascades into a bad order for the payments join.

-- RIGHT: join on native types so indexes and stats apply
SELECT u.name, o.id
FROM users u
JOIN orders o   ON o.user_id = u.id
JOIN payments p ON p.order_id = o.id;
```

---

## 10. Performance Profile

### 10.1 Cost Drivers Unique to Multi-Joins

| Factor | Effect |
|---|---|
| Number of tables N | Planning cost grows ~`O(2^N)` (DP) until GEQO kicks in |
| Join order | Can change execution time by 10–1000× |
| Fan-out compounding | Intermediate rows can explode multiplicatively |
| Number of Hash Joins | Each allocates up to `work_mem`; N−1 joins → up to N−1 hash tables |
| Collapse limits | Above them, planner honors written order (good or bad) |
| GEQO threshold | Above it, order is heuristic and possibly non-deterministic |

### 10.2 Scaling at 1M / 10M / 100M Rows

Consider a 5-way star join (fact + 4 small dimensions), fact table at each scale, dimensions constant and RAM-resident:

| Fact rows | Plan | Typical warm runtime | Bottleneck |
|---|---|---|---|
| 1M | Hash join star, 1 fact scan | 30–80 ms | CPU (hash probes) |
| 10M | Hash join star, 1 fact scan | 300–900 ms | fact seq scan I/O + CPU |
| 100M | Hash join star, parallel seq scan | 2–8 s | fact I/O; benefits hugely from parallelism |

For a 5-way **chain** join (each table linked only to its neighbor) with fan-out, runtimes are far more sensitive to order and can vary from seconds to "never finishes" at 100M if the planner mis-orders.

### 10.3 Memory Accounting

Each Hash Join builds a hash table sized up to `work_mem`. A 6-table query with 5 Hash Joins can, worst case, hold **5 hash tables at once** → up to `5 × work_mem`. With `work_mem = 256MB` that's 1.25GB **per query execution**, and PostgreSQL allocates `work_mem` **per node, per parallel worker**. At 20 concurrent executions with 4 parallel workers each, the math is frightening. Mitigations:

- Keep dimension hash tables small (they usually are).
- Watch for `Batches > 1` (spill to disk) — either raise `work_mem` for that query via `SET LOCAL`, or reduce the build side.
- Prefer pre-aggregation to shrink intermediate relations before they hit a hash join.

### 10.4 Index Interactions

- **Every join key on the "probed"/inner side wants an index** (for Nested Loop) — especially the fact→dimension foreign keys.
- **The most selective WHERE predicate wants an index** so the planner can start the join tree from the smallest relation. In §7.1, `orders(status, created_at)` is what let the planner begin from 8,200 orders instead of 10M.
- **Covering indexes** (`INCLUDE`) can turn an Index Scan + heap fetch into an index-only scan, cutting buffers on the driving table.
- **Foreign-key indexes are not automatic in PostgreSQL** — declaring a FK does *not* create an index on the referencing column. For multi-joins this is a frequent miss: `order_items(order_id)`, `orders(user_id)`, `products(category_id)` all need explicit indexes.

### 10.5 Optimization Techniques Specific to Multi-Joins

1. **Start the tree from the most selective relation** — ensure it has an index for its filter.
2. **Pre-aggregate one-to-many branches** before joining (correctness *and* speed).
3. **Tune `join_collapse_limit`/`from_collapse_limit`** for large analytical queries.
4. **Manage GEQO** for 12+ table queries (disable for batch, keep for OLTP).
5. **Fix statistics first** (`ANALYZE`, extended statistics, higher `default_statistics_target`).
6. **Enable parallelism** for big fact scans (`max_parallel_workers_per_gather`).
7. **Materialize** repeated multi-join subresults into a table/materialized view if run often.

---

## 11. Node.js Integration

### 11.1 A Parameterized Multi-Join Endpoint

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Order detail: order + customer + line items + product + category
async function getOrderDetail(orderId) {
  const { rows } = await pool.query(
    `SELECT
       o.id            AS order_id,
       o.created_at    AS order_date,
       u.name          AS customer,
       u.email,
       p.name          AS product,
       cat.name        AS category,
       oi.quantity,
       oi.unit_price,
       oi.quantity * oi.unit_price AS line_total
     FROM orders o
     INNER JOIN users u        ON u.id = o.user_id
     INNER JOIN order_items oi ON oi.order_id = o.id
     INNER JOIN products p     ON p.id = oi.product_id
     INNER JOIN categories cat ON cat.id = p.category_id
     WHERE o.id = $1
     ORDER BY oi.id`,
    [orderId]
  );
  return rows;   // one row per line item; multiple rows per order
}
```

### 11.2 Pre-Aggregated Report (Avoiding Fan-Out in App Code)

```javascript
// Monthly department report — pushes the aggregation into SQL (CTEs),
// so the app receives one clean row per (department, month).
async function getDepartmentMonthlyReport(monthsBack = 12) {
  const { rows } = await pool.query(
    `WITH item_rollup AS (
       SELECT order_id,
              SUM(quantity)                 AS units,
              SUM(quantity * unit_price)    AS gross_revenue
       FROM order_items GROUP BY order_id
     )
     SELECT d.name                              AS department,
            DATE_TRUNC('month', o.created_at)   AS month,
            COUNT(DISTINCT o.id)                AS orders,
            SUM(ir.units)                       AS units_sold,
            SUM(ir.gross_revenue)               AS gross_revenue
     FROM orders o
     INNER JOIN users u        ON u.id = o.user_id
     INNER JOIN departments d  ON d.id = u.department_id
     INNER JOIN item_rollup ir ON ir.order_id = o.id
     WHERE o.status = 'completed'
       AND o.created_at >= DATE_TRUNC('month', CURRENT_DATE - ($1 || ' months')::INTERVAL)
     GROUP BY d.name, DATE_TRUNC('month', o.created_at)
     ORDER BY department, month`,
    [monthsBack]
  );
  return rows;
}
```

### 11.3 Tuning the Planner Per-Query From Node

```javascript
// For a known-heavy 12+ table analytical query, pin planner behavior
// inside a single transaction so SET LOCAL scopes only to this work.
async function runHeavyAnalytics(client, params) {
  await client.query('BEGIN');
  try {
    await client.query('SET LOCAL join_collapse_limit = 12');
    await client.query('SET LOCAL from_collapse_limit = 12');
    await client.query('SET LOCAL geqo = off');           // deterministic, optimal order
    await client.query('SET LOCAL work_mem = \'256MB\'');  // avoid hash spill this query only
    const { rows } = await client.query(HEAVY_ANALYTICS_SQL, params);
    await client.query('COMMIT');
    return rows;
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  }
}
// Use a dedicated client from the pool so SET LOCAL doesn't leak:
//   const client = await pool.connect();
//   try { return await runHeavyAnalytics(client, params); }
//   finally { client.release(); }
```

**Note on the multi-row hydration problem:** because a multi-join with one-to-many relations returns one row per leaf (e.g., one row per order item), your Node code must **regroup** rows into nested objects (order → items[]). Doing that reshaping in JS is fine for small result sets; for large ones, prefer aggregating into JSON in SQL (`json_agg`) so the wire transfers one row per order. Most ORMs' `include`/eager-loading exists precisely to hide this reshaping — but they often issue N+1 queries to do it (Topic 18).

---

## 12. ORM Comparison

### Prisma

**Can Prisma do multi-table joins?** — Yes, via nested `include`/`select`. Prisma models relationships, so a 4-level join is expressed as nested includes. Historically Prisma issued **separate queries per relation** (a controlled batch, not N+1) and stitched them in the client; the newer relation-join mode (`relationLoadStrategy: "join"`) emits a single SQL statement with `LATERAL` joins and JSON aggregation.

```typescript
const orders = await prisma.order.findMany({
  where: { status: 'completed' },
  include: {
    user: true,                                   // orders → users
    orderItems: {                                 // orders → order_items
      include: {
        product: { include: { category: true } }, // → products → categories
      },
    },
  },
  // Prisma 5+: force a single joined SQL statement instead of batched queries:
  relationLoadStrategy: 'join',
});
```

**Where it breaks:** join *order* is entirely the DB planner's job — Prisma exposes no control over `join_collapse_limit`, GEQO, or join algorithm. Multi-branch aggregations that need pre-aggregation (§6.7) cannot be expressed relationally without producing fan-out; you must use `$queryRaw`. Explicit INNER vs LEFT per relation is inferred from FK nullability, not chosen.

**Verdict:** Great for hydrating nested resources; use `relationLoadStrategy: 'join'` to avoid multi-query round trips. Drop to `$queryRaw` for any analytical multi-join with aggregation or planner tuning.

---

### Drizzle ORM

**Can Drizzle do multi-table joins?** — Yes, fully and explicitly. You chain `.innerJoin()` / `.leftJoin()` in the exact order you want, mirroring SQL.

```typescript
import { db } from './db';
import { orders, users, orderItems, products, categories } from './schema';
import { eq, and } from 'drizzle-orm';

const rows = await db
  .select({
    orderId: orders.id,
    customer: users.name,
    product: products.name,
    category: categories.name,
    quantity: orderItems.quantity,
  })
  .from(orders)
  .innerJoin(users,      eq(users.id, orders.userId))
  .innerJoin(orderItems, eq(orderItems.orderId, orders.id))
  .innerJoin(products,   eq(products.id, orderItems.productId))
  .innerJoin(categories, eq(categories.id, products.categoryId))
  .where(and(eq(orders.status, 'completed')));
```

**Where it breaks:** Drizzle emits the joins in written order, but the *planner* still reorders INNER joins — Drizzle does not (and cannot portably) set `join_collapse_limit`; run `db.execute(sql\`SET LOCAL ...\`)` yourself. Mixing INNER/LEFT and the WHERE-vs-ON trap is on you to get right, exactly as in raw SQL. Pre-aggregation uses subquery builders (`db.$with` / `.as()`).

**Verdict:** The closest ORM to hand-written SQL for multi-joins — full control, full type safety. Best choice when you care about the exact query shape.

---

### Sequelize

**Can Sequelize do multi-table joins?** — Yes, via nested `include`. **Default is LEFT OUTER JOIN**; set `required: true` per include for INNER.

```javascript
const orders = await Order.findAll({
  where: { status: 'completed' },
  include: [
    { model: User, required: true },                        // INNER JOIN
    {
      model: OrderItem, required: true,                     // INNER JOIN
      include: [
        {
          model: Product, required: true,
          include: [{ model: Category, required: true }],   // nested INNER JOIN
        },
      ],
    },
  ],
});
```

**Where it breaks:** the `required` default (LEFT) means forgetting it in a 4-level nest silently changes your semantics and row counts. Deeply nested includes generate large, sometimes surprising SQL, and `separate: true` on a hasMany flips a branch into a *separate query* (N+1-ish) — great for avoiding fan-out but easy to misuse. Aggregations across a multi-join need `sequelize.query()`.

**Verdict:** Workable for CRUD-style nested loads; audit the generated SQL and always be explicit with `required`. Use raw queries for reports.

---

### TypeORM

**Can TypeORM do multi-table joins?** — Yes, via QueryBuilder `innerJoin`/`leftJoin` (+ `...AndSelect` to hydrate).

```typescript
const rows = await dataSource
  .getRepository(Order)
  .createQueryBuilder('o')
  .innerJoinAndSelect('o.user', 'u')
  .innerJoinAndSelect('o.orderItems', 'oi')
  .innerJoinAndSelect('oi.product', 'p')
  .innerJoinAndSelect('p.category', 'cat')
  .where('o.status = :status', { status: 'completed' })
  .getMany();
```

**Where it breaks:** `innerJoinAndSelect` on a one-to-many (`o.orderItems`) triggers TypeORM's fan-out de-duplication in memory, which interacts badly with `.take()/.skip()` pagination (it paginates *rows*, not *entities*, unless you use the split-query mode). No planner-order control. Raw string conditions lose type safety.

**Verdict:** Fine for entity graphs; beware pagination + one-to-many. Use `.getRawMany()` or `dataSource.query()` for analytical multi-joins.

---

### Knex.js

**Can Knex do multi-table joins?** — Yes, transparently. `.join()` = INNER; `.leftJoin()` = LEFT. You write the order; the planner still reorders INNER joins.

```javascript
const rows = await knex('orders as o')
  .innerJoin('users as u',        'u.id',  'o.user_id')
  .innerJoin('order_items as oi', 'oi.order_id', 'o.id')
  .innerJoin('products as p',     'p.id',  'oi.product_id')
  .innerJoin('categories as c',   'c.id',  'p.category_id')
  .where('o.status', 'completed')
  .select('o.id as order_id', 'u.name as customer',
          'p.name as product', 'c.name as category', 'oi.quantity');

// Planner tuning is just another statement on the same connection:
await knex.raw('SET LOCAL join_collapse_limit = 12');
```

**Where it breaks:** Knex is a query builder, not an ORM — it won't reshape one-row-per-item into nested objects; you do that yourself (or use `json_agg`). No relationship modeling, so nested hydration is manual.

**Verdict:** Most SQL-transparent option; ideal when you want exact control over the multi-join and are comfortable owning the SQL semantics.

---

### ORM Summary Table

| ORM | Multi-Join Method | Default Join | Planner Control | Pre-Aggregation | Verdict |
|---|---|---|---|---|---|
| Prisma | nested `include` | inferred from FK nullability | none | `$queryRaw` | Great for hydration; raw for reports |
| Drizzle | chained `.innerJoin/.leftJoin` | must specify | via `sql` `SET LOCAL` | subquery builders | Closest to raw SQL |
| Sequelize | nested `include` | LEFT (danger) | none | `sequelize.query()` | Be explicit with `required` |
| TypeORM | `innerJoinAndSelect` | explicit | none | `getRawMany()` | Watch pagination + hasMany |
| Knex | chained `.join/.leftJoin` | INNER | `knex.raw('SET LOCAL...')` | manual subqueries | Most transparent |

---

## 13. Practice Exercises

### Exercise 1 — Easy

Given:
- `users(id, name, department_id)`
- `orders(id, user_id, total_amount, status, created_at)`
- `order_items(id, order_id, product_id, quantity, unit_price)`
- `products(id, name, category_id)`
- `categories(id, name)`

Write a single query that returns, for every completed order, the customer name, the product name, the category name, and the line quantity — one row per line item. Order by order id, then line item id.

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines Topics 11, 12, 20)

Using the same tables plus `payments(id, order_id, amount, status)`:

Return, for each customer, `name`, `total_orders` (distinct completed orders), `gross_revenue` (sum of quantity × unit_price across their completed orders), and `total_captured` (sum of captured payment amounts). Customers with completed orders but **no captured payments** must still appear, with `total_captured = 0`. Beware: `order_items` and `payments` are both one-to-many off `orders` — do not let them cross-multiply.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

You have 8 tables to join for a supplier scorecard: `order_items → orders → users → departments`, and `order_items → products → categories → suppliers → warehouses`. Company policy: the report must include **every supplier**, even those with zero sales in the period, and must not double-count revenue when a product appears across multiple order items.

1. Write the query returning `supplier_name`, `warehouse_name`, `distinct_products_sold`, and `revenue` for the last quarter, keeping suppliers with no sales.
2. The naive all-INNER version silently drops zero-sales suppliers and (if you also join a second one-to-many branch) inflates revenue. Explain both bugs.
3. The query has 8 join items. Given `join_collapse_limit = 8`, what could go wrong, and what would you `SET` to be safe?

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

A 14-table nightly analytics query "randomly" takes anywhere from 8 seconds to 90 seconds with no data changes. `EXPLAIN ANALYZE` shows a different join order on different runs. Write the diagnostic steps and the exact `SET` commands you would apply, and explain *why* the runtime varies. Then describe how you'd make the fix permanent without editing every call site.

```sql
-- Write your reasoning and SET commands here
```

---

## 14. Interview Questions

### Junior Level

**Q: How many join operations does a query joining 5 tables perform?**

A: Four. N tables always require N−1 pairwise joins, because PostgreSQL only ever joins two relations at a time — a 5-table query is executed as a tree of 4 binary joins, each feeding the next.

**Interviewer follow-up:** *Does the order you write the joins in determine the execution order?* — For all-INNER joins, no: the planner reorders them freely because INNER join is commutative and associative. It picks the order it estimates is cheapest. Written order only becomes binding when you exceed `join_collapse_limit` or introduce OUTER joins that constrain reordering.

---

**Q: You have `users LEFT JOIN orders LEFT JOIN products` and add `WHERE products.category_id = 5`. Some users disappeared. Why?**

A: Putting a predicate on the nullable side of a LEFT JOIN in the `WHERE` clause converts it to an INNER JOIN. For a user with no orders, `products.category_id` is NULL after the outer joins; `NULL = 5` is UNKNOWN, so the row is filtered out. The fix is to move `category_id = 5` into the `ON` clause of the products join, so unmatched rows survive with NULLs.

**Interviewer follow-up:** *Which columns are safe to filter in WHERE?* — Columns from tables on the **preserved (left) side** of every outer join above them. A filter on the driving/INNER-side table is always safe in WHERE.

---

**Q: What does `INNER JOIN` vs `LEFT JOIN` change about which tables the planner may reorder?**

A: INNER joins can be reordered arbitrarily (commutative + associative), giving the planner full freedom. OUTER joins are not fully commutative/associative, so they act as partial barriers — the planner can only reorder within the constraints that preserve outer-join semantics, shrinking the search space.

---

### Principal Level

**Q: A 4-table join ran in 200ms yesterday and takes 40 seconds today with no code change. Walk me through diagnosing it.**

A: This is the classic stale-statistics/wrong-order failure. Steps: (1) `EXPLAIN (ANALYZE, BUFFERS)` and find the node with the biggest gap between estimated `rows=` and `actual rows=` — typically a scan whose estimate is orders of magnitude too low. (2) That underestimate makes the planner choose a Nested Loop with a huge `loops=` count instead of a Hash Join. (3) Check `pg_stat_user_tables.last_analyze` — if a bulk load or large delete happened, the stats are stale. (4) Run `ANALYZE` on the affected table; the estimate corrects and the planner flips back to the good order. (5) If the estimate is wrong due to *correlated columns* rather than staleness, add extended statistics (`CREATE STATISTICS`) or raise `default_statistics_target`. I fix the *estimates*, not pin the plan — pinning (via `join_collapse_limit=1` or `pg_hint_plan`) is a last resort because it goes stale as data evolves.

**Interviewer follow-up:** *What if `ANALYZE` doesn't fix it?* — Then the estimate error is structural (cross-table correlation the planner can't model). Options in order: extended statistics on the correlated set, restructure via pre-aggregated subqueries to give the planner smaller, better-estimated inputs, and only as a last resort force order with `join_collapse_limit=1` or hints.

---

**Q: Explain `join_collapse_limit` and `geqo_threshold`. When would you change each?**

A: `join_collapse_limit` (default 8) caps how many explicit JOIN items the planner flattens into a single reorderable list. Below it, the System-R dynamic-programming search finds the optimal order; above it, the planner stops collapsing and honors your written order for the excess — so a query with 9+ joins and a bad written order can silently get a bad plan. I raise it (per-session/transaction) for important analytical queries where planning time is negligible versus execution time. Setting it to 1 disables reordering entirely to pin a hand-chosen order. `geqo_threshold` (default 12) is the point at which the exhaustive DP search is replaced by the Genetic Query Optimizer — a heuristic, non-deterministic search that trades optimality for planning speed. For a batch report where planning cost doesn't matter, I disable GEQO (or raise the threshold) so I get a deterministic, optimal order; for OLTP with many tables I leave it on so planning stays fast.

**Interviewer follow-up:** *What's the risk of raising `join_collapse_limit` to 20?* — Planning time grows roughly `O(2^N)`; for a 20-way join, planning could take longer than executing, and it's paid on every query execution unless the plan is cached. You raise it deliberately for specific heavy queries, not globally.

---

**Q: You join `orders` to both `order_items` and `payments`, both one-to-many, and `SUM(oi.quantity)` and `SUM(pay.amount)` are both inflated. Why, and how do you fix it without changing the numbers' meaning?**

A: The two independent one-to-many branches form a Cartesian product per order: an order with 4 items and 2 payments yields 4×2 = 8 rows. Each item row is duplicated once per payment (inflating the item sum 2×) and each payment row is duplicated once per item (inflating the payment sum 4×). You cannot fix this with `DISTINCT` because the duplicated values may legitimately repeat. The correct fix is to pre-aggregate each branch to exactly one row per order in separate subqueries/CTEs, then join those one-row-per-order rollups to `orders`. Now neither fan-out touches the other and both sums are correct. This also usually runs faster because the fact tables are scanned once inside their aggregates rather than materialized as an exploded intermediate.

---

## 15. Mental Model Checkpoint

1. A query joins 6 tables. How many physical pairwise join operations does PostgreSQL perform, and why can't it do a single 6-way join?

2. You write `FROM a JOIN b JOIN c JOIN d` with all INNER joins. The planner executes them in a completely different order than written. Is your result still correct? What property guarantees that?

3. You change one of those joins to a LEFT JOIN. Explain why the planner now has fewer legal orderings to choose from.

4. A query with 9 explicit joins runs slowly. `join_collapse_limit` is 8. What is the planner doing with the 9th join, and how might that hurt you?

5. Your 13-table report produces a different `EXPLAIN` plan on two consecutive runs against identical data. What feature is responsible, and how do you make it deterministic?

6. You join `orders` to `order_items` and to `payments` (both one-to-many) and both your `SUM`s are wrong. Describe the row-level mechanics of why, and the structural fix.

7. In `EXPLAIN ANALYZE` you see a Nested Loop with `loops=8000000` and an inner scan estimated at `rows=1000` but `actual rows=5000000`. What single command most likely fixes this, and what is the planner's mistake?

---

## 16. Quick Reference Card

```sql
-- N tables → N-1 joins. Executed as a tree of PAIRWISE joins.
-- All-INNER: planner reorders freely (commutative + associative).
-- Any OUTER join: constrains reordering; written order starts to matter.

-- ── Mixing INNER + OUTER: filter placement ──
-- Filter on OUTER (nullable/right) side → put in ON, else LEFT collapses to INNER:
LEFT JOIN products p ON p.id = oi.product_id AND p.category_id = 5   -- correct
-- WHERE p.category_id = 5   ← WRONG: silently drops unmatched rows

-- ── Two one-to-many branches: PRE-AGGREGATE, don't flat-join ──
JOIN (SELECT order_id, SUM(quantity) u FROM order_items GROUP BY order_id) i USING(order_id)
JOIN (SELECT order_id, SUM(amount)   p FROM payments    GROUP BY order_id) p USING(order_id)
-- Flat-joining both cross-multiplies and corrupts BOTH sums.

-- ── Planner reordering budget ──
SHOW join_collapse_limit;   -- 8  : max JOIN items freely reordered
SHOW from_collapse_limit;   -- 8  : max subquery flattening
SET join_collapse_limit = 12;   -- explore more orderings (heavy analytics)
SET join_collapse_limit = 1;    -- DISABLE reordering; honor written order exactly

-- ── GEQO (Genetic Query Optimizer) ──
SHOW geqo_threshold;   -- 12 : at/above this many items → heuristic genetic search
SET geqo = off;              -- force exhaustive DP (deterministic, optimal, slower plan)
SET geqo_threshold = 20;     -- raise the DP ceiling

-- ── Diagnosing a wrong join order ──
EXPLAIN (ANALYZE, BUFFERS) <query>;
--   look for: estimated rows ≪ actual rows on a scan  → stale stats  → ANALYZE
--             Nested Loop with huge loops=            → underestimated outer side
--             Batches > 1 on a Hash node              → work_mem spill → raise work_mem
--             Rows Removed by Filter (high)           → missing index / push filter down
ANALYZE order_items;                         -- fix #1 for most order bugs
CREATE STATISTICS s (dependencies) ON a, b FROM t;   -- correlated columns
ALTER TABLE t ALTER COLUMN c SET STATISTICS 1000;    -- skewed column, then ANALYZE

-- ── Memory rule of thumb ──
-- Up to (N-1) hash tables live at once, each up to work_mem, per parallel worker.

-- ── Node.js: scope planner tweaks with SET LOCAL in a txn ──
-- BEGIN; SET LOCAL join_collapse_limit=12; SET LOCAL geqo=off; <query>; COMMIT;

-- ── Interview one-liners ──
-- "N tables = N-1 pairwise joins, executed as a binary tree."
-- "INNER joins reorder freely; the planner picks order from row ESTIMATES."
-- "Wrong order = bad estimates. Fix stats before pinning plans."
-- "Above join_collapse_limit, the planner honors your written order — good or bad."
-- "Above geqo_threshold(12), order is genetic & non-deterministic."
-- "Two one-to-many branches cross-multiply: pre-aggregate each first."
-- "Filter an outer-joined column in WHERE and your LEFT JOIN becomes an INNER JOIN."
```

---

## Connected Topics

- **Topic 11 — INNER JOIN in Depth**: The pairwise algorithms (Nested Loop, Hash, Merge) and commutativity/associativity that make multi-join reordering possible.
- **Topic 12 — LEFT JOIN and RIGHT JOIN**: The ON-vs-WHERE trap that becomes far easier to trigger — and far more damaging — in a mixed multi-join.
- **Topic 15 — Self JOIN**: A table joined to itself is still just another node in the multi-join tree; everything here applies.
- **Topic 17 — JOIN Performance Deep Dive**: The next topic — full treatment of choosing and forcing join algorithms, `work_mem`, parallelism, and reading complex plans.
- **Topic 18 — The N+1 Query Problem**: What ORMs do *instead* of a real multi-join, and why one good multi-join usually beats many small queries.
- **Topic 20 — GROUP BY Fundamentals**: Aggregation over a fanned-out multi-join; the pre-aggregation pattern (§6.7) lives at the intersection of these topics.
- **Internals — The Query Planner & `pg_statistic`**: Selectivity estimation, dynamic-programming join search, GEQO, and the collapse-limit GUCs this topic builds on.
