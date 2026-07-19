# Topic 29 — Recursive CTEs
### SQL Mastery Curriculum — Phase 5: Subqueries and CTEs

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you are standing at the very bottom of a corporate org chart, and someone hands you a single business card — the CEO's. Your job: find every single person in the company, from the CEO all the way down to the newest intern, and write out the full chain of command for each one.

You can't see the whole org chart at once. All you have is one rule and one starting card:

- **Starting card**: "The CEO reports to nobody." (This is your seed.)
- **The rule**: "Given anyone whose card I already hold, go find everyone who reports *directly* to them, and hand me their cards."

So you start with the CEO's card. You apply the rule: who reports to the CEO? You get five VP cards. Now you apply the rule again to *those five cards*: who reports to each VP? You get thirty director cards. Apply the rule again: who reports to each director? And so on. Each round you only look at the cards you got in the *previous* round — never the whole pile — and you keep going until a round produces zero new cards (you've reached the interns, nobody reports to them). Then you stop.

That is exactly what a **recursive CTE** does. You give it:

1. A **starting point** (the anchor member — "the CEO reports to nobody").
2. A **rule that references the results of the previous round** (the recursive member — "join what I found last round to the table to find the next level").
3. An implicit **stopping condition** (stop when a round produces no new rows).

The magic — and the thing that trips everyone up — is that the recursive part of the query *refers to itself*. The CTE's own name appears inside its own definition. That is not a typo or an error. It is a controlled, bounded loop expressed as a single SQL statement. The database does the round-by-round iteration for you; you just describe the seed and the step.

The word "recursive" scares people, but the mental model is dead simple: **seed row(s), then repeat a step against the previous step's output, until nothing new comes out.**

---

## 2. Connection to SQL Internals

A recursive CTE is not "real" recursion in the call-stack sense — PostgreSQL does not push stack frames. Under the hood it is **iteration** driven by a **working table**. The SQL standard even calls the recursive member's evaluation a fixed-point iteration. Here is what the engine actually maintains:

1. **The result table (`R`)** — accumulates every row the CTE will eventually return.
2. **The working table (`W`)** — holds only the rows produced by the *most recent* iteration. This is the key structure. Each recursive step reads from `W`, not from the entire accumulated result.
3. **An intermediate table (`I`)** — scratch space where the current iteration's output lands before being swapped into `W`.

The algorithm PostgreSQL runs (this is literally the documented evaluation, `RecursiveUnion` in the plan):

```
1. Evaluate the anchor (non-recursive) term.
2. Put those rows into both R and W.
3. Loop:
     a. Evaluate the recursive term, but every reference to the CTE name reads from W (not R).
     b. For UNION (not ALL): discard rows that duplicate anything already in R.
     c. Put the surviving rows into R and into a fresh intermediate I.
     d. Replace W with I.  (W now = only this round's new rows.)
     e. If W is empty, stop.
```

The internal structures this touches:

- **`RecursiveUnion` node** in the planner — the physical operator that owns the working-table loop. Below it you will see two children: the anchor plan and the recursive plan, and a **`WorkTable Scan`** node representing the read from `W`.
- **`tuplestore`** — PostgreSQL materializes the working table and intermediate table as tuplestores. If they exceed `work_mem`, they spill to temporary disk files (`base/pgsql_tmp`). A deep or wide recursion is a classic `work_mem` pressure point.
- **No index on the working table** — the working table is transient and *unindexed*. Every recursive step that joins `W` to a base table relies on an index *on the base table's* join column, never on `W`. This is why the base table's join key must be indexed for recursion to scale.
- **MVCC snapshot** — the entire recursive CTE executes within one statement snapshot (Topic on MVCC). The base tables are read at a single consistent point in time; concurrent writes are invisible mid-recursion.
- **CTEs and the optimization fence** — before PostgreSQL 12, *all* CTEs were an optimization fence (always materialized). From 12 onward, *non-recursive* CTEs can be inlined, but **recursive CTEs are always materialized** — the `RecursiveUnion` is never inlined into the outer query. You cannot `MATERIALIZED`/`NOT MATERIALIZED` your way out of that for the recursive term.

So when you read a recursive CTE, picture a `while` loop over a shrinking-then-emptying working table, each iteration doing one relational step and appending to an ever-growing result tuplestore.

---

## 3. Logical Execution Order Context

A recursive CTE lives in the `WITH` clause, which is logically evaluated **before** the main query's `FROM` sees it. But *within* the CTE there is a second, nested execution order that is unique to recursion:

```
WITH RECURSIVE cte AS (
    <anchor query>          ← evaluated ONCE, first
    UNION [ALL]
    <recursive query>       ← evaluated REPEATEDLY, reading the working table
)
SELECT ...                  ← the outer query, runs AFTER the CTE reaches fixed point
FROM cte
WHERE ...                   ← filters applied to the FULLY MATERIALIZED cte result
GROUP BY ... HAVING ...
ORDER BY ... LIMIT ...
```

The critical ordering facts, and why each matters here:

- **The CTE is fully computed before the outer `WHERE`/`ORDER BY`/`LIMIT` run.** A `LIMIT 10` in the outer query does **not** stop the recursion early in the general case — the recursion runs to its natural termination, materializes every row, and only then does the outer query slice off 10. (There is a narrow exception: if the outer query has no `ORDER BY` and the plan can stream from the `RecursiveUnion`, PostgreSQL *can* stop early once the LIMIT is satisfied. Do not rely on it; put the bound *inside* the recursion.)
- **`UNION` vs `UNION ALL` changes the termination logic itself**, not just deduplication. With `UNION` (no `ALL`), the dedup-against-`R` step is what stops cycles from looping forever. With `UNION ALL`, there is no dedup, so a cyclic graph loops until it exhausts memory unless you add explicit cycle detection.
- **Ordering across iterations is breadth-first by construction**, but *not guaranteed* in output. The working-table algorithm naturally produces rows level-by-level (all depth-1, then all depth-2, …), but PostgreSQL does not promise this order in the final result. If you need hierarchical or depth ordering, you must carry a `depth` or `path` column and `ORDER BY` it in the outer query.
- **The anchor's column list defines the CTE's column types.** The recursive term must produce a union-compatible row (same column count, compatible types). Type mismatches — e.g., the anchor emits `integer` for a `depth` column and the recursive term emits `integer + 1` which is fine, but the anchor emitting a text literal `''` for a path while the recursive term emits `varchar` — trigger "recursive query ... column N has type X in non-recursive term but type Y overall" errors. You often must cast in the anchor.

Where recursion sits relative to everything else: it is a `FROM`-phase construct (a named subquery), so it runs before `WHERE`/`GROUP BY`/`SELECT` of the outer statement, exactly like the non-recursive CTEs from Topic 28 — but internally it carries its own bounded loop that the rest of the pipeline never sees.

---

## 4. What Is a Recursive CTE?

A **recursive CTE** is a common table expression whose definition references itself, allowing a single SQL statement to iterate over hierarchical or graph-structured data until a fixed point (no new rows) is reached. It is composed of a **non-recursive anchor member**, a set operator (`UNION` or `UNION ALL`), and a **recursive member** that references the CTE by name.

```sql
WITH RECURSIVE subordinates AS (
    -- ┌── ANCHOR MEMBER: the seed. Runs once. No self-reference.
    SELECT
        e.id,                    -- │ columns here DEFINE the CTE's output shape & types
        e.name,
        e.manager_id,
        1 AS depth               -- │ synthesized column: tracks recursion depth
    FROM employees e             -- │ base table scan
    WHERE e.id = 42              -- │ the starting point (a specific root)

    UNION ALL                    -- └── set op: ALL = keep dups (fast, no cycle guard)
                                 --     UNION (no ALL) = dedup each round (cycle-safe, slower)

    -- ┌── RECURSIVE MEMBER: references `subordinates` (itself). Runs repeatedly.
    SELECT
        e.id,                    -- │ same column shape as the anchor
        e.name,
        e.manager_id,
        s.depth + 1              -- │ increment depth using the PREVIOUS round's row
    FROM employees e             -- │ base table, re-scanned each iteration
    INNER JOIN subordinates s    -- │ THE SELF-REFERENCE: reads the working table (last round)
        ON e.manager_id = s.id   -- │ the "step" rule: children of last round's rows
    -- WHERE s.depth < 100       -- │ (optional) explicit depth cap — bound the loop
)
SELECT id, name, manager_id, depth
FROM subordinates                -- outer query: reads the fully materialized result
ORDER BY depth, id;              -- carry a depth column to reconstruct level order
```

