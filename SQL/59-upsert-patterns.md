# Topic 59 — Upsert Patterns

### SQL Mastery Curriculum — Phase 9: Advanced SQL Patterns

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a coat-check counter at a theatre. Each guest hands you a coat and a numbered ticket. Your rule is simple: **if that ticket number already has a coat on its hook, replace the old coat with the new one; if the hook is empty, hang the new coat.**

You never stop to ask "does this hook already have something?" as a separate step. You just walk up to the hook, and the hook itself enforces the rule — one coat per number. The number on the ticket is the thing that decides "same guest or new guest." That number is the **conflict target**.

Now suppose two attendants both try to hang a coat on hook #42 at the exact same instant. Only one hook exists. The coat rack itself — the physical constraint — forces them to take turns. One wins, hangs their coat; the other sees "hook occupied" and follows the replace-or-skip rule. Neither attendant had to coordinate in advance; the **hook (the unique constraint)** did the coordination for them.

That is exactly what an **upsert** is:

- "Update or insert" — hence *up-sert*.
- You describe the row you *want to exist*, and the database figures out whether that means an INSERT (empty hook) or an UPDATE (occupied hook).
- The decision is made by a **unique index or constraint**, not by you running a `SELECT` first.
- Concurrency is safe *because* the unique constraint is a real physical thing that two transactions cannot both satisfy independently.

The naive alternative — "check if it exists, then insert or update" — is like an attendant who looks at the hook, walks away to fetch the coat, and comes back to hang it. In that gap, another attendant grabbed the hook. That gap is the **race condition** that upserts exist to eliminate.

---

## 2. Connection to SQL Internals

An upsert is not a new storage operation — it is an INSERT with a **conflict-resolution branch** wired directly into the executor. Understanding what happens underneath explains every gotcha in this topic.

**The unique index is the load-bearing structure.** `INSERT ... ON CONFLICT` cannot function without a unique index (or the implicit unique index behind a `PRIMARY KEY` / `UNIQUE` constraint) on the conflict target. When PostgreSQL inserts a candidate tuple, it descends the B-tree of that unique index to find where the key belongs. If it finds a live tuple with the same key, that is the "conflict" — and the executor switches from insert mode to the `DO NOTHING` / `DO UPDATE` action. There is no separate lookup step you write; the index probe *is* the check.

**Speculative insertion (the internal mechanism).** PostgreSQL implements `ON CONFLICT` using a two-phase technique called *speculative insertion*. The executor first inserts a "speculative token" tuple into the heap and the unique index. If another transaction is concurrently inserting the same key, the speculative insert detects it, **kills its own speculative tuple**, waits for the other transaction to commit or abort, and then re-checks. This is what makes upserts concurrency-safe without you writing explicit locking. It happens in the heap (`heap_insert` with a speculative token) and in `_bt_doinsert` in the B-tree code.

**MVCC and the EXCLUDED tuple.** In `DO UPDATE`, the row you tried to insert is exposed as the pseudo-relation `EXCLUDED`, and the existing conflicting row is the normal target row. The UPDATE creates a **new heap tuple version** (MVCC — remember from Topic 40, PostgreSQL never updates in place), marks the old version dead with your transaction's `xmax`, and inserts new index entries. So an upsert that updates costs a dead tuple that VACUUM must later reclaim — an upsert-heavy table bloats exactly like an update-heavy table.

**WAL and durability.** Both the INSERT branch and the UPDATE branch emit WAL records (Topic 41). A `DO UPDATE` that fires emits an update WAL record; a `DO NOTHING` that skips emits nothing for the skipped row. High-volume upserts are WAL-heavy, which matters for replication lag and `checkpoint` pressure.

**The planner barely participates.** An upsert is almost always a trivial plan: `Insert on <table>` with a `Conflict Resolution` node. There is no join reordering, no algorithm choice — the "plan" is fixed. The interesting cost is entirely in the index probe, the heap write, and the WAL.

**Row locks.** When `DO UPDATE` fires on an existing row, that row is locked (`FOR UPDATE`-style) for the duration of the transaction. Concurrent upserts targeting the *same* key serialize on that row lock — this is the throughput ceiling for hot-key upserts.

---

## 3. Logical Execution Order Context

An upsert is a write statement, so it does not slot into the `FROM → WHERE → GROUP BY → SELECT → ORDER BY` pipeline the way a query does. Its internal order is its own:

```
1. Evaluate the VALUES / SELECT source rows        ← produce candidate tuples
2. For each candidate, probe the conflict target   ← the unique index descent
3a. No conflict  → perform the INSERT
3b. Conflict     → evaluate ON CONFLICT action:
      DO NOTHING → skip this row
      DO UPDATE  → evaluate SET using EXCLUDED + existing row
                 → apply optional WHERE (the "DO UPDATE WHERE")
                 → if WHERE is TRUE  → write new tuple version
                 → if WHERE is FALSE → skip (row untouched, NOT inserted)
4. Evaluate RETURNING (if present) on the affected rows
```

Two ordering facts that trip people up:

- **The `WHERE` on `DO UPDATE` runs *after* the conflict is detected**, not before. It cannot prevent the conflict; it can only decide whether the already-conflicting row gets updated. A row that conflicts and fails the `WHERE` is neither inserted nor updated — it silently does nothing.
- **`EXCLUDED` is materialized before the UPDATE.** Inside `SET` and inside the `DO UPDATE WHERE`, `EXCLUDED.col` refers to the value you *tried to insert*, and the bare `col` (or `table.col`) refers to the value *currently in the row*. Both are available simultaneously — this is what lets you write `SET count = table.count + EXCLUDED.count`.

If the source is a multi-row `INSERT ... SELECT`, rows are processed as a set; **a single statement cannot conflict with two different existing rows and merge them** — and critically, a single statement *cannot upsert the same key twice* (the "cardinality violation," covered in §6).

---

## 4. What Is Upsert?

An **upsert** is a single atomic statement that inserts a row, and — if inserting would violate a specified unique constraint — instead either skips the insert or updates the existing conflicting row. PostgreSQL spells it `INSERT ... ON CONFLICT`, standardized in PostgreSQL 9.5. PostgreSQL 15 added the SQL-standard `MERGE` as a second, more general mechanism.

```sql
INSERT INTO products (sku, name, price, updated_at)
--          │        └── the columns being supplied
--          └── target table
VALUES ('ABC-123', 'Widget', 9.99, NOW())
--     └── the candidate row(s) to insert
ON CONFLICT (sku)
--          │  └── the CONFLICT TARGET: a column list matching a UNIQUE index/constraint
--          └── begins the conflict-resolution clause
DO UPDATE
--  └── the ACTION when a conflict occurs (alternative: DO NOTHING)
SET name       = EXCLUDED.name,
--             │  └── EXCLUDED = the row we TRIED to insert
--             └── assign new values to the existing row's columns
    price      = EXCLUDED.price,
    updated_at = EXCLUDED.updated_at
WHERE products.price IS DISTINCT FROM EXCLUDED.price
--    │        └── optional predicate: only update when something actually changed
--    └── qualify with the TABLE name (or alias), not EXCLUDED
RETURNING id, (xmax = 0) AS was_inserted;
--        │                 └── trick: xmax = 0 means the row was freshly inserted
--        └── optional: return columns from affected rows
```

### The three forms in one place

```sql
-- Form 1: idempotent insert (skip duplicates silently)
INSERT INTO t (...) VALUES (...) ON CONFLICT (key) DO NOTHING;

-- Form 2: insert-or-replace (last write wins)
INSERT INTO t (...) VALUES (...) ON CONFLICT (key)
  DO UPDATE SET col = EXCLUDED.col;

-- Form 3: insert-or-merge (accumulate / conditional)
INSERT INTO t (...) VALUES (...) ON CONFLICT (key)
  DO UPDATE SET count = t.count + EXCLUDED.count
  WHERE <some predicate>;
```

### Conflict target variants

```sql
ON CONFLICT (sku)                              -- column list
ON CONFLICT (lower(email))                     -- expression (needs matching expression index)
ON CONFLICT (tenant_id, sku)                   -- composite
ON CONFLICT ON CONSTRAINT products_sku_key     -- named constraint
ON CONFLICT (sku) WHERE discontinued = false   -- partial index target (index predicate)
ON CONFLICT DO NOTHING                          -- no target: matches ANY unique violation
```

