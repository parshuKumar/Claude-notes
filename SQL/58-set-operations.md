# Topic 58 — Set Operations
### SQL Mastery Curriculum — Phase 9: Advanced SQL Patterns

---

## 1. ELI5 — The Analogy That Makes It Click

You run two email newsletters, one for a cooking blog and one for a gardening blog. Each has its own signup sheet — a physical clipboard with names written down.

One morning your boss asks four different questions, and each one is a different "set operation":

1. **"Give me one combined mailing list of everyone across both blogs."**
   You stack both clipboards, copy every name into one big list. But wait — some people signed up for *both* blogs. Do you write their name once or twice? "Once," says your boss. So you go through, and every time you're about to write a duplicate, you skip it. That deduplication step is `UNION`. It's careful, and it's slow, because to know whether "Alice" already appears you have to keep checking against everything you've written so far.

2. **"Actually, I don't care about duplicates — just give me the raw combined stack, fast."**
   Now you just copy List A, then copy List B underneath it, and hand it over. Alice appears twice. You didn't check anything. That's `UNION ALL` — it just concatenates. It is dramatically faster because there is no checking step. This is almost always what you actually want, and paying for deduplication you don't need is the single most common set-operation mistake.

3. **"Who signed up for BOTH blogs?"**
   You go through List A, and for each name, check if it's also on List B. Only the names on both lists make the cut. That's `INTERSECT`.

4. **"Who signed up for the cooking blog but NOT the gardening blog?"**
   You go through List A, and cross off anyone who also appears on List B. What remains is List A minus List B. That's `EXCEPT` (called `MINUS` in Oracle).

The whole family of set operations is just these four verbs: *combine* (with or without dedup), *keep the overlap*, and *subtract*. The one insight that saves you the most money in production: **`UNION` secretly does a sort-or-hash to remove duplicates, and if you didn't need that, you should have written `UNION ALL`.**

---

## 2. Connection to SQL Internals

Set operations are fundamentally about **combining the row streams of two or more independent queries** and then applying a set-theoretic rule. Underneath, PostgreSQL implements each one differently, and the differences are exactly where the performance lives.

- **`UNION ALL`** is the cheap one. The executor uses an **Append** node (or **Merge Append** if a common sort order can be exploited). It runs the first subquery to completion, streaming its rows out, then runs the second, streaming those out. No buffering of the whole result, no hashing, no sorting. Memory is O(1) beyond each child's own needs. It is essentially free glue.

- **`UNION`** (without ALL) is `UNION ALL` plus a **deduplication** step. The planner sits a **HashAggregate** (hash the whole combined stream, keep one row per distinct value) or a **Unique** node over a **Sort** (sort the whole combined stream, then discard adjacent duplicates) on top of the Append. Either way you now pay: a hash table sized to the number of distinct rows (bounded by `work_mem`, spilling to disk in batches if it overflows), or an O(N log N) sort of the *entire* combined output. This is the "expensive sort/hash" the description warns about.

- **`INTERSECT`** and **`EXCEPT`** are implemented with a **SetOp** executor node fed by a **Sort** (or a **HashSetOp** fed by a hash table). Postgres tags each input row with a marker for which side it came from, sorts/hashes the combined stream, and then within each group of equal rows counts how many came from the left versus the right. The set rule decides what to emit: INTERSECT emits a row if both sides contributed; EXCEPT emits it if only the left side did. The `ALL` variants change the *count* arithmetic (see §6) rather than the algorithm.

Three internal facts drive everything in this topic:

1. **Duplicate elimination is not free** — it is a sort or a hash over the *combined* cardinality, and it is the dividing line between `UNION` and `UNION ALL`, and between the plain and `ALL` variants of INTERSECT/EXCEPT.
2. **NULLs are treated as equal to each other** for the purpose of set operations — the opposite of their behaviour in `WHERE` and `JOIN ... ON` (Topic 07). Two NULL rows are duplicates and collapse under `UNION`; a NULL row on both sides matches under `INTERSECT`.
3. **Set operations compare rows positionally by column, using the resolved common type**, not by column name. The engine coerces both branches to one output row type before it can hash or sort them.

---

## 3. Logical Execution Order Context

Set operations sit at an unusual place in the logical pipeline. Each branch of a set operation is a *complete* `SELECT` — it runs its own `FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT` internally and produces a finished row set. Only then are the branches combined.

```
( FROM → WHERE → GROUP BY → HAVING → SELECT )   ← branch 1 runs fully
        UNION / UNION ALL / INTERSECT / EXCEPT   ← branches combined here
( FROM → WHERE → GROUP BY → HAVING → SELECT )   ← branch 2 runs fully
        ↓
   set-level dedup (for UNION / INTERSECT / EXCEPT)
        ↓
   ORDER BY        ← applies to the WHOLE combined result, ONCE, at the end
        ↓
   LIMIT / OFFSET  ← applies to the WHOLE combined result
```

Consequences that trip people up constantly:

- **`ORDER BY` belongs to the whole statement, not a branch.** You may write exactly one `ORDER BY`, and it goes after the *last* branch. Putting `ORDER BY` inside a branch is either a syntax error or requires parentheses, and even then it only orders that branch's input — the set operation is free to reorder. To order the final combined result you place `ORDER BY` at the very end (see §6.6).
- **`ORDER BY` after a set operation can only reference the output columns**, by the first branch's column name or by ordinal position (`ORDER BY 1`). It cannot see a column that isn't in the SELECT list, because the branches have already been merged into an anonymous combined type.
- **`LIMIT` also applies to the combined result.** `SELECT ... UNION SELECT ... LIMIT 10` limits the merged output, not either branch.
- **The output column names come from the first branch.** Branch 2's aliases are ignored for naming purposes.
- **Deduplication happens after both branches are fully materialised logically** — which is why `UNION` can't start returning rows until it has seen enough to know what's distinct (a blocking step for the Sort-based plan; HashAggregate can stream once built).

Set operations also nest: `A UNION B EXCEPT C` is evaluated left-to-right by default, but `UNION`/`EXCEPT` have **lower precedence than `INTERSECT`**. `A UNION B INTERSECT C` means `A UNION (B INTERSECT C)`. When in doubt, parenthesise — it costs nothing and removes all ambiguity.

---

## 4. What Is a Set Operation?

A set operation takes two or more `SELECT` result sets that are **union-compatible** (same number of columns, with pairwise-compatible types) and combines them by a set-theoretic rule: union, intersection, or difference. By default (`UNION`, `INTERSECT`, `EXCEPT`) the result contains only **distinct** rows; the `ALL` modifier keeps duplicates and skips the deduplication work.

```sql
SELECT col_a, col_b FROM table_1          -- ┐ branch 1: a complete SELECT producing N columns
WHERE ...                                 -- │
                                          -- │
UNION [ALL]                               -- ← set operator
--  │    └── ALL      → keep every row, no dedup (fast: Append only)
--  └────── UNION     → combine both branches into one result set
--          INTERSECT → keep only rows present in BOTH branches
--          EXCEPT    → keep rows in branch 1 that are NOT in branch 2

SELECT col_x, col_y FROM table_2          -- ┐ branch 2: another complete SELECT
WHERE ...                                 -- │  must have the SAME column count as branch 1
                                          -- │  col_x must be type-compatible with col_a, etc.
ORDER BY col_a                            -- ← ONE order-by for the whole combined result
LIMIT 100;                                -- ← applies to the combined result
```

### The four operators, precisely

| Operator | Keeps | Duplicates | Set-theory meaning |
|----------|-------|-----------|--------------------|
| `UNION ALL` | all rows from both | **kept** | multiset sum (A ⊎ B) |
| `UNION` | rows from both, deduplicated | removed | set union (A ∪ B) |
| `INTERSECT` | rows in both, deduplicated | removed | set intersection (A ∩ B) |
| `INTERSECT ALL` | rows in both | kept per multiplicity | multiset intersection |
| `EXCEPT` | rows in A not in B, deduplicated | removed | set difference (A − B) |
| `EXCEPT ALL` | rows in A not in B | kept per multiplicity | multiset difference |