Anatomy, precisely:

- **`WITH RECURSIVE`** — the keyword `RECURSIVE` is required and applies to the whole `WITH` list, not a single CTE. (Curiously, you can then define non-recursive CTEs in the same `WITH RECURSIVE` block; the keyword just *permits* self-reference.)
- **Anchor member** — any `SELECT` with no reference to the CTE. It seeds the working table. There can be more than one anchor member, combined by `UNION`/`UNION ALL`, though one is typical.
- **`UNION` / `UNION ALL`** — mandatory separator. `UNION ALL` keeps every produced row; `UNION` eliminates duplicates against the accumulated result each iteration, which doubles as automatic cycle protection.
- **Recursive member** — a `SELECT` that references the CTE name exactly once (PostgreSQL restriction: the self-reference may appear only once, and not inside a subquery, `LEFT JOIN` outer side, aggregate, `LIMIT`, etc.).
- **Termination** — implicit: the loop stops when an iteration produces zero rows. There is no explicit `STOP` keyword; you engineer termination via the join not matching, a `depth <` guard, or `UNION`'s dedup breaking a cycle.

---

## 5. Why Recursive CTE Mastery Matters in Production

1. **Hierarchies are everywhere and adjacency lists are the default schema.** Org charts, category trees, threaded comments, folder structures, bill-of-materials, permission inheritance — the overwhelmingly common storage model is `(id, parent_id)`. Without recursion you cannot answer "give me the entire subtree" in a single query; you fall back to N+1 application-side loops (Topic 18) that make one query per level per node. A recursive CTE replaces hundreds of round-trips with one.

2. **The alternative is worse.** Before recursive CTEs, teams used stored procedures with `WHILE` loops, or the nested-set / materialized-path models that require rewriting large swaths of the table on every insert. Recursive CTEs let you keep the simple adjacency list and still query it efficiently.

3. **Cycle bugs cause production outages.** A miscoded `manager_id` that points in a loop (A reports to B, B reports to A) will spin a `UNION ALL` recursion until it exhausts `work_mem`, spills gigabytes to disk, and either times out or takes the box down. Knowing when you need `UNION` vs the `CYCLE` clause vs a manual path guard is the difference between a robust query and a landmine.

4. **Unbounded recursion is a resource DoS.** A recursive CTE with no depth cap over user-controllable data (e.g., "expand this graph the user uploaded") is a denial-of-service vector. Depth limiting is a security control, not just a performance tweak.

5. **Performance cliffs are non-obvious.** Recursion re-scans the base table every iteration. Without an index on the join column, a 10-level recursion over a million-row table is ten sequential scans. Understanding that the working table is unindexed and the base table's index is what saves you is essential to keeping these queries sub-second.

6. **They express things JOINs simply cannot.** Transitive closure ("is A connected to B through any number of hops?"), shortest-path, running date series, generating gap-free sequences, parsing/splitting recursively — these have no fixed-JOIN equivalent because you do not know the depth in advance. This is the unique capability recursion buys you (contrast in Topic 30).

---

## 6. Deep Technical Content

### 6.1 The Working-Table Algorithm, Step by Step

Consider this table:

```
employees:
id | name    | manager_id
---+---------+-----------
1  | Root    | NULL
2  | Alice   | 1
3  | Bob     | 1
4  | Carol   | 2
5  | Dan     | 2
6  | Eve     | 4
```

Query: full subtree under `id = 1`, `UNION ALL`, tracking depth.

```sql
WITH RECURSIVE tree AS (
    SELECT id, name, manager_id, 1 AS depth
    FROM employees WHERE id = 1
  UNION ALL
    SELECT e.id, e.name, e.manager_id, t.depth + 1
    FROM employees e JOIN tree t ON e.manager_id = t.id
)
SELECT * FROM tree ORDER BY depth, id;
```

Iteration trace:

```
Anchor:        R = {(1,Root,NULL,1)}          W = {(1,Root,NULL,1)}
Iteration 1:   children of W (mgr=1): Alice(2), Bob(3)
               R += {(2,Alice,1,2),(3,Bob,1,2)}   W = {those 2}
Iteration 2:   children of {2,3}: Carol(4),Dan(5) [mgr=2]; none for 3
               R += {(4,Carol,2,3),(5,Dan,2,3)}   W = {those 2}
Iteration 3:   children of {4,5}: Eve(6) [mgr=4]
               R += {(6,Eve,4,4)}                 W = {(6,...)}
Iteration 4:   children of {6}: none
               W = {}  → STOP
```

Final result: 6 rows, depths 1..4. Note iteration 1 read `W` (just the root), *not* all of `R`. That is the entire trick.

### 6.2 UNION ALL vs UNION — Semantics and Cycle Behavior

- **`UNION ALL`**: no deduplication. Fastest. **Loops forever on a cycle.** Use when the graph is guaranteed acyclic (a strict tree with a single root path) or when you add your own cycle guard.
- **`UNION`**: deduplicates the newly produced rows against everything already in `R` each iteration. A row that has been seen before is dropped, so a cycle A→B→A produces A, then B, then A-again-is-a-dup-and-dropped → working table empties → terminates. The cost: an implicit dedup (hash/sort) every iteration, and it only dedups on the *entire row*, so if you carry a `depth` or `path` column that differs, the "same" node is no longer a duplicate and the cycle protection silently fails.

This is the subtle trap: people add `UNION` for cycle safety, then also add `depth + 1`, and now `(A, depth=1)` and `(A, depth=3)` are different rows, so `UNION` never sees a duplicate and the loop runs forever anyway. **`UNION` only protects you if the columns that repeat in a cycle are identical.**

```sql
-- BROKEN cycle protection: depth makes every revisit "unique"
WITH RECURSIVE walk AS (
    SELECT id, 1 AS depth FROM nodes WHERE id = 1
  UNION                          -- looks safe...
    SELECT n.id, w.depth + 1     -- ...but depth differs each loop → no dup ever
    FROM edges e JOIN walk w ON e.src = w.id
    JOIN nodes n ON n.id = e.dst
)
SELECT * FROM walk;              -- STILL loops forever on a cycle
```

### 6.3 Cycle Detection — Three Techniques

**(a) Manual path array + guard.** Carry the visited-node path; refuse to revisit.

```sql
WITH RECURSIVE walk AS (
    SELECT n.id, ARRAY[n.id] AS path, FALSE AS is_cycle
    FROM nodes n WHERE n.id = 1
  UNION ALL
    SELECT n.id,
           w.path || n.id,               -- append current node to path
           n.id = ANY(w.path)            -- have we seen this node before?
    FROM walk w
    JOIN edges e ON e.src = w.id
    JOIN nodes n ON n.id = e.dst
    WHERE NOT w.is_cycle                  -- stop expanding once a cycle is flagged
)
SELECT * FROM walk WHERE NOT is_cycle;
```

The `WHERE NOT w.is_cycle` in the recursive term is what actually terminates: once a row is flagged as closing a cycle, its children are never generated.

**(b) The `CYCLE` clause (SQL:1999, PostgreSQL 14+).** Declarative, does the same bookkeeping for you.

```sql
WITH RECURSIVE walk AS (
    SELECT id, name FROM nodes WHERE id = 1
  UNION ALL
    SELECT n.id, n.name
    FROM walk w JOIN edges e ON e.src = w.id
    JOIN nodes n ON n.id = e.dst
)
CYCLE id SET is_cycle USING path
--    │        │              └── column that stores the traversal path (an array)
--    │        └── boolean column set TRUE when a cycle is detected
--    └── the column(s) that identify a node for cycle-checking
SELECT * FROM walk WHERE NOT is_cycle;
```

`CYCLE id SET is_cycle USING path` tells PostgreSQL: track uniqueness on `id`, expose a boolean `is_cycle`, and materialize the path in a column named `path`. When a node's `id` reappears on its own path, the row is emitted with `is_cycle = true` and is **not** expanded further. This is the cleanest option on PG 14+.

**(c) `UNION` (whole-row dedup).** Works only when the recurring columns are identical across a cycle — see 6.2. Rarely the right tool once you carry depth/path.

### 6.4 Depth Limiting

Even on acyclic data, unbounded recursion over user data is dangerous. Two ways to cap:

```sql
-- (a) Guard inside the recursive term (preferred — bounds the loop itself)
WITH RECURSIVE tree AS (
    SELECT id, manager_id, 1 AS depth FROM employees WHERE id = 42
  UNION ALL
    SELECT e.id, e.manager_id, t.depth + 1
    FROM employees e JOIN tree t ON e.manager_id = t.id
    WHERE t.depth < 10            -- stop after 10 levels; loop cannot exceed it
)
SELECT * FROM tree;

-- (b) Filter in the outer query (WRONG for bounding — recursion still runs full!)
WITH RECURSIVE tree AS ( ... no guard ... )
SELECT * FROM tree WHERE depth <= 10;   -- computes ALL depths, then filters. Wasteful.
```

Form (a) bounds the *computation*; form (b) only bounds the *output* after the full recursion already ran. Always put the depth guard in the recursive member.

### 6.5 Multiple Anchor Rows / Forest Traversal

The anchor need not be a single row. Seed with all roots to traverse a forest:

```sql
WITH RECURSIVE tree AS (
    SELECT id, name, parent_id, 1 AS depth,
           ARRAY[id] AS path
    FROM categories WHERE parent_id IS NULL   -- ALL roots, not one
  UNION ALL
    SELECT c.id, c.name, c.parent_id, t.depth + 1,
           t.path || c.id
    FROM categories c JOIN tree t ON c.parent_id = t.id
)
SELECT repeat('  ', depth - 1) || name AS indented, depth
FROM tree
ORDER BY path;    -- ORDER BY the path array = correct depth-first tree layout
```

`ORDER BY path` (an integer array) produces a proper depth-first, indented tree — sibling order preserved, children immediately under parents. This is the standard idiom for rendering a tree.

### 6.6 Sequence and Date-Series Generation

Recursion can manufacture rows that do not exist in any table.

```sql
-- Generate integers 1..10
WITH RECURSIVE nums AS (
    SELECT 1 AS n
  UNION ALL
    SELECT n + 1 FROM nums WHERE n < 10
)
SELECT n FROM nums;

-- Generate a gap-free daily date series for a report axis
WITH RECURSIVE dates AS (
    SELECT DATE '2026-01-01' AS d
  UNION ALL
    SELECT d + 1 FROM dates WHERE d < DATE '2026-01-31'
)
SELECT d FROM dates;
```

In PostgreSQL you would normally use `generate_series()` for exactly this — it is faster and clearer. Recursion for sequences matters mainly (a) in databases lacking `generate_series` (older MySQL, SQL Server historically), and (b) when each step depends on the previous computed value in a non-linear way (e.g., a running compound-interest balance, Fibonacci, amortization schedules) where `generate_series` cannot help.

```sql
-- Loan amortization: each row depends on the previous balance (not expressible via generate_series)
WITH RECURSIVE schedule AS (
    SELECT 1 AS period,
           10000.00::numeric AS balance,
           10000.00 * 0.01 AS interest,     -- 1% monthly
           500.00 AS payment
  UNION ALL
    SELECT period + 1,
           balance - (payment - balance * 0.01),
           (balance - (payment - balance * 0.01)) * 0.01,
           500.00
    FROM schedule
    WHERE balance > 0 AND period < 360
)
SELECT * FROM schedule;
```

### 6.7 Graph Traversal and Transitive Closure

Reachability — "everything reachable from node X" — is the canonical graph use case.

```sql
-- All nodes reachable from node 1 in a directed graph (cycle-safe)
WITH RECURSIVE reachable AS (
    SELECT 1 AS node, ARRAY[1] AS path
  UNION ALL
    SELECT e.dst, r.path || e.dst
    FROM reachable r
    JOIN edges e ON e.src = r.node
    WHERE e.dst <> ALL(r.path)     -- do not revisit nodes on the current path
)
SELECT DISTINCT node FROM reachable WHERE node <> 1;
```

Shortest path (unweighted, breadth-first — the first time a node is reached is via a shortest path):

```sql
WITH RECURSIVE bfs AS (
    SELECT 1 AS node, 0 AS dist, ARRAY[1] AS path
  UNION ALL
    SELECT e.dst, b.dist + 1, b.path || e.dst
    FROM bfs b JOIN edges e ON e.src = b.node
    WHERE e.dst <> ALL(b.path)
)
SELECT node, MIN(dist) AS shortest_hops
FROM bfs
GROUP BY node
ORDER BY node;
```

### 6.8 Aggregation Restrictions in the Recursive Term

PostgreSQL forbids certain constructs in the recursive member because they break the fixed-point model:

- No aggregate functions (`SUM`, `COUNT`, …) referencing the recursive CTE.
- No `GROUP BY` / `HAVING` on the recursive self-reference.
- No window functions over the recursive term.
- No `DISTINCT` on the union between anchor and recursive term (use `UNION` for that instead).
- No `LIMIT`/`OFFSET` in the recursive term.
- No subquery, `LEFT`/`RIGHT`/`FULL OUTER JOIN`, `EXCEPT`, or `INTERSECT` that references the CTE.
- The self-reference may appear **only once** in the recursive `FROM`.

If you need to aggregate, do it in the *outer* query over the finished CTE, or accumulate a running value column by column inside the recursion (as the amortization example does).

### 6.9 Running Totals via Recursion (and Why You Usually Should Not)

You *can* compute running totals recursively:

```sql
WITH RECURSIVE running AS (
    SELECT id, amount, amount AS total
    FROM (SELECT id, amount, ROW_NUMBER() OVER (ORDER BY id) rn FROM payments) x
    WHERE rn = 1
  UNION ALL
    ...
)
```

But this is an anti-pattern in PostgreSQL: a window function `SUM(amount) OVER (ORDER BY id)` does it in one pass, O(N), no per-row iteration. Recursion here is O(N) iterations each materializing the working table — far slower. Reserve recursion for genuinely hierarchical/graph problems; use window functions for ordered running computations.

### 6.10 Mutual Recursion and Multiple CTEs

Only *one* CTE in a `WITH RECURSIVE` list needs to be recursive; others can be plain. You can reference an earlier CTE from the recursive one:

```sql
WITH RECURSIVE
  roots AS (                             -- non-recursive helper
    SELECT id FROM employees WHERE manager_id IS NULL
  ),
  tree AS (
    SELECT e.id, e.manager_id, 1 AS depth
    FROM employees e JOIN roots r ON e.id = r.id
  UNION ALL
    SELECT e.id, e.manager_id, t.depth + 1
    FROM employees e JOIN tree t ON e.manager_id = t.id
  )
SELECT * FROM tree;
```

True mutual recursion (two CTEs referencing each other) is **not** supported by PostgreSQL; only self-reference within a single CTE is allowed.

### 6.11 SEARCH Clause — Ordered Traversal (PG 14+)

Alongside `CYCLE`, PostgreSQL 14 added `SEARCH` to formalize depth-first or breadth-first ordering without hand-rolling a path/depth column:

```sql
WITH RECURSIVE tree AS (
    SELECT id, name, parent_id FROM categories WHERE parent_id IS NULL
  UNION ALL
    SELECT c.id, c.name, c.parent_id
    FROM categories c JOIN tree t ON c.parent_id = t.id
)
SEARCH DEPTH FIRST BY id SET ordercol   -- or SEARCH BREADTH FIRST BY id SET ordercol
SELECT id, name FROM tree ORDER BY ordercol;
```

`SEARCH DEPTH FIRST BY id SET ordercol` adds a hidden ordering column `ordercol` you can `ORDER BY` to get a correct DFS/BFS layout, replacing the manual `path` array idiom.

---

## 7. EXPLAIN — Recursion in the Plan

### Bounded tree traversal with an index on the join column

```sql
EXPLAIN (ANALYZE, BUFFERS)
WITH RECURSIVE tree AS (
    SELECT id, name, manager_id, 1 AS depth
    FROM employees WHERE id = 42
  UNION ALL
    SELECT e.id, e.name, e.manager_id, t.depth + 1
    FROM employees e JOIN tree t ON e.manager_id = t.id
    WHERE t.depth < 20
)
SELECT * FROM tree;
```