The bare `ON CONFLICT DO NOTHING` (no target) is special — it catches a conflict on *any* unique constraint on the table. This is the only form that lets you omit the target, and only `DO NOTHING` allows it (`DO UPDATE` always needs a target, because PostgreSQL must know *which* row to update).

---

## 5. Why Upsert Mastery Matters in Production

1. **Idempotency is the backbone of reliable APIs.** Every retried HTTP request, every re-delivered message-queue event, every webhook that fires twice must not create duplicate rows. `ON CONFLICT DO NOTHING` / `DO UPDATE` turns "insert" into "ensure this row exists" — the single most important primitive for at-least-once delivery systems. Without it, you get duplicate orders, double-charged payments, and phantom users.

2. **The check-then-insert race is a real, exploitable bug.** The naive `SELECT ... then INSERT or UPDATE` pattern has a window between the check and the write where a concurrent transaction slips in. Under load this produces duplicate-key errors (best case, a 500) or duplicate rows (worst case, if there's no unique constraint). Upsert closes the window at the storage layer.

3. **It collapses two round-trips into one.** Check-then-write is two statements and often two network round-trips from Node.js. An upsert is one statement, one round-trip, one plan, one lock acquisition. At 10K writes/sec this halves your database round-trips.

4. **Bulk ingestion and sync jobs depend on it.** ETL, catalog syncs, upserting analytics counters, "materialize this external dataset into our table" — all are naturally expressed as one bulk upsert. The alternative (delete-all-then-insert, or row-by-row check) is slower and less safe.

5. **Getting the conflict target wrong causes silent data corruption or hard errors.** If your `ON CONFLICT (email)` doesn't have a matching unique index, PostgreSQL raises an error at runtime — not at deploy time. If you target the wrong column, you either overwrite the wrong row or never detect the conflict you meant to. This is a class of bug that only appears under real data.

6. **Upsert-heavy tables bloat.** Every `DO UPDATE` leaves a dead tuple. A counter table hammered with upserts needs aggressive autovacuum tuning or it bloats and slows down — a production incident waiting to happen if you don't know it's coming.

---

## 6. Deep Technical Content

### 6.1 The Conflict Target Must Match a Unique Arbiter

The conflict target (`ON CONFLICT (cols)`) is not just "columns to check." PostgreSQL uses it to select an **arbiter index** — a unique index whose definition matches the target. If no unique index matches, you get:

```
ERROR:  there is no unique or exclusion constraint matching the ON CONFLICT specification
```

The match must be exact in structure:

```sql
-- Requires a unique index on (tenant_id, sku)
CREATE UNIQUE INDEX ON products (tenant_id, sku);
INSERT INTO products (...) ON CONFLICT (tenant_id, sku) DO UPDATE ...;  -- OK

-- Order of columns does NOT matter for arbiter selection:
ON CONFLICT (sku, tenant_id) DO UPDATE ...;  -- also matches the same index

-- But you cannot target a NON-unique index:
CREATE INDEX ON products (category_id);
ON CONFLICT (category_id) DO UPDATE ...;  -- ERROR: no unique constraint matches
```

For **expression indexes** and **partial indexes**, the target must reproduce the expression / predicate:

```sql
-- Case-insensitive unique email
CREATE UNIQUE INDEX users_email_lower ON users (lower(email));
INSERT INTO users (email) VALUES ('Bob@X.com')
ON CONFLICT (lower(email)) DO UPDATE SET email = EXCLUDED.email;  -- must repeat lower()

-- Partial unique index: only one active subscription per user
CREATE UNIQUE INDEX one_active_sub ON subscriptions (user_id) WHERE status = 'active';
INSERT INTO subscriptions (user_id, status) VALUES (7, 'active')
ON CONFLICT (user_id) WHERE status = 'active' DO UPDATE SET renewed_at = NOW();
--                    └── the index predicate must be restated here
```

### 6.2 The EXCLUDED Pseudo-Table

Inside `DO UPDATE`, two versions of the row are in scope:

| Reference | Meaning |
|-----------|---------|
| `EXCLUDED.col` | the value from the row you *tried to insert* (was "excluded" from insertion) |
| `table.col` or `alias.col` | the value *currently stored* in the conflicting row |

This dual visibility is the whole point — it lets you compute the new value from both:

```sql
-- Last-write-wins: overwrite with incoming value
SET price = EXCLUDED.price

-- Accumulate: add incoming to existing (counters, running totals)
SET view_count = page_stats.view_count + EXCLUDED.view_count

-- Keep the max: don't let a stale event lower a high-water mark
SET last_seen = GREATEST(devices.last_seen, EXCLUDED.last_seen)

-- Coalesce: only fill in nulls, never overwrite existing data
SET phone = COALESCE(users.phone, EXCLUDED.phone)

-- Merge JSON: shallow-merge incoming metadata into existing
SET metadata = users.metadata || EXCLUDED.metadata
```

`EXCLUDED` also carries columns you did **not** name in the SET but which came from the INSERT — including defaults evaluated for the candidate row. `EXCLUDED.created_at` reflects whatever `created_at` the insert would have used (its DEFAULT if not supplied).

### 6.3 The WHERE Clause on DO UPDATE

`DO UPDATE ... WHERE <predicate>` decides whether the already-conflicting row is actually updated. It runs *after* the conflict is detected. Two independent uses:

**(a) Avoid no-op writes (reduce bloat and WAL).** An update that sets a column to the value it already holds still writes a new tuple version. Guard it:

```sql
INSERT INTO products (sku, price) VALUES ('ABC', 9.99)
ON CONFLICT (sku) DO UPDATE SET price = EXCLUDED.price
WHERE products.price IS DISTINCT FROM EXCLUDED.price;
--    └── if price is unchanged, skip the write entirely — no dead tuple, no WAL
```

Use `IS DISTINCT FROM`, not `<>`, so that NULLs compare correctly (Topic 07 — `NULL <> NULL` is UNKNOWN, which would wrongly skip).

**(b) Conditional / ordered updates (reject stale data).** Only apply the update if the incoming row is "newer" or otherwise wins:

```sql
-- Only update if the incoming event is more recent (out-of-order event guard)
INSERT INTO device_state (device_id, state, event_ts)
VALUES ($1, $2, $3)
ON CONFLICT (device_id) DO UPDATE
  SET state = EXCLUDED.state, event_ts = EXCLUDED.event_ts
WHERE EXCLUDED.event_ts > device_state.event_ts;
--    └── a late-arriving old event fails this and is ignored
```

**The crucial subtlety:** when the `WHERE` is false on a conflict, the statement does **nothing** for that row — it does not fall back to inserting (the row already exists), and it does not update. `RETURNING` returns no row for it. Many developers wrongly assume a failed `DO UPDATE WHERE` re-attempts an insert; it does not.

There is also a separate `WHERE` on the **conflict target** for partial indexes (§6.1) — do not confuse `ON CONFLICT (x) WHERE <index predicate>` (arbiter selection) with `DO UPDATE ... WHERE <action predicate>` (update gating). They are different clauses in different positions.

### 6.4 DO NOTHING Semantics

`DO NOTHING` skips the row on conflict — no error, no update.

```sql
INSERT INTO event_dedup (event_id, payload) VALUES ($1, $2)
ON CONFLICT (event_id) DO NOTHING
RETURNING id;
```

Key behaviors:

- **`RETURNING` returns nothing for skipped rows.** If the row already existed, `RETURNING` yields zero rows. This is how you detect "was this a fresh insert?" — a returned row means inserted; empty means it already existed (see §6.7).
- **Bare `DO NOTHING` (no target) swallows *all* unique/exclusion violations.** Convenient but dangerous: it can hide a conflict on a constraint you didn't mean to ignore. Prefer naming the target: `ON CONFLICT (event_id) DO NOTHING`.
- **`DO NOTHING` does not lock the existing row.** Unlike `DO UPDATE`, which takes a row lock, `DO NOTHING` on an existing row just moves on. (It still had to attempt speculative insertion and detect the conflict.)

### 6.5 The Cardinality Violation — Duplicate Keys in One Statement

A single `INSERT ... ON CONFLICT DO UPDATE` **cannot** contain two source rows that map to the same conflict key:

```sql
INSERT INTO stock (sku, qty) VALUES
  ('ABC', 5),
  ('ABC', 3)          -- same sku twice in one statement
ON CONFLICT (sku) DO UPDATE SET qty = stock.qty + EXCLUDED.qty;
```

```
ERROR:  ON CONFLICT DO UPDATE command cannot affect row a second time
HINT:   Ensure that no rows proposed for insertion within the same command
        have duplicate constrained values.
```

This is because the update on the first `ABC` produces a row the second `ABC` would have to touch again in the same command — disallowed to prevent nondeterministic results. **The fix is to de-duplicate and pre-aggregate the source before upserting:**

```sql
INSERT INTO stock (sku, qty)
SELECT sku, SUM(qty)
FROM (VALUES ('ABC', 5), ('ABC', 3)) AS v(sku, qty)
GROUP BY sku
ON CONFLICT (sku) DO UPDATE SET qty = stock.qty + EXCLUDED.qty;
```

This is one of the most common real-world upsert bugs in bulk ingestion — the batch contains the same key twice, and the statement blows up under production data.

### 6.6 Multiple Unique Constraints — Which One Arbitrates?

If a table has several unique constraints, the conflict target picks exactly one arbiter. A conflict on a *different* constraint is **not** handled — it raises the normal duplicate-key error:

```sql
CREATE TABLE accounts (
  id     bigint PRIMARY KEY,
  email  text UNIQUE,
  handle text UNIQUE
);

INSERT INTO accounts (id, email, handle) VALUES (1, 'a@x.com', 'alice')
ON CONFLICT (email) DO UPDATE SET handle = EXCLUDED.handle;
-- If a row with the SAME handle but DIFFERENT email exists,
-- this raises: duplicate key value violates unique constraint "accounts_handle_key"
-- because we only told it to arbitrate on email.
```

You cannot upsert against two arbiters in one `INSERT ... ON CONFLICT`. If you truly need "match either email or handle," you need application logic, a `MERGE`, or a redesign. Bare `ON CONFLICT DO NOTHING` (no target) is the only construct that tolerates any-constraint conflicts, and only by skipping.

### 6.7 Detecting Insert vs Update

A frequent requirement: "did this upsert create a new row or update an existing one?" Two techniques:

```sql
-- Technique 1: the xmax trick (works for DO UPDATE)
INSERT INTO users (email, name) VALUES ($1, $2)
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name
RETURNING id, (xmax = 0) AS was_inserted;
-- Freshly inserted rows have xmax = 0; updated rows have a non-zero xmax.
```

```sql
-- Technique 2: system column via a flag in an expression
INSERT INTO users (email, name) VALUES ($1, $2)
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name
RETURNING id, (created_at = updated_at) AS was_inserted;
-- Works only if you set both timestamps identically on insert
-- and always bump updated_at on update.
```

The `xmax = 0` trick is idiomatic but relies on an implementation detail (it can misreport under certain subtransaction/rollback scenarios). For business-critical distinctions, prefer an explicit column or `MERGE`'s `WHEN MATCHED`/`WHEN NOT MATCHED` reporting.

### 6.8 Sequences Are Consumed Even on Conflict

If the target has a `SERIAL` / `IDENTITY` / `GENERATED` id, the sequence is advanced **for every candidate row, including rows that end up conflicting and doing nothing/updating.** Sequences are non-transactional and never roll back.

```sql
CREATE TABLE t (id bigserial PRIMARY KEY, sku text UNIQUE, name text);

-- Every attempted insert grabs a nextval('t_id_seq'), even if it conflicts on sku
INSERT INTO t (sku, name) VALUES ('ABC', 'x')
ON CONFLICT (sku) DO UPDATE SET name = EXCLUDED.name;
-- Result: gaps in the id sequence over time. This is normal and expected.
```

**Implication:** never treat a serial primary key as gap-free or as a count of successful inserts. Upsert-heavy tables burn through sequence values. Use `bigint` ids, not `int`, if you upsert at volume.

### 6.9 Triggers and Upsert

Which triggers fire depends on which branch runs:

- INSERT branch → `BEFORE INSERT` and `AFTER INSERT` triggers fire.
- UPDATE branch (`DO UPDATE` that applies) → `BEFORE INSERT` fires *first* (the row was attempted as an insert), then `BEFORE UPDATE` and `AFTER UPDATE` fire.

That ordering surprises people: a `BEFORE INSERT` trigger runs even for rows that end up as updates, because the executor genuinely attempts the insert first (speculative insertion) before discovering the conflict. If your `BEFORE INSERT` trigger has side effects, they occur on the update path too. `MERGE` gives cleaner trigger semantics (it fires triggers matching the action actually taken).

### 6.10 MERGE (PostgreSQL 15+)

`MERGE` is the SQL-standard, more general statement. It joins a target table to a source and runs `WHEN MATCHED` / `WHEN NOT MATCHED` actions. It is strictly more expressive than `ON CONFLICT` (can DELETE, can have multiple conditional branches, joins on arbitrary conditions), but has **different concurrency semantics**.

```sql
MERGE INTO inventory AS tgt
USING (VALUES ($1::text, $2::int)) AS src(sku, delta)
ON tgt.sku = src.sku
WHEN MATCHED AND tgt.qty + src.delta <= 0 THEN
  DELETE
WHEN MATCHED THEN
  UPDATE SET qty = tgt.qty + src.delta
WHEN NOT MATCHED THEN
  INSERT (sku, qty) VALUES (src.sku, src.delta);
```

Line-by-line:
- `MERGE INTO ... USING ... ON` — join target to source on an arbitrary condition (not restricted to a unique index).
- `WHEN MATCHED AND ...` — multiple matched branches, evaluated top-to-bottom; first satisfied branch wins.
- `WHEN MATCHED THEN DELETE` — something `ON CONFLICT` cannot do.
- `WHEN NOT MATCHED THEN INSERT` — the insert branch.

**Critical difference — MERGE is NOT a drop-in concurrency-safe upsert.** `ON CONFLICT` uses speculative insertion and the unique index to be race-proof. `MERGE` evaluates its `ON` condition against the snapshot; under concurrency, two `MERGE`s can both see "not matched," both try to insert, and one hits a **unique-violation error** (if a unique constraint exists) or creates a **duplicate** (if it doesn't). PostgreSQL's docs state plainly that `MERGE` can raise a serialization/unique error under concurrent modification and that you may need retry logic or `SERIALIZABLE` isolation.

**Rule of thumb:** for a plain idempotent single-key upsert under concurrency, prefer `INSERT ... ON CONFLICT` — it is purpose-built and race-proof. Reach for `MERGE` when you need its extra power (conditional DELETE, multiple branches, join-based matching) and either the workload is not concurrent on the same keys or you add retry/serializable handling.

### 6.11 Upsert and Foreign Keys / RETURNING chains

`RETURNING` from an upsert can feed a CTE, which is the idiomatic way to upsert a parent and use its id for children:

```sql
WITH up AS (
  INSERT INTO users (email, name) VALUES ($1, $2)
  ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name
  RETURNING id
)
INSERT INTO audit_logs (user_id, action)
SELECT id, 'profile_sync' FROM up;
```

Note: with `DO NOTHING`, the CTE returns **no row** when the user already existed, so the child insert would insert nothing. If you always need the id, use `DO UPDATE SET email = EXCLUDED.email` (a self-assignment that always fires and always returns the row) or a `DO NOTHING` + fallback `SELECT` union — see §9.

---

## 7. EXPLAIN — Upsert in the Plan

Upsert plans are simple but reveal the index probe and the conflict branch.

### DO NOTHING, no conflict (fresh insert)

```sql
EXPLAIN (ANALYZE, BUFFERS)
INSERT INTO event_dedup (event_id, payload)
VALUES ('550e8400-e29b-41d4-a716-446655440000', '{"k":1}')
ON CONFLICT (event_id) DO NOTHING;
```

```
Insert on event_dedup  (cost=0.00..0.01 rows=0 width=0)
                        (actual time=0.089..0.089 rows=0 loops=1)
  Conflict Resolution: NOTHING
  Conflict Arbiter Indexes: event_dedup_event_id_key
  Tuples Inserted: 1
  Conflicting Tuples: 0
  ->  Result  (cost=0.00..0.01 rows=1 width=48)
              (actual time=0.004..0.005 rows=1 loops=1)
  Buffers: shared hit=5 dirtied=2
Planning Time: 0.061 ms
Execution Time: 0.140 ms
```

**Reading it:**
- `Conflict Resolution: NOTHING` — this is `DO NOTHING`.
- `Conflict Arbiter Indexes: event_dedup_event_id_key` — the unique index chosen as arbiter. If this line is missing or wrong, your target didn't resolve as intended.
- `Tuples Inserted: 1`, `Conflicting Tuples: 0` — one new row, no conflict.
- `dirtied=2` — the heap page and index page were modified.

### DO UPDATE, conflict fires

```sql
EXPLAIN (ANALYZE, BUFFERS)
INSERT INTO products (sku, name, price)
VALUES ('ABC-123', 'Widget v2', 12.50)
ON CONFLICT (sku) DO UPDATE
  SET name = EXCLUDED.name, price = EXCLUDED.price
WHERE products.price IS DISTINCT FROM EXCLUDED.price;
```

```
Insert on products  (cost=0.00..0.01 rows=0 width=0)
                     (actual time=0.201..0.201 rows=0 loops=1)
  Conflict Resolution: UPDATE
  Conflict Arbiter Indexes: products_sku_key
  Tuples Inserted: 0
  Conflicting Tuples: 1
  ->  Result  (cost=0.00..0.01 rows=1 width=44)
              (actual time=0.006..0.007 rows=1 loops=1)
  Buffers: shared hit=9 dirtied=3
Planning Time: 0.144 ms
Execution Time: 0.260 ms
```

**Reading it:**
- `Conflict Resolution: UPDATE` — `DO UPDATE` path.
- `Tuples Inserted: 0`, `Conflicting Tuples: 1` — nothing new inserted; one existing row was hit and updated (the `WHERE` passed).
- Higher `Buffers` than the insert case: the arbiter index was probed, the old tuple was marked dead, a new tuple version written, and index entries updated → more pages dirtied.
- If the `WHERE` had failed, you would still see `Conflicting Tuples: 1` but the row would not be rewritten (fewer dirtied buffers).

### Bulk upsert (INSERT ... SELECT)

```sql
EXPLAIN (ANALYZE, BUFFERS)
INSERT INTO daily_counters (day, metric, n)
SELECT day, metric, SUM(n)
FROM staging_counters
GROUP BY day, metric
ON CONFLICT (day, metric) DO UPDATE SET n = daily_counters.n + EXCLUDED.n;
```

```
Insert on daily_counters  (cost=1943.00..2205.00 rows=0 width=0)
                          (actual time=142.3..142.3 rows=0 loops=1)
  Conflict Resolution: UPDATE
  Conflict Arbiter Indexes: daily_counters_day_metric_key
  Tuples Inserted: 8123
  Conflicting Tuples: 41877
  ->  HashAggregate  (cost=1943.00..2075.00 rows=50000 width=20)
                     (actual time=98.1..118.4 rows=50000 loops=1)
        Group Key: staging_counters.day, staging_counters.metric
        Batches: 1  Memory Usage: 4096kB
        ->  Seq Scan on staging_counters
              (actual time=0.01..24.6 rows=250000 loops=1)
  Buffers: shared hit=120334 read=842 dirtied=6210
Planning Time: 0.30 ms
Execution Time: 168.9 ms
```

**Reading it:**
- The `HashAggregate` (`GROUP BY`) is the de-duplication that avoids the cardinality violation from §6.5 — essential in bulk upserts.
- `Tuples Inserted: 8123` + `Conflicting Tuples: 41877` = 50,000 distinct keys processed. Most were updates.
- `dirtied=6210` — the cost of ~42K updates leaving dead tuples; this is the bloat pressure to watch.

### What to look for

| Signal in plan | Meaning / action |
|----------------|------------------|
| `Conflict Arbiter Indexes:` names the wrong index | your target resolved to an unintended constraint |
| `Conflicting Tuples` ≫ `Tuples Inserted` | mostly updates → bloat/vacuum concern |
| High `dirtied` on a hot table | add a `DO UPDATE ... WHERE` no-op guard (§6.3) |
| Sort/HashAggregate absent before bulk upsert | risk of cardinality-violation on duplicate source keys |

---

## 8. Query Examples

### Example 1 — Basic: Idempotent Insert

```sql
-- Ensure a tag exists; if it already does, do nothing.
-- Classic "get or create" without a race.
INSERT INTO categories (slug, name)
VALUES ('electronics', 'Electronics')
ON CONFLICT (slug) DO NOTHING;
-- Safe to run 1000 times; creates the row exactly once.
```

### Example 2 — Intermediate: Insert-or-Update a User Profile

```sql
-- Sync a user from an external identity provider.
-- On repeat syncs, refresh mutable fields but keep created_at intact,
-- and only write when something actually changed.
INSERT INTO users (email, name, avatar_url, updated_at)
VALUES ($1, $2, $3, NOW())
ON CONFLICT (email) DO UPDATE
  SET name       = EXCLUDED.name,
      avatar_url = EXCLUDED.avatar_url,
      updated_at = NOW()
WHERE users.name       IS DISTINCT FROM EXCLUDED.name
   OR users.avatar_url IS DISTINCT FROM EXCLUDED.avatar_url
RETURNING id, (xmax = 0) AS was_created;
-- Returns the id and whether this call created the user.
-- No-op syncs (nothing changed) return zero rows and write nothing.
```

### Example 3 — Production Grade: Bulk Counter Upsert with Accumulation

**Context:** an analytics pipeline flushes per-minute event counts from a staging table into a rolled-up `metric_counters(bucket, metric, count)` table. The staging batch can contain the same `(bucket, metric)` many times. Target table: ~80M rows, unique index on `(bucket, metric)`. Expected: sub-second for a 200K-row staging flush; must be idempotent so a re-run of the same batch (after a crash) doesn't double-count — which we handle by draining staging in the same transaction.

```sql
BEGIN;

-- 1. Aggregate the staging batch FIRST to avoid the cardinality violation
--    (one statement cannot touch the same key twice).
WITH batch AS (
  SELECT bucket, metric, SUM(delta) AS delta
  FROM staging_events
  WHERE flushed = false
  GROUP BY bucket, metric
)
INSERT INTO metric_counters (bucket, metric, count, updated_at)
SELECT bucket, metric, delta, NOW()
FROM batch
ON CONFLICT (bucket, metric) DO UPDATE
  SET count      = metric_counters.count + EXCLUDED.count,
      updated_at = NOW();
--    └── accumulate: existing + incoming (this is why we pre-summed)

-- 2. Mark the staging rows drained in the same transaction (atomic with the upsert).
UPDATE staging_events SET flushed = true WHERE flushed = false;

COMMIT;
```

**EXPLAIN (the upsert step):**

```
Insert on metric_counters  (cost=4021.0..4998.0 rows=0 width=0)
                           (actual time=210.4..210.4 rows=0 loops=1)
  Conflict Resolution: UPDATE
  Conflict Arbiter Indexes: metric_counters_bucket_metric_key
  Tuples Inserted: 1204
  Conflicting Tuples: 18796
  ->  Subquery Scan on batch  (actual rows=20000 loops=1)
        ->  HashAggregate  (actual time=88.2..140.1 rows=20000 loops=1)
              Group Key: staging_events.bucket, staging_events.metric
              ->  Seq Scan on staging_events (actual rows=200000 loops=1)
  Buffers: shared hit=88012 read=1902 dirtied=3140
Execution Time: 233.7 ms
```

- The `HashAggregate` collapses 200K staging rows into 20K distinct keys — no cardinality violation, and each key is touched once.
- 1,204 new counters, 18,796 accumulated. The whole thing is one atomic transaction with the staging drain, so a crash-and-retry re-reads only un-flushed rows: idempotent by construction.
- `dirtied=3140` is the update bloat; this table needs a tuned autovacuum (`autovacuum_vacuum_scale_factor` lowered) because it is update-dominated.

---

## 9. Wrong → Right Patterns

### Wrong 1: Check-then-insert (the race condition)

```sql
-- WRONG: two statements with a gap between them
SELECT id FROM users WHERE email = 'a@x.com';   -- returns nothing
-- ... another transaction inserts 'a@x.com' right here ...
INSERT INTO users (email, name) VALUES ('a@x.com', 'Alice');
-- ERROR: duplicate key value violates unique constraint "users_email_key"
```

**Why it's wrong at the execution level:** the `SELECT` reads a snapshot; between it and the `INSERT`, a concurrent transaction commits the same key. There is no lock held across the gap. Under load this throws duplicate-key errors (or, without a unique constraint, silently creates duplicates).

```sql
-- RIGHT: one atomic statement, race closed by the unique index
INSERT INTO users (email, name) VALUES ('a@x.com', 'Alice')
ON CONFLICT (email) DO NOTHING
RETURNING id;
-- Speculative insertion serializes concurrent inserters on the index.
```

### Wrong 2: ON CONFLICT with no matching unique index

```sql
-- WRONG: email has only a plain (non-unique) index
CREATE INDEX users_email_idx ON users (email);   -- NOT unique
INSERT INTO users (email, name) VALUES ('a@x.com', 'Alice')
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;
-- ERROR: there is no unique or exclusion constraint matching
--        the ON CONFLICT specification
```

**Why it's wrong:** the conflict target must resolve to a *unique* arbiter index. A non-unique index cannot arbitrate. This error surfaces only at runtime.

```sql
-- RIGHT: make the arbiter unique
CREATE UNIQUE INDEX users_email_key ON users (email);
INSERT INTO users (email, name) VALUES ('a@x.com', 'Alice')
ON CONFLICT (email) DO UPDATE SET name = EXCLUDED.name;
```

### Wrong 3: Duplicate keys in the same bulk statement

```sql
-- WRONG: same sku twice in one command
INSERT INTO stock (sku, qty) VALUES ('A', 5), ('A', 3)
ON CONFLICT (sku) DO UPDATE SET qty = stock.qty + EXCLUDED.qty;
-- ERROR: ON CONFLICT DO UPDATE command cannot affect row a second time
```

**Why it's wrong:** the first `A` produces a row the second `A` would update again in the same statement — disallowed. Common in ingestion where the batch isn't pre-deduplicated.

```sql
-- RIGHT: pre-aggregate the source so each key appears once
INSERT INTO stock (sku, qty)
SELECT sku, SUM(qty) FROM (VALUES ('A', 5), ('A', 3)) v(sku, qty)
GROUP BY sku
ON CONFLICT (sku) DO UPDATE SET qty = stock.qty + EXCLUDED.qty;
```

### Wrong 4: Referencing the wrong row version in SET

```sql
-- WRONG: trying to add, but reading EXCLUDED on both sides
INSERT INTO counters (k, n) VALUES ('hits', 1)
ON CONFLICT (k) DO UPDATE SET n = EXCLUDED.n + EXCLUDED.n;
-- BUG: this sets n = 1 + 1 = 2 every time, ignoring the stored value.
-- The counter never accumulates past 2.
```

**Why it's wrong:** `EXCLUDED.n` is the *incoming* value only. To accumulate you must read the *existing* value via the table name.

```sql
-- RIGHT: existing value from the table, increment from EXCLUDED
INSERT INTO counters (k, n) VALUES ('hits', 1)
ON CONFLICT (k) DO UPDATE SET n = counters.n + EXCLUDED.n;
```

### Wrong 5: Expecting DO UPDATE WHERE to fall back to insert

```sql
-- WRONG mental model
INSERT INTO device_state (device_id, state, event_ts) VALUES ($1, $2, $3)
ON CONFLICT (device_id) DO UPDATE
  SET state = EXCLUDED.state, event_ts = EXCLUDED.event_ts
WHERE EXCLUDED.event_ts > device_state.event_ts
RETURNING id;
-- Developer assumes: "if WHERE fails, at least insert." It does NOT.
-- On a stale event, WHERE is false → row untouched, RETURNING empty.
-- That's actually CORRECT here — but only if you understand it.
```

**Why the confusion is dangerous:** if you *needed* the row to exist and relied on the "fall back to insert" that never happens, you'd have a missing row. Here the row already exists (we're in the conflict branch), so "do nothing on stale event" is the intended behavior — but you must know the `WHERE` gates the update only, never re-triggers an insert.

```sql
-- RIGHT: same query, correct expectation documented — a false WHERE means
-- "keep the existing (newer) row." Handle the empty RETURNING in app code
-- as "no-op, existing data was newer."
```

---

## 10. Performance Profile

### Cost per row

An upsert's cost is: one unique-index descent (B-tree, ~`log(N)` page accesses) + heap write + WAL. For the update branch, add: old-tuple invalidation, new-tuple heap insert, and index maintenance for every index on the table.

| Operation | Dominant cost | Memory |
|-----------|--------------|--------|
| `DO NOTHING`, no conflict | index probe + heap insert + WAL | O(1) |
| `DO NOTHING`, conflict | index probe (then skip) | O(1) |
| `DO UPDATE`, conflict | index probe + new tuple + all-index update + WAL | O(1) |
| Bulk upsert (N rows) | N × above + one HashAggregate to dedup | O(distinct keys) for the sort/hash |

### Scaling

| Table size | Single upsert | Bulk upsert (per 100K rows) | Notes |
|-----------|---------------|------------------------------|-------|
| 1M rows | < 0.3 ms | ~100–300 ms | index fits in cache; trivial |
| 10M rows | < 0.5 ms | ~200–600 ms | arbiter index still cache-friendly |
| 100M rows | < 1 ms (if index cached) | ~0.5–2 s | index may not fully fit in RAM; random I/O on probes dominates |

The single-row upsert stays fast even at 100M rows *as long as the unique index's upper levels stay in the buffer pool* — a B-tree descent is a handful of page reads regardless of table size. The bulk case scales with the number of rows and, critically, with how random the keys are (sequential keys keep index writes hot; random UUIDs scatter writes and hurt cache locality — a reason to prefer `uuid v7` / time-ordered keys for high-volume upsert targets).

### The bloat problem (upsert-specific)

Every `DO UPDATE` leaves a dead tuple (MVCC). A hot counter table receiving 10K upserts/sec generates 10K dead tuples/sec. If autovacuum can't keep up:
- The table and its indexes bloat.
- Index probes touch more pages → upserts slow down → positive feedback loop.

Mitigations:
- **Guard no-op updates** with `DO UPDATE ... WHERE col IS DISTINCT FROM EXCLUDED.col` — a skipped update creates no dead tuple.
- **Tune autovacuum per-table**: `ALTER TABLE counters SET (autovacuum_vacuum_scale_factor = 0.02, autovacuum_vacuum_cost_delay = 0);`
- **Consider `fillfactor < 100`** (e.g. 80) so HOT updates can reuse space on the same page when no indexed column changed — HOT updates avoid index bloat entirely.
- **Batch upserts** rather than one-row-per-statement: one statement of 1,000 keys amortizes planning and WAL flushing over 1,000 rows.

### Lock contention on hot keys

`DO UPDATE` locks the conflicting row for the transaction's duration. If thousands of concurrent transactions upsert the **same** key (a single global counter), they serialize on that row lock — throughput collapses to "one update per row-lock round trip." Fix by sharding the hot key (e.g. `counter_shard_{0..N}` rows summed at read time) so writes spread across many rows.

---

## 11. Node.js Integration

### 11.1 Basic idempotent insert with `pg`

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Get-or-create a category; returns the row whether it existed or not.
async function ensureCategory(slug, name) {
  const { rows } = await pool.query(
    `INSERT INTO categories (slug, name)
     VALUES ($1, $2)
     ON CONFLICT (slug) DO UPDATE SET slug = EXCLUDED.slug
     RETURNING id, slug, name`,
    [slug, name]
  );
  // Note the self-assignment `SET slug = EXCLUDED.slug`: it always fires,
  // so RETURNING always yields the row (DO NOTHING would return nothing
  // when the row already existed).
  return rows[0];
}
```

### 11.2 Upsert with insert-vs-update detection

```javascript
async function syncUser(email, name, avatarUrl) {
  const { rows } = await pool.query(
    `INSERT INTO users (email, name, avatar_url, updated_at)
     VALUES ($1, $2, $3, NOW())
     ON CONFLICT (email) DO UPDATE
       SET name = EXCLUDED.name,
           avatar_url = EXCLUDED.avatar_url,
           updated_at = NOW()
     WHERE users.name       IS DISTINCT FROM EXCLUDED.name
        OR users.avatar_url IS DISTINCT FROM EXCLUDED.avatar_url
     RETURNING id, (xmax = 0) AS was_created`,
    [email, name, avatarUrl]
  );
  if (rows.length === 0) {
    // No row returned = conflict occurred but WHERE failed (nothing changed).
    return { changed: false };
  }
  return { changed: true, id: rows[0].id, created: rows[0].was_created };
}
```

### 11.3 Idempotent webhook handler (the canonical use case)

```javascript
// At-least-once delivery: the same webhook may arrive twice.
// Dedup on the provider's event id so we process each event once.
async function handleWebhook(eventId, payload) {
  const { rows } = await pool.query(
    `INSERT INTO processed_events (event_id, payload, received_at)
     VALUES ($1, $2, NOW())
     ON CONFLICT (event_id) DO NOTHING
     RETURNING id`,
    [eventId, payload]
  );
  if (rows.length === 0) {
    return { duplicate: true };  // already processed — ack and skip work
  }
  await doExpensiveProcessing(payload);  // only runs once per event
  return { duplicate: false };
}
```

### 11.4 Bulk upsert with `UNNEST` (efficient multi-row)

```javascript
// Upsert many counters in one round-trip. Pre-aggregate in JS to avoid
// the cardinality violation (§6.5), then pass parallel arrays via UNNEST.
async function bulkUpsertCounters(entries /* [{bucket, metric, delta}] */) {
  // De-duplicate in the app: sum deltas per (bucket, metric).
  const merged = new Map();
  for (const e of entries) {
    const k = `${e.bucket}|${e.metric}`;
    merged.set(k, (merged.get(k) ?? 0) + e.delta);
  }
  const buckets = [], metrics = [], deltas = [];
  for (const [k, delta] of merged) {
    const [bucket, metric] = k.split('|');
    buckets.push(bucket); metrics.push(metric); deltas.push(delta);
  }

  await pool.query(
    `INSERT INTO metric_counters (bucket, metric, count, updated_at)
     SELECT b, m, d, NOW()
     FROM UNNEST($1::timestamptz[], $2::text[], $3::bigint[]) AS t(b, m, d)
     ON CONFLICT (bucket, metric) DO UPDATE
       SET count = metric_counters.count + EXCLUDED.count,
           updated_at = NOW()`,
    [buckets, metrics, deltas]
  );
}
```

`UNNEST` with array parameters is the standard high-performance bulk-upsert pattern in `pg` — one statement, one parse, one plan, N rows, and it sidesteps building a giant parameter list.

**ORM note:** most ORMs expose upsert helpers, but they cover only the common `DO NOTHING` / simple `DO UPDATE` cases. Accumulation (`count = count + EXCLUDED.count`), conditional `DO UPDATE WHERE`, `MERGE`, and `UNNEST`-based bulk upserts almost always require the raw-SQL escape hatch — see §12.

---

## 12. ORM Comparison

### Prisma

**Can Prisma upsert?** Yes — `upsert()` for single rows, and `createMany`/`create...skipDuplicates` for `DO NOTHING`.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// Single-row upsert — maps to INSERT ... ON CONFLICT DO UPDATE
const user = await prisma.user.upsert({
  where:  { email: 'a@x.com' },       // must be a unique field → conflict target
  create: { email: 'a@x.com', name: 'Alice' },
  update: { name: 'Alice' },
});

// Bulk "DO NOTHING"
await prisma.processedEvent.createMany({
  data: events,
  skipDuplicates: true,               // → ON CONFLICT DO NOTHING
});

// Accumulation needs raw SQL — Prisma cannot express count = count + x
await prisma.$executeRaw`
  INSERT INTO metric_counters (bucket, metric, count)
  VALUES (${bucket}, ${metric}, ${delta})
  ON CONFLICT (bucket, metric)
  DO UPDATE SET count = metric_counters.count + EXCLUDED.count`;
```

**Where it breaks:** `upsert`'s `where` must be a unique field; you cannot target a partial or expression index. No `DO UPDATE WHERE` (conditional update). No accumulation, no `MERGE`, no bulk-`DO UPDATE`. Historically Prisma's `upsert` could itself race (it once did a find-then-write); modern versions emit `ON CONFLICT`, but verify on your version.

**Verdict:** great for simple upserts; drop to `$executeRaw`/`$queryRaw` for counters, conditional updates, or bulk accumulate.

---

### Drizzle ORM

**Can Drizzle upsert?** Yes — `.onConflictDoNothing()` and `.onConflictDoUpdate()`, the closest-to-SQL mapping of any ORM.

```typescript
import { db } from './db';
import { metricCounters, users } from './schema';
import { sql } from 'drizzle-orm';

// DO NOTHING
await db.insert(users)
  .values({ email: 'a@x.com', name: 'Alice' })
  .onConflictDoNothing({ target: users.email });

// DO UPDATE with accumulation + conditional WHERE
await db.insert(metricCounters)
  .values({ bucket, metric, count: delta })
  .onConflictDoUpdate({
    target: [metricCounters.bucket, metricCounters.metric],
    set: { count: sql`${metricCounters.count} + excluded.count` },
    setWhere: sql`${metricCounters.count} + excluded.count >= 0`,
  });

// Bulk upsert: pass an array of values; Drizzle emits one multi-row INSERT
await db.insert(metricCounters)
  .values(rows)  // pre-dedupe in JS to avoid cardinality violation
  .onConflictDoUpdate({
    target: [metricCounters.bucket, metricCounters.metric],
    set: { count: sql`${metricCounters.count} + excluded.count` },
  });
```

**Where it breaks:** you still hand-write the `sql\`... excluded ...\`` fragment for accumulation, and you must pre-deduplicate bulk arrays yourself. No first-class `MERGE`.

**Verdict:** best-in-class. Exposes conflict target, `EXCLUDED`, and the `DO UPDATE WHERE` (`setWhere`) directly. Use confidently.

---

### Sequelize

**Can Sequelize upsert?** Yes — `Model.upsert()` and `bulkCreate(..., { updateOnDuplicate })`.

```javascript
const { User } = require('./models');

// Single upsert; returns [instance, created]
const [user, created] = await User.upsert(
  { email: 'a@x.com', name: 'Alice' },
  { conflictFields: ['email'] }        // Postgres: sets the conflict target
);

// Bulk upsert
await MetricCounter.bulkCreate(rows, {
  updateOnDuplicate: ['count', 'updatedAt'],   // → DO UPDATE SET these
  conflictAttributes: ['bucket', 'metric'],
});
```

**Where it breaks:** `updateOnDuplicate` can only set columns to `EXCLUDED` values — it **cannot** express `count = count + EXCLUDED.count` (accumulation) or any `DO UPDATE WHERE`. The `created` flag from `upsert()` is unreliable across dialects. For accumulation or conditional upserts, use `sequelize.query()` with raw SQL.

**Verdict:** fine for last-write-wins upserts; raw SQL for anything conditional or accumulative.

---

### TypeORM

**Can TypeORM upsert?** Yes — `repository.upsert()` and `QueryBuilder.orUpdate()` / `orIgnore()`.

```typescript
import { AppDataSource } from './data-source';
import { User } from './entities/User';

// Simple upsert
await AppDataSource.getRepository(User).upsert(
  { email: 'a@x.com', name: 'Alice' },
  { conflictPaths: ['email'] }         // conflict target
);

// QueryBuilder for accumulation via raw overwrite expression
await AppDataSource.createQueryBuilder()
  .insert()
  .into('metric_counters')
  .values({ bucket, metric, count: delta })
  .orUpdate(
    ['count'],
    ['bucket', 'metric'],
    { overwriteCondition: { /* limited */ } }
  )
  .setParameter('d', delta)
  // For real accumulation you often must inject raw SQL:
  .onConflict(`("bucket","metric") DO UPDATE SET count = metric_counters.count + EXCLUDED.count`)
  .execute();
```

**Where it breaks:** `upsert()` overwrites with incoming values only; accumulation and `DO UPDATE WHERE` require the lower-level `.onConflict()` raw string (deprecated in newer versions) or `.orUpdate()` with hand-written expressions. No `MERGE`.

**Verdict:** workable for standard upserts; conditional/accumulative logic drops to raw conflict strings or `dataSource.query()`.

---

### Knex.js

**Can Knex upsert?** Yes — `.onConflict().ignore()` and `.onConflict().merge()`.

```javascript
const knex = require('knex')({ client: 'pg', connection: process.env.DATABASE_URL });

// DO NOTHING
await knex('users')
  .insert({ email: 'a@x.com', name: 'Alice' })
  .onConflict('email')
  .ignore();

// DO UPDATE (last write wins) — merge() with no args updates all inserted cols
await knex('users')
  .insert({ email: 'a@x.com', name: 'Alice' })
  .onConflict('email')
  .merge(['name']);

// Accumulation + conditional: use knex.raw in merge
await knex('metric_counters')
  .insert({ bucket, metric, count: delta })
  .onConflict(['bucket', 'metric'])
  .merge({ count: knex.raw('metric_counters.count + EXCLUDED.count') })
  .whereRaw('metric_counters.count + EXCLUDED.count >= 0');  // DO UPDATE WHERE
```

**Where it breaks:** the conflict target for partial/expression indexes needs `knex.raw`. `merge()` semantics differ subtly across dialects, but on Postgres it maps cleanly to `DO UPDATE`.

**Verdict:** most SQL-transparent after Drizzle. `merge()` + `knex.raw` + `whereRaw` covers accumulation and conditional updates without leaving the builder.

---

### ORM Summary Table

| ORM | Upsert method | DO NOTHING | Accumulate (`c=c+x`) | DO UPDATE WHERE | Verdict |
|-----|--------------|-----------|----------------------|-----------------|---------|
| Prisma | `upsert()`, `createMany skipDuplicates` | Yes | Raw SQL | Raw SQL | Simple cases only |
| Drizzle | `.onConflictDoUpdate()` | Yes | `sql\`excluded\`` | `setWhere` | Best support |
| Sequelize | `upsert()`, `updateOnDuplicate` | Yes | Raw SQL | Raw SQL | Last-write-wins only |
| TypeORM | `upsert()`, `.orUpdate()` | `.orIgnore()` | `.onConflict()` raw | raw string | Workable |
| Knex | `.onConflict().merge()/.ignore()` | Yes | `knex.raw` | `whereRaw` | Transparent |

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given `sessions(id, user_id, token, last_seen_at, created_at)` with a unique index on `token`.

Write an upsert that records a session: insert a new session row for the token, or — if the token already exists — just bump `last_seen_at` to `NOW()` (do not change `created_at`). Return the session `id`.

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines Topic 20 GROUP BY and Topic 58 Set Operations)