### Union compatibility — the two hard rules

1. **Same number of columns in every branch.** `SELECT a, b` cannot be unioned with `SELECT a, b, c`. Error: *"each UNION query must have the same number of columns."*
2. **Pairwise type compatibility, by position.** Column 1 of branch 2 must be coercible to a common type with column 1 of branch 1, and so on. `integer` and `numeric` resolve to `numeric`; `text` and `varchar` resolve to `text`; `integer` and `text` do **not** resolve and raise *"UNION types integer and text cannot be matched."* Names are irrelevant — only position and type.

The resolved output type follows the same type-resolution rules as a `CASE` expression: Postgres picks the type each branch can be implicitly cast to. The output column *name* is taken from the first branch.

---

## 5. Why Set Operations Mastery Matters in Production

1. **`UNION` vs `UNION ALL` is a silent performance tax.** The two are one keyword apart, look almost identical, and `UNION` is shorter — so developers reach for it by habit. But `UNION` forces a sort or hash over the *combined* row count on every execution. On a query stitching together six partitions of a 100M-row table, that's a multi-gigabyte sort you never asked for. When the branches are already provably disjoint (different date ranges, different status values, different source tables with no overlap), `UNION ALL` gives an identical result for a fraction of the cost. **This is the number-one set-operation optimisation in real systems.**

2. **Wrong results from accidental deduplication.** The flip side: `UNION` silently collapses rows that are genuinely distinct events but happen to have identical column values. If you `UNION` two logs of "payment attempts" and two legitimate attempts had the same amount, user, and truncated timestamp, `UNION` throws one away and your count is wrong. Knowing when dedup is semantically correct versus destructive is the difference between a right and a wrong report.

3. **`EXCEPT` and `INTERSECT` are the clearest way to express "what changed" / "what's common".** Reconciliation jobs (rows in the source not yet in the target), diffing two snapshots, finding customers in segment A but not segment B — these are naturally set difference. Written as `NOT EXISTS` or anti-joins they're more verbose and easier to get wrong; `EXCEPT` says exactly what it means. Knowing the NULL semantics (§6.4) is what makes them trustworthy.

4. **Type and column-count mismatches fail loudly — but the coercion can also succeed *silently and wrongly*.** If two branches both have a `NULL` literal in a position, or an `integer` that gets promoted to `numeric`, the query runs but the output type may not be what you assumed, breaking downstream code or an index-only comparison. Understanding the resolution rules prevents subtle type surprises.

5. **Set operations often out-perform the `OR` / big-`IN` alternative — or under-perform a JOIN.** A query that can't use an index because of an `OR` across two different columns can sometimes be rewritten as a `UNION` of two index-friendly branches (each branch uses its own index), a classic optimisation. Conversely, an `INTERSECT` that could have been a single indexed `JOIN` may leave performance on the table. Knowing which shape the planner handles best (§10) is a senior-level skill.

---

## 6. Deep Technical Content

### 6.1 `UNION ALL` — the fast concatenation

`UNION ALL` is the baseline. It appends branch 2's rows after branch 1's, keeping every row including duplicates. There is no sort, no hash, no dedup.

```sql
-- Combine current and archived orders into one stream — they are disjoint by design
SELECT id, customer_id, total_amount, created_at, 'live'    AS source FROM orders
UNION ALL
SELECT id, customer_id, total_amount, created_at, 'archive' AS source FROM orders_archive;
```

Because the two tables share no primary keys (archived rows were moved out of `orders`), there are no duplicates to remove. `UNION ALL` is not just faster here — it is *more correct*, because `UNION` would waste effort proving a disjointness you already know holds. Execution is an `Append` node; rows stream out as each child produces them, so `LIMIT` can short-circuit (see §7).

Row count of `UNION ALL` is exactly `count(branch1) + count(branch2)`. Always.

### 6.2 `UNION` — dedup and what "duplicate" means

`UNION` removes duplicate rows from the combined output. A "duplicate" is a row whose **entire tuple** is equal to another row's, column by column. Two rows that differ in any single column are distinct and both survive.

```sql
-- Distinct set of user IDs who did EITHER of two actions
SELECT user_id FROM sessions      WHERE started_at >= CURRENT_DATE
UNION
SELECT user_id FROM audit_logs    WHERE action = 'login' AND created_at >= CURRENT_DATE;
-- A user who both had a session and logged in appears ONCE
```

The engine must compare whole rows. It does this with either:

- **HashAggregate**: build a hash table keyed on the whole row; each distinct row is inserted once. O(distinct rows) memory. Streams output once the hash is built. Chosen when the estimated distinct count fits in `work_mem`.
- **Sort + Unique**: sort the combined stream on all columns, then a `Unique` node discards rows equal to their predecessor. O(N log N) time, O(N) sort memory (spills to disk if over `work_mem`). Chosen when a sort is cheap or already needed (e.g. for a following `ORDER BY`).

**Key equivalence:** `SELECT DISTINCT` over a `UNION ALL` is logically identical to `UNION`:

```sql
SELECT DISTINCT * FROM (
  SELECT user_id FROM sessions   ...
  UNION ALL
  SELECT user_id FROM audit_logs ...
) t;
-- == the UNION version above, and the planner may produce the same plan
```

Understanding this makes the cost obvious: `UNION` = `UNION ALL` + `DISTINCT`. If you don't need the `DISTINCT`, don't pay for it.

### 6.3 The `UNION` vs `UNION ALL` decision — a rule you can apply mechanically

Ask: **can the two branches ever produce an identical row?**

- **No, provably disjoint** → use `UNION ALL`. Examples: different tables with no shared identity; the same table split by a partitioning predicate (`status = 'open'` vs `status = 'closed'`); date ranges that don't overlap; branches that each select a distinct literal tag column. The tag-column trick (`'live'`/`'archive'` above) *guarantees* disjointness because the tag differs, so `UNION ALL` is always correct there.
- **Yes, and you want one copy** → use `UNION`. Example: "any user who appears in either list," where the same user genuinely can appear in both.
- **Yes, and duplicates are meaningful** → use `UNION ALL`. Example: combining two streams of events where two events legitimately coincide and both must be counted.

If you catch yourself writing `UNION` "just to be safe," stop: either the branches are disjoint (so `UNION ALL` is safe and faster) or they aren't (so you must think about whether dedup is what you actually want).

### 6.4 NULL handling — set operations treat NULL as equal to NULL

This is the single most important semantic distinction from the rest of SQL. Recall from Topic 07 that `NULL = NULL` is `UNKNOWN`, so a `JOIN ... ON a = b` drops NULL keys and `WHERE a = NULL` matches nothing. **Set operations do the opposite.** For deduplication, intersection, and difference, **two NULLs are considered equal (not distinct)** — this is "not distinct from" semantics, the same rule `GROUP BY` and `DISTINCT` use.

```sql
-- UNION collapses the two NULL rows into one
SELECT NULL::int AS x
UNION
SELECT NULL::int;
--  x
-- ----
-- (null)          -- ONE row, not two — the NULLs were treated as equal duplicates
```

```sql
-- INTERSECT: a NULL on both sides matches
SELECT NULL::int, 'a'::text
INTERSECT
SELECT NULL::int, 'a'::text;
-- returns one row (NULL, 'a') — because NULL is-not-distinct-from NULL here
```

```sql
-- EXCEPT: a NULL row in the left that also appears in the right is removed
SELECT NULL::int
EXCEPT
SELECT NULL::int;
-- returns ZERO rows — the left NULL was cancelled by the right NULL
```

And when comparing multi-column rows, the whole tuple is compared with NULL-equals-NULL semantics per column:

```sql
SELECT 1, NULL::int
INTERSECT
SELECT 1, NULL::int;   -- matches → one row (1, NULL)

SELECT 1, NULL::int
INTERSECT
SELECT 1, 5;           -- no match → zero rows (NULL ≠ 5)
```

