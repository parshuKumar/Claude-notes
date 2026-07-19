# Topic 45 — Partial Indexes
### SQL Mastery Curriculum — Phase 7: Indexes and Query Optimisation

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you manage a giant library with 10 million books. Most of them are archived in a warehouse across town — nobody ever asks for them. Only about 50,000 books are on the "new arrivals" shelf, and those are the ones people actually request all day long.

Now, your boss says: "Build me an index card catalog so patrons can find books fast."

**The naive approach**: make an index card for all 10 million books. The card catalog is enormous. It fills an entire room. Every time a book is added, moved, or removed — even an archived warehouse book nobody cares about — you have to update a card. And when a patron searches for a new arrival, they still have to flip past millions of irrelevant archive cards to get to the section they need.

**The smart approach**: make index cards **only for the 50,000 new arrivals**. The catalog now fits in a single drawer. It's tiny, it stays in your hands (in memory), and every lookup is instant. When an archived warehouse book changes, you don't touch the catalog at all — it was never in there. And because 99% of your patrons only ever ask about new arrivals, this tiny catalog answers almost every question you'll ever get.

That drawer — a catalog covering only the rows that matter — is a **partial index**. You add a `WHERE` clause to `CREATE INDEX`, and PostgreSQL builds an index over just the subset of rows that satisfy that condition. Smaller on disk, cheaper to maintain, and often the difference between a query that scans a bloated 10M-entry index and one that scans a lean 50K-entry index.

The whole idea: **don't index rows you never query.**

---

## 2. Connection to SQL Internals

A partial index is a standard PostgreSQL B-tree (or GiST, GIN, BRIN, hash — any access method) whose construction is constrained by a predicate. Everything you learned about B-trees in Topic 40 still applies; the only difference is **which heap tuples get an entry**.

Here is what happens underneath:

1. **Index build (`CREATE INDEX ... WHERE pred`)**: PostgreSQL scans the heap, and for each live tuple it evaluates `pred`. Only tuples where `pred` returns TRUE get an index entry inserted into the B-tree. Rows where the predicate is FALSE or NULL are skipped entirely — they never consume a leaf-page slot.

2. **B-tree structure**: The resulting tree has fewer leaf entries, therefore fewer leaf pages, therefore often **one fewer level of internal (branch) pages**. A full index on 10M rows might be 4 levels deep; a partial index on 50K of those rows might be 2–3 levels. Fewer levels = fewer buffer reads per lookup.

3. **Heap-tuple maintenance (INSERT/UPDATE/DELETE)**: When a row is inserted or updated, PostgreSQL evaluates the partial-index predicate against the new tuple. If the predicate is FALSE, **no index entry is created or maintained** — the write skips the index entirely. This is the maintenance win: writes to rows outside the predicate never touch the index, so they generate no index WAL, no index-page dirtying, and no B-tree page splits.

4. **MVCC interaction**: Like every PostgreSQL index, a partial index entry points at a heap tuple (a `ctid`). Because of MVCC (Topic on MVCC), an `UPDATE` creates a new tuple version. If a row transitions across the predicate boundary — e.g. `status` changes from `'pending'` to `'shipped'` and the index is `WHERE status = 'pending'` — the new version is simply not indexed, and the old version's index entry becomes dead and is later reclaimed by VACUUM. Partial indexes therefore naturally "shed" rows as they leave the interesting subset.

5. **Planner predicate proving**: The planner can only use a partial index for a query if it can **prove**, using the query's `WHERE` clause and any `CHECK` constraints, that every row the query wants is present in the index. This proof uses PostgreSQL's predicate refutation/implication machinery (`predicate_implied_by` in the source). This is the single most important internal concept for partial indexes and the source of most "why isn't my partial index being used?" confusion — covered exhaustively in Section 6.

6. **Visibility map & index-only scans**: A partial index participates in index-only scans (Topic 43) exactly like a full index — if the referenced heap pages are all-visible, the heap fetch is skipped.

The mental model: a partial index is a B-tree that agreed to only remember a slice of the table, and the planner is a lawyer who must prove the slice covers your query before it's allowed to use it.

---

## 3. Logical Execution Order Context

Partial indexes are an **access-path** construct. They do not appear anywhere in the logical clause order (`FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → ORDER BY → LIMIT`) — indexes are never written in a query. Instead, they influence how the **physical plan** implements the `FROM` and `WHERE` phases.

The critical interaction is with `WHERE`:

```
FROM orders
WHERE status = 'pending'      ← the planner matches THIS predicate
                                against the partial index's WHERE clause
  AND created_at > now() - interval '7 days'
```

When the planner processes the `FROM`/`WHERE` phase and is choosing an access path for `orders`, it asks: "Is there an index I can use? For each candidate index, does the query's filter guarantee I'll find every needed row inside it?" For a partial index `WHERE status = 'pending'`, the planner checks whether the query's `WHERE` clause **implies** `status = 'pending'`. Because the query literally filters `status = 'pending'`, the implication holds, and the partial index is a legal access path.

Two ordering consequences matter:

1. **The query predicate must be present at plan time.** If `status = 'pending'` is only known at runtime (e.g. it arrives as a parameter, or is buried in a subquery the planner can't see through), the planner may fail to prove implication and fall back to a sequential scan. Placement in `ON` vs `WHERE` can matter for outer joins (recall Topic 12) because it changes what the planner can prove about which rows survive.

2. **The partial-index predicate is applied at build/maintenance time, not query time.** By the time a query runs, the filtering already happened when rows were written. The query's `WHERE` is only used to *prove access legality* and to filter any residual conditions, not to re-apply the partial predicate.

So although partial indexes are invisible in the logical order, they are deeply tied to what the `WHERE` clause can prove — which is why matching the query predicate to the index predicate (Section 6.4) is the whole game.

---

## 4. What Is a Partial Index?

A **partial index** is an index built over only the subset of a table's rows that satisfy a boolean predicate supplied in a `WHERE` clause on the `CREATE INDEX` statement. Rows for which the predicate is FALSE or NULL receive no index entry. The result is a smaller, cheaper-to-maintain index that the planner may use only when it can prove the query targets rows within that subset.

```sql
CREATE INDEX idx_orders_pending
    ON orders (created_at)
    WHERE status = 'pending';
--     │        │  │          │
--     │        │  │          └── the PARTIAL predicate: only rows where
--     │        │  │              status = 'pending' get an index entry.
--     │        │  │              Rows with any other status are skipped.
--     │        │  └── the indexed COLUMN(S) (the "key"): created_at is
--     │        │      what the B-tree is ordered by, used for lookups,
--     │        │      range scans, and ORDER BY satisfaction.
--     │        └── the TABLE the index is built on.
--     └── the index NAME. Convention: idx_<table>_<purpose>.
```

Full anatomy with every option annotated:

```sql
CREATE UNIQUE INDEX CONCURRENTLY idx_users_active_email
    ON users USING btree (lower(email))
    INCLUDE (id)
    WHERE deleted_at IS NULL;
--  │      │          │                │       │        │
--  │      │          │                │       │        └── PARTIAL predicate.
--  │      │          │                │       │            Only non-deleted users
--  │      │          │                │       │            are indexed. This scopes
--  │      │          │                │       │            the UNIQUE constraint too —
--  │      │          │                │       │            uniqueness is enforced ONLY
--  │      │          │                │       │            among rows matching the predicate.
--  │      │          │                │       └── INCLUDE: non-key payload columns
--  │      │          │                │           stored in the leaf for index-only
--  │      │          │                │           scans (Topic 43). Not part of the key.
--  │      │          │                └── the key expression. Here an EXPRESSION
--  │      │          │                    (lower(email)) — combining a partial index
--  │      │          │                    with an expression index (Topic 46).
--  │      │          └── USING btree: the access method. btree is default;
--  │      │              gist/gin/brin/hash also support partial indexes.
--  │      └── CONCURRENTLY: build without an ACCESS EXCLUSIVE lock, so
--  │          production writes continue during the build. Cannot run in
--  │          a transaction block; may leave an INVALID index on failure.
--  └── UNIQUE: enforce uniqueness. Combined with WHERE = "partial uniqueness".
```

### Exact Semantics

Given an `orders` table:

```
id | customer_id | status     | created_at
---+-------------+------------+------------
1  | 10          | pending    | 2026-07-01
2  | 10          | shipped    | 2026-06-15
3  | 11          | pending    | 2026-07-10
4  | 12          | cancelled  | 2026-05-01
5  | 13          | delivered  | 2026-04-01
```

With `CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'pending';`, the index physically contains **only two entries** — for rows `id = 1` and `id = 3`. Rows 2, 4, and 5 are completely absent from the index structure.

- A query `WHERE status = 'pending' AND created_at > '2026-07-05'` **can** use this index (the planner proves `status = 'pending'` is implied) and finds row 3.
- A query `WHERE status = 'shipped'` **cannot** use this index — row 2 isn't in it, so the index would give a wrong (incomplete) answer. The planner won't even consider it.
- A query `WHERE created_at > '2026-07-05'` with **no status filter** cannot use it either — it would miss any shipped/delivered rows in that date range. The planner refuses it and falls back to a seq scan or another index.

That last case is the number-one partial-index gotcha: **the index is only usable when the query guarantees it's asking about the indexed subset.**

---

## 5. Why Partial Index Mastery Matters in Production

1. **Skewed-status tables are everywhere.** Order queues, job queues, notification outboxes, and workflow tables almost always have a tiny "hot" subset (`pending`, `unprocessed`, `active`) and a huge "cold" archive (`completed`, `shipped`, `done`). A full index on `status` wastes 95%+ of its space and I/O on rows nobody queries. A partial index on just the hot subset can be 20–100× smaller, stays entirely in the buffer pool, and turns a slow queue poll into a sub-millisecond lookup.

2. **Write amplification on high-churn tables.** Every full index must be maintained on every INSERT/UPDATE to indexed columns — that's extra WAL, extra page dirtying, extra VACUUM pressure. On a table doing 10K writes/sec where 99% of rows are "cold", a partial index only maintains entries for the 1% hot rows, slashing index-maintenance write amplification.

3. **Enforcing conditional uniqueness.** Business rules like "a user may have only one *active* subscription" or "only one *primary* address per customer" or "email must be unique among *non-deleted* users" cannot be expressed with a plain `UNIQUE` constraint. A partial unique index (`UNIQUE ... WHERE ...`) is the canonical PostgreSQL solution — and getting it wrong silently allows duplicates.

4. **Soft-delete correctness and performance.** Tables with `deleted_at` accumulate tombstones forever. Full indexes bloat with dead-but-not-deleted rows. Partial indexes `WHERE deleted_at IS NULL` keep every index lean and, critically, let a partial unique index enforce uniqueness only among live rows — so a "deleted" email can be reused.

5. **Cost-model honesty.** A smaller index has lower estimated cost, which nudges the planner toward the right plan. A bloated full index sometimes loses to a seq scan in the cost model even when an index scan would be faster in practice; the lean partial index wins cleanly.

6. **The silent-failure trap.** The dangerous part of partial indexes is that a *slightly mismatched* query predicate silently prevents index use — no error, just a seq scan and a 500ms query where you expected 0.5ms. Engineers who don't understand predicate implication ship these regressions constantly. Mastery means you can look at a query and its partial index and instantly know whether the planner can prove usability.

Without this knowledge you get: bloated indexes, write-amplified hot tables, duplicate "active" rows that corrupt business invariants, reusable-email bugs after soft delete, and mysterious query regressions that EXPLAIN "should" have avoided.

---

## 6. Deep Technical Content

### 6.1 The Anatomy of the Predicate

The `WHERE` clause on `CREATE INDEX` accepts any expression that could appear in a query `WHERE`, with restrictions:

- It may reference **only columns of the table being indexed** (no joins, no subqueries to other tables).
- It must be **IMMUTABLE or STABLE-safe in the way PostgreSQL requires** — crucially, it **cannot** reference anything non-immutable in a way that changes which rows belong. `now()`, `current_timestamp`, `random()` are **rejected** for the predicate because the set of matching rows would change over time and the index would silently become wrong.

```sql
-- REJECTED: now() is not immutable — the "recent" set drifts every second.
CREATE INDEX idx_recent ON orders (id)
    WHERE created_at > now() - interval '7 days';
-- ERROR:  functions in index predicate must be marked IMMUTABLE
```

You *can* use immutable comparisons to constants:

```sql
-- ALLOWED: constant boundary, immutable.
CREATE INDEX idx_orders_2026 ON orders (created_at)
    WHERE created_at >= '2026-01-01';
```

Common valid predicate shapes:

```sql
WHERE status = 'pending'                    -- equality to constant
WHERE status IN ('pending', 'processing')   -- IN list of constants
WHERE deleted_at IS NULL                    -- NULL test (soft delete)
WHERE amount > 0                            -- range against constant
WHERE is_active                             -- boolean column directly
WHERE status <> 'archived'                  -- inequality
WHERE (status = 'open' AND priority >= 5)   -- compound boolean
```

### 6.2 Partial vs Full Index — Size and Maintenance

Consider `orders` with 10,000,000 rows, of which 40,000 are `status = 'pending'` at any moment (a queue that drains).

```sql
-- Full index on status
CREATE INDEX idx_full_status ON orders (status);
-- ~10M entries. On disk perhaps 210 MB. 4 B-tree levels.

-- Partial index on just the pending subset
CREATE INDEX idx_pending ON orders (created_at) WHERE status = 'pending';
-- ~40K entries. On disk perhaps 1.2 MB. 2 B-tree levels. Fits in cache trivially.
```

The maintenance difference is even more dramatic than the size difference. Every time an order is created and later marked `shipped`:

- **Full index**: insert entry at creation, then on the status UPDATE, if `status` is indexed, the old index entry is marked dead and a new one inserted. Every lifecycle transition touches the index.
- **Partial index `WHERE status = 'pending'`**: insert entry when the order becomes/starts pending. When it transitions to `shipped`, the predicate `status = 'pending'` is now FALSE for the new tuple version, so **no new entry is made** — the row silently leaves the index. The old entry becomes dead and VACUUM reclaims it. The index self-prunes to always hold ~40K live entries.

This self-pruning property is why partial indexes are the correct tool for **queue tables**: the index size tracks the *queue depth*, not the *table size*.

### 6.3 The Planner's Predicate-Implication Proof (the core mechanism)

The planner may use a partial index for a query **only if it can prove the query returns a subset of the indexed rows**. Formally: the query's restriction clauses must **imply** the index predicate. PostgreSQL implements this with `predicate_implied_by(index_predicate, query_clauses)`.

Rules the prover understands (non-exhaustive, but the ones you'll rely on):

| Index predicate | Query WHERE contains | Implied? | Usable? |
|---|---|---|---|
| `status = 'pending'` | `status = 'pending'` | Yes | Yes |
| `status = 'pending'` | `status = 'pending' AND x > 5` | Yes (AND only narrows) | Yes |
| `status = 'pending'` | `status = 'pending' OR x > 5` | No (OR widens) | No |
| `status = 'pending'` | `status IN ('pending','shipped')` | No (broader set) | No |
| `x > 100` | `x > 200` | Yes (200>100 range ⊂) | Yes |
| `x > 100` | `x > 50` | No (50-range ⊄) | No |
| `x > 100` | `x = 150` | Yes (150 satisfies >100) | Yes |
| `deleted_at IS NULL` | `deleted_at IS NULL` | Yes | Yes |
| `deleted_at IS NULL` | (no mention of deleted_at) | No | No |
| `is_active` | `is_active = true` | Yes | Yes |
| `is_active` | `is_active IS TRUE` | Yes | Yes |
| `status <> 'archived'` | `status = 'open'` | Yes (open ≠ archived) | Yes |

Key intuitions:

- **AND narrows** — adding more conjuncts to the query can only shrink its result set, so if any conjunct matches the index predicate (or implies it), the proof succeeds.
- **OR widens** — a disjunction can pull in rows outside the indexed subset, so it breaks the proof.
- **Range containment** — the prover does real interval math for `>`, `>=`, `<`, `<=` against constants. A tighter query range inside a looser index range is provable.
- **The query must mention the predicate columns.** If the query says nothing about `deleted_at`, the planner cannot assume `deleted_at IS NULL`, so a `WHERE deleted_at IS NULL` partial index is unusable for that query even if, in reality, all matching rows happen to be non-deleted.

### 6.4 Matching the Query Predicate to the Index Predicate

This is the single most important practical skill. The partial index is only an asset if your real queries can be **proven** to target its subset. Three tactics:

**Tactic 1 — Make the constant identical.** If the index is `WHERE status = 'pending'`, ensure hot queries literally contain `status = 'pending'`. Not `status IN ('pending')` (usually fine, PG normalizes single-element IN), not `status != 'shipped' AND status != 'delivered' ...` (won't prove), just `status = 'pending'`.

**Tactic 2 — Beware parameterized predicates.** This is the classic production regression:

```sql
-- App sends status as a bind parameter $1.
SELECT id FROM orders WHERE status = $1 AND created_at > $2;
```

At plan time, `$1` is unknown. The planner cannot prove `$1 = 'pending'`, so it **cannot use** `idx_pending WHERE status = 'pending'`. With a generic plan you lose the partial index. Fixes: (a) hard-code the constant when the query is genuinely always for pending rows; (b) rely on custom plans (PostgreSQL may build a custom plan for the first few executions where `$1`'s value *is* known — see Section 10); or (c) use a full index if the parameter varies widely.

**Tactic 3 — Align ranges deliberately.** If your index is `WHERE created_at >= '2026-01-01'` (a "current year" partial index), your queries must filter `created_at >= <something ≥ 2026-01-01>` for the proof to hold. A query `WHERE created_at >= '2025-12-01'` cannot use it.

### 6.5 Partial Unique Indexes — Conditional Uniqueness

A `UNIQUE` partial index enforces uniqueness **only among the rows matching the predicate**. This is the canonical way to express "unique when active / primary / not-deleted".

**Example: one active subscription per user.**

```sql
CREATE UNIQUE INDEX uq_one_active_sub_per_user
    ON subscriptions (user_id)
    WHERE status = 'active';
```

Now `user_id = 10` may have any number of `cancelled` or `expired` subscriptions, but at most **one** `active`. Inserting a second active subscription for user 10 raises a unique-violation; inserting a cancelled one is fine.

**Example: one primary address per customer.**

```sql
CREATE UNIQUE INDEX uq_one_primary_address
    ON addresses (customer_id)
    WHERE is_primary;
```

**Example: email unique among live users (soft delete).**

```sql
CREATE UNIQUE INDEX uq_users_email_live
    ON users (lower(email))
    WHERE deleted_at IS NULL;
```

A user soft-deleted (`deleted_at` set) frees their email — a new signup can reuse it, because deleted rows aren't in the unique index. A plain `UNIQUE(email)` would forever block reuse.

**NULL subtlety.** Recall that a standard `UNIQUE` constraint treats NULLs as distinct (many NULLs allowed). Partial unique indexes interact with this: the predicate decides *membership*, and within members, standard uniqueness (including NULL-distinctness, unless `NULLS NOT DISTINCT` in PG15+) applies.

### 6.6 The `ON CONFLICT` Interaction (upsert)

Partial unique indexes can serve as arbiters for `INSERT ... ON CONFLICT`, but you must supply the **matching predicate** in the conflict target:

```sql
INSERT INTO subscriptions (user_id, status, plan)
VALUES (10, 'active', 'pro')
ON CONFLICT (user_id) WHERE status = 'active'
DO UPDATE SET plan = EXCLUDED.plan;
--          │           │
--          │           └── REQUIRED: the partial predicate must be
--          │               repeated so PG can identify WHICH unique
--          │               index is the arbiter. Omit it and you get:
--          │               "there is no unique or exclusion constraint
--          │                matching the ON CONFLICT specification".
--          └── the conflict target columns must match the index key.
```

Without the `WHERE status = 'active'` clause, PostgreSQL cannot match the partial unique index and the upsert errors out. This trips up many teams migrating from full unique constraints to partial ones.

### 6.7 Combining Partial with Composite and Expression Indexes

Partial is orthogonal to the key definition. You can combine it with:

- **Composite keys** (Topic 44): `CREATE INDEX ON orders (customer_id, created_at) WHERE status = 'pending';` — a multi-column index over the pending subset, ideal for "list a customer's pending orders newest-first".
- **Expression keys** (Topic 46): `CREATE INDEX ON users (lower(email)) WHERE deleted_at IS NULL;` — case-insensitive lookups over live users only.
- **INCLUDE columns** (covering, Topic 43): `CREATE INDEX ON orders (created_at) INCLUDE (customer_id, total) WHERE status = 'pending';` — index-only scans over the pending queue.

Ordering rule with composite partial indexes is identical to Topic 44: leading columns must be usable by the query's equality/range for the B-tree to be efficient. The partial predicate is *additional* to, not a substitute for, good column ordering.

### 6.8 Multiple Partial Indexes as a Partitioning-Lite Strategy

You can create several partial indexes over the same table, each covering a different slice:

```sql
CREATE INDEX idx_orders_pending    ON orders (created_at) WHERE status = 'pending';
CREATE INDEX idx_orders_processing ON orders (created_at) WHERE status = 'processing';
CREATE INDEX idx_orders_failed     ON orders (updated_at) WHERE status = 'failed';
```

Each index is tiny and independently maintained. Queries for each status use their dedicated lean index. This is a "poor man's partitioning" for the index layer — it doesn't split the heap (real partitioning does, see partitioning topics), but it gives you per-slice index performance without the operational overhead of table partitioning. Downside: more indexes to maintain on writes that *do* fall into those slices, and more objects to manage.

### 6.9 Boolean Column Predicates and the `IS TRUE` Nuance

For a boolean column `is_active`:

```sql
CREATE INDEX idx_active ON users (id) WHERE is_active;
```

The planner proves usability for queries containing `WHERE is_active`, `WHERE is_active = true`, or `WHERE is_active IS TRUE`. But `WHERE is_active IS NOT FALSE` (which also matches NULLs) will **not** prove, because the index excludes NULL `is_active` rows and `IS NOT FALSE` would include them. Always match the exact boolean semantics.

### 6.10 What Partial Indexes Cannot Do / Limitations

- **Predicate can't reference other tables.** No joins in the `WHERE`.
- **Predicate must be immutable-safe.** No `now()`, `random()`, `current_user`, volatile functions.
- **The planner must prove implication at plan time.** Runtime-only knowledge (parameters, opaque subqueries) can defeat it.
- **Not a substitute for a WHERE clause.** The index doesn't auto-filter your query; your query still needs its own `WHERE`. The partial predicate only limits what's *in* the index.
- **Foreign keys can't target a partial unique index.** A `REFERENCES` FK requires a full unique constraint/index over the referenced columns; a partial unique index does not satisfy an FK target.
- **Can't be a `PRIMARY KEY` or table-level `UNIQUE` constraint.** Those require full indexes. Partial uniqueness is index-only, not constraint-syntax.

### 6.11 Interaction with VACUUM and Bloat

Because rows leave a partial index when they exit the predicate (e.g. an order stops being pending), the index accumulates **dead entries** that VACUUM must clean. On a fast-draining queue, the index can churn heavily: entries constantly inserted (row becomes pending) and killed (row leaves pending). Autovacuum keeps it lean, but on extreme-throughput queues you may see index bloat between vacuums. Monitor with `pg_stat_user_indexes` and consider more aggressive autovacuum settings on the table, or periodic `REINDEX CONCURRENTLY`. The index stays *logically* small (few live entries) but can grow *physically* with dead tuples until vacuumed.

---

## 7. EXPLAIN — Partial Index in the Plan

### 7.1 Partial index used (predicate proven)

Table: `orders`, 10,000,000 rows, ~40,000 with `status = 'pending'`.

```sql
CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'pending';

EXPLAIN (ANALYZE, BUFFERS)
SELECT id, customer_id, created_at
FROM orders
WHERE status = 'pending'
  AND created_at > '2026-07-01';
```

```
Index Scan using idx_orders_pending on orders
    (cost=0.29..812.44 rows=9800 width=20)
    (actual time=0.031..3.914 rows=9712 loops=1)
  Index Cond: (created_at > '2026-07-01'::date)
  Buffers: shared hit=142
Planning Time: 0.147 ms
Execution Time: 4.302 ms
```

**Reading it:**
- `Index Scan using idx_orders_pending` — the partial index was chosen.
- `Index Cond: (created_at > '2026-07-01')` — note the `status = 'pending'` predicate is **NOT** in `Index Cond`. It doesn't need to be: every entry in this index is already `status = 'pending'`, so the planner dropped it as redundant. This is the tell-tale sign a partial index is doing its job — the predicate "disappears" from the plan because it's baked into the index.
- `Buffers: shared hit=142` — only 142 pages touched, all from cache. The whole partial index is a handful of pages.
- 4.3 ms for a 10M-row table because we only ever touched the ~40K-entry index.

### 7.2 The same query without the partial index (full seq scan)

```
Seq Scan on orders  (cost=0.00..229055.00 rows=9800 width=20)
    (actual time=0.019..1187.402 rows=9712 loops=1)
  Filter: ((status = 'pending'::text) AND (created_at > '2026-07-01'::date))
  Rows Removed by Filter: 9990288
  Buffers: shared hit=1088 read=127967
Planning Time: 0.101 ms
Execution Time: 1188.6 ms
```

**Reading it:**
- `Seq Scan` over all 10M rows; `Rows Removed by Filter: 9990288` — 99.9% of the work was discarding non-pending rows.
- `Buffers: ... read=127967` — 128K pages read from disk (~1 GB of heap I/O).
- 1188 ms vs 4.3 ms — a **276× speedup** from the partial index. This is the headline number for skewed-status tables.

### 7.3 Partial index NOT used because predicate can't be proven

```sql
-- Query omits the status filter entirely:
EXPLAIN
SELECT id FROM orders WHERE created_at > '2026-07-01';
```

```
Seq Scan on orders  (cost=0.00..229055.00 rows=520000 width=8)
  Filter: (created_at > '2026-07-01'::date)
```

The planner **cannot** use `idx_orders_pending` — without `status = 'pending'` in the query, the index would miss all the shipped/delivered/cancelled rows in that date range. So even though the index exists and covers `created_at`, it's ignored. This is the mismatch trap: the index is present but useless for this query.

### 7.4 Partial index defeated by a parameter (generic plan)

```sql
PREPARE q (text) AS
  SELECT id FROM orders WHERE status = $1 AND created_at > '2026-07-01';
EXPLAIN EXECUTE q('pending');
```

If PostgreSQL has switched to a **generic plan** (after ~5 executions), `$1` is a placeholder and the planner cannot prove `$1 = 'pending'`:

```
Seq Scan on orders  (cost=0.00..254055.00 rows=50000 width=8)
  Filter: ((status = $1) AND (created_at > '2026-07-01'::date))
```

Under a **custom plan** (early executions, value known), the same statement uses the partial index:

```
Index Scan using idx_orders_pending on orders  (cost=0.29..812.44 rows=9800 width=8)
  Index Cond: (created_at > '2026-07-01'::date)
```

This is why prepared statements + partial indexes can regress after warm-up. Mitigation in Section 10/11.

### 7.5 Partial unique index enforcing a constraint (INSERT plan)

You don't usually EXPLAIN an INSERT's constraint check, but you can see the arbiter in `ON CONFLICT`:

```sql
EXPLAIN
INSERT INTO subscriptions (user_id, status, plan) VALUES (10, 'active', 'pro')
ON CONFLICT (user_id) WHERE status = 'active' DO UPDATE SET plan = EXCLUDED.plan;
```

```
Insert on subscriptions  (cost=0.00..0.01 rows=0 width=0)
  Conflict Resolution: UPDATE
  Conflict Arbiter Indexes: uq_one_active_sub_per_user
  ->  Result  (cost=0.00..0.01 rows=1 width=44)
```

`Conflict Arbiter Indexes: uq_one_active_sub_per_user` confirms the partial unique index is the arbiter — proof that your predicate matched.

---

## 8. Query Examples

### Example 1 — Basic: Index the Active Subset

```sql
-- Business: users table, 5M rows, ~200K currently active.
-- We constantly look up active users by email.

CREATE INDEX idx_users_active_email
    ON users (email)
    WHERE is_active;               -- only active users are indexed

-- This query PROVES is_active and uses the lean 200K-entry index:
SELECT id, email, created_at
FROM users
WHERE is_active                    -- matches the partial predicate exactly
  AND email = 'ana@example.com';
```

### Example 2 — Intermediate: Unprocessed Queue Poll with Composite Partial Index

```sql
-- Job queue: jobs table, 20M lifetime rows, ~5K unprocessed at any time.
-- Worker poll: grab the oldest unprocessed jobs for a given queue.

CREATE INDEX idx_jobs_unprocessed
    ON jobs (queue_name, created_at)   -- composite key: filter by queue, order by age
    WHERE processed_at IS NULL;        -- only pending jobs indexed

-- Worker query — proves processed_at IS NULL, filters queue, orders by age:
SELECT id, payload
FROM jobs
WHERE processed_at IS NULL            -- matches partial predicate
  AND queue_name = 'emails'           -- leading key column, equality
ORDER BY created_at                   -- second key column, satisfied by index order
LIMIT 100
FOR UPDATE SKIP LOCKED;               -- safe concurrent claim (queue pattern)
```

The index holds only ~5K entries. As jobs are processed (`processed_at` set), they leave the index automatically. Index size tracks queue depth, not the 20M-row history.

### Example 3 — Production Grade: Soft-Delete + Partial Uniqueness + Covering

```sql
-- Context:
--   users: 8,000,000 rows total, ~7,600,000 live (deleted_at IS NULL),
--          ~400,000 soft-deleted tombstones.
--   Requirements:
--     (a) email must be unique AMONG LIVE users (deleted users free their email),
--     (b) fast case-insensitive login lookup for live users,
--     (c) login lookup should be an index-only scan (no heap fetch).
--   Expected: login lookup < 1 ms even at 8M rows.

-- Partial UNIQUE + EXPRESSION + COVERING index, built without locking writes:
CREATE UNIQUE INDEX CONCURRENTLY uq_users_live_email
    ON users (lower(email))            -- expression key: case-insensitive
    INCLUDE (id, password_hash)        -- covering payload for login (index-only scan)
    WHERE deleted_at IS NULL;          -- partial: only live users; enforces (a)

-- Login lookup — proves deleted_at IS NULL, uses lower(email):
SELECT id, password_hash
FROM users
WHERE deleted_at IS NULL              -- REQUIRED to prove partial-index usability
  AND lower(email) = lower($1);       -- matches the expression key

-- EXPLAIN (ANALYZE, BUFFERS):
--
-- Index Only Scan using uq_users_live_email on users
--     (cost=0.42..4.44 rows=1 width=45)
--     (actual time=0.028..0.030 rows=1 loops=1)
--   Index Cond: (lower(email) = lower('ana@example.com'))
--   Heap Fetches: 0
--   Buffers: shared hit=4
-- Planning Time: 0.093 ms
-- Execution Time: 0.061 ms
--
-- Reading it: Index Only Scan (Heap Fetches: 0) — the INCLUDE columns
-- satisfied the SELECT with zero heap access. deleted_at IS NULL vanished
-- from Index Cond (baked into the partial index). 4 buffer hits, 0.06 ms.
-- The 400K soft-deleted tombstones are entirely absent from the index.
```

---

## 9. Wrong → Right Patterns

### Wrong 1: Query omits the partial predicate — index silently unused

```sql
-- Index built to accelerate pending-order polling:
CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'pending';

-- WRONG: the query forgot to include status = 'pending'.
SELECT id, customer_id FROM orders
WHERE created_at > '2026-07-01'
ORDER BY created_at
LIMIT 100;

-- WRONG RESULT (not incorrect data — incorrect PERFORMANCE):
--   Seq Scan on orders, Rows Removed by Filter: ~9.99M, ~1200 ms.
-- WHY at the execution level: the planner cannot prove that every row
-- with created_at > '2026-07-01' is a pending row. Non-pending rows in
-- that date range exist and would be missing from the index, so using
-- idx_orders_pending would return a WRONG (incomplete) answer. The
-- planner correctly refuses it and seq-scans instead.

-- RIGHT: include the predicate so implication holds.
SELECT id, customer_id FROM orders
WHERE status = 'pending'          -- proves partial-index usability
  AND created_at > '2026-07-01'
ORDER BY created_at
LIMIT 100;
-- Index Scan using idx_orders_pending, ~4 ms.
```

### Wrong 2: Broadening the query with OR or IN defeats the proof

```sql
CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'pending';

-- WRONG: IN widens the target set beyond the indexed subset.
SELECT id FROM orders
WHERE status IN ('pending', 'processing')
  AND created_at > '2026-07-01';
-- WHY: 'processing' rows are NOT in idx_orders_pending. The planner
-- cannot prove the query stays within the pending subset, so the partial
-- index is unusable → seq scan. The IN silently widened past the index.

-- RIGHT (option A): one partial index per status, union the results.
CREATE INDEX idx_orders_processing ON orders (created_at) WHERE status = 'processing';
SELECT id FROM orders WHERE status = 'pending'    AND created_at > '2026-07-01'
UNION ALL
SELECT id FROM orders WHERE status = 'processing' AND created_at > '2026-07-01';
-- Each branch proves its own partial index; both use lean index scans.

-- RIGHT (option B): if you frequently query BOTH together, make the
-- index cover both from the start:
CREATE INDEX idx_orders_open ON orders (created_at)
    WHERE status IN ('pending', 'processing');
SELECT id FROM orders
WHERE status IN ('pending', 'processing')   -- now matches the index predicate
  AND created_at > '2026-07-01';
```

### Wrong 3: Partial unique index doesn't prevent the duplicates you think

```sql
-- Requirement: at most one ACTIVE subscription per user.

-- WRONG: a plain unique index blocks ALL duplicate user_ids, including
-- historical cancelled ones — users can never re-subscribe.
CREATE UNIQUE INDEX uq_sub_user ON subscriptions (user_id);
-- INSERT of a second row for user 10 (even status='cancelled' first,
-- then 'active') fails: "duplicate key value violates unique constraint".
-- The business rule (one ACTIVE, many cancelled) is broken.

-- RIGHT: scope uniqueness to active rows only.
CREATE UNIQUE INDEX uq_one_active_sub
    ON subscriptions (user_id)
    WHERE status = 'active';
-- user 10 may have many cancelled/expired rows, at most one active.
-- Attempting a 2nd active row → unique violation (correct).
```

### Wrong 4: `ON CONFLICT` on a partial unique index without repeating the predicate

```sql
CREATE UNIQUE INDEX uq_one_active_sub ON subscriptions (user_id) WHERE status = 'active';

-- WRONG: conflict target omits the partial predicate.
INSERT INTO subscriptions (user_id, status, plan) VALUES (10, 'active', 'pro')
ON CONFLICT (user_id) DO UPDATE SET plan = EXCLUDED.plan;
-- ERROR: there is no unique or exclusion constraint matching the
--        ON CONFLICT specification
-- WHY: without the WHERE, PG can't identify the PARTIAL index as arbiter;
-- it looks for a FULL unique index on (user_id), which doesn't exist.

-- RIGHT: repeat the predicate in the conflict target.
INSERT INTO subscriptions (user_id, status, plan) VALUES (10, 'active', 'pro')
ON CONFLICT (user_id) WHERE status = 'active'
DO UPDATE SET plan = EXCLUDED.plan;
```

### Wrong 5: Non-immutable predicate rejected (or silently wrong intent)

```sql
-- WRONG: trying to index "recent" orders with a moving boundary.
CREATE INDEX idx_recent ON orders (id)
    WHERE created_at > now() - interval '7 days';
-- ERROR: functions in index predicate must be marked IMMUTABLE
-- WHY: now() changes every call; the set of "recent" rows would drift,
-- but the index membership is fixed at write time — it would rot.

-- RIGHT: use a fixed boundary and rebuild periodically, OR index a
-- stable derived flag maintained by the application/trigger.
CREATE INDEX idx_orders_h2_2026 ON orders (id)
    WHERE created_at >= '2026-07-01';   -- immutable constant boundary
-- For a true rolling window, prefer a maintained boolean column
-- (e.g. is_recent) updated by a scheduled job, then:
--   CREATE INDEX ... WHERE is_recent;
```

---

## 10. Performance Profile

### Size and Depth Scaling

| Table rows | Matching subset | Full index size | Partial index size | Partial B-tree depth |
|---|---|---|---|---|
| 1,000,000 | 1% (10K) | ~21 MB | ~0.3 MB | 2 levels |
| 10,000,000 | 0.4% (40K) | ~214 MB | ~1.2 MB | 2 levels |
| 100,000,000 | 0.1% (100K) | ~2.1 GB | ~3 MB | 2–3 levels |

The partial index size scales with the **subset**, not the table. At 100M rows with a 0.1% hot subset, the partial index is ~700× smaller than the full index and stays permanently cached, while the full index likely spills the cache and incurs disk reads on lookups.

### CPU and Memory

- **Lookups**: a partial index with 2 B-tree levels needs ~2–4 buffer reads per point lookup, essentially always cache hits. CPU is a couple of comparisons per level. Negligible.
- **Build**: `CREATE INDEX` still scans the whole heap once to evaluate the predicate (it must check every row), so build time is proportional to *table* size, not subset size — but the resulting index is small so the sort/write phase is cheap. Use `CONCURRENTLY` in production to avoid locking.
- **Maintenance memory**: writes to non-matching rows skip the index entirely — zero index maintenance cost for the cold majority.

### Write Amplification (the big win)

On a table doing heavy writes where most rows are cold:

| Scenario | Full index maintenance | Partial index maintenance |
|---|---|---|
| Insert a cold row (never pending) | Index entry inserted + WAL | **No index work at all** |
| Update a cold row's indexed col | Old entry dead + new entry + WAL | **No index work** (predicate FALSE) |
| Row enters the hot subset | Entry maintained | Entry inserted (needed) |
| Row leaves the hot subset | Entry maintained | Old entry dead, no new entry |

For a queue table where 99% of writes are to cold rows, the partial index cuts index-maintenance WAL and page dirtying by ~99%, directly reducing checkpoint pressure, replication lag, and VACUUM load.

### Scaling at 1M / 10M / 100M rows

- **1M rows**: even a full index is fine; partial helps write-heavy tables and enables partial uniqueness. Gains modest but real.
- **10M rows**: partial index is the difference between a cached lean scan (single-digit ms) and a bloated index or seq scan (hundreds of ms). Clear win for skewed queries.
- **100M rows**: often mandatory. A full index may not fit in cache; the partial index for the hot subset stays resident. Combined with table partitioning for the heap, partial indexes on the active partition keep the working set tiny.

### Optimization Techniques Specific to Partial Indexes

1. **Cover the hot query** — add `INCLUDE` columns so the hot-subset query is index-only (Heap Fetches: 0).
2. **Composite key inside the subset** — order key columns per Topic 44 for the queries that run against the subset.
3. **One index per hot status** rather than one full index on `status`.
4. **Watch prepared-statement generic plans** — a partial index can be abandoned when `$1` hides the constant. Consider `plan_cache_mode = force_custom_plan` for sessions that rely on partial indexes, or inline the constant.
5. **Autovacuum tuning** — high-churn partial indexes accumulate dead entries; lower `autovacuum_vacuum_scale_factor` on the table to keep the index physically small.
6. **REINDEX CONCURRENTLY** periodically on extreme-throughput queue indexes to reclaim bloat without downtime.

---

## 11. Node.js Integration

### 11.1 Creating a partial index from a migration

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Build CONCURRENTLY so production writes aren't blocked.
// NOTE: CREATE INDEX CONCURRENTLY cannot run inside a transaction,
// so do NOT wrap this in BEGIN/COMMIT. Run it on its own.
async function createPendingIndex() {
  await pool.query(`
    CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_orders_pending
      ON orders (created_at)
      WHERE status = 'pending'
  `);
}
```

### 11.2 Hot queue poll that PROVES the predicate

```javascript
// Worker poll. The literal status = 'pending' must be present so the
// planner can prove usability of idx_orders_pending. We pass only the
// varying values ($1, $2) as parameters — the predicate stays a constant.
async function claimPendingOrders(limit = 100) {
  const { rows } = await pool.query(
    `SELECT id, customer_id, payload
       FROM orders
      WHERE status = 'pending'            -- constant → partial index provable
        AND created_at > $1               -- varying value as parameter
      ORDER BY created_at
      LIMIT $2
      FOR UPDATE SKIP LOCKED`,
    [new Date(Date.now() - 24 * 3600 * 1000), limit]
  );
  return rows;
}
```

### 11.3 The parameterized-status trap and its fix

```javascript
// RISKY: status is a bind parameter. If PG switches to a generic plan,
// it can no longer prove $1 = 'pending' and abandons the partial index.
async function riskyByStatus(status, since) {
  const { rows } = await pool.query(
    `SELECT id FROM orders WHERE status = $1 AND created_at > $2`,
    [status, since]
  );
  return rows;
}

// FIX A: if the code path is ALWAYS for pending, inline the constant.
async function safePending(since) {
  const { rows } = await pool.query(
    `SELECT id FROM orders WHERE status = 'pending' AND created_at > $1`,
    [since]
  );
  return rows;
}

// FIX B: force a custom plan for this session so PG re-plans with the
// actual value each time (worth it when the partial index is critical).
async function safeByStatusForcedCustom(status, since) {
  const client = await pool.connect();
  try {
    await client.query(`SET LOCAL plan_cache_mode = 'force_custom_plan'`);
    const { rows } = await client.query(
      `SELECT id FROM orders WHERE status = $1 AND created_at > $2`,
      [status, since]
    );
    return rows;
  } finally {
    client.release();
  }
}
```

### 11.4 Upsert against a partial unique index

```javascript
// The partial predicate MUST be repeated in ON CONFLICT ... WHERE.
async function upsertActiveSubscription(userId, plan) {
  const { rows } = await pool.query(
    `INSERT INTO subscriptions (user_id, status, plan)
     VALUES ($1, 'active', $2)
     ON CONFLICT (user_id) WHERE status = 'active'
     DO UPDATE SET plan = EXCLUDED.plan
     RETURNING id, user_id, plan`,
    [userId, plan]
  );
  return rows[0];
}
```

### 11.5 Handling the partial-unique violation on soft-deleted email reuse

```javascript
// Signup: email must be unique among live users (uq_users_live_email
// WHERE deleted_at IS NULL). A previously soft-deleted user's email is
// reusable. On conflict, surface a clean 409 rather than a 500.
async function registerUser(email, passwordHash) {
  try {
    const { rows } = await pool.query(
      `INSERT INTO users (email, password_hash, deleted_at)
       VALUES ($1, $2, NULL)
       RETURNING id, email`,
      [email, passwordHash]
    );
    return { ok: true, user: rows[0] };
  } catch (err) {
    if (err.code === '23505' /* unique_violation */) {
      // Only collides with a LIVE user (deleted ones aren't in the index).
      return { ok: false, reason: 'email_taken_by_active_user' };
    }
    throw err;
  }
}
```

**Do ORMs handle partial indexes?** Only for *declaring* them in migrations (most do). None of the major ORMs automatically *prove* the predicate for you at query time — you must write your query's `WHERE` to include the partial predicate yourself, exactly as in raw SQL. The ORM will happily generate a query that silently fails to use your partial index. See Section 12.

---

## 12. ORM Comparison

The two questions for every ORM: **(1) Can it declare a partial index in migrations?** and **(2) Does its query builder let you write the `WHERE` that proves the predicate?** The second is where teams get burned — ORMs never add the partial predicate for you.

### Prisma

**Can Prisma declare partial indexes?** — Yes, via the `@@index` attribute... but only recently and partially. Prisma's schema does **not** natively express arbitrary `WHERE` predicates on indexes for PostgreSQL as of most versions; you declare them with raw SQL in a migration file.

```prisma
// schema.prisma — model, but the partial index is NOT expressible here.
model Order {
  id         Int      @id @default(autoincrement())
  customerId Int
  status     String
  createdAt  DateTime @default(now())
  // @@index([createdAt]) would be a FULL index — no WHERE support.
}
```

```sql
-- prisma/migrations/xxxx_partial/migration.sql — hand-written:
CREATE INDEX "idx_orders_pending" ON "Order" ("createdAt") WHERE status = 'pending';
```

```typescript
// Query side: you MUST include the predicate yourself.
const pending = await prisma.order.findMany({
  where: {
    status: 'pending',                      // ← required to use the partial index
    createdAt: { gt: new Date('2026-07-01') },
  },
  orderBy: { createdAt: 'asc' },
  take: 100,
});

// Partial-unique enforcement and ON CONFLICT arbiter → raw SQL:
await prisma.$executeRaw`
  INSERT INTO subscriptions (user_id, status, plan) VALUES (${userId}, 'active', ${plan})
  ON CONFLICT (user_id) WHERE status = 'active' DO UPDATE SET plan = EXCLUDED.plan
`;
```

**Where it breaks**: Prisma cannot express the partial `WHERE` in schema, can't use a partial unique index for its `upsert()` conflict resolution (needs `$executeRaw`), and Prisma's `@unique` always maps to a full unique constraint. Introspection (`prisma db pull`) may not round-trip a hand-written partial index cleanly.

**Verdict**: Declare partial indexes via raw SQL migrations; write `findMany` `where` clauses that include the predicate; drop to `$queryRaw`/`$executeRaw` for partial-unique upserts.

---

### Drizzle ORM

**Can Drizzle declare partial indexes?** — Yes, natively and cleanly, which is a standout.

```typescript
import { pgTable, serial, integer, text, timestamp, index, uniqueIndex } from 'drizzle-orm/pg-core';
import { sql, eq, and, gt, isNull } from 'drizzle-orm';

export const orders = pgTable('orders', {
  id: serial('id').primaryKey(),
  customerId: integer('customer_id').notNull(),
  status: text('status').notNull(),
  createdAt: timestamp('created_at').defaultNow(),
}, (t) => ({
  // Native partial index via .where():
  pendingIdx: index('idx_orders_pending')
    .on(t.createdAt)
    .where(sql`${t.status} = 'pending'`),
}));

export const subscriptions = pgTable('subscriptions', {
  id: serial('id').primaryKey(),
  userId: integer('user_id').notNull(),
  status: text('status').notNull(),
}, (t) => ({
  // Native partial UNIQUE index:
  oneActive: uniqueIndex('uq_one_active_sub')
    .on(t.userId)
    .where(sql`${t.status} = 'active'`),
}));

// Query — include the predicate to prove usability:
const pending = await db.select()
  .from(orders)
  .where(and(eq(orders.status, 'pending'), gt(orders.createdAt, new Date('2026-07-01'))))
  .orderBy(orders.createdAt)
  .limit(100);
```

**Where it breaks**: `ON CONFLICT` with a partial predicate needs `.onConflictDoUpdate({ target: ..., targetWhere: sql\`status = 'active'\`, set: {...} })` — the `targetWhere` is easy to forget. Otherwise Drizzle's partial-index support is the best of the group.

**Verdict**: First-class partial index support in schema and queries. Preferred ORM when partial indexes are central.

---

### Sequelize

**Can Sequelize declare partial indexes?** — Yes, via the `where` option on an index definition.

```javascript
const Order = sequelize.define('Order', {
  status: DataTypes.STRING,
  createdAt: DataTypes.DATE,
}, {
  indexes: [
    {
      name: 'idx_orders_pending',
      fields: ['createdAt'],
      where: { status: 'pending' },     // ← partial predicate
    },
  ],
});

// Partial unique index:
const Subscription = sequelize.define('Subscription', {
  userId: DataTypes.INTEGER,
  status: DataTypes.STRING,
}, {
  indexes: [
    { name: 'uq_one_active_sub', unique: true, fields: ['userId'], where: { status: 'active' } },
  ],
});

// Query — must include the predicate:
const { Op } = require('sequelize');
const pending = await Order.findAll({
  where: { status: 'pending', createdAt: { [Op.gt]: new Date('2026-07-01') } },
  order: [['createdAt', 'ASC']],
  limit: 100,
});
```

**Where it breaks**: Sequelize's `upsert()` and `findOrCreate()` don't let you specify a partial conflict target; you need `sequelize.query()` for `ON CONFLICT (col) WHERE ...`. Complex predicates (ranges, `IS NULL`, expressions) in the index `where` sometimes need `Sequelize.literal()`.

**Verdict**: Good declarative support for partial indexes; raw query needed for partial-unique upserts.

---

### TypeORM

**Can TypeORM declare partial indexes?** — Yes, via the `where` option on `@Index`.

```typescript
import { Entity, PrimaryGeneratedColumn, Column, Index } from 'typeorm';

@Entity()
@Index('idx_orders_pending', ['createdAt'], { where: `status = 'pending'` })
export class Order {
  @PrimaryGeneratedColumn() id: number;
  @Column() status: string;
  @Column() createdAt: Date;
}

@Entity()
@Index('uq_one_active_sub', ['userId'], { unique: true, where: `status = 'active'` })
export class Subscription {
  @PrimaryGeneratedColumn() id: number;
  @Column() userId: number;
  @Column() status: string;
}
```

```typescript
// Query — include the predicate:
const pending = await dataSource.getRepository(Order)
  .createQueryBuilder('o')
  .where('o.status = :s', { s: 'pending' })
  .andWhere('o.createdAt > :d', { d: new Date('2026-07-01') })
  .orderBy('o.createdAt', 'ASC')
  .limit(100)
  .getMany();
```

**Where it breaks**: The `where` string is raw SQL — no type checking, and you must keep it byte-compatible with your queries for the proof to hold. TypeORM's `upsert()` (`orUpdate`) doesn't support a partial conflict target; use `.query()` for `ON CONFLICT ... WHERE`. Migrations sometimes drop/recreate the index on unrelated schema changes.

**Verdict**: Declarative partial indexes work; watch the raw `where` string and use raw queries for partial-unique upserts.

---

### Knex.js

**Can Knex declare partial indexes?** — Not via the fluent schema builder directly for the `WHERE`; you use `.whereRaw`-style raw SQL or `knex.raw` in a migration.

```javascript
// Migration — partial index via raw SQL (fluent builder has no .where on index):
exports.up = async (knex) => {
  await knex.raw(`
    CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'pending'
  `);
  await knex.raw(`
    CREATE UNIQUE INDEX uq_one_active_sub ON subscriptions (user_id) WHERE status = 'active'
  `);
};

// Query — include the predicate (fluent builder is fine here):
const pending = await knex('orders')
  .where('status', 'pending')
  .andWhere('created_at', '>', new Date('2026-07-01'))
  .orderBy('created_at', 'asc')
  .limit(100)
  .select('id', 'customer_id');

// Partial-unique upsert — .onConflict().merge() supports a WHERE on the target:
await knex('subscriptions')
  .insert({ user_id: userId, status: 'active', plan })
  .onConflict(knex.raw('(user_id) WHERE status = \'active\''))
  .merge({ plan });
```

**Where it breaks**: No fluent API for the index `WHERE` — always raw. The `onConflict` partial target must be expressed with `knex.raw`. But because Knex is SQL-transparent, nothing is hidden.

**Verdict**: Most transparent. Partial indexes are declared with `knex.raw`, queried with the normal builder, and the partial-unique upsert works via `knex.raw` in `onConflict`.

---

### ORM Summary Table

| ORM | Declare partial index | Partial unique | Partial `ON CONFLICT` | Proves predicate for you? | Verdict |
|-----|----------------------|----------------|----------------------|--------------------------|---------|
| Prisma | Raw SQL migration only | Raw SQL | `$executeRaw` | No | Schema can't express it; raw migrations |
| Drizzle | Native `.where()` | Native `uniqueIndex().where()` | `targetWhere` | No | Best native support |
| Sequelize | `indexes: [{ where }]` | `unique + where` | `sequelize.query()` | No | Good declarative, raw upsert |
| TypeORM | `@Index(..., { where })` | `{ unique, where }` | `.query()` | No | Works; watch raw `where` string |
| Knex | `knex.raw` migration | `knex.raw` | `onConflict(knex.raw(...))` | No | Most transparent, all raw |

**Universal rule**: No ORM adds the partial predicate to your query automatically. You are always responsible for writing a `WHERE` that the planner can prove implies the index predicate.

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given:
- `users(id, email, is_active, created_at)` — 5,000,000 rows, ~150,000 active.

Tasks:
1. Create a partial index that makes "look up an active user by email" fast, indexing only active users.
2. Write the lookup query so that it actually uses your index.
3. Write a second query (looking up a user by email regardless of active status) and explain why it cannot use your partial index.

```sql
-- Write your query here
```

---

### Exercise 2 — Medium (combines Topic 43 covering + Topic 44 composite)

Given:
- `jobs(id, queue_name, status, priority, created_at, payload)` — 30M lifetime rows, ~8,000 with `status = 'queued'`.

Tasks:
1. Create ONE partial index that lets a worker efficiently run: "for a given `queue_name`, fetch the highest-priority, then oldest, `queued` jobs" — and make it an **index-only scan** for a `SELECT id` poll (recall covering indexes, Topic 43, and composite column order, Topic 44).
2. Write the poll query (with `FOR UPDATE SKIP LOCKED`) so it proves the predicate, uses the composite order, and is covered.
3. Explain, referencing the B-tree order, why `priority DESC, created_at ASC` in the query must match the index definition to avoid a Sort node.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (production simulation; naive answer is wrong)

A `subscriptions(id, user_id, status, plan, started_at, cancelled_at)` table must enforce:
"A user may have **at most one active** subscription, but unlimited cancelled/expired ones, and cancelled subscriptions must **not** block a user from subscribing again."

A developer ships:

```sql
ALTER TABLE subscriptions ADD CONSTRAINT uq_user UNIQUE (user_id);
```

Tasks:
1. Explain precisely what breaks with this constraint (give the failing scenario).
2. Replace it with the correct partial unique index.
3. Write the upsert (`INSERT ... ON CONFLICT`) that activates or re-plans a user's active subscription, and explain the one clause that is mandatory for it to work against your partial index.
4. There's a race: two concurrent transactions both try to insert an active subscription for the same user. Explain what the partial unique index guarantees and what error the loser gets.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

You are handed this in a code review. The `orders` table has 40M rows; ~50K are `status = 'pending'`. Someone added:

```sql
CREATE INDEX idx_orders_status ON orders (status);

-- Hot query, runs 2,000×/sec from workers:
SELECT id, payload FROM orders
WHERE status = $1                 -- always 'pending' in practice
ORDER BY created_at
LIMIT 50;
```

Tasks:
1. Critique the index choice. Is `idx_orders_status` a good fit? What does its selectivity look like, and will the query even use it well given the `ORDER BY`?
2. Propose a better index using a partial index, and justify the key columns and predicate.
3. The query passes `status` as `$1`. Explain the prepared-statement generic-plan risk against your partial index and give two concrete fixes.
4. State the expected before/after latency and the buffer/I-O difference at 40M rows.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What is a partial index and when would you use one?**

A junior answer: "It's an index with a `WHERE` clause, so it only indexes some rows. You use it when you only query part of the table — like active users or pending orders — to make the index smaller and faster."

The principal answer: All of that, plus the *why it's smaller AND cheaper to maintain*: only rows satisfying the predicate get a B-tree entry, so the index size tracks the interesting subset (not the table), it has fewer levels (fewer buffer reads per lookup), and — critically — **writes to non-matching rows skip the index entirely**, cutting write amplification and VACUUM load. The canonical use cases are skewed-status tables (queues, outboxes), soft-delete (`WHERE deleted_at IS NULL`), and conditional uniqueness. And I'd immediately flag the catch: the planner can only use it when it can *prove* the query targets the indexed subset.

**Follow-up the interviewer asks**: "How much smaller, concretely, for a 10M-row table with 40K pending rows?" — ~214 MB full vs ~1.2 MB partial, and the partial one stays permanently cached.

---

**Q: Why does the `status = 'pending'` condition disappear from `Index Cond` in the EXPLAIN output when I use a partial index?**

A junior answer: "Not sure — maybe the planner optimized it away?"

The principal answer: Because *every entry in the index is already `status = 'pending'`* — the predicate is baked into the index at build time. The planner recognizes the condition is redundant with the index's own predicate and drops it from `Index Cond`. Its absence is actually the confirmation that the partial index is being used correctly. If you still saw `status = 'pending'` re-checked as a `Filter`, that would suggest the planner used a *different* index and re-filtered.

**Follow-up**: "What would you see if the partial index were NOT used?" — a `Seq Scan` with `Filter: (status = 'pending' AND ...)` and a large `Rows Removed by Filter`.

---

### Mid Level

**Q: I have `CREATE INDEX ... WHERE status = 'pending'`. My query is `WHERE status IN ('pending','shipped')`. Does it use the index? Why or why not?**

A junior answer: "Yes, because 'pending' is in the list."

The principal answer: No. The planner needs to prove that *every* row the query returns is in the index. `status IN ('pending','shipped')` includes shipped rows, which are **not** in a `WHERE status = 'pending'` index — using it would return an incomplete result. `IN`/`OR` *widen* the target set and break predicate implication. Only conditions that *narrow* to a subset of the indexed rows (like adding `AND` conjuncts, or a tighter range) preserve the proof. To serve both statuses you'd either union two single-status partial-index scans or build one index `WHERE status IN ('pending','shipped')`.

**Follow-up**: "What about `WHERE status = 'pending' AND priority > 5`?" — Still uses it: `AND` only narrows, so the proof holds; `priority > 5` becomes a residual filter or index cond.

---

### Principal Level

**Q: A team migrated a hot query from a full index to a partial index and latency got *worse* in production, even though it tested fast. What's your first hypothesis?**

A junior answer: "Maybe the partial index is bad and they should revert."

The principal answer: My first hypothesis is a **prepared-statement generic plan** hiding the predicate constant. The query almost certainly passes `status = $1`. In their test (a fresh `psql` with a literal, or the first few executions), PostgreSQL builds a *custom plan* with `$1 = 'pending'` known, proves the partial index, and it's fast. In production, after ~5 executions the driver's prepared statement flips to a *generic plan* where `$1` is opaque; the planner can no longer prove `$1 = 'pending'`, abandons the partial index, and falls back to a seq scan or the old full index — hence the regression that only appears under sustained load. I'd confirm with `EXPLAIN EXECUTE` on a warmed statement, then fix by either inlining the constant (`status = 'pending'`) when the path is always-pending, setting `plan_cache_mode = force_custom_plan`, or reverting to a full index if the parameter genuinely varies. Secondary hypotheses: predicate-string mismatch between ORM-generated SQL and the index `WHERE`, or index bloat on a high-churn queue between vacuums.

**Follow-up**: "How would `force_custom_plan` hurt you?" — It re-plans every execution, adding planning CPU per call; on a 2,000 QPS path that planning overhead can itself become the bottleneck, so I'd prefer inlining the constant.

---

**Q: You need "email unique among non-deleted users" with soft deletes. Walk me through the design and every edge case.**

A junior answer: "Add a unique index with `WHERE deleted_at IS NULL`."

The principal answer: The core is `CREATE UNIQUE INDEX uq_users_live_email ON users (lower(email)) WHERE deleted_at IS NULL` — partial (live only) plus expression (case-insensitive). Edge cases: (1) **Reuse after delete** — when a user is soft-deleted, their row leaves the index, so the email frees up and a new signup can take it; that's the desired behavior but I'd confirm the product wants it. (2) **Undelete collision** — if we ever "undelete" a user by setting `deleted_at = NULL`, and someone else grabbed the email meanwhile, the undelete UPDATE will fail with a unique violation; the app must handle that. (3) **Login queries must include `deleted_at IS NULL`** or they won't use the index — and they *should* filter it anyway to avoid logging in deleted users. (4) **FK targets** — no foreign key can reference this partial unique index; if other tables FK to users, they reference the PK, not email, so that's fine. (5) **NULL emails** — if email is nullable, decide via `NULLS NOT DISTINCT` (PG15+) whether multiple NULL live emails are allowed. (6) **Case/whitespace** — `lower(email)` handles case; I'd also trim at the app layer. (7) **ON CONFLICT** upserts must repeat `WHERE deleted_at IS NULL` in the conflict target.

**Follow-up**: "How do you enforce this if some code path hard-deletes instead of soft-deletes?" — Hard delete removes the row entirely, so it also frees the index entry; behavior is consistent. The risk is *inconsistent* deletion strategies across the codebase, which I'd standardize.

---

## 15. Mental Model Checkpoint

1. A table has 50M rows; 20K satisfy `status = 'queued'`. You build `CREATE INDEX ON jobs (created_at) WHERE status = 'queued'`. Roughly how many entries are in the index, and does the build scan 20K rows or 50M rows? Why?

2. Your partial index is `WHERE status = 'pending'`. Explain, in terms of predicate implication, why `WHERE status = 'pending' AND amount > 100` can use it but `WHERE status = 'pending' OR amount > 100` cannot.

3. In an EXPLAIN plan that uses a partial index, the index's own predicate is missing from `Index Cond`. Is something wrong? What does its absence tell you?

4. A row currently indexed by `WHERE status = 'pending'` is UPDATEd to `status = 'shipped'`. Walk through what happens to its index entry and why no new entry is created. What eventually removes the old one?

5. Why is `CREATE INDEX ... WHERE created_at > now() - interval '7 days'` rejected, and what does that tell you about *when* the predicate is evaluated versus a query's `WHERE`?

6. You define `CREATE UNIQUE INDEX ON subscriptions (user_id) WHERE status = 'active'`. A user has three cancelled subscriptions and you insert a fourth active one. Does it succeed? Now you insert a second active one — what happens, and at what layer (constraint vs application)?

7. A prepared statement `WHERE status = $1` stops using your partial index after the app has been running for a minute, though it worked at startup. What changed inside PostgreSQL, and name two fixes?

---

## 16. Quick Reference Card

```sql
-- SYNTAX ------------------------------------------------------------
CREATE INDEX idx_name ON tbl (col)         WHERE predicate;   -- partial
CREATE UNIQUE INDEX uq ON tbl (col)        WHERE predicate;   -- partial UNIQUE
CREATE INDEX idx ON tbl (a, b) INCLUDE (c) WHERE predicate;   -- partial+composite+covering
CREATE INDEX CONCURRENTLY ...              WHERE predicate;   -- no write lock (not in txn)

-- VALID PREDICATES (immutable only) --------------------------------
WHERE status = 'pending'                 -- equality to constant
WHERE deleted_at IS NULL                 -- soft delete
WHERE is_active                          -- boolean column
WHERE amount > 0                         -- range vs constant
WHERE status IN ('pending','processing') -- constant set
-- REJECTED: WHERE created_at > now() - interval '7 days'  (not immutable)

-- WHEN THE PLANNER CAN USE IT (predicate implication) --------------
-- Index WHERE status='pending' is usable when the query's WHERE IMPLIES it:
--   status = 'pending'                     -> yes (exact)
--   status = 'pending' AND x > 5           -> yes (AND narrows)
--   x > 200   (index WHERE x > 100)        -> yes (range containment)
-- NOT usable:
--   status IN ('pending','shipped')        -> no  (widens)
--   status = 'pending' OR x > 5            -> no  (OR widens)
--   (no mention of status)                 -> no  (can't assume)
--   status = $1  (generic plan)            -> no  (constant hidden)

-- EXPLAIN TELLS --------------------------------------------------
-- Index Scan using idx_partial ... Index Cond does NOT show the
--   predicate  -> partial index working (predicate baked in).
-- Seq Scan ... Filter: (status='pending' AND ...) Rows Removed by
--   Filter: huge  -> partial index NOT used (proof failed).

-- PARTIAL UNIQUENESS (conditional constraints) -------------------
CREATE UNIQUE INDEX uq_one_active ON subs (user_id) WHERE status='active';
CREATE UNIQUE INDEX uq_primary   ON addr (cust_id) WHERE is_primary;
CREATE UNIQUE INDEX uq_live_email ON users (lower(email)) WHERE deleted_at IS NULL;

-- UPSERT against a partial unique index (REPEAT the predicate) ----
INSERT INTO subs (user_id, status, plan) VALUES (10,'active','pro')
ON CONFLICT (user_id) WHERE status='active' DO UPDATE SET plan=EXCLUDED.plan;

-- PERF RULES OF THUMB --------------------------------------------
-- * Index size ~ subset size, not table size. Hot subset stays cached.
-- * Writes to non-matching rows skip the index entirely (low write amp).
-- * Best for skewed status, queues, soft-delete, conditional uniqueness.
-- * Query MUST include the predicate (as a constant) to be provable.
-- * Beware prepared-statement generic plans hiding the constant.
-- * High-churn queue index -> tune autovacuum / REINDEX CONCURRENTLY.

-- INTERVIEW ONE-LINERS -------------------------------------------
-- "Don't index rows you never query."
-- "Index size tracks the queue depth, not the table history."
-- "The predicate must NARROW; AND is fine, OR/IN break the proof."
-- "The predicate vanishing from Index Cond means it's working."
-- "A hidden $1 constant defeats a partial index via generic plans."
-- "Partial unique = conditional uniqueness: one active, many cancelled."
```

---

## Connected Topics

- **Topic 40 — B-Tree Index Internals**: A partial index is a normal B-tree with fewer entries; everything about levels, leaf pages, and lookups carries over.
- **Topic 43 — Covering Indexes and Index-Only Scans**: Combine `INCLUDE` with a partial predicate to make hot-subset queries heap-free (Heap Fetches: 0).
- **Topic 44 — Composite Index Strategy** (previous): Column-ordering rules apply to the *key* of a partial index; partial is orthogonal to composite and stacks with it.
- **Topic 46 — Expression Indexes** (next): Partial and expression indexes combine — e.g. `(lower(email)) WHERE deleted_at IS NULL` — and both rely on the planner matching the query to the index definition.
- **Topic 12 — LEFT JOIN and RIGHT JOIN**: The `ON` vs `WHERE` placement that affects what the planner can prove about surviving rows also affects partial-index provability in joins.
- **MVCC / VACUUM internals**: Rows self-prune out of a partial index as they leave the predicate; understanding dead tuples and autovacuum explains partial-index bloat on high-churn queues.
- **Prepared Statements / Generic vs Custom Plans**: The `plan_cache_mode` behavior that can silently defeat a partial index when a constant is passed as `$1`.
- **Topic on Upserts / ON CONFLICT**: Partial unique indexes require the predicate repeated in the conflict target to act as an arbiter.