Given a staging table `staging_sales(product_id, sold_on, units)` that may contain the same `(product_id, sold_on)` multiple times, and a target `daily_sales(product_id, sold_on, units)` with a unique index on `(product_id, sold_on)`.

Write a single bulk upsert that folds the staging batch into `daily_sales`, **adding** units to any existing day's total. Make sure it does not raise a cardinality violation when the staging batch repeats a key.

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

You maintain `device_state(device_id, firmware_version, state, event_ts)` with a unique index on `device_id`. Events arrive out of order over an unreliable network. A naive `ON CONFLICT (device_id) DO UPDATE SET state = EXCLUDED.state, event_ts = EXCLUDED.event_ts` corrupts state because a late-arriving *old* event overwrites a newer one.

Write the upsert that:
1. Inserts the device row if unseen.
2. On conflict, updates state/firmware/event_ts **only if** the incoming `event_ts` is strictly newer than the stored one.
3. Avoids writing a new tuple version when the incoming event is stale (no bloat).
4. Returns the `device_id` and a boolean `applied` indicating whether the update took effect.

Then explain: what does `RETURNING` yield when a stale event arrives, and how should the Node.js caller interpret it?

```sql
-- Write your query here
```

### Exercise 4 — Interview Simulation

On a video call, you're asked: "We have a `wallet_balances(user_id, balance)` table. Thousands of concurrent requests each add or subtract from a user's balance. Some hot users get 5,000 concurrent updates/second. Write the upsert, then tell me why it will fall over at scale and how you'd fix it — at the schema level."