**Practical consequence:** `EXCEPT` is a *safe* anti-join even in the presence of NULLs, whereas `NOT IN (subquery)` is notoriously broken when the subquery yields a NULL (it returns no rows at all). If you need "rows in A whose key isn't in B" and B might contain NULLs, `EXCEPT` behaves intuitively where `NOT IN` betrays you. (`NOT EXISTS`, Topic 27, is the other safe choice.)

### 6.5 `INTERSECT`, `EXCEPT`, and their `ALL` variants — the multiplicity arithmetic

The plain forms operate on sets (each row present 0 or 1 times in the output). The `ALL` forms operate on **multisets**, and the rule is counting-based. Let `m` = number of times a given row appears in the left branch, `n` = number of times it appears in the right branch.

| Operator | Output multiplicity of that row |
|----------|--------------------------------|
| `INTERSECT` | `1` if `m ≥ 1 AND n ≥ 1`, else `0` |
| `INTERSECT ALL` | `min(m, n)` |
| `EXCEPT` | `1` if `m ≥ 1 AND n = 0`, else `0` |
| `EXCEPT ALL` | `max(m − n, 0)` |
| `UNION` | `1` if `m ≥ 1 OR n ≥ 1` |
| `UNION ALL` | `m + n` |

Worked example. Left branch produces the value `7` three times; right branch produces `7` once.

```sql
-- Left: 7,7,7   Right: 7
VALUES (7),(7),(7)  INTERSECT      VALUES (7);   -- → 7            (set: present in both)
VALUES (7),(7),(7)  INTERSECT ALL  VALUES (7);   -- → 7            (min(3,1) = 1)
VALUES (7),(7),(7)  EXCEPT         VALUES (7);   -- → (nothing)    (7 is in right, so removed)
VALUES (7),(7),(7)  EXCEPT ALL     VALUES (7);   -- → 7,7          (max(3-1,0) = 2)
```