```
CTE Scan on tree  (cost=412.55..414.57 rows=101 width=44)
                  (actual time=0.030..2.114 rows=350 loops=1)
  CTE tree
    ->  Recursive Union  (cost=0.29..412.55 rows=101 width=44)
                         (actual time=0.028..1.985 rows=350 loops=1)
          ->  Index Scan using employees_pkey on employees
                (cost=0.29..8.30 rows=1 width=44)
                (actual time=0.020..0.021 rows=1 loops=1)
                Index Cond: (id = 42)
          ->  Nested Loop  (cost=0.29..40.22 rows=10 width=44)
                           (actual time=0.008..0.180 rows=44 loops=8)
                ->  WorkTable Scan on tree t
                      (cost=0.00..0.20 rows=10 width=8)
                      (actual time=0.001..0.004 rows=44 loops=8)
                      Filter: (depth < 20)
                ->  Index Scan using employees_manager_id_idx on employees e
                      (cost=0.29..3.98 rows=1 width=40)
                      (actual time=0.002..0.003 rows=1 loops=350)
                      Index Cond: (manager_id = t.id)
Buffers: shared hit=712
Planning Time: 0.181 ms
Execution Time: 2.210 ms
```

**Reading it, line by line:**

- **`Recursive Union`** — the operator that owns the working-table loop. Its two children are the anchor and the recursive step.
- **First child (`Index Scan ... Index Cond: (id = 42)`)** — the anchor, run once, one row.
- **Second child (`Nested Loop ... loops=8`)** — the recursive term. `loops=8` means the loop iterated 8 times (tree depth ≈ 8 before the working table emptied). Each iteration is one nested loop.
- **`WorkTable Scan on tree t`** — this is the read of the working table `W` (last round's rows). It is **not** an index scan and never will be — the working table has no indexes. `Filter: (depth < 20)` is the depth guard being applied.
- **`Index Scan using employees_manager_id_idx`** — the crucial line. For each working-table row, the child level is fetched via the **base-table index on `manager_id`**, `loops=350` (once per row produced overall). This index is what makes recursion fast.
- **`CTE Scan on tree`** — the outer query reading the fully materialized result (350 rows). The recursion is materialized into a tuplestore; the outer query streams from it.

### The same query with NO index on manager_id (the pathological case)

```
          ->  Nested Loop  (actual time=4.1..1120.5 rows=44 loops=8)
                ->  WorkTable Scan on tree t (actual rows=44 loops=8)
                ->  Seq Scan on employees e   (actual rows=1 loops=350)
                      Filter: (manager_id = t.id)
                      Rows Removed by Filter: 999999
...
Execution Time: 9004.7 ms
```

`Seq Scan on employees` with `loops=350` and `Rows Removed by Filter: 999999` — every recursive step scans the entire million-row table. 350 full scans → ~9 seconds. The fix is a single `CREATE INDEX ON employees(manager_id)`.

### What to watch for in a recursive plan

| Symptom | Meaning | Fix |
|---------|---------|-----|
| `Seq Scan` under `Recursive Union` recursive child | No index on base-table join column | Index the join column (`parent_id`/`manager_id`/`edges.src`) |
| Very high `loops=` on the Nested Loop | Deep recursion (many iterations) | Add a `depth <` guard; verify it is not a cycle |
| `rows` in result far exceeds table size | Cyclic graph exploding via `UNION ALL` | Add `CYCLE` clause or path guard |
| `Sort`/`HashAggregate` inside `Recursive Union` per iteration | `UNION` (dedup) overhead each round | Use `UNION ALL` + explicit cycle guard if data is acyclic |
| Temp file / disk buffers under the union | Working/result tuplestore spilling `work_mem` | Raise `work_mem` for the statement, or narrow the result |

---

## 8. Query Examples

### Example 1 — Basic: Full Subtree of a Manager

```sql
-- Every employee under manager 42, at any depth, with their level.
WITH RECURSIVE subordinates AS (
    -- anchor: the manager themselves at depth 0
    SELECT id, name, manager_id, 0 AS depth
    FROM employees
    WHERE id = 42
  UNION ALL
    -- recursive step: direct reports of everyone found last round
    SELECT e.id, e.name, e.manager_id, s.depth + 1
    FROM employees e
    INNER JOIN subordinates s ON e.manager_id = s.id
)
SELECT id, name, depth
FROM subordinates
WHERE id <> 42            -- exclude the manager themselves from the report
ORDER BY depth, name;
```

### Example 2 — Intermediate: Indented Category Tree with Path Ordering

```sql
-- Render the full category tree, indented, in correct depth-first order.
-- categories(id, name, parent_id)
WITH RECURSIVE category_tree AS (
    SELECT
        c.id,
        c.name,
        c.parent_id,
        1 AS depth,
        ARRAY[c.id] AS path,                 -- integer path for ordering
        c.name::text AS name_path            -- human-readable breadcrumb
    FROM categories c
    WHERE c.parent_id IS NULL                -- all top-level roots
  UNION ALL
    SELECT
        c.id,
        c.name,
        c.parent_id,
        ct.depth + 1,
        ct.path || c.id,
        ct.name_path || ' > ' || c.name
    FROM categories c
    INNER JOIN category_tree ct ON c.parent_id = ct.id
    WHERE ct.depth < 20                      -- safety cap
)
SELECT
    repeat('    ', depth - 1) || name AS tree_label,
    name_path AS breadcrumb,
    depth
FROM category_tree
ORDER BY path;                               -- depth-first, siblings in id order
```

### Example 3 — Production Grade: Reporting-Line Rollup with Cycle Safety

Scenario: an `employees` table of ~250,000 rows, `employees(id PK, name, manager_id FK→employees.id, department_id)`, index on `manager_id`. HR occasionally introduces bad data (a manager cycle). We need, for a given executive, the full downstream org with headcount per level, and the query must **never** loop even if the data is corrupt. Target: < 50 ms for a subtree of a few thousand people.

```sql
-- Full org rollup under an executive, cycle-safe, with per-level headcount.
-- Table sizes: employees ~250K rows. Index: employees(manager_id).
-- Expectation: single-digit ms for typical subtrees; bounded even on cyclic data.
WITH RECURSIVE org AS (
    SELECT
        e.id,
        e.name,
        e.manager_id,
        e.department_id,
        0 AS depth,
        ARRAY[e.id] AS path,
        FALSE AS is_cycle
    FROM employees e
    WHERE e.id = $1                          -- the executive (parameterized)
  UNION ALL
    SELECT
        e.id,
        e.name,
        e.manager_id,
        e.department_id,
        o.depth + 1,
        o.path || e.id,
        e.id = ANY(o.path)                   -- cycle flag: seen this id already?
    FROM employees e
    INNER JOIN org o ON e.manager_id = o.id
    WHERE NOT o.is_cycle                     -- stop expanding once a cycle closes
      AND o.depth < 30                       -- hard depth cap (defense in depth)
)
SELECT
    depth,
    COUNT(*)                              AS headcount,
    COUNT(DISTINCT department_id)         AS departments_at_level,
    ARRAY_AGG(name ORDER BY name)         AS people
FROM org
WHERE NOT is_cycle
  AND id <> $1
GROUP BY depth
ORDER BY depth;
```

Its EXPLAIN (typical, subtree of ~2,000 people, index present):

```
GroupAggregate  (cost=520.10..540.80 rows=12 width=72)
                (actual time=6.9..7.4 rows=9 loops=1)
  Group Key: org.depth
  ->  Sort  (actual time=6.8..6.9 rows=2013 loops=1)
        Sort Key: org.depth
        Sort Method: quicksort  Memory: 412kB
        ->  CTE Scan on org  (actual time=0.03..5.9 rows=2013 loops=1)
                             Filter: ((NOT is_cycle) AND (id <> $1))
              CTE org
                ->  Recursive Union (actual time=0.02..4.8 rows=2015 loops=1)
                      ->  Index Scan using employees_pkey on employees e
                            Index Cond: (id = $1)
                      ->  Nested Loop (actual time=0.01..0.4 rows=250 loops=8)
                            ->  WorkTable Scan on org o
                                  Filter: ((NOT is_cycle) AND (depth < 30))
                            ->  Index Scan using employees_manager_id_idx on employees e
                                  Index Cond: (manager_id = o.id)
Buffers: shared hit=4120
Execution Time: 7.6 ms
```

The `Index Scan using employees_manager_id_idx` inside the recursive child is what keeps this at 7.6 ms over a 250K-row table. Remove that index and you get 8 sequential scans of 250K rows.

---

## 9. Wrong → Right Patterns

### Wrong 1: UNION ALL over a graph that can contain cycles

```sql
-- WRONG: no cycle protection on a directed graph
WITH RECURSIVE reach AS (
    SELECT src, dst FROM edges WHERE src = 1
  UNION ALL
    SELECT e.src, e.dst
    FROM edges e JOIN reach r ON e.src = r.dst
)
SELECT * FROM reach;
-- RESULT: if edges contain 1→2, 2→3, 3→1, this loops forever,
-- spills tuplestore to disk, and eventually errors:
--   ERROR: could not write to temporary file / or statement_timeout kills it.
```

Why it breaks: `UNION ALL` never deduplicates. The working table keeps producing "new" rows by cycling 1→2→3→1→2→3… indefinitely. There is no termination condition because the join always matches.

```sql
-- RIGHT: track the path and refuse to revisit (or use the CYCLE clause on PG14+)
WITH RECURSIVE reach AS (
    SELECT src, dst, ARRAY[src, dst] AS path
    FROM edges WHERE src = 1
  UNION ALL
    SELECT e.src, e.dst, r.path || e.dst
    FROM edges e JOIN reach r ON e.src = r.dst
    WHERE e.dst <> ALL(r.path)          -- never step onto a node already visited
)
SELECT DISTINCT dst FROM reach;
```

### Wrong 2: Trusting UNION to break a cycle while carrying a depth column

```sql
-- WRONG: UNION "for cycle safety" defeated by the depth column
WITH RECURSIVE walk AS (
    SELECT id, 0 AS depth FROM nodes WHERE id = 1
  UNION                                   -- intends to dedup revisited nodes
    SELECT n.id, w.depth + 1              -- but (id, depth) differs each loop
    FROM edges e JOIN walk w ON e.src = w.id
    JOIN nodes n ON n.id = e.dst
)
SELECT * FROM walk;
-- RESULT: still infinite on a cycle. (1,0),(2,1),(1,2),(2,3),(1,4)... all distinct rows,
-- so UNION never sees a duplicate → never terminates.
```

Why it breaks: `UNION` dedups on the *whole* row. Because `depth` increases every iteration, no two rows are ever identical, so the dedup never fires. The cycle protection is an illusion.

```sql
-- RIGHT: dedup on identity only, keep depth outside the dedup key via a path guard
WITH RECURSIVE walk AS (
    SELECT id, 0 AS depth, ARRAY[id] AS path FROM nodes WHERE id = 1
  UNION ALL
    SELECT n.id, w.depth + 1, w.path || n.id
    FROM edges e JOIN walk w ON e.src = w.id
    JOIN nodes n ON n.id = e.dst
    WHERE n.id <> ALL(w.path)            -- identity-based cycle guard
)
SELECT * FROM walk;
```

### Wrong 3: Depth cap placed in the outer query instead of the recursive term

```sql
-- WRONG: recursion runs to full depth, THEN filters — huge wasted work
WITH RECURSIVE tree AS (
    SELECT id, manager_id, 1 AS depth FROM employees WHERE id = 42
  UNION ALL
    SELECT e.id, e.manager_id, t.depth + 1
    FROM employees e JOIN tree t ON e.manager_id = t.id
    -- no guard here
)
SELECT * FROM tree WHERE depth <= 3;      -- filters AFTER computing all depths
-- On a 15-level org this computes all 15 levels, then throws away 12.
```

Why it breaks: the outer `WHERE depth <= 3` cannot prune the recursion — the CTE is fully materialized before the outer filter runs (Section 3). You pay for the entire tree.

```sql
-- RIGHT: bound the loop itself
WITH RECURSIVE tree AS (
    SELECT id, manager_id, 1 AS depth FROM employees WHERE id = 42
  UNION ALL
    SELECT e.id, e.manager_id, t.depth + 1
    FROM employees e JOIN tree t ON e.manager_id = t.id
    WHERE t.depth < 3                     -- recursion stops at level 3
)
SELECT * FROM tree;
```

### Wrong 4: Type mismatch between anchor and recursive term

```sql
-- WRONG: anchor emits a text literal, recursive term emits varchar concatenation
WITH RECURSIVE tree AS (
    SELECT id, name, '' AS path FROM categories WHERE parent_id IS NULL
  UNION ALL
    SELECT c.id, c.name, t.path || '/' || c.name
    FROM categories c JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree;
-- ERROR: recursive query "tree" column 3 has type text in non-recursive term
--        but type character varying overall
-- (exact message varies; the point: the anchor's inferred type conflicts)
```

Why it breaks: the anchor's `''` and the recursive term's expression resolve to different types, and PostgreSQL requires the union to be type-consistent, with the anchor defining the column type.

```sql
-- RIGHT: cast the anchor explicitly so both terms agree
WITH RECURSIVE tree AS (
    SELECT id, name, name::text AS path FROM categories WHERE parent_id IS NULL
  UNION ALL
    SELECT c.id, c.name, (t.path || '/' || c.name)::text
    FROM categories c JOIN tree t ON c.parent_id = t.id
)
SELECT * FROM tree;
```

### Wrong 5: Aggregate inside the recursive term

```sql
-- WRONG: attempting to aggregate the self-reference
WITH RECURSIVE rollup AS (
    SELECT id, parent_id, amount FROM accounts WHERE parent_id IS NULL
  UNION ALL
    SELECT a.id, a.parent_id, SUM(r.amount)      -- aggregate over the CTE
    FROM accounts a JOIN rollup r ON a.parent_id = r.id
    GROUP BY a.id, a.parent_id
)
SELECT * FROM rollup;
-- ERROR: aggregate functions are not allowed in a recursive query's recursive term
```

Why it breaks: aggregation breaks the fixed-point iteration model (Section 6.8) — the recursive term must be a straightforward set-producing SELECT.

```sql
-- RIGHT: recurse first (carry raw rows), aggregate in the outer query
WITH RECURSIVE tree AS (
    SELECT id, parent_id, amount, id AS root_id FROM accounts WHERE parent_id IS NULL
  UNION ALL
    SELECT a.id, a.parent_id, a.amount, t.root_id
    FROM accounts a JOIN tree t ON a.parent_id = t.id
)
SELECT root_id, SUM(amount) AS subtree_total
FROM tree
GROUP BY root_id;
```

---

## 10. Performance Profile

### Cost model

A recursive CTE's cost is roughly:

```
total ≈ anchor_cost + Σ (per-iteration cost)
per-iteration cost ≈ |W_i| × probe_cost(base table)
```

where `|W_i|` is the number of rows in the working table at iteration `i` and `probe_cost` is one index lookup (with an index) or one full scan (without). The total number of base-table probes equals the total number of rows the recursion emits.

### Scaling table

| Scenario | Base table rows | Result rows | With index on join col | Without index |
|----------|----------------|-------------|-------------------------|---------------|
| Org subtree, 8 levels, ~2K people | 250K | ~2K | ~7 ms | ~3–9 s (8 seq scans) |
| Category tree, full, 5 levels | 50K | 50K | ~40 ms | ~1–2 s |
| Graph reachability, dense, cycle-safe | 1M edges | up to 1M | 200 ms–1 s | minutes / OOM |
| Date series, 10 years daily | n/a (generated) | ~3,650 | ~5 ms | n/a |
| Cyclic graph, UNION ALL, no guard | any | ∞ | OOM / timeout | OOM / timeout |

### Memory

- The **result tuplestore** holds *every* emitted row for the life of the statement. A recursion that emits 5M rows holds 5M rows in memory (or spills to `pgsql_tmp`). Narrow your SELECT list — carry only the columns you need through the recursion.
- The **working table** holds one level at a time; for a wide tree (one level with 100K nodes) that level must fit in `work_mem` or it spills.
- **`UNION` (dedup)** adds a hash or sort structure per iteration — more memory and CPU than `UNION ALL`.
- Path arrays (`ARRAY[...]`) grow linearly with depth and are carried on every row — a 30-deep path on 1M rows is significant. For very deep/wide traversals, consider whether you truly need the full path or just a cycle flag.

### Optimization techniques specific to recursion

1. **Index the join column on the base table** — `parent_id`, `manager_id`, `edges(src)`. This is the single biggest lever (Section 7). Turns per-iteration full scans into index probes.
2. **Bound the recursion** with a `depth <` guard inside the recursive term — caps worst-case work and is a security control against unbounded user data.
3. **Prefer `UNION ALL` + explicit cycle guard** over `UNION` when you must dedup — you control exactly what identifies a duplicate (node id) rather than paying whole-row dedup.
4. **Select narrow** — do not drag wide text columns through every iteration; join to fetch them once in the outer query using the collected ids.
5. **Raise `work_mem` for the statement** (`SET LOCAL work_mem = '256MB'`) when a large traversal spills, but bound the result first — throwing memory at an unbounded cycle just delays the crash.
6. **Consider materialized path / closure table** for read-heavy, rarely-changing hierarchies. A `closure_table(ancestor, descendant, depth)` answers subtree queries with a plain indexed JOIN and no recursion — at the cost of maintenance on writes. Recursive CTEs win when the tree changes often; precomputed closure wins when reads dominate.
7. **Push filters into the anchor**, not the outer query, so the recursion starts from the smallest possible seed.

---

## 11. Node.js Integration

### 11.1 Basic subtree query with pg

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function getSubtree(rootEmployeeId, maxDepth = 20) {
  const { rows } = await pool.query(
    `WITH RECURSIVE subordinates AS (
       SELECT id, name, manager_id, 0 AS depth
       FROM employees
       WHERE id = $1
     UNION ALL
       SELECT e.id, e.name, e.manager_id, s.depth + 1
       FROM employees e
       INNER JOIN subordinates s ON e.manager_id = s.id
       WHERE s.depth < $2
     )
     SELECT id, name, manager_id, depth
     FROM subordinates
     WHERE id <> $1
     ORDER BY depth, name`,
    [rootEmployeeId, maxDepth]
  );
  return rows;
}
```

Note `$2` (the depth cap) is parameterized and lives **inside** the recursive term — the loop is bounded, and the bound is caller-controlled, which is important when `rootEmployeeId` comes from user input.

### 11.2 Cycle-safe graph reachability, returning a clean node list

```javascript
async function reachableNodes(startNodeId) {
  const { rows } = await pool.query(
    `WITH RECURSIVE reach AS (
       SELECT $1::int AS node, ARRAY[$1::int] AS path
     UNION ALL
       SELECT e.dst, r.path || e.dst
       FROM reach r
       JOIN edges e ON e.src = r.node
       WHERE e.dst <> ALL(r.path)          -- cycle guard: no revisits
     )
     SELECT DISTINCT node
     FROM reach
     WHERE node <> $1
     ORDER BY node`,
    [startNodeId]
  );
  return rows.map((r) => r.node);
}
```

The `path` array comes back to Node as a JavaScript array (pg parses `int[]`), though here we only return `node`. Guard against pathological inputs by also capping depth if `edges` is user-supplied.

### 11.3 Building a nested tree object from a flat recursive result

```javascript
// One query returns the flat rows; assemble the nested structure in JS.
async function getCategoryTree() {
  const { rows } = await pool.query(
    `WITH RECURSIVE t AS (
       SELECT id, name, parent_id, 1 AS depth, ARRAY[id] AS path
       FROM categories WHERE parent_id IS NULL
     UNION ALL
       SELECT c.id, c.name, c.parent_id, t.depth + 1, t.path || c.id
       FROM categories c JOIN t ON c.parent_id = t.id
       WHERE t.depth < 20
     )
     SELECT id, name, parent_id, depth FROM t ORDER BY path`
  );

  const byId = new Map(rows.map((r) => [r.id, { ...r, children: [] }]));
  const roots = [];
  for (const node of byId.values()) {
    if (node.parent_id == null) roots.push(node);
    else byId.get(node.parent_id)?.children.push(node);
  }
  return roots; // nested tree, built from ONE round-trip instead of N+1 queries
}
```

This is the payoff versus the N+1 pattern (Topic 18): one query, one round-trip, tree assembled in memory.

### 11.4 Guarding against runaway recursion at the connection level

```javascript
// For any recursive query over user-controllable graph data, set a hard
// statement timeout so a missed cycle guard cannot hang a connection.
async function safeTraverse(startId) {
  const client = await pool.connect();
  try {
    await client.query('SET LOCAL statement_timeout = 3000'); // 3s ceiling
    const { rows } = await client.query(
      `WITH RECURSIVE reach AS (
         SELECT $1::int AS node, ARRAY[$1::int] AS path, 0 AS depth
       UNION ALL
         SELECT e.dst, r.path || e.dst, r.depth + 1
         FROM reach r JOIN edges e ON e.src = r.node
         WHERE e.dst <> ALL(r.path) AND r.depth < 50
       )
       SELECT DISTINCT node FROM reach`,
      [startId]
    );
    return rows.map((r) => r.node);
  } finally {
    client.release();
  }
}
```

Defense in depth: `path` cycle guard + `depth < 50` cap + `statement_timeout`. Any one failing, the others contain the blast radius.

### A note on ORMs

No mainstream JS ORM has first-class support for `WITH RECURSIVE`. All of them require raw SQL for the recursive part. The ORM is still useful for the surrounding CRUD and for typing the result rows, but the recursion itself is hand-written SQL passed through a raw-query escape hatch. See Section 12.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do recursive CTEs?** — Not through its query API. There is no `include`-style construct for arbitrary-depth traversal. You must use `$queryRaw`.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

type OrgRow = { id: number; name: string; manager_id: number | null; depth: number };

const rootId = 42;
const rows = await prisma.$queryRaw<OrgRow[]>`
  WITH RECURSIVE subordinates AS (
    SELECT id, name, manager_id, 0 AS depth
    FROM employees WHERE id = ${rootId}
  UNION ALL
    SELECT e.id, e.name, e.manager_id, s.depth + 1
    FROM employees e JOIN subordinates s ON e.manager_id = s.id
    WHERE s.depth < 20
  )
  SELECT id, name, manager_id, depth FROM subordinates WHERE id <> ${rootId}
  ORDER BY depth, name
`;
```

**Where it breaks**: no type generation for the recursive result (you annotate the row type yourself), and `${rootId}` is safely parameterized by Prisma's tagged template — but if you build the SQL string dynamically you lose that protection. **Verdict**: use `$queryRaw` with an explicit row type; Prisma adds nothing to the recursion but is fine as the transport.

### Drizzle ORM

**Can Drizzle do recursive CTEs?** — Yes, uniquely among these it has a typed builder: `db.$with(...)` combined with `.union`/`.unionAll` and a self-referencing lambda, though many teams still prefer `sql` for clarity.

```typescript
import { sql } from 'drizzle-orm';
import { db } from './db';

// Typed-ish builder form
const subtree = db.$withRecursive('subtree').as((qb) =>
  qb
    .select({ id: employees.id, name: employees.name, depth: sql<number>`0`.as('depth') })
    .from(employees)
    .where(eq(employees.id, 42))
    .unionAll((self) =>
      qb
        .select({ id: employees.id, name: employees.name, depth: sql<number>`${self.depth} + 1` })
        .from(employees)
        .innerJoin(self, eq(employees.managerId, self.id))
        .where(sql`${self.depth} < 20`)
    )
);
const rows = await db.with(subtree).select().from(subtree);

// Pragmatic raw form (most teams use this)
const rows2 = await db.execute(sql`
  WITH RECURSIVE subtree AS (
    SELECT id, name, manager_id, 0 AS depth FROM employees WHERE id = 42
  UNION ALL
    SELECT e.id, e.name, e.manager_id, s.depth + 1
    FROM employees e JOIN subtree s ON e.manager_id = s.id WHERE s.depth < 20
  )
  SELECT * FROM subtree
`);
```

**Where it breaks**: the typed recursive builder is the most fragile part of Drizzle's API and its shape shifts across versions; the `sql` escape hatch always works. **Verdict**: best-in-class *attempt* at typed recursion, but for anything non-trivial use `db.execute(sql\`...\`)`.

### Sequelize

**Can Sequelize do recursive CTEs?** — No builder support. Raw query only.

```javascript
const { QueryTypes } = require('sequelize');
const rows = await sequelize.query(
  `WITH RECURSIVE subordinates AS (
     SELECT id, name, manager_id, 0 AS depth FROM employees WHERE id = :rootId
   UNION ALL
     SELECT e.id, e.name, e.manager_id, s.depth + 1
     FROM employees e JOIN subordinates s ON e.manager_id = s.id
     WHERE s.depth < :maxDepth
   )
   SELECT id, name, depth FROM subordinates WHERE id <> :rootId ORDER BY depth, name`,
  { replacements: { rootId: 42, maxDepth: 20 }, type: QueryTypes.SELECT }
);
```

**Where it breaks**: named replacements (`:rootId`) are interpolated as parameters — safe — but there is zero model mapping; you get plain objects. **Verdict**: raw query with `replacements`; Sequelize is just the driver here.

### TypeORM

**Can TypeORM do recursive CTEs?** — Two paths. It has a `@Tree` entity decorator (`materialized-path`, `closure-table`, `nested-set`, `adjacency-list`) that *hides* recursion behind `TreeRepository.findDescendants()`, and it has `query()` for raw SQL. The QueryBuilder also gained a limited `.addCommonTableExpression(..., { recursive: true })`.

```typescript
// (a) Tree entity — TypeORM manages a closure table for you
@Entity() @Tree('closure-table')
class Category {
  @PrimaryGeneratedColumn() id: number;
  @Column() name: string;
  @TreeChildren() children: Category[];
  @TreeParent() parent: Category;
}
const repo = dataSource.getTreeRepository(Category);
const subtree = await repo.findDescendantsTree(rootCategory); // no hand-written SQL

// (b) Raw recursive CTE when you need control
const rows = await dataSource.query(
  `WITH RECURSIVE subordinates AS (
     SELECT id, name, manager_id, 0 AS depth FROM employees WHERE id = $1
   UNION ALL
     SELECT e.id, e.name, e.manager_id, s.depth + 1
     FROM employees e JOIN subordinates s ON e.manager_id = s.id WHERE s.depth < 20
   )
   SELECT * FROM subordinates WHERE id <> $1`,
  [42]
);
```

**Where it breaks**: the `@Tree` closure-table approach maintains a separate table on every write (extra I/O, transactional coupling) and only supports the patterns it ships with — arbitrary graph traversal still needs raw SQL. **Verdict**: `@Tree('closure-table')` is genuinely useful for stable category trees; drop to `dataSource.query()` for graphs, cycle detection, or custom depth logic.

### Knex.js

**Can Knex do recursive CTEs?** — Yes, via `withRecursive`.

```javascript
const rows = await knex
  .withRecursive('subordinates', (qb) => {
    qb.select('id', 'name', 'manager_id')
      .select(knex.raw('0 AS depth'))
      .from('employees')
      .where('id', 42)
      .unionAll((qb2) => {
        qb2
          .select('e.id', 'e.name', 'e.manager_id')
          .select(knex.raw('s.depth + 1'))
          .from('employees as e')
          .join('subordinates as s', 'e.manager_id', 's.id')
          .where('s.depth', '<', 20);
      });
  })
  .select('*')
  .from('subordinates')
  .whereNot('id', 42)
  .orderBy(['depth', 'name']);
```

**Where it breaks**: the union member is a closure and reads a little awkwardly; the synthesized `depth` column relies on `knex.raw`. Cycle-detection arrays (`ARRAY[...] || x`, `x <> ALL(path)`) drop to `knex.raw` anyway. **Verdict**: Knex has real `withRecursive` support and stays close to SQL — the best builder-level story here alongside Drizzle; use `knex.raw` for path arrays and `CYCLE`.

### ORM Summary Table

| ORM | Recursive CTE support | Mechanism | Cycle/path arrays | Verdict |
|-----|----------------------|-----------|-------------------|---------|
| Prisma | None (raw only) | `$queryRaw` | Raw SQL | Transport only; annotate row type |
| Drizzle | Typed builder + raw | `$withRecursive` / `sql` | `sql` escape hatch | Best typed attempt; raw for non-trivial |
| Sequelize | None (raw only) | `sequelize.query` | Raw SQL | Driver only, use `replacements` |
| TypeORM | `@Tree` + raw | closure/nested-set or `query()` | Raw SQL | `@Tree` for stable trees; raw for graphs |
| Knex | Real builder | `withRecursive` | `knex.raw` | Closest to SQL alongside Drizzle |

Across the board: the *recursive step's* interesting parts — path arrays, `CYCLE`, depth guards — are raw SQL in every ORM. Treat recursion as a raw-SQL feature you wrap, not something the ORM abstracts.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given:
- `categories(id, name, parent_id)` — a category tree; roots have `parent_id IS NULL`.

Write a recursive CTE that returns every descendant category of the category named `'Electronics'` (at any depth), showing `id`, `name`, and `depth` (Electronics itself is depth 0). Order by depth, then name.

```sql
-- Write your query here
```

Then: how would you also return the full breadcrumb path (e.g. `Electronics > Computers > Laptops`) for each row?

---

### Exercise 2 — Medium (combines prior topics)

Given:
- `employees(id, name, manager_id, department_id, salary)`
- `departments(id, name)`

For the executive with `id = 7`, return, for each level of their reporting tree, the level `depth`, the `headcount`, the `total_salary`, and the department name that has the most people at that level. Combine recursion (this topic), aggregation and `GROUP BY` (Topic 20), and a JOIN to `departments` (Topic 11). Exclude the executive themselves.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation, naive answer is wrong)