```sql
-- Write your upsert, then describe the scaling problem and your fix.
```

---

## 14. Interview Questions

### Junior Level

**Q: What is an upsert and how do you write one in PostgreSQL?**

*Junior answer:* "It's insert-or-update. You use `INSERT ... ON CONFLICT (column) DO UPDATE SET ...`. If the row exists it updates, otherwise it inserts."

*Principal answer:* "An upsert is a single atomic statement that inserts a row, or — if the insert would violate a specified unique constraint — either skips (`DO NOTHING`) or updates the conflicting row (`DO UPDATE`). The critical detail is the **conflict target**: `ON CONFLICT (col)` must resolve to a unique or exclusion index that acts as the *arbiter*; without one, PostgreSQL errors at runtime. Inside `DO UPDATE`, the incoming row is exposed as the `EXCLUDED` pseudo-table and the existing row via the table name, so you can compute the new value from both. It's atomic and concurrency-safe because it's implemented via speculative insertion against the unique index, not a separate check."

*Interviewer follow-up:* "What happens if there's no unique index on the conflict target column?"
→ Runtime error: `there is no unique or exclusion constraint matching the ON CONFLICT specification`.

---

**Q: What's the difference between `DO NOTHING` and `DO UPDATE`?**

*Junior answer:* "`DO NOTHING` skips it, `DO UPDATE` changes the row."