(In real Postgres you'd wrap these as `SELECT * FROM (VALUES ...) v(x)` or use `SELECT`; shown compact for the arithmetic.) The `EXCEPT ALL` result of two 7s is the surprising one and the reason `ALL` exists: it does *proportional* subtraction, useful for reconciling quantities (e.g., "expected shipments minus actual shipments, item by item").

`INTERSECT ALL` and `EXCEPT ALL` are far rarer than plain `INTERSECT`/`EXCEPT`, but when you need multiplicity-aware diffing they are exactly right and hard to reproduce with joins.

### 6.6 Ordering the combined result

`ORDER BY` attaches to the whole set operation and must be the last clause. It can reference output columns by the **first branch's column name** or by **ordinal position**.

```sql
SELECT id, name, 'employee' AS kind FROM employees
UNION ALL
SELECT id, name, 'contractor'       FROM contractors_view   -- alias here is ignored
ORDER BY name ASC, id;                 -- refers to branch-1 names; applies to whole result
```

```sql
-- Ordinal ordering is unambiguous and immune to the naming rule
SELECT id, total_amount FROM orders
UNION ALL
SELECT id, total_amount FROM orders_archive
ORDER BY 2 DESC;                        -- order by the 2nd output column (total_amount)
```

To sort *within* a branch before combining (rare — usually to feed a `LIMIT` per branch), you must parenthesise the branch:

```sql
( SELECT id, score FROM game_a ORDER BY score DESC LIMIT 3 )
UNION ALL
( SELECT id, score FROM game_b ORDER BY score DESC LIMIT 3 )
ORDER BY score DESC;                    -- final ordering of the 6 combined rows
```

Without the final `ORDER BY`, the combined order of a `UNION ALL` is **not guaranteed** — it happens to be branch-1-then-branch-2 today but is an implementation detail. Never rely on it.

### 6.7 Column names, types, and coercion in detail

- **Output names come from branch 1 only.** `SELECT a AS foo ... UNION SELECT b AS bar ...` yields a column named `foo`.
- **Type resolution is per-column across all branches**, using the same algorithm as `UNION`/`CASE`/`COALESCE`: find a type all branches can implicitly cast to. `smallint`+`integer`→`integer`; `integer`+`numeric`→`numeric`; `date`+`timestamp`→`timestamp`; `text`+`varchar(n)`→`text`. Incompatible pairs (e.g. `integer`+`text`, `json`+`jsonb` in some versions) error out.
- **Untyped NULL and untyped literals** take their type from the other branch. `SELECT NULL UNION SELECT 1` gives an `integer` column. `SELECT NULL UNION SELECT NULL` is `text` (Postgres's default for a wholly-unknown type) — pin it with `NULL::int` if downstream code cares.
- **Explicit casts remove ambiguity.** When branches differ, cast the narrower one: `SELECT amount::numeric ... UNION ALL SELECT 0::numeric`.

### 6.8 Number of branches and precedence

You can chain any number of branches. Evaluation is left-associative, **except** that `INTERSECT` binds tighter than `UNION` and `EXCEPT`:

```sql
A UNION B UNION C            -- ((A ∪ B) ∪ C)
A EXCEPT B EXCEPT C          -- ((A − B) − C)
A UNION B INTERSECT C        -- A ∪ (B ∩ C)   ← INTERSECT first!
(A UNION B) INTERSECT C      -- parenthesise to force the other grouping
```

Chained `UNION` (no ALL) deduplicates **once** over the whole thing, not pairwise — the planner typically stacks one Append with a single HashAggregate on top, which is efficient. Mixing `UNION` and `UNION ALL` in a chain gives per-operator semantics: `A UNION ALL B UNION C` first appends A and B, then deduplicates the whole against C's contribution — effectively deduplicating everything, because `UNION` dedups its combined left input too. If you meant "keep A and B duplicates but dedup against C," you need explicit parentheses and careful thought; usually it's clearer to restructure.

### 6.9 `CORRESPONDING`, `VALUES`, and generating rows

- Standard SQL has a `CORRESPONDING` clause to union by column name; **PostgreSQL does not implement it** — set ops are strictly positional. Align your SELECT lists yourself.
- `VALUES` lists are set-operation compatible and handy for building a constant reference set to `EXCEPT`/`INTERSECT` against:

```sql
-- Which of these expected status codes are MISSING from the table?
SELECT status FROM (VALUES ('open'),('paid'),('shipped'),('closed')) v(status)
EXCEPT
SELECT DISTINCT status FROM orders;
-- returns statuses that no order currently has
```

### 6.10 Set operations vs `DISTINCT`, `GROUP BY`, and joins — mental map

- `UNION` = `UNION ALL` + `DISTINCT` (dedup over combined rows).
- `INTERSECT` ≈ an inner semi-join on *all columns* + dedup — but expresses "match on the whole row" far more cleanly than spelling out every column in an `ON`.
- `EXCEPT` ≈ an anti-join on all columns + dedup, and crucially it is **NULL-safe** where `NOT IN` is not.
- `UNION ALL` has no join equivalent — it's pure vertical concatenation, orthogonal to joins (which are horizontal).

---

## 7. EXPLAIN — Set Operations in the Plan

### `UNION ALL` — just an Append (cheap)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, customer_id, total_amount FROM orders
UNION ALL
SELECT id, customer_id, total_amount FROM orders_archive;
```

```
Append  (cost=0.00..4360.00 rows=200000 width=20)
        (actual time=0.02..41.6 rows=200000 loops=1)
  ->  Seq Scan on orders          (cost=0.00..1730.00 rows=100000 width=20)
                                   (actual time=0.02..12.1 rows=100000 loops=1)
  ->  Seq Scan on orders_archive  (cost=0.00..1730.00 rows=100000 width=20)
                                   (actual time=0.01..12.0 rows=100000 loops=1)
Buffers: shared hit=1460
Planning Time: 0.18 ms
Execution Time: 48.9 ms
```

**Reading it:**
- A single `Append` node with two `Seq Scan` children — no dedup node anywhere.
- `rows=200000` = exactly the sum of the two children. No rows removed.
- No `Sort`, no `HashAggregate`, no extra memory line — this is why `UNION ALL` is cheap.

### `UNION` — Append plus a HashAggregate (pays for dedup)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT customer_id FROM orders
UNION
SELECT customer_id FROM orders_archive;
```

```
HashAggregate  (cost=5360.00..5410.00 rows=5000 width=8)
               (actual time=71.2..72.9 rows=4800 loops=1)
  Group Key: orders.customer_id
  Batches: 1  Memory Usage: 785kB
  ->  Append  (cost=0.00..4860.00 rows=200000 width=8)
              (actual time=0.02..40.1 rows=200000 loops=1)
        ->  Seq Scan on orders          (actual rows=100000 loops=1)
        ->  Seq Scan on orders_archive  (actual rows=100000 loops=1)
Buffers: shared hit=1460
Planning Time: 0.20 ms
Execution Time: 74.1 ms
```

**Reading it:**
- The `Append` still produces 200,000 rows, but a `HashAggregate` sits on top and collapses them to 4,800 distinct `customer_id`s.
- `Group Key: orders.customer_id` — the dedup is a group-by on the whole output row.
- `Memory Usage: 785kB`, `Batches: 1` — the hash table fit in `work_mem`. If `Batches > 1`, the hash spilled to disk and the query got much slower — the classic sign that `work_mem` is too small for a big `UNION`.
- Execution time rose from ~49ms (ALL) to ~74ms here on identical inputs; the gap widens dramatically as row counts grow and the hash spills.

### `UNION` that falls back to Sort + Unique

```sql
EXPLAIN (ANALYZE)
SELECT customer_id FROM orders
UNION
SELECT customer_id FROM orders_archive
ORDER BY customer_id;
```

```
Unique  (cost=24567.10..25067.10 rows=5000 width=8)
        (actual time=118.4..129.7 rows=4800 loops=1)
  ->  Sort  (cost=24567.10..25067.10 rows=200000 width=8)
            (actual time=118.4..124.9 rows=200000 loops=1)
        Sort Key: orders.customer_id
        Sort Method: quicksort  Memory: 14232kB
        ->  Append  (actual rows=200000 loops=1)
              ->  Seq Scan on orders          (actual rows=100000 loops=1)
              ->  Seq Scan on orders_archive  (actual rows=100000 loops=1)
Execution Time: 131.0 ms
```

**Reading it:**
- Because an `ORDER BY customer_id` was requested, the planner chose `Sort` + `Unique` (the sort serves double duty: dedup and ordering).
- `Sort Method: quicksort  Memory: 14232kB` — the whole 200k-row combined stream was sorted in memory. A large `UNION` here would show `Sort Method: external merge  Disk: NNNNkB` — a disk spill, the expensive case.
- Note this is the *slowest* of the three plans, illustrating that dedup-by-sort over a large combined set is the worst case.

### `INTERSECT` / `EXCEPT` — SetOp over a Sort

```sql
EXPLAIN (ANALYZE)
SELECT product_id FROM order_items WHERE order_id IN (SELECT id FROM orders WHERE status='paid')
INTERSECT
SELECT product_id FROM order_items WHERE order_id IN (SELECT id FROM orders WHERE status='refunded');
```

```
HashSetOp Intersect  (cost=... rows=1200 width=8)
                     (actual time=33.1..35.0 rows=1150 loops=1)
  ->  Append  (actual rows=42000 loops=1)
        ->  Subquery Scan on "*SELECT* 1"  (actual rows=25000 loops=1)
              ->  Hash Join  ...
        ->  Subquery Scan on "*SELECT* 2"  (actual rows=17000 loops=1)
              ->  Hash Join  ...
Execution Time: 36.2 ms
```

**Reading it:**
- `HashSetOp Intersect` is the node that implements the set rule. Each branch's rows are tagged by origin, hashed, and the node emits a value only if both branches contributed (§6.5).
- An `EXCEPT` shows `HashSetOp Except`; a sort-based plan shows `SetOp` under a `Sort`. `Intersect All`/`Except All` appear as `HashSetOp Intersect All` etc.

### What to look for

| Symptom in the plan | Meaning | Fix |
|---------------------|---------|-----|
| `HashAggregate`/`Unique` on top of `Append` | You used `UNION`, paying for dedup | If branches are disjoint, use `UNION ALL` |
| `Sort Method: external merge Disk:` under a UNION | Dedup sort spilled to disk | Raise `work_mem`, or switch to `UNION ALL` |
| `Batches > 1` in the dedup HashAggregate | Hash spilled to disk | Raise `work_mem`, or use `UNION ALL` |
| `Append` only, no dedup node | `UNION ALL` — optimal | nothing to do |
| `Merge Append` | Ordered children merged cheaply | good — an index provided the order |

---

## 8. Query Examples

### Example 1 — Basic: combine two disjoint sources with `UNION ALL`

```sql
-- Unified activity feed from two tables that never share a row.
-- Disjoint by construction, so UNION ALL is both correct and fast.
SELECT
  s.user_id,
  'session_start'      AS event_type,
  s.started_at         AS event_time
FROM sessions s
WHERE s.started_at >= CURRENT_DATE - INTERVAL '7 days'

UNION ALL

SELECT
  a.user_id,
  a.action             AS event_type,     -- alias ignored for naming; matches by position
  a.created_at         AS event_time
FROM audit_logs a
WHERE a.created_at >= CURRENT_DATE - INTERVAL '7 days'

ORDER BY event_time DESC                   -- one ORDER BY for the whole combined feed
LIMIT 100;
```

### Example 2 — Intermediate: `EXCEPT` for a reconciliation / "what's missing" check

```sql
-- Which active products have NEVER been ordered in a completed order?
-- EXCEPT is NULL-safe and reads exactly like the requirement.
SELECT p.id
FROM products p
WHERE p.active = TRUE

EXCEPT

SELECT oi.product_id                        -- product_id may be NULL for adjustments;
FROM order_items oi                         -- EXCEPT handles NULL sanely, unlike NOT IN
INNER JOIN orders o ON o.id = oi.order_id
WHERE o.status = 'completed';
-- Result: product ids that appear in the first set but not the second.
```

### Example 3 — Production Grade: index-friendly `UNION ALL` rewrite of an un-indexable `OR`

**Scenario.** `orders` has 50M rows. There are two separate B-tree indexes: `orders(customer_id)` and `orders(assigned_agent_id)`. The dashboard needs orders where a given person is *either* the customer *or* the assigned agent. A single `WHERE customer_id = $1 OR assigned_agent_id = $1` cannot use both indexes efficiently — Postgres either seq-scans or does a BitmapOr that reads both index ranges but still combines awkwardly. Splitting into two index-friendly branches and concatenating is often dramatically faster.

```sql
-- Index available: orders(customer_id), orders(assigned_agent_id), both B-tree.
-- Expectation: two Index Scans + Append, ~1-3ms, versus a 50M-row Seq Scan (~seconds).
SELECT id, customer_id, assigned_agent_id, total_amount, created_at, 'as_customer' AS role
FROM orders
WHERE customer_id = $1

UNION ALL

SELECT id, customer_id, assigned_agent_id, total_amount, created_at, 'as_agent' AS role
FROM orders
WHERE assigned_agent_id = $1
  AND customer_id <> $1          -- avoid double-counting a self-order; keeps branches disjoint
                                 -- (also lets us safely keep UNION ALL)
ORDER BY created_at DESC
LIMIT 50;
```

```sql
EXPLAIN (ANALYZE, BUFFERS)  -- for the query above
```

```
Limit  (cost=8.90..9.02 rows=50 width=44) (actual time=0.31..0.44 rows=50 loops=1)
  ->  Sort  (cost=8.90..9.15 rows=98 width=44) (actual time=0.31..0.36 rows=50 loops=1)
        Sort Key: orders.created_at DESC
        Sort Method: top-N heapsort  Memory: 34kB
        ->  Append  (cost=0.56..5.62 rows=98 width=44) (actual time=0.04..0.24 rows=91 loops=1)
              ->  Index Scan using orders_customer_id_idx on orders
                    Index Cond: (customer_id = $1)          (actual rows=61 loops=1)
              ->  Index Scan using orders_assigned_agent_id_idx on orders orders_1
                    Index Cond: (assigned_agent_id = $1)
                    Filter: (customer_id <> $1)             (actual rows=30 loops=1)
Buffers: shared hit=14
Planning Time: 0.22 ms
Execution Time: 0.52 ms
```

**Why it wins:** each branch uses its own dedicated index (`Index Cond`), the `Append` stitches the two small result sets, and `top-N heapsort` handles the `LIMIT 50` without sorting everything. Buffers touched: 14 pages, versus millions for a seq scan. This is the canonical "`OR` across two indexed columns → `UNION ALL` of two branches" optimisation.

---

## 9. Wrong → Right Patterns

### Wrong 1: `UNION` where `UNION ALL` was meant — silent performance tax

```sql
-- WRONG: stitching 12 monthly partitions with UNION
SELECT * FROM orders_2025_01
UNION
SELECT * FROM orders_2025_02
UNION
-- ... ten more ...
SELECT * FROM orders_2025_12;
-- The partitions are DISJOINT (a row lives in exactly one month), so there are
-- ZERO duplicates to remove. But UNION still hashes/sorts the ENTIRE combined
-- result — potentially hundreds of millions of rows — to prove that.
-- EXPLAIN shows a giant HashAggregate or an external-merge Sort on top of Append.
```

**Why it's wrong at the execution level:** the planner has no way to know the partitions are disjoint, so it dutifully deduplicates. That's an O(N log N) sort or an O(N) hash over the full union cardinality, plus likely a disk spill (`Batches > 1` / `external merge`). You pay seconds for a guarantee you didn't need.

```sql
-- RIGHT: the branches are provably disjoint → UNION ALL
SELECT * FROM orders_2025_01
UNION ALL
SELECT * FROM orders_2025_02
UNION ALL
-- ...
SELECT * FROM orders_2025_12;
-- Plan is a plain Append: rows stream straight through, no dedup node, O(1) extra memory.
-- (Better still, use a native partitioned table so the planner appends automatically.)
```

### Wrong 2: `UNION` silently deleting legitimately-distinct rows

```sql
-- WRONG: combining two logs of payment attempts with UNION
SELECT user_id, amount, attempted_at::date AS day FROM gateway_a_attempts
UNION
SELECT user_id, amount, attempted_at::date AS day FROM gateway_b_attempts;
-- BUG: if the same user tried the same amount on the same day through BOTH
-- gateways, those are TWO real attempts, but UNION collapses them into ONE.
-- Any COUNT(*) over this is now undercounted.
```

**Why it's wrong:** `UNION` defines "duplicate" as "identical tuple," but here identical tuples are distinct real-world events. Dedup is semantically destructive.

```sql
-- RIGHT: keep every attempt with UNION ALL (add a source tag if you need provenance)
SELECT user_id, amount, attempted_at::date AS day, 'a' AS gateway FROM gateway_a_attempts
UNION ALL
SELECT user_id, amount, attempted_at::date AS day, 'b' AS gateway FROM gateway_b_attempts;
-- Every attempt survives; the gateway tag also makes the branches provably disjoint.
```

### Wrong 3: `NOT IN` anti-join broken by NULL — where `EXCEPT` is safe

```sql
-- WRONG: "customers who have never placed an order"
SELECT id FROM customers
WHERE id NOT IN (SELECT customer_id FROM orders);
-- BUG: orders.customer_id can be NULL (orphaned orders). If the subquery returns
-- even ONE NULL, `id NOT IN (... NULL ...)` evaluates to UNKNOWN for every row,
-- so the whole query returns ZERO rows. Silent, catastrophic.
```

**Why it's wrong at the execution level:** `x NOT IN (a, b, NULL)` is `x<>a AND x<>b AND x<>NULL`; the last term is `UNKNOWN`, dragging the whole conjunction to `UNKNOWN` (never TRUE). This is the three-valued-logic trap from Topic 07.

```sql
-- RIGHT: EXCEPT uses NULL-equals-NULL set semantics and is immune to the trap
SELECT id FROM customers
EXCEPT
SELECT customer_id FROM orders;      -- a NULL in the right side does not poison the result
-- (NOT EXISTS, Topic 27, is the other correct choice and often the fastest.)
```

### Wrong 4: mismatched column count / order

```sql
-- WRONG: branches don't line up positionally
SELECT id, name, email FROM users
UNION ALL
SELECT id, email FROM contractors;      -- 3 columns vs 2
-- ERROR: each UNION query must have the same number of columns

-- WRONG (worse — runs but wrong): columns in different order, same count/type family
SELECT id, name FROM users
UNION ALL
SELECT name, id FROM legacy_users;      -- if both are text/int-compatible this may COERCE
-- and silently interleave names into the id column. Positional matching bites hard.
```

```sql
-- RIGHT: align by position, explicitly, with matching types
SELECT id, name FROM users
UNION ALL
SELECT id, name FROM legacy_users
ORDER BY name;
```

### Wrong 5: `ORDER BY` inside a branch expecting it to order the result

```sql
-- WRONG: ORDER BY misplaced; also many engines reject it mid-statement
SELECT id, created_at FROM orders ORDER BY created_at   -- ← error or ignored here
UNION ALL
SELECT id, created_at FROM orders_archive;
-- The set operator does not preserve a branch's internal order.

-- RIGHT: one ORDER BY, at the very end, referencing an output column or ordinal
SELECT id, created_at FROM orders
UNION ALL
SELECT id, created_at FROM orders_archive
ORDER BY created_at DESC;      -- or: ORDER BY 2 DESC
```

---

## 10. Performance Profile

### Cost model per operator

| Operator | Extra work beyond scanning branches | Memory | Blocking? |
|----------|-------------------------------------|--------|-----------|
| `UNION ALL` | none (Append) | O(1) | no — streams |
| `UNION` | dedup: HashAggregate or Sort+Unique over combined rows | O(distinct) hash, or O(N) sort | hash: after build; sort: fully blocking |
| `INTERSECT`/`EXCEPT` | SetOp/HashSetOp over combined rows | O(distinct) | similar to UNION |
| `INTERSECT ALL`/`EXCEPT ALL` | same node, multiplicity arithmetic | O(distinct) | similar |

### Scaling with row count

Let the combined input be N rows.

| N (combined) | `UNION ALL` | `UNION` (hash, fits work_mem) | `UNION` (sort spills to disk) |
|--------------|-------------|-------------------------------|-------------------------------|
| 1M | ~instant append | +10–40ms hash build | n/a (fits) |
| 10M | streams, bounded by scan | +150–500ms, ~hundreds of MB hash | external merge, 2–5× slower |
| 100M | scan-bound only | hash likely spills → `Batches > 1` | multi-GB external sort, seconds–minutes |

The takeaway: `UNION ALL` scales with the cost of reading the branches and nothing more. `UNION` adds a term proportional to N that, once it exceeds `work_mem`, jumps to disk and degrades sharply. At 100M rows the difference between `UNION` and `UNION ALL` can be the difference between a sub-second query and a minute.

### `work_mem` and spilling

- The dedup hash / sort for `UNION`, `INTERSECT`, `EXCEPT` is bounded by `work_mem` (per-node, per-connection).
- Overflow → `HashAggregate` shows `Batches > 1`, or `Sort` shows `Sort Method: external merge Disk: NNNNkB`. Both mean disk I/O in the hot path.
- Mitigations: (a) eliminate the dedup entirely by proving disjointness and using `UNION ALL`; (b) raise `work_mem` for the session (`SET work_mem = '256MB'`) if the dedup is genuinely required; (c) reduce N by filtering *inside* each branch before combining.

### Index interactions

- **`UNION ALL` preserves each branch's ability to use its own indexes** — this is the whole point of the "OR → UNION ALL" rewrite (§8, Example 3). Each branch is planned independently.
- **`UNION`'s dedup can sometimes be satisfied by an index** that already delivers sorted, unique output, letting the planner use `Merge Append` + `Unique` cheaply instead of a big hash — but only if the branch outputs are individually ordered on the dedup key.
- **A trailing `ORDER BY` + `LIMIT` over `UNION ALL`** can use per-branch ordered index scans feeding a `Merge Append`, then stop early — near-optimal for "top N across several tables."
- `INTERSECT`/`EXCEPT` generally cannot push their whole-row comparison into an index unless the branches are pre-sorted on all output columns; usually they hash.

### Optimisation checklist specific to set operations

1. Default to `UNION ALL`; use `UNION` only when dedup is *semantically required*.
2. Filter early — put the most selective `WHERE` inside each branch so N is small before combining.
3. Add a constant tag column to make branches provably disjoint (and to keep provenance) — this justifies `UNION ALL`.
4. For "top N across sources," give each branch an index on the `ORDER BY` column so `Merge Append` can short-circuit under the `LIMIT`.
5. Watch EXPLAIN for `Batches > 1` / `external merge` under a `UNION`; that's your signal to switch to `UNION ALL` or raise `work_mem`.
6. Prefer a native **partitioned table** over hand-written `UNION ALL` across partition tables — the planner appends and prunes automatically.

---

## 11. Node.js Integration

### 11.1 Basic `UNION ALL` with `pg`

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Unified recent-activity feed from two disjoint sources
async function getRecentActivity(days = 7, limit = 100) {
  const { rows } = await pool.query(
    `SELECT s.user_id, 'session_start' AS event_type, s.started_at AS event_time
       FROM sessions s
      WHERE s.started_at >= NOW() - ($1 || ' days')::INTERVAL
     UNION ALL
     SELECT a.user_id, a.action AS event_type, a.created_at AS event_time
       FROM audit_logs a
      WHERE a.created_at >= NOW() - ($1 || ' days')::INTERVAL
     ORDER BY event_time DESC
     LIMIT $2`,
    [days, limit]                 // $1 reused across both branches — fully parameterised
  );
  return rows;
}
```

Note that a bind parameter (`$1`) can be referenced in **both** branches — the parameter list is per-statement, not per-branch. This keeps the "OR across two columns" rewrite clean:

### 11.2 The index-friendly `UNION ALL` rewrite, parameterised

```javascript
// Orders where a person is either the customer OR the assigned agent — each branch
// hits its own index. See §8 Example 3 for the plan.
async function ordersInvolvingPerson(personId, limit = 50) {
  const { rows } = await pool.query(
    `SELECT id, customer_id, assigned_agent_id, total_amount, created_at, 'as_customer' AS role
       FROM orders
      WHERE customer_id = $1
     UNION ALL
     SELECT id, customer_id, assigned_agent_id, total_amount, created_at, 'as_agent' AS role
       FROM orders
      WHERE assigned_agent_id = $1 AND customer_id <> $1
     ORDER BY created_at DESC
     LIMIT $2`,
    [personId, limit]
  );
  return rows;
}
```

### 11.3 `EXCEPT` for a safe anti-join

```javascript
// Customers who have never placed an order — NULL-safe, unlike NOT IN.
async function customersWithNoOrders() {
  const { rows } = await pool.query(
    `SELECT id FROM customers
     EXCEPT
     SELECT customer_id FROM orders`
  );
  return rows.map(r => r.id);
}
```

### 11.4 Building the disjoint-tag pattern dynamically (careful with identifiers)

```javascript
// Combine an arbitrary set of monthly partition tables with UNION ALL.
// Table names come from a TRUSTED allow-list, never user input — identifiers
// cannot be bound with $1; only values can. Validate against a whitelist.
const ALLOWED = new Set(['orders_2025_01', 'orders_2025_02', 'orders_2025_03']);

async function unionMonths(tableNames) {
  for (const t of tableNames) {
    if (!ALLOWED.has(t)) throw new Error(`refusing unknown table ${t}`);
  }
  const branches = tableNames
    .map(t => `SELECT id, customer_id, total_amount, created_at FROM ${t}`)
    .join('\nUNION ALL\n');
  const { rows } = await pool.query(`${branches}\nORDER BY created_at DESC LIMIT 200`);
  return rows;
}
```

**A note on ORMs:** most ORMs do *not* model set operations as first-class query-builder methods, or model them only partially (see §12). For anything beyond the simplest `UNION`, the idiomatic Node path is a raw `pool.query()` / `$queryRaw`. Set operations are one of the areas where dropping to SQL is the norm, not a workaround.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do set operations?** — Not through its type-safe query API. There is no `.union()` / `.intersect()` / `.except()` builder. You must use `$queryRaw`.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

// UNION ALL via raw SQL — the only route in Prisma
const feed = await prisma.$queryRaw<Array<{ user_id: number; event_type: string; event_time: Date }>>`
  SELECT s.user_id, 'session_start' AS event_type, s.started_at AS event_time
    FROM sessions s
   WHERE s.started_at >= NOW() - INTERVAL '7 days'
  UNION ALL
  SELECT a.user_id, a.action, a.created_at
    FROM audit_logs a
   WHERE a.created_at >= NOW() - INTERVAL '7 days'
  ORDER BY event_time DESC
  LIMIT 100
`;
```

**Where it breaks:** no compile-time column/type checking of the union branches; you hand-type the result. Interpolating dynamic table names needs `Prisma.raw()` with your own allow-listing.

**Verdict:** Set operations are always raw SQL in Prisma. Accept it and keep the SQL in a well-named repository function.

---

### Drizzle ORM

**Can Drizzle do set operations?** — Yes, and it's the standout here. Drizzle exposes `union`, `unionAll`, `intersect`, `intersectAll`, `except`, `exceptAll` as typed combinators.

```typescript
import { db } from './db';
import { sessions, auditLogs, customers, orders } from './schema';
import { union, unionAll, except } from 'drizzle-orm/pg-core';
import { gte, sql, eq } from 'drizzle-orm';

// UNION ALL — branches are type-checked to have matching column shapes
const feed = await unionAll(
  db.select({
      userId: sessions.userId,
      eventType: sql<string>`'session_start'`.as('event_type'),
      eventTime: sessions.startedAt,
    })
    .from(sessions)
    .where(gte(sessions.startedAt, sql`NOW() - INTERVAL '7 days'`)),
  db.select({
      userId: auditLogs.userId,
      eventType: auditLogs.action,
      eventTime: auditLogs.createdAt,
    })
    .from(auditLogs)
    .where(gte(auditLogs.createdAt, sql`NOW() - INTERVAL '7 days'`)),
).orderBy(sql`event_time DESC`).limit(100);

// EXCEPT — NULL-safe anti-join, fully typed
const noOrderCustomers = await except(
  db.select({ id: customers.id }).from(customers),
  db.select({ id: orders.customerId }).from(orders),
);
```

**Where it breaks:** the two branches must produce structurally identical selection shapes or the types won't line up — which is a feature, catching the column-mismatch bug at compile time. Ordinal `ORDER BY` still goes through `sql`.

**Verdict:** Best-in-class. If set operations matter to your app and you're picking an ORM, Drizzle handles them natively and safely.

---

### Sequelize

**Can Sequelize do set operations?** — No native builder. Use `sequelize.query()` with raw SQL.

```javascript
const { QueryTypes } = require('sequelize');

const rows = await sequelize.query(
  `SELECT id FROM customers
   EXCEPT
   SELECT customer_id FROM orders`,
  { type: QueryTypes.SELECT }
);
```

**Where it breaks:** everything about the set operation is opaque to the ORM — no model mapping, no eager loading across the union. You get plain objects.

**Verdict:** Raw query only. Wrap it in a model static method for tidiness.

---

### TypeORM

**Can TypeORM do set operations?** — Only partially. The QueryBuilder has no `.union()` method (as of current stable releases); the pragmatic approach is `dataSource.query()` raw SQL, or hand-composing sub-query SQL strings.

```typescript
// Raw SQL is the reliable path
const rows = await dataSource.query(
  `SELECT product_id FROM order_items oi
     JOIN orders o ON o.id = oi.order_id AND o.status = 'paid'
   INTERSECT
   SELECT product_id FROM order_items oi
     JOIN orders o ON o.id = oi.order_id AND o.status = 'refunded'`
);
```

Some versions let you feed a raw union string into `.from(qb => ..., alias)` sub-selects, but it's brittle. **Where it breaks:** no type safety, entity hydration doesn't span the set operation.

**Verdict:** Use `dataSource.query()`. Don't fight the QueryBuilder for this.

---

### Knex.js

**Can Knex do set operations?** — Yes: `.union()`, `.unionAll()`, `.intersect()`. `.except()` support depends on version/dialect; for Postgres you can fall back to `.with()`/raw if missing.

```javascript
// Knex builds each branch as a callback; unionAll takes an array of them.
const feed = await knex
  .select('user_id', knex.raw(`'session_start' AS event_type`), 'started_at AS event_time')
  .from('sessions')
  .where('started_at', '>=', knex.raw(`NOW() - INTERVAL '7 days'`))
  .unionAll([
    knex
      .select('user_id', 'action AS event_type', 'created_at AS event_time')
      .from('audit_logs')
      .where('created_at', '>=', knex.raw(`NOW() - INTERVAL '7 days'`)),
  ])
  .orderBy('event_time', 'desc')
  .limit(100);

// The boolean arg toggles UNION vs UNION ALL: .union(cb, true) === UNION ALL
```

**Where it breaks:** the branch column lists must align positionally — Knex won't check this for you. `ORDER BY`/`LIMIT` on the outer builder correctly apply to the combined result. `EXCEPT ALL`/`INTERSECT ALL` usually need raw.

**Verdict:** Solid, SQL-transparent support for `UNION`/`UNION ALL`/`INTERSECT`. Drop to `knex.raw()` for the `ALL` variants of intersect/except.

---

### ORM Summary Table

| ORM | Set-op support | `UNION ALL` | `INTERSECT`/`EXCEPT` | Type-checked branches | Verdict |
|-----|----------------|-------------|----------------------|-----------------------|---------|
| Prisma | None (raw only) | `$queryRaw` | `$queryRaw` | No | Always raw SQL |
| Drizzle | Native, all six | `unionAll()` | `intersect()`/`except()` (+`All`) | **Yes** | Best support |
| Sequelize | None (raw only) | `sequelize.query()` | `sequelize.query()` | No | Raw query |
| TypeORM | Minimal | `dataSource.query()` | `dataSource.query()` | No | Use raw query |
| Knex | Native `union`/`unionAll`/`intersect` | `.unionAll()` | `.intersect()`, `except` varies | No | Good; raw for `ALL` variants |

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given:
- `employees(id, name, email, department_id)`
- `contractors(id, name, email, agency)`

Write a single query returning one combined directory of `name` and `email` for **everyone** — employees and contractors — sorted by `name`. Assume an employee and a contractor never share an identity, so no deduplication is needed. Which operator do you use, and why is the other one wrong here on performance grounds?

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines prior topics)

Given:
- `orders(id, customer_id, status, created_at)`
- `order_items(id, order_id, product_id, quantity)`

Write a query that returns the `product_id`s that appear in **both** completed orders **and** refunded orders (i.e., products that were bought and also refunded), deduplicated. Then write a second version that instead returns products in completed orders **but never** refunded. Use set operations, and make sure the join to `orders` for status filtering happens inside each branch (recall INNER JOIN, Topic 11).

```sql
-- Write your query here (INTERSECT version)


-- Write your query here (EXCEPT version)
```

---

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

`orders` has 80M rows with two B-tree indexes: `orders(customer_id)` and `orders(referrer_id)`. A support tool must return, for a given `person_id`, every order where the person is **either** the customer **or** the referrer, newest first, limited to 100. A teammate wrote:

```sql
SELECT * FROM orders
WHERE customer_id = $1 OR referrer_id = $1
ORDER BY created_at DESC
LIMIT 100;
```

It runs in 6 seconds. (1) Explain from the execution plan why it's slow. (2) Rewrite it so each condition uses its own index and it returns in single-digit milliseconds. (3) Your rewrite must not return the same order twice when the person is *both* customer and referrer of an order — do this without paying for a full dedup. Explain why your approach is still correct.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

You're given two snapshots of a `product_catalog` table exported on consecutive days:
- `catalog_yesterday(sku, price, active)`
- `catalog_today(sku, price, active)`

Using set operations only (no joins), produce a "diff":
1. Rows that are **new or changed** today (present today, not identical yesterday).
2. Rows that were **removed or changed** since yesterday (present yesterday, not identical today).

Then explain: how does NULL in the `price` column behave in your `EXCEPT`, and why is that the behaviour you want here (contrast with what `NOT IN` would do)?

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1 — What is the difference between `UNION` and `UNION ALL`, and which should be the default?

**Junior answer:** "`UNION` removes duplicates and `UNION ALL` keeps them."

**Principal answer:** Both concatenate the rows of two union-compatible queries. `UNION ALL` is a pure `Append` — it streams the branches out one after another with no extra work, O(1) additional memory. `UNION` is `UNION ALL` plus a deduplication step: the planner puts a `HashAggregate` (or `Sort` + `Unique`) over the combined output to remove whole-row duplicates, which costs a hash or sort over the *entire* combined cardinality and spills to disk past `work_mem`. So the correct default is **`UNION ALL`** — you reach for `UNION` only when duplicates can actually occur *and* removing them is semantically what you want. Reaching for `UNION` reflexively is a silent, per-execution performance tax, and worse, it can delete legitimately-distinct rows that happen to share column values.

**Interviewer follow-up:** *"You have twelve monthly partition tables and you union them. Which operator, and how would you confirm your choice in production?"* — `UNION ALL`, because a row lives in exactly one month so the branches are disjoint and there is nothing to dedup. Confirm with `EXPLAIN ANALYZE`: the `UNION ALL` plan is a bare `Append`; a mistaken `UNION` shows a `HashAggregate`/`Unique` node and, on large data, `Batches > 1` or `Sort Method: external merge Disk:` — the tell-tale of an unnecessary spill.

---

### Q2 — How do set operations treat NULL, and why does that make `EXCEPT` safer than `NOT IN`?

**Junior answer:** "NULL is unknown so it probably gets dropped."

**Principal answer:** Set operations use "not distinct from" semantics — **NULL is considered equal to NULL** for dedup, intersection, and difference. This is the opposite of `=`/`JOIN ... ON`/`WHERE`, where `NULL = NULL` is `UNKNOWN`. So `UNION` collapses two NULL rows into one, `INTERSECT` matches a NULL on both sides, and `EXCEPT` cancels a left NULL against a right NULL. This is exactly why `EXCEPT` is a reliable anti-join: it does the intuitive thing with NULLs. `NOT IN (subquery)`, by contrast, is broken when the subquery yields any NULL — `x NOT IN (..., NULL)` becomes `UNKNOWN` for every `x`, so the query silently returns zero rows. `EXCEPT` (and `NOT EXISTS`) sidestep that trap entirely.

**Interviewer follow-up:** *"So are `EXCEPT` and `NOT EXISTS` interchangeable?"* — Semantically close for the "rows in A not in B" case, but not identical: `EXCEPT` compares the *entire* selected tuple and deduplicates its output, whereas `NOT EXISTS` compares whatever you write in its correlated predicate and preserves A's duplicates. `NOT EXISTS` is often faster (it can anti-join with an index and stop at the first match) and lets you compare on a subset of columns; `EXCEPT` is terser when you truly mean whole-row difference.

---

### Q3 — Explain the `ALL` variants of `INTERSECT` and `EXCEPT`. When would you use `EXCEPT ALL`?

**Junior answer:** "They keep duplicates like `UNION ALL` does."

**Principal answer:** They switch from set to multiset semantics with counting arithmetic. If a row appears `m` times on the left and `n` times on the right: `INTERSECT ALL` emits it `min(m, n)` times; `EXCEPT ALL` emits it `max(m − n, 0)` times (the plain forms emit it 0 or 1 time based on presence). `EXCEPT ALL` is the tool for **quantity reconciliation** — e.g. "expected shipments minus fulfilled shipments, line by line": if you expected an item 3 times and shipped it once, `EXCEPT ALL` correctly leaves 2 outstanding, which neither a join nor plain `EXCEPT` expresses cleanly.

**Interviewer follow-up:** *"Left has value 7 three times, right has it once. What does each of the four intersect/except forms return?"* — `INTERSECT` → one 7; `INTERSECT ALL` → one 7 (min(3,1)); `EXCEPT` → nothing (7 is present on the right); `EXCEPT ALL` → two 7s (max(3−1,0)).

---

### Q4 — A query filters with `WHERE a = $1 OR b = $1` and is slow despite indexes on both `a` and `b`. Why, and how do set operations help?

**Junior answer:** "Add a composite index on (a, b)."

**Principal answer:** A composite `(a, b)` index doesn't help an `OR` across the two columns — the two predicates target different leading columns, so a single index scan can't satisfy both. Postgres either seq-scans or builds a `BitmapOr` of two index scans, which on a very large table with a `LIMIT`/`ORDER BY` often can't short-circuit well. Rewriting as `SELECT ... WHERE a = $1 UNION ALL SELECT ... WHERE b = $1 AND a <> $1` lets **each branch use its own index** independently, `Append`s the two small result sets, and with a trailing `ORDER BY ... LIMIT` uses a top-N heapsort or `Merge Append` to stop early. The `a <> $1` in the second branch keeps the branches disjoint so `UNION ALL` stays correct without paying for dedup. This turns a multi-second seq scan into a millisecond indexed lookup.

**Interviewer follow-up:** *"Why `UNION ALL` with a guard predicate instead of plain `UNION`?"* — Plain `UNION` would also produce correct results by deduping the overlap (rows where `a = $1 AND b = $1`), but it forces a hash/sort over the combined output. The `a <> $1` guard makes the branches provably disjoint, so `UNION ALL` gives the identical result with zero dedup cost.

---

## 15. Mental Model Checkpoint

1. Branch 1 returns 1,000,000 rows and branch 2 returns 500,000 rows, and the two sets overlap in 200,000 identical rows. How many rows does `UNION ALL` return? How many does `UNION` return? Which one does extra work, and what work?

2. You write `SELECT NULL::int UNION SELECT NULL::int`. How many rows come back, and how does this contradict what `WHERE x = NULL` would do?

3. You need "products in completed orders but not in refunded orders." You write it with `NOT IN` and it returns zero rows even though you know there are such products. What's the most likely cause, and which set operator fixes it for free?

4. In `A UNION B INTERSECT C`, which operation happens first? How would you force `(A UNION B)` to happen first instead?

5. Your `UNION` query's `EXPLAIN ANALYZE` shows `Sort Method: external merge  Disk: 512000kB`. What does that tell you, and what are your two options to fix it?

6. Left branch has the value `x` four times; right branch has it once. What does `EXCEPT ALL` return for `x`, and what does plain `EXCEPT` return? Why do they differ?

7. You put `ORDER BY created_at` inside the first branch of a `UNION ALL`, expecting the whole result sorted. Why isn't the combined output reliably sorted, and where must the `ORDER BY` go?

---

## 16. Quick Reference Card

```sql
-- ── The four operators ──────────────────────────────────────────
A UNION ALL B      -- concatenate, keep dupes      → Append (fast, O(1) mem)
A UNION B          -- concatenate, drop dupes       → Append + HashAgg/Sort (pays!)
A INTERSECT B      -- rows in BOTH, deduped
A EXCEPT B         -- rows in A not in B, deduped   (NULL-safe anti-join)
A INTERSECT ALL B  -- min(m,n) copies of each row
A EXCEPT ALL B     -- max(m-n,0) copies of each row

-- ── The default rule ────────────────────────────────────────────
-- Use UNION ALL unless duplicates can occur AND you want them removed.
-- UNION = UNION ALL + DISTINCT. Don't pay for DISTINCT you don't need.

-- ── Union compatibility ─────────────────────────────────────────
-- 1. Every branch: SAME number of columns.
-- 2. Columns match by POSITION, not name. Types must be coercible.
-- 3. Output column NAMES come from the FIRST branch. Branch-2 aliases ignored.

-- ── NULL semantics (opposite of WHERE/JOIN) ─────────────────────
NULL is EQUAL to NULL here → UNION collapses NULLs, INTERSECT matches them,
EXCEPT cancels them. This is why EXCEPT is safe where NOT IN breaks on NULL.

-- ── Ordering & limiting the COMBINED result ─────────────────────
A UNION ALL B
ORDER BY 1 DESC        -- one ORDER BY, last clause, output-column or ordinal
LIMIT 50;              -- applies to the whole combined result
-- To order/limit a branch: parenthesise it: (SELECT ... ORDER BY ... LIMIT n)

-- ── Precedence ──────────────────────────────────────────────────
-- INTERSECT binds tighter than UNION/EXCEPT:
A UNION B INTERSECT C   ==  A UNION (B INTERSECT C)   -- parenthesise if unsure

-- ── The OR → UNION ALL index rewrite ────────────────────────────
-- Slow: WHERE a = $1 OR b = $1   (can't use both indexes)
SELECT ... WHERE a = $1
UNION ALL
SELECT ... WHERE b = $1 AND a <> $1   -- guard keeps branches disjoint

-- ── EXPLAIN signals ─────────────────────────────────────────────
-- Append only .................... UNION ALL, optimal
-- HashAggregate/Unique on Append .. UNION dedup — needed? else use ALL
-- Sort Method: external merge Disk. dedup sort spilled → raise work_mem / use ALL
-- Batches > 1 in HashAggregate .... hash spilled to disk
-- HashSetOp Intersect/Except ...... INTERSECT/EXCEPT node

-- ── One-liners ──────────────────────────────────────────────────
-- "What's missing?"  → VALUES(...) EXCEPT SELECT ... FROM t
-- "What's common?"   → SELECT ... INTERSECT SELECT ...
-- "Combine feeds"    → SELECT ...,'tagA' UNION ALL SELECT ...,'tagB'  (tag = disjoint)
```

---

## Connected Topics

- **Topic 07 — NULL in Depth**: Set operations invert NULL's usual behaviour — here `NULL` is *not distinct from* `NULL`. This is the root of why `EXCEPT` is a NULL-safe anti-join while `NOT IN` is not.
- **Topic 11 — INNER JOIN in Depth**: `INTERSECT`/`EXCEPT` are whole-row semi/anti-joins; set operations are the *vertical* combine, joins the *horizontal* one. Each branch typically contains its own joins.
- **Topic 27 — EXISTS and NOT EXISTS**: The other NULL-safe alternative to `NOT IN`; often faster than `EXCEPT` because it stops at the first match and preserves duplicates.
- **Topic 20 — GROUP BY Fundamentals**: `UNION`'s dedup uses the same "not distinct from" grouping semantics as `GROUP BY`/`DISTINCT`; `UNION` ≡ `UNION ALL` + `DISTINCT`.
- **Internals — HashAggregate, Sort/Unique, Append, SetOp nodes**: The physical operators that implement dedup and combination; watching for their appearance (and disk spills) in `EXPLAIN` is how you audit a set-operation query.
- **Internals — `work_mem`**: Bounds the dedup hash/sort; overflow triggers the disk spill that makes a large `UNION` slow.
- **Topic 57 — Transaction Patterns in Node.js** (previous): Set-operation reporting queries often run inside read transactions; the parameterised `pool.query()` patterns carry over directly.
- **Topic 59 — Upsert Patterns** (next): `EXCEPT`/`INTERSECT` express *which* rows differ between source and target — the reconciliation step that frequently precedes an `INSERT ... ON CONFLICT` upsert.