You maintain a `pages(id, title, redirect_to_id)` table for a wiki. `redirect_to_id` points to another page this page redirects to (or `NULL` if it is a real content page). Redirects can chain: A → B → C. Some pages, due to editor mistakes, form redirect **cycles** (A → B → A).

Write a query that, for every page that is a redirect, returns the *final* destination page it ultimately resolves to (following the chain to the end). For pages caught in a cycle, return `NULL` as the destination and flag them as broken. The naive `UNION ALL` chase will loop forever on the cyclic data — your answer must terminate and correctly distinguish "resolved" from "in a cycle."

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

Given `edges(src, dst)` representing a directed dependency graph of build tasks (an edge `a → b` means "a must run before b"), write a single query that:
1. Computes, for a given start task `$1`, all tasks transitively dependent on it (everything reachable).
2. Reports the minimum number of hops from `$1` to each such task.
3. Is safe against cycles in the data (a real build graph should be a DAG, but you cannot trust the data).

State your assumptions about indexes and explain, in a sentence, why your query terminates.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1: Explain how a recursive CTE actually executes. Is it real recursion?

**Junior answer**: "It calls itself repeatedly, like a recursive function, until it runs out of rows."

**Principal answer**: It is *iteration*, not call-stack recursion. PostgreSQL maintains a **working table** and a **result table**. The anchor (non-recursive) member runs once, seeding both. Then it loops: each iteration evaluates the recursive member with every self-reference reading *only the working table* (last iteration's output, not the whole accumulated result), appends the new rows to the result, and replaces the working table with just those new rows. The loop stops when an iteration produces zero rows. With `UNION` (not `UNION ALL`), each iteration also deduplicates the new rows against the accumulated result, which is what breaks cycles. In the plan this is the `Recursive Union` node with a `WorkTable Scan` child. Critically, the working table is unindexed and transient, so scaling depends on an index on the *base table's* join column.

**Follow-up the interviewer asks**: "If the working table is unindexed, why isn't every recursive step a full scan?" — Because the recursive step joins the working table *to the base table*; the base table's index on the join column (e.g., `manager_id`) turns each working-table row into an index probe. The working table itself is small (one level) and is scanned fully, which is fine.

---

### Q2: When do you need `UNION` vs `UNION ALL` in a recursive CTE, and what breaks if you choose wrong?

**Junior answer**: "`UNION ALL` is faster because it doesn't remove duplicates, so use it unless you want distinct rows."

**Principal answer**: The choice is about *cycle safety*, not just duplicates. `UNION ALL` never deduplicates, so on any cyclic graph it loops forever, spilling the tuplestore to disk until it times out or OOMs. `UNION` deduplicates each iteration's output against the accumulated result, so a revisited node is dropped and the working table eventually empties — automatic cycle protection. **But** `UNION` only dedups on the *entire row*, so the moment you carry a `depth` or `path` column that changes every iteration, the "same" node is no longer a duplicate and the protection silently fails. In practice I use `UNION ALL` plus an explicit cycle guard — a `path` array with `WHERE new_node <> ALL(path)`, or the `CYCLE` clause on PG 14+ — because that dedups on *node identity* which is what I actually mean, and it is cheaper than whole-row dedup.

**Follow-up**: "You added `UNION` for safety and your query still hangs. Why?" — Almost certainly a `depth` or path column making every row unique, defeating the whole-row dedup. Switch to identity-based cycle detection.

---

### Q3: A recursive CTE that traverses a 5-million-row org table is taking 20 seconds. Walk me through diagnosing it.

**Junior answer**: "Add a `LIMIT`, or increase `work_mem`."

**Principal answer**: First, `EXPLAIN (ANALYZE, BUFFERS)`. Look at the recursive child under `Recursive Union`. The classic culprit is a **`Seq Scan` there with high `loops=` and huge `Rows Removed by Filter`** — meaning there is no index on the join column (`manager_id`/`parent_id`), so every iteration full-scans the whole table. Fix: `CREATE INDEX ON employees(manager_id)`. Second, check the `loops` count and result `rows` — if `rows` exceeds the table size, the graph has a **cycle** exploding via `UNION ALL`; add a `CYCLE` clause or path guard. Third, check for temp-file buffers — a very wide level or a fat SELECT list spilling the tuplestore; narrow the columns carried through the recursion and only join for wide columns in the outer query. A `LIMIT` in the outer query does *not* help because the CTE materializes fully first; the bound must go *inside* the recursive term as a `depth <` guard. Increasing `work_mem` only helps a legitimately large-but-bounded result — it does nothing for a missing index or a cycle.

**Follow-up**: "You added the index and it's still slow." — Then either the traversal genuinely emits millions of rows (narrow the seed, cap depth, or precompute a closure table), or it is a cycle (verify with a path guard and check whether result rows exceed node count).

---

### Q4: How would you detect and safely handle cycles, and what does the `CYCLE` clause do under the hood?

**Junior answer**: "Use `UNION` instead of `UNION ALL`."

**Principal answer**: Two robust approaches. Manually: carry an array of visited node ids and add `WHERE candidate_id <> ALL(visited_path)` in the recursive term, plus append the current id to the path each step. This dedups on identity and terminates because a node can appear on a path at most once. Declaratively on PG 14+: `CYCLE id SET is_cycle USING path` — PostgreSQL maintains the `path` array for you, sets `is_cycle = true` when a node's `id` reappears on its own path, and *stops expanding that branch*. It is exactly the manual path-guard bookkeeping, standardized. `UNION`'s whole-row dedup is the fragile option because any monotonic column (depth) defeats it. In production over untrusted graph data I also add a hard `depth <` cap and a `statement_timeout` as defense in depth.

**Follow-up**: "What's the performance cost of the path array?" — It grows linearly with depth and is stored on every emitted row, so a 30-deep traversal over a million rows carries meaningful memory. If you only need termination and not the actual path, that cost is the price of safety; alternatives are the `CYCLE` clause (same cost, cleaner) or a precomputed closure table if the graph is stable.

---

## 15. Mental Model Checkpoint

1. In a recursive CTE, the recursive member joins to the CTE by name. When it reads that name during iteration 3, is it reading *all* rows produced so far, or only the rows produced by iteration 2? Why does the distinction matter for correctness and performance?

2. You write a graph traversal with `UNION` (no `ALL`) specifically to avoid infinite loops, but you also add a `depth + 1` column. The query hangs on cyclic data. Explain precisely why the cycle protection failed.

3. You put `WHERE depth <= 5` in the outer query rather than inside the recursive term. The result is correct but the query is slow on a 15-level tree. What extra work did the engine do, and where should the bound have gone?

4. A colleague claims "adding `LIMIT 100` to the outer SELECT makes the recursion stop after 100 rows." Under what circumstances is this true, and why should you never rely on it?

5. Your recursive query over a 2-million-row table runs in 8 ms in staging but 12 seconds in production. Same query, same-ish data volume. What is the single most likely difference, and how would you confirm it from `EXPLAIN`?

6. Why is it impossible to put an aggregate function like `SUM()` over the self-reference inside the recursive term, and what is the standard workaround to compute a subtree total?

7. You need a running compound-interest balance where each month's balance depends on the previous month's. Why can a window function *not* express this, making a recursive CTE the right tool — unlike the "running total" case where a window function is strictly better?

---

## 16. Quick Reference Card

```sql
-- ── Canonical shape ──────────────────────────────────────────────
WITH RECURSIVE cte AS (
    <anchor>                     -- seed; runs once; no self-reference
  UNION [ALL]                    -- ALL = fast/no dedup; UNION = dedup each round
    <recursive>                  -- references `cte`; runs until it yields 0 rows
)
SELECT ... FROM cte;             -- outer query reads FULLY materialized result

-- ── Termination ──────────────────────────────────────────────────
-- Implicit: stops when an iteration produces no new rows.
-- Engineer it: join stops matching, OR depth guard, OR UNION dedup breaks cycle.

-- ── Depth limiting (bound the LOOP, not the output) ─────────────
    ... JOIN cte c ... WHERE c.depth < 20      -- INSIDE recursive term  ✔
SELECT * FROM cte WHERE depth < 20;            -- outer filter = ran full first �’

-- ── Cycle detection ─────────────────────────────────────────────
-- (a) path guard (works everywhere):
    SELECT ..., c.path || n.id
    FROM cte c JOIN edges e ... JOIN nodes n ...
    WHERE n.id <> ALL(c.path)
-- (b) CYCLE clause (PG 14+):
) CYCLE id SET is_cycle USING path
  SELECT * FROM cte WHERE NOT is_cycle;
-- (c) UNION whole-row dedup — ONLY if no monotonic column (depth) is carried.

-- ── Ordered output ──────────────────────────────────────────────
ORDER BY path         -- depth-first, siblings preserved (carry ARRAY[id] path)
ORDER BY depth, ...   -- level order (carry a depth column)
) SEARCH DEPTH FIRST BY id SET ordercol  -- PG 14+ built-in ordering

-- ── Forbidden in the recursive term ────────────────────────────
-- aggregates, GROUP BY, HAVING, window funcs, DISTINCT, LIMIT/OFFSET,
-- OUTER JOIN / subquery referencing the CTE, self-reference more than once.

-- ── Sequence / series ──────────────────────────────────────────
WITH RECURSIVE s AS (SELECT 1 n UNION ALL SELECT n+1 FROM s WHERE n<10) ...
-- Prefer generate_series() unless each step depends on the previous value.

-- ── Perf rules of thumb ────────────────────────────────────────
-- 1. INDEX the base-table join column (parent_id/manager_id/edges.src). #1 lever.
-- 2. Bound depth INSIDE the recursion; it's also a security control.
-- 3. UNION ALL + identity cycle guard  >  UNION whole-row dedup.
-- 4. SELECT narrow — don't drag wide columns through every iteration.
-- 5. Recursive CTEs are ALWAYS materialized (never inlined) — outer LIMIT
--    does not prune the recursion in general.

-- ── Interview one-liners ───────────────────────────────────────
-- "It's iteration over a working table, not call-stack recursion."
-- "The working table is unindexed; the BASE table's index makes it fast."
-- "UNION dedups whole rows; a depth column defeats its cycle protection."
-- "Bound the loop inside the recursive term — the outer WHERE runs too late."
-- "Recursive CTEs are always materialized; you can't NOT MATERIALIZED them."
```

---

## Connected Topics

- **Topic 28 — Common Table Expressions (CTEs)**: The non-recursive foundation — `WITH` syntax, the materialization fence, readability. Recursive CTEs are the `RECURSIVE` extension of exactly that machinery.
- **Topic 30 — Subquery vs CTE vs JOIN**: When recursion is the *only* option (unknown-depth traversal, transitive closure) versus when a plain JOIN or subquery suffices; the decision framework that positions this topic.
- **Topic 11 — INNER JOIN in Depth**: Each recursive step is an INNER JOIN of the working table to the base table — the join algorithms, NULL behavior, and index rules carry over directly.
- **Topic 18 — The N+1 Query Problem**: The application-side per-level loop that recursive CTEs replace — one query instead of one-query-per-node.
- **Topic 20 — GROUP BY Fundamentals**: Aggregation is forbidden in the recursive term, so subtree rollups aggregate the finished CTE in the outer query.
- **Topic on Window Functions**: The right tool for *ordered running computations* (running totals, ranks) — contrast with recursion, which you reserve for hierarchy/graph problems and step-dependent series.
- **Internals — `RecursiveUnion`, WorkTable Scan, tuplestore, `work_mem`**: The physical operators and memory structures that make the working-table algorithm concrete in `EXPLAIN`.
- **Internals — B-tree indexes**: The index on the base table's join column is the single biggest performance lever for any recursive traversal.