*Principal answer:* "`DO NOTHING` silently skips the conflicting row — no error, no write, and `RETURNING` yields nothing for it, which you can use to detect 'already existed.' It also doesn't take a row lock. `DO UPDATE` updates the existing row, requires an explicit conflict target (it must know *which* row), takes a row-level lock on the conflicting row for the transaction, and creates a new MVCC tuple version — so it produces dead tuples and WAL. Also, bare `ON CONFLICT DO NOTHING` with no target catches *any* unique-constraint violation, whereas `DO UPDATE` always needs a named target."

*Interviewer follow-up:* "You need `RETURNING id` to always give you the id even when the row already exists. `DO NOTHING` returns nothing on conflict — how do you fix it?"
→ Use `DO UPDATE SET col = EXCLUDED.col` (a self-assignment that always fires and returns the row), or wrap in a CTE with a fallback `SELECT`.

---

### Principal Level

**Q: Your team uses `SELECT ... then INSERT or UPDATE` in the application. It works in dev but throws duplicate-key errors in production under load. Diagnose and fix.**

*Principal answer:* "That's a check-then-act race. The `SELECT` reads an MVCC snapshot; between reading 'not found' and executing the `INSERT`, a concurrent transaction commits the same key. No lock is held across the gap, so both transactions try to insert and one hits the unique violation. In dev, concurrency is low enough that the window is rarely hit; in production it's hit constantly. The fix is to collapse it into one atomic `INSERT ... ON CONFLICT ... DO UPDATE`, which uses speculative insertion: it inserts a speculative token, detects the concurrent inserter via the unique index, waits for it to commit, and then takes the update branch. The race window is eliminated at the storage layer. If they can't change the query shape, the alternative is `SERIALIZABLE` isolation with retry-on-serialization-failure — but `ON CONFLICT` is simpler and doesn't need retries for this case."

*Interviewer follow-up:* "Does `ON CONFLICT` need retry logic like `MERGE` might?"
→ No. `ON CONFLICT` is race-proof by design. `MERGE` (PG15+) evaluates its `ON` against a snapshot and *can* raise a unique violation or create duplicates under concurrency, so `MERGE`-based upserts may need retries or `SERIALIZABLE`.

---

**Q: A counter table using upserts has become slow and bloated. Walk me through why and how you'd fix it.**

*Principal answer:* "Every `DO UPDATE` is an MVCC update: it marks the old tuple dead and writes a new version, plus updates every index. A high-rate counter generates dead tuples faster than autovacuum reclaims them, so the table and indexes bloat; bloated indexes mean more pages per probe, which slows the upserts, which is a feedback loop. Fixes, in order: (1) guard no-op writes with `DO UPDATE ... WHERE col IS DISTINCT FROM EXCLUDED.col` so unchanged rows don't churn; (2) tune per-table autovacuum aggressively — lower `autovacuum_vacuum_scale_factor`, zero the cost delay; (3) set `fillfactor` to ~80 so HOT updates reuse space on-page and skip index maintenance when no indexed column changed; (4) if it's a single hot key with massive concurrency, the real problem is row-lock serialization, so shard the counter into N rows and sum at read time."

*Interviewer follow-up:* "Why `IS DISTINCT FROM` and not `<>` in the guard?"
→ `<>` returns UNKNOWN when either side is NULL, so a NULL-involving change would be wrongly skipped. `IS DISTINCT FROM` treats NULLs as comparable, giving correct true/false.

---

**Q: When would you choose `MERGE` over `INSERT ... ON CONFLICT`, and what's the concurrency trade-off?**

*Principal answer:* "`MERGE` (PG15+) is more general: it joins the target to an arbitrary source on any condition — not just a unique index — and supports multiple `WHEN MATCHED`/`WHEN NOT MATCHED` branches including `DELETE`. So I'd use it when I need conditional deletes, several conditional update branches, or matching on a non-unique join condition. The trade-off is concurrency: `ON CONFLICT` is race-proof via speculative insertion against the unique index. `MERGE` evaluates its `ON` condition against a snapshot; two concurrent `MERGE`s can both decide 'not matched' and both insert — hitting a unique violation if a constraint exists, or silently duplicating if it doesn't. So `MERGE` may need retry logic or `SERIALIZABLE` isolation for concurrent same-key workloads. For a plain idempotent single-key upsert under concurrency, `ON CONFLICT` is the correct, simpler tool."

*Interviewer follow-up:* "Can `ON CONFLICT` delete a row like `MERGE`'s `WHEN MATCHED THEN DELETE`?"
→ No. `ON CONFLICT` only inserts, skips, or updates. Deleting-on-match requires `MERGE` or a separate statement.

---

## 15. Mental Model Checkpoint

1. You run `INSERT ... ON CONFLICT (email) DO UPDATE ...` but `email` has only a non-unique index. What happens, and at what point (parse, plan, execute) does it fail?

2. Inside `DO UPDATE SET total = ??? `, you want the running sum of existing plus incoming. Which side reads the stored value and which reads the new value? What does `SET total = EXCLUDED.total + EXCLUDED.total` actually compute?

3. A bulk `INSERT ... ON CONFLICT DO UPDATE` fails with "cannot affect row a second time." What's in your source data, and what's the one-line fix?

4. A `DO UPDATE ... WHERE incoming_ts > stored_ts` fails the `WHERE` for a stale event. Is the row inserted? Updated? Untouched? What does `RETURNING` give the caller?

5. Your `SERIAL` primary key has thousands of gaps on an upsert-heavy table. Is this a bug? Why or why not?

6. Two transactions concurrently run the *same* `INSERT ... ON CONFLICT (id) DO UPDATE` for the same id. Neither errors and no duplicate appears. What internal mechanism made that safe, and where would the equivalent `MERGE` differ?

7. You need `RETURNING id` to always return the id, even when the row already existed. Why does `DO NOTHING` fail this requirement, and what's the idiomatic workaround?

---

## 16. Quick Reference Card

```sql
-- ┌─────────────────────────── THE THREE FORMS ───────────────────────────┐
-- Idempotent insert (skip dupes)
INSERT INTO t (k, v) VALUES ($1, $2) ON CONFLICT (k) DO NOTHING;

-- Insert-or-replace (last write wins)
INSERT INTO t (k, v) VALUES ($1, $2)
ON CONFLICT (k) DO UPDATE SET v = EXCLUDED.v;

-- Insert-or-accumulate (existing + incoming)
INSERT INTO t (k, n) VALUES ($1, $2)
ON CONFLICT (k) DO UPDATE SET n = t.n + EXCLUDED.n;
-- └───────────────────────────────────────────────────────────────────────┘

-- CONFLICT TARGET must match a UNIQUE/exclusion index (the arbiter):
ON CONFLICT (col)                       -- column list
ON CONFLICT (lower(email))              -- expression index (repeat the expr)
ON CONFLICT (a, b)                      -- composite
ON CONFLICT (col) WHERE active          -- partial index (repeat the predicate)
ON CONFLICT ON CONSTRAINT my_uk         -- by constraint name
ON CONFLICT DO NOTHING                  -- NO target = catch ANY unique violation

-- EXCLUDED = the row you TRIED to insert;  table.col = the existing row
SET v = EXCLUDED.v                      -- overwrite
SET n = t.n + EXCLUDED.n                -- accumulate
SET ts = GREATEST(t.ts, EXCLUDED.ts)    -- high-water mark
SET meta = t.meta || EXCLUDED.meta      -- JSON merge
SET phone = COALESCE(t.phone, EXCLUDED.phone)  -- fill nulls only

-- DO UPDATE WHERE: gate the update (never falls back to insert)
... DO UPDATE SET v = EXCLUDED.v
    WHERE t.v IS DISTINCT FROM EXCLUDED.v      -- skip no-op writes (anti-bloat)
    -- or WHERE EXCLUDED.ts > t.ts             -- reject stale data

-- Detect insert vs update:
RETURNING id, (xmax = 0) AS was_inserted

-- Bulk: PRE-AGGREGATE to avoid "cannot affect row a second time"
INSERT INTO t (k, n)
SELECT k, SUM(n) FROM src GROUP BY k
ON CONFLICT (k) DO UPDATE SET n = t.n + EXCLUDED.n;

-- MERGE (PG15+): more power, NOT race-proof
MERGE INTO t USING src ON t.k = src.k
WHEN MATCHED AND ... THEN DELETE
WHEN MATCHED THEN UPDATE SET ...
WHEN NOT MATCHED THEN INSERT (...) VALUES (...);
```

**Performance rules of thumb**
- Single-row upsert stays sub-millisecond at 100M rows *if the unique index's top levels are cached*.
- Every `DO UPDATE` = one dead tuple → tune autovacuum, use `fillfactor 80`, guard no-op writes.
- Hot single key + high concurrency = row-lock serialization → shard the key.
- Sequences advance on conflicting rows too → expect gaps; use `bigint` ids.
- `ON CONFLICT` is race-proof; `MERGE` may need retries/`SERIALIZABLE`.

**Interview one-liners**
- "Upsert = INSERT with a conflict branch wired to a unique index arbiter."
- "The conflict target selects the arbiter index; no matching unique index = runtime error."
- "`EXCLUDED` is the row you tried to insert; the table name is the row that's already there."
- "`DO UPDATE WHERE` gates the update — a false predicate does nothing, it never re-inserts."
- "Can't touch the same key twice in one statement — pre-aggregate the source."
- "`ON CONFLICT` uses speculative insertion and is race-proof; `MERGE` isn't."

---

## Connected Topics

- **Topic 58 — Set Operations** (previous): `UNION`/`INTERSECT`/`EXCEPT` combine result sets; upserts combine *source rows with existing table state*. Pre-aggregating a bulk-upsert source often uses set-style `GROUP BY`/`UNION ALL` of staging batches.
- **Topic 60 — Bulk Operations** (next): high-volume `COPY`, multi-row `INSERT`, and `UNNEST`-based ingestion — where bulk upserts live and where cardinality-violation de-duplication becomes critical.
- **Topic 07 — NULL in Depth**: why `IS DISTINCT FROM` (not `<>`) is the correct no-op guard in `DO UPDATE WHERE`; three-valued logic in conflict predicates.
- **Topic 40 — MVCC Internals**: why every `DO UPDATE` writes a new tuple version and leaves a dead one — the source of upsert-driven bloat.
- **Topic 41 — WAL and Durability**: upserts are WAL-heavy; the update branch emits update WAL records that drive replication lag.
- **Topic 42 — Indexes and B-trees**: the unique index is the arbiter; the B-tree descent *is* the conflict check.
- **Topic 20 — GROUP BY Fundamentals**: pre-aggregating a bulk-upsert source to avoid the "cannot affect row a second time" error.
- **Topic 44 — Transaction Isolation**: why `MERGE` may need `SERIALIZABLE` + retry while `ON CONFLICT` does not; row-lock serialization on hot keys.
- **Topic 11 — INNER JOIN**: `MERGE`'s `USING ... ON` is a join between target and source — the same matching semantics, applied to a write.
