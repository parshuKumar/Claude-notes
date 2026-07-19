# Topic 54 — SELECT FOR UPDATE / FOR SHARE
### SQL Mastery Curriculum — Phase 8: Transactions and Concurrency in SQL

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine a shared library with exactly **one physical copy** of a rare book. Two students, Alice and Bob, both want to write notes in the margins of that book. If they both grab it, scribble their notes, and put it back, one student's notes get erased by the other — whoever finishes last wins. Neither of them knew the other was writing.

Now the library adds a rule. Before you write in the book, you go to the front desk and say: **"I'm going to modify this book — hold it for me."** The librarian puts a physical claim ticket on it. While Alice holds the claim ticket, Bob can still walk up and *read* the book over her shoulder — but the moment Bob says "I also want to modify it," the librarian tells him: **"Wait. Alice has the modify-claim. Stand here until she's done."** Bob physically cannot proceed. He blocks at the desk. When Alice returns the book (commits), the librarian hands the modify-claim to Bob, who now sees Alice's finished notes and writes his own on top of them — safely, in order.

That claim ticket is **`SELECT ... FOR UPDATE`**. It's a *pessimistic lock*: you assume a conflict *will* happen, so you lock the row up front, before anyone else can touch it, and you hold that lock until your transaction ends.

There's a softer version too. Sometimes you don't want to *write*, you just want to guarantee that **nobody changes the book while you're reading it** — you need the price on page 5 to stay exactly what it is until you finish your calculation. You tell the librarian: **"Let others read too, but don't let anyone modify it until I'm done."** That's **`SELECT ... FOR SHARE`** — a shared read-lock. Many readers can hold it at once, but any would-be writer must wait for all of them to leave.

The whole topic is about these claim tickets: which ones exist, who blocks whom, how to skip a locked book and grab the next available one (SKIP LOCKED), and how to say "if it's locked, don't make me wait — just tell me immediately" (NOWAIT).

---

## 2. Connection to SQL Internals

`SELECT ... FOR UPDATE` and `FOR SHARE` are **row-level locks** implemented in PostgreSQL's storage and MVCC layer. Understanding them requires naming the exact machinery:

1. **MVCC and tuple headers.** Remember from Topic 52 that every row version (tuple) carries hidden system columns: `xmin` (the transaction that created it), `xmax` (the transaction that deleted/locked it), and a set of infomask bits. A row lock is *not* stored in a separate lock table for the common case — it is written **directly into the tuple's `xmax` field** and infomask bits. When you `SELECT ... FOR UPDATE` a row, PostgreSQL stamps your transaction ID (or a MultiXact ID) into that tuple's `xmax` and sets infomask flags like `HEAP_XMAX_EXCL_LOCK` or `HEAP_XMAX_KEYSHR_LOCK`.

2. **This means row locks touch the heap.** Locking a row *dirties the heap page* and generates a WAL record (a lightweight one). This is why `SELECT FOR UPDATE` is not free — it writes. On a standby/replica, this is also why you cannot take these locks (read-only transactions on a hot standby reject `FOR UPDATE`).

3. **MultiXact IDs.** A single `xmax` field can only hold one transaction ID. But multiple transactions can hold a `FOR SHARE` lock on the same row simultaneously. PostgreSQL solves this with **MultiXact** — a separate structure (`pg_multixact`) that represents a *set* of transactions plus their lock modes. When two sessions `FOR SHARE` the same row, `xmax` holds a MultiXactId pointing to the group. MultiXact has its own SLRU cache and can be a source of contention and wraparound at extreme scale.

4. **The heavyweight lock manager (`pg_locks`).** Beyond the tuple, PostgreSQL also uses the in-memory lock manager for a *tuple lock* (a transient lock on the physical tuple to serialize who gets to write `xmax` next) and for the transaction-ID locks that make waiters block. When session B waits for session A's row lock, B is actually waiting on a lock whose holder is A's transaction ID (`transactionid` lock in `pg_locks`). This is the mechanism that implements "block until the other transaction ends."

5. **The four row-lock strengths.** Internally PostgreSQL defines a lattice of four row-level lock modes with a conflict table: `FOR KEY SHARE` (weakest) < `FOR SHARE` < `FOR NO KEY UPDATE` < `FOR UPDATE` (strongest). The distinction between "key" and "non-key" exists specifically to make **foreign key checks** cheaper — `FOR KEY SHARE` locks a parent row against key changes/deletion without blocking ordinary non-key updates to it.

6. **The planner and executor.** Locking clauses attach to the plan as a `LockRows` node sitting above the scan. The rows flow up from the scan, hit `LockRows`, which acquires the lock on each tuple *as it is returned*. If a row was updated by a concurrent committed transaction after your snapshot (an MVCC conflict under READ COMMITTED), the executor performs an **EPQ (EvalPlanQual)** re-check: it fetches the latest row version, re-applies the WHERE clause, and either re-locks the new version or drops the row.

So `FOR UPDATE` sits at the intersection of MVCC, the heap, WAL, MultiXact, the lock manager, and the executor's EPQ machinery. It is one of the most internally interconnected features in the engine.

---

## 3. Logical Execution Order Context

```
FROM / JOIN            ← rows sourced
WHERE                  ← rows filtered
GROUP BY / HAVING      ← (NOT allowed with locking clauses)
SELECT (projection)
DISTINCT               ← (NOT allowed with locking clauses)
ORDER BY
LIMIT
FOR UPDATE / FOR SHARE ← locking applied LAST, to the final surviving rows
```

The locking clause is the **final step** of statement processing, and this ordering has sharp consequences:

- **Locks are acquired only on rows that survive `WHERE`.** Rows filtered out are never locked. This is what makes `SELECT ... WHERE status='queued' ... FOR UPDATE` a targeted lock rather than a table lock.

- **`FOR UPDATE` runs conceptually after `ORDER BY` and `LIMIT` in terms of *which rows* are locked** — but there is a critical subtlety. With `LIMIT n`, PostgreSQL locks the rows it returns. Combined with `SKIP LOCKED`, the executor locks-and-skips as it goes, so the *effective* interaction of LIMIT + SKIP LOCKED is "return the first `n` rows that I can successfully lock." This is the beating heart of the job-queue pattern in Section 6.

- **You cannot use `FOR UPDATE` with aggregation.** `GROUP BY`, `HAVING`, `DISTINCT`, `UNION`, window functions, and set-returning aggregates make the output rows not correspond one-to-one with base-table rows — so there is nothing concrete to lock. PostgreSQL raises an error: `FOR UPDATE is not allowed with GROUP BY clause`.

- **In a JOIN, you choose *which tables* to lock.** `FOR UPDATE OF table_alias` restricts the lock to specific tables in the FROM list. Without `OF`, every table contributing a row to the result is locked.

The mental rule: **filter first, lock last, hold until commit.** The lock's lifetime is the transaction, not the statement — this is the single biggest difference from a plain SELECT.

---

## 4. What Is SELECT FOR UPDATE / FOR SHARE?

A **locking clause** (`FOR UPDATE`, `FOR NO KEY UPDATE`, `FOR SHARE`, `FOR KEY SHARE`) appended to a `SELECT` acquires a **row-level lock** on every selected row and holds it until the current transaction commits or rolls back. It is PostgreSQL's mechanism for **pessimistic concurrency control**: reserve the rows now so that concurrent transactions cannot modify (or, for the stronger modes, cannot even read-lock) them until you are done.

```sql
SELECT column_list
FROM   table_a a
JOIN   table_b b ON b.a_id = a.id
WHERE  a.status = 'active'
ORDER BY a.priority DESC
LIMIT  10
FOR UPDATE                    -- lock strength
  OF a                        -- restrict lock to table_a only
  SKIP LOCKED;                -- non-blocking: ignore already-locked rows
```

### Annotated syntax breakdown

```sql
FOR UPDATE
│   └── Lock strength keyword. One of:
│         FOR UPDATE          → strongest. Blocks all other locks + writes on the row.
│                               Used when you WILL update or delete the row.
│         FOR NO KEY UPDATE   → like FOR UPDATE but does NOT block FOR KEY SHARE.
│                               Taken automatically by UPDATEs that don't touch key columns.
│         FOR SHARE           → shared read-lock. Multiple sessions can hold it.
│                               Blocks writers (FOR UPDATE/NO KEY UPDATE) but not other readers.
│         FOR KEY SHARE       → weakest. Only blocks key changes / deletes.
│                               Taken automatically by foreign-key checks on the parent row.
│
  OF a
│  └── Optional. Names which table(s) from the FROM clause to lock.
│      Without OF, ALL tables producing an output row are locked.
│      With OF a, only rows of table_a are locked; table_b rows are read normally.
│
  SKIP LOCKED
   └── Optional wait-policy modifier (mutually exclusive with NOWAIT):
         (default)     → WAIT: block until the conflicting lock is released.
         NOWAIT        → don't wait; if any target row is locked, raise error 55P03 immediately.
         SKIP LOCKED   → don't wait; silently omit any row currently locked by another txn.
```

### The four lock modes and their conflict table

This is the single most important reference in the topic. A row-level lock in mode X held by transaction A **blocks** transaction B from acquiring mode Y iff the cell is ✗:

| A holds ↓ / B wants → | KEY SHARE | SHARE | NO KEY UPDATE | UPDATE |
|-----------------------|:---------:|:-----:|:-------------:|:------:|
| **FOR KEY SHARE**     |     ✓     |   ✓   |       ✓       |   ✗    |
| **FOR SHARE**         |     ✓     |   ✓   |       ✗       |   ✗    |
| **FOR NO KEY UPDATE** |     ✓     |   ✗   |       ✗       |   ✗    |
| **FOR UPDATE**        |     ✗     |   ✗   |       ✗       |   ✗    |

Read it as: `FOR KEY SHARE` conflicts only with `FOR UPDATE`. `FOR UPDATE` conflicts with everything. The two "share" modes are compatible with each other (many readers), the two "update" modes conflict with each other and with everything below them on their side. A plain `UPDATE`/`DELETE` on key columns takes `FOR UPDATE`; an `UPDATE` on non-key columns takes `FOR NO KEY UPDATE`; a foreign-key validation takes `FOR KEY SHARE` on the parent.

---

## 5. Why SELECT FOR UPDATE Mastery Matters in Production

1. **It is the only correct fix for read-modify-write races.** The classic bug: read a balance, compute `balance - amount` in application code, write it back. Under concurrency, two transactions read the same starting balance and one update is lost. `SELECT ... FOR UPDATE` serializes the two transactions on that row so the second reads the first's committed result. Without it, you silently corrupt money. This is not theoretical — it is the most common data-integrity bug in fintech and inventory systems.

2. **It powers reliable job queues without a message broker.** `SELECT ... FOR UPDATE SKIP LOCKED` lets N worker processes pull disjoint sets of jobs from a single Postgres table with zero coordination and zero double-processing. Many teams run their entire background-job system on this one pattern instead of adopting Redis/RabbitMQ/SQS. Getting it wrong means either workers stampede and block each other, or the same job runs twice.

3. **It determines whether your API blocks or fails fast.** The difference between `FOR UPDATE`, `FOR UPDATE NOWAIT`, and `FOR UPDATE SKIP LOCKED` is the difference between a request that hangs for 30 seconds, one that returns a clean 409 in 2ms, and one that quietly grabs the next available work item. Choosing wrong turns a concurrency spike into a connection-pool exhaustion outage.

4. **It interacts with foreign keys in ways that cause mysterious deadlocks.** Before the `FOR KEY SHARE` / `FOR NO KEY UPDATE` split (added in PostgreSQL 9.3), any update to a child row took a full share-lock on the parent, and concurrent inserts into two children of the same parent could deadlock. Understanding the key/non-key distinction is essential to diagnosing FK-related lock contention — which still surprises engineers who assume "I only touched the child table."

5. **It is the boundary between optimistic and pessimistic strategies.** A principal engineer must know *when* pessimistic locking is right (short transactions, high contention on specific rows, correctness-critical) and when it is wrong (long-held locks, low contention, read-heavy) — where a `version` column with optimistic concurrency (Topic on optimistic locking) or `SERIALIZABLE` isolation (Topic 53) is the better tool. Reaching for `FOR UPDATE` reflexively creates lock convoys and throughput collapse.

---

## 6. Deep Technical Content

### 6.1 The Read-Modify-Write Race and Why FOR UPDATE Fixes It

Consider decrementing inventory. Under `READ COMMITTED` (the default, Topic 53), a plain read does not lock:

```sql
-- Session A                          -- Session B (concurrent)
BEGIN;                                BEGIN;
SELECT stock FROM products            SELECT stock FROM products
  WHERE id = 10;  -- returns 5          WHERE id = 10;  -- ALSO returns 5
-- app computes 5 - 3 = 2             -- app computes 5 - 4 = 1
UPDATE products SET stock = 2         UPDATE products SET stock = 1
  WHERE id = 10;                        WHERE id = 10;  -- blocks until A commits
COMMIT;                               -- then overwrites: stock = 1
                                      COMMIT;
-- Final stock = 1. But 3 + 4 = 7 units sold from 5. LOST UPDATE + oversell.
```

The fix is to acquire the lock at read time so the second reader cannot even read-to-decide until the first commits:

```sql
-- Session A                          -- Session B
BEGIN;                                BEGIN;
SELECT stock FROM products            SELECT stock FROM products
  WHERE id = 10                         WHERE id = 10
  FOR UPDATE;    -- returns 5, LOCKS    FOR UPDATE;   -- BLOCKS here
UPDATE products SET stock = 2         -- (still waiting on A's row lock)
  WHERE id = 10;
COMMIT;                               -- unblocks; SELECT now returns 2 (A's commit)
                                      -- app computes 2 - 4 = -2 → app rejects: insufficient stock
                                      ROLLBACK;
```

Under `READ COMMITTED`, when B's `FOR UPDATE` unblocks, PostgreSQL re-reads the **latest committed version** of the row (EvalPlanQual) and returns `stock = 2`, not the stale `5`. This is the crucial guarantee: a blocked `FOR UPDATE` returns fresh data, not the snapshot value.

### 6.2 FOR UPDATE vs FOR NO KEY UPDATE

Both are "write intent" locks, but they differ in what they block on the *weak* side:

- **`FOR UPDATE`** blocks `FOR KEY SHARE`. That means while you hold `FOR UPDATE` on a parent row, another transaction cannot insert a child row that references it (because the FK check needs `FOR KEY SHARE`). Use it when you intend to **delete** the row or **change a key column** (primary key or a column referenced by a unique index used as an FK target).

- **`FOR NO KEY UPDATE`** does *not* block `FOR KEY SHARE`. So you can hold it on a parent while others still insert children referencing it. This is what a regular `UPDATE` that changes only non-key columns takes automatically. When you explicitly `SELECT ... FOR NO KEY UPDATE`, you signal "I'll update this row but not its identity," allowing more concurrency with FK activity.

```sql
-- You will change the user's email (non-key). Prefer the weaker lock:
SELECT * FROM users WHERE id = $1 FOR NO KEY UPDATE;
-- Allows concurrent INSERT INTO orders(user_id) VALUES ($1) to proceed (FK KEY SHARE).

-- You will DELETE the user or change users.id. You need the strong lock:
SELECT * FROM users WHERE id = $1 FOR UPDATE;
-- Blocks concurrent child inserts referencing this user until you commit.
```

### 6.3 FOR SHARE vs FOR KEY SHARE

- **`FOR SHARE`** is a read-lock that says "this row must not change while I hold this." Multiple sessions can hold `FOR SHARE` simultaneously (compatible with each other), but any writer (`FOR UPDATE` / `FOR NO KEY UPDATE`, or a plain UPDATE/DELETE) blocks until all sharers release. Use it to pin a row you're reading as an invariant for a multi-row calculation without preventing other readers.

- **`FOR KEY SHARE`** is the weakest lock. It only prevents the row's *key* from changing and prevents deletion. It permits concurrent non-key updates. It is exactly what foreign-key enforcement uses on parent rows. You rarely write it explicitly, but understanding it explains FK lock behavior.

```sql
-- Pin exchange rate rows so nobody edits them mid-calculation, but let others read:
BEGIN;
SELECT rate FROM fx_rates WHERE pair IN ('USD/EUR','USD/GBP') FOR SHARE;
-- ... perform multi-currency settlement using guaranteed-stable rates ...
COMMIT;  -- releases the share locks
```

### 6.4 SKIP LOCKED — Building a Job Queue

`SKIP LOCKED` changes the wait policy from "block" to "pretend the locked rows don't exist for this query." Combined with `LIMIT` and `FOR UPDATE`, it yields the canonical concurrent-consumer pattern:

```sql
-- Each worker runs this in its own transaction:
BEGIN;
SELECT id, payload
FROM   jobs
WHERE  status = 'queued'
ORDER BY priority DESC, created_at ASC
FOR UPDATE SKIP LOCKED
LIMIT 1;
-- ... process the job in application code ...
UPDATE jobs SET status = 'done', finished_at = now() WHERE id = $1;
COMMIT;  -- releases the row lock
```

How it works internally: the `LockRows` node tries to lock each candidate row as it flows up. If a row is already locked by another transaction, instead of waiting it **skips** it and asks the scan for the next row. With `LIMIT 1`, worker A locks job #1, worker B (running the identical query concurrently) finds #1 locked, skips it, and locks job #2. No two workers ever get the same job, and none of them block. This scales near-linearly with worker count until you saturate the index scan or the table's hot rows.

Critical details:
- The `ORDER BY` is honored *before* skipping, so you still get highest-priority available jobs — but you do **not** get a strict global priority order across workers (worker B might process #2 before worker A finishes #1). This is usually acceptable for queues.
- `SKIP LOCKED` only skips rows locked by *other* transactions. Rows you've filtered by `WHERE` are gone before locking, so a queue must mark rows `status='processing'` inside the same transaction (or rely on the row lock itself) to prevent a *third* pattern of double-fetch across separate polling loops.
- It generates no error and no wait — the safest wait policy for high-throughput pollers.

### 6.5 NOWAIT — Fail Fast Instead of Blocking

`NOWAIT` makes the statement raise an error immediately (`SQLSTATE 55P03`, `lock_not_available`) if *any* target row is currently locked, rather than waiting:

```sql
BEGIN;
SELECT * FROM seats
WHERE id = $1
FOR UPDATE NOWAIT;   -- if someone else is booking this seat, error out at once
-- ERROR: could not obtain lock on row in relation "seats"
```

Use `NOWAIT` when the correct product behavior is "tell the user immediately that this resource is busy, try again" rather than making them wait behind a potentially long transaction. It converts a latency problem (blocked request) into a clean, retryable error. Node handlers catch `error.code === '55P03'` and return HTTP 409 Conflict.

Difference from `SKIP LOCKED`: `NOWAIT` errors on the *first* locked row and aborts the whole statement; `SKIP LOCKED` silently omits locked rows and returns the rest. `NOWAIT` is for "I want *this specific row* or nothing"; `SKIP LOCKED` is for "give me *any* available rows."

### 6.6 Lock Lifetime, Release, and Savepoints

Row locks are held until **transaction end** — there is no explicit `UNLOCK ROW`. They release on `COMMIT` or `ROLLBACK`. This has design implications:

- **Keep FOR UPDATE transactions short.** A row locked at the start of a 5-second transaction blocks every writer to that row for 5 seconds. Do external I/O (HTTP calls, sending email) *outside* the locked window.
- **Savepoints and partial rollback.** A `ROLLBACK TO SAVEPOINT` releases locks acquired *after* the savepoint. This lets you attempt a `NOWAIT` lock, catch the failure, roll back to a savepoint, and continue the transaction — the pattern behind retryable operations inside one transaction.

```sql
BEGIN;
SAVEPOINT try_lock;
SELECT * FROM accounts WHERE id = 7 FOR UPDATE NOWAIT;
-- if it errors, in app code: ROLLBACK TO try_lock; then try a different account
```

### 6.7 FOR UPDATE with JOINs and OF

By default a locking clause locks **every table that contributed a row**. That is often too much:

```sql
-- Locks BOTH orders AND users rows — probably not intended:
SELECT o.*, u.email
FROM   orders o
JOIN   users u ON u.id = o.user_id
WHERE  o.id = $1
FOR UPDATE;

-- Lock only the order; read the user without locking it:
SELECT o.*, u.email
FROM   orders o
JOIN   users u ON u.id = o.user_id
WHERE  o.id = $1
FOR UPDATE OF o;
```

`OF o` restricts the lock to `orders`. You can even mix strengths in one statement:

```sql
SELECT ...
FROM orders o JOIN users u ON u.id = o.user_id
WHERE o.id = $1
FOR UPDATE OF o          -- strong lock on the order
FOR SHARE  OF u;         -- shared lock pinning the user row
```

### 6.8 FOR UPDATE and Aggregates / DISTINCT / GROUP BY — Disallowed

Locking clauses require a one-to-one mapping between output rows and lockable base-table rows. These constructs break that mapping and are rejected:

```sql
SELECT count(*) FROM orders FOR UPDATE;
-- ERROR: FOR UPDATE is not allowed with aggregate functions

SELECT DISTINCT user_id FROM orders FOR UPDATE;
-- ERROR: FOR UPDATE is not allowed with DISTINCT clause

SELECT user_id, count(*) FROM orders GROUP BY user_id FOR UPDATE;
-- ERROR: FOR UPDATE is not allowed with GROUP BY clause

SELECT * FROM orders UNION SELECT * FROM archived_orders FOR UPDATE;
-- ERROR: FOR UPDATE is not allowed with UNION/INTERSECT/EXCEPT
```

Workaround: lock in a subquery/CTE that returns the base rows, then aggregate outside — or lock the concrete rows first, then run the aggregate in the same transaction.

```sql
-- Lock the concrete rows, aggregate them separately in the same txn:
WITH locked AS (
  SELECT id, amount FROM invoices
  WHERE customer_id = $1 AND status = 'open'
  FOR UPDATE
)
SELECT sum(amount) FROM locked;
-- The FOR UPDATE is inside the CTE where rows map 1:1; aggregation is outside.
```

### 6.9 FOR UPDATE Under Different Isolation Levels

- **READ COMMITTED (default):** A blocked `FOR UPDATE` uses EvalPlanQual to re-read the latest committed row version when it unblocks, then re-checks the WHERE clause on that new version. If the new version no longer matches WHERE, the row is dropped from the result. This can cause the surprising "the row I locked disappeared" behavior.

- **REPEATABLE READ / SERIALIZABLE (Topic 53):** These use a snapshot fixed at transaction start. If a `FOR UPDATE` targets a row that another transaction has *already updated and committed* since your snapshot, PostgreSQL cannot silently re-read (that would violate snapshot isolation). Instead it aborts your transaction with a **serialization failure** (`SQLSTATE 40001`, `could not serialize access due to concurrent update`). Your application must catch `40001` and **retry the whole transaction**. This is the single most important operational fact about `FOR UPDATE` under higher isolation.

```sql
-- Under REPEATABLE READ, this may raise:
-- ERROR: could not serialize access due to concurrent update
-- The app must retry the transaction from BEGIN.
```

### 6.10 Foreign Keys and the Lock Split (Historical + Practical)

Before PostgreSQL 9.3, updating *any* column of a child row took a `SELECT FOR SHARE` on the referenced parent. Two transactions each inserting a child of the same parent would both try to share-lock the parent, and if either later needed to strengthen, they deadlocked. This caused frequent, mysterious deadlocks in high-write systems with shared parent rows (e.g., many orders referencing one popular product).

The 9.3 fix introduced `FOR KEY SHARE` and `FOR NO KEY UPDATE`:
- FK checks now take only `FOR KEY SHARE` on the parent (weakest).
- Non-key updates take only `FOR NO KEY UPDATE`.
- `FOR KEY SHARE` and `FOR NO KEY UPDATE` **do not conflict** (see the table in 6.4 — wait, the conflict table in Section 4).

Result: concurrent child inserts referencing the same parent no longer block each other, and a non-key update of the parent no longer blocks child inserts. You only get blocking when someone genuinely tries to change the parent's key or delete it (which would orphan children). Knowing this lets you diagnose "why is my child insert waiting on a parent row lock?" — the answer is almost always that someone is holding `FOR UPDATE` (or updating a key column / deleting) on that parent.

### 6.11 MultiXact — When Many Share One Row

When two or more transactions hold locks on the *same* row and at least one is a share-type lock (or a lock plus an update), PostgreSQL packs them into a **MultiXact ID** stored in the tuple's `xmax`. MultiXacts are stored in the `pg_multixact` SLRU. Under extreme contention on hot rows with many concurrent `FOR SHARE`/`FOR KEY SHARE` lockers, MultiXact allocation and its own SLRU cache can become a bottleneck, and in pathological cases MultiXact ID wraparound forces aggressive autovacuum. Symptoms: high `multixact` waits in `pg_stat_activity.wait_event`, or wraparound-prevention autovacuum warnings. The practical takeaway: heavy `FOR SHARE`/`FOR KEY SHARE` on a few very hot rows is not free even though the locks are "compatible."

### 6.12 Deadlocks (Preview of Topic 55)

`FOR UPDATE` on multiple rows in inconsistent orders across transactions causes deadlocks: A locks row 1 then waits for row 2; B locks row 2 then waits for row 1. PostgreSQL's deadlock detector (default `deadlock_timeout = 1s`) breaks the cycle by aborting one transaction with `SQLSTATE 40P01`. The prevention rule: **always lock rows in a consistent, deterministic order** (e.g., `ORDER BY id` before locking, or always lock the lower account id first in a transfer). Topic 55 covers this in full.

---

## 7. EXPLAIN — LockRows in the Plan

The locking clause appears as a `LockRows` node above the scan. Consider the queue-poll query:

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, payload
FROM   jobs
WHERE  status = 'queued'
ORDER BY priority DESC, created_at ASC
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

```
Limit  (cost=0.43..2.65 rows=1 width=48)
       (actual time=0.052..0.053 rows=1 loops=1)
  Buffers: shared hit=5
  ->  LockRows  (cost=0.43..889.10 rows=400 width=48)
                (actual time=0.051..0.051 rows=1 loops=1)
        Buffers: shared hit=5
        ->  Index Scan using jobs_status_priority_created_idx on jobs
                (cost=0.43..885.10 rows=400 width=48)
                (actual time=0.030..0.031 rows=1 loops=1)
              Index Cond: (status = 'queued'::text)
              Buffers: shared hit=4
Planning Time: 0.18 ms
Execution Time: 0.078 ms
```

**Reading it:**
- `LockRows` sits directly above the `Index Scan` — this is where `FOR UPDATE SKIP LOCKED` is applied. Each row the scan returns is passed to `LockRows`, which attempts the lock; under `SKIP LOCKED`, already-locked rows are discarded here and the scan is asked for the next.
- `width=48` is the same as the scan — `LockRows` doesn't change the row shape, only its lock state.
- The `Index Scan` uses `jobs_status_priority_created_idx` — a partial or composite index on `(status, priority DESC, created_at)` lets the scan return already-ordered rows so `ORDER BY` needs no separate `Sort` node. This is essential: without the index you'd get a `Sort` node that must materialize *all* queued rows before `LockRows` sees any, defeating cheap skipping.
- `actual rows=1` at every level because `LIMIT 1` short-circuits once one row is successfully locked.
- `Buffers: shared hit=5` — locking dirties heap buffers (row locks write to the tuple), so a `FOR UPDATE` touches more buffers than the equivalent plain SELECT.

Now the same table **without** a supporting index (anti-pattern):

```sql
EXPLAIN (ANALYZE)
SELECT id, payload FROM jobs
WHERE status = 'queued'
ORDER BY priority DESC, created_at ASC
FOR UPDATE SKIP LOCKED
LIMIT 1;
```

```
Limit  (cost=18542.10..18542.10 rows=1 width=48)
       (actual time=142.310..142.311 rows=1 loops=1)
  ->  LockRows  (cost=18542.10..18546.10 rows=400 width=48)
                (actual time=142.309..142.309 rows=1 loops=1)
        ->  Sort  (cost=18542.10..18543.10 rows=400 width=48)
                  (actual time=142.290..142.295 rows=380 loops=1)
              Sort Key: priority DESC, created_at
              Sort Method: quicksort  Memory: 74kB
              ->  Seq Scan on jobs
                    (cost=0.00..18525.00 rows=400 width=48)
                    (actual time=0.02..138.9 rows=100000 loops=1)
                    Filter: (status = 'queued'::text)
                    Rows Removed by Filter: 900000
Execution Time: 142.4 ms
```

**What's wrong:** the `Sort` node is *below* `LockRows`, so PostgreSQL must scan 1M rows, filter to 100K queued, sort all of them, and only then does `LockRows` start locking. Every worker does this full sort on every poll — 142ms instead of 0.078ms. The fix is the composite/partial index that eliminates both the `Seq Scan` and the `Sort`, letting `LockRows` see pre-ordered rows and stop after one.

**Signals to watch:**

| Symptom in plan | Meaning | Fix |
|---|---|---|
| `Sort` below `LockRows` | Whole result materialized before locking | Add index matching `ORDER BY` |
| `Seq Scan` under `LockRows` on large table | No index on WHERE/ORDER columns | Add partial index `WHERE status='queued'` |
| High `Buffers` on `LockRows` vs scan | Expected — locks write to heap | Keep locked txns short |
| `rows` estimate far above `LIMIT` | Planner over-scans before short-circuit | Ensure index provides order |

---

## 8. Query Examples

### Example 1 — Basic: Lock a single row for a balance update

```sql
-- Safely debit an account balance under concurrency (READ COMMITTED).
BEGIN;

-- Lock the row: any other FOR UPDATE / UPDATE on this account now blocks.
SELECT balance
FROM   accounts
WHERE  id = 42
FOR UPDATE;          -- returns current committed balance, e.g. 500

-- Application checks 500 >= 120, then:
UPDATE accounts
SET    balance = balance - 120
WHERE  id = 42;

COMMIT;              -- lock released; next waiter reads 380
```

### Example 2 — Intermediate: Non-blocking seat reservation with NOWAIT

```sql
-- Reserve a specific seat. If someone else is mid-booking it, fail fast.
BEGIN;

SELECT id, status
FROM   seats
WHERE  event_id = $1 AND seat_no = $2
FOR UPDATE NOWAIT;   -- raises 55P03 immediately if the seat row is locked

-- If we got the lock and status = 'free':
UPDATE seats
SET    status = 'reserved', reserved_by = $3, reserved_at = now()
WHERE  id = $4;

COMMIT;
-- The application maps SQLSTATE 55P03 → HTTP 409 "seat is being booked, retry".
```

### Example 3 — Production Grade: Concurrent job queue with SKIP LOCKED

```sql
-- Table: jobs(id bigserial, status text, priority int, run_after timestamptz,
--             attempts int, payload jsonb, created_at timestamptz, locked_by text)
-- Size: ~8M rows total, ~40K in status='queued' at peak.
-- Index (CRITICAL): partial composite index so the poll is index-only-ish and pre-sorted:
--   CREATE INDEX jobs_poll_idx ON jobs (priority DESC, run_after, created_at)
--     WHERE status = 'queued';
-- Perf expectation: sub-millisecond poll per worker, linear scaling to ~50 workers.

BEGIN;

-- Grab up to 10 due jobs that no other worker currently holds.
WITH next_jobs AS (
  SELECT id
  FROM   jobs
  WHERE  status = 'queued'
    AND  run_after <= now()
  ORDER BY priority DESC, run_after ASC, created_at ASC
  FOR UPDATE SKIP LOCKED
  LIMIT 10
)
UPDATE jobs j
SET    status     = 'processing',
       locked_by  = $1,           -- worker id
       attempts   = j.attempts + 1,
       started_at = now()
FROM   next_jobs n
WHERE  j.id = n.id
RETURNING j.id, j.payload;

COMMIT;   -- rows now 'processing'; lock released. Worker processes payloads,
          -- then a later txn sets status='done' or 'queued' (for retry).
```

Its EXPLAIN (the CTE portion):

```
Update on jobs j  (cost=... rows=10 ...)  (actual time=0.21..0.24 rows=10 loops=1)
  ->  Nested Loop  (actual rows=10 loops=1)
        ->  Subquery Scan on next_jobs  (actual rows=10 loops=1)
              ->  LockRows  (actual time=0.09..0.15 rows=10 loops=1)
                    ->  Limit  (actual rows=10 loops=1)
                          ->  Index Scan using jobs_poll_idx on jobs
                                Index Cond: (run_after <= now())
                                (actual rows=10 loops=1)
        ->  Index Scan using jobs_pkey on jobs j
              Index Cond: (id = next_jobs.id)
Execution Time: 0.31 ms
```

The `LockRows` above the index scan applies `SKIP LOCKED`; `jobs_poll_idx` supplies rows already in priority order so no `Sort` and only 10 rows are touched even with 40K queued.

---

## 9. Wrong → Right Patterns

### Wrong 1: Read without lock, then update (lost update)

```sql
-- WRONG: plain SELECT does not lock under READ COMMITTED
BEGIN;
SELECT stock FROM products WHERE id = 10;   -- returns 5 (no lock)
-- concurrent txn also reads 5, both write; one decrement is lost → oversell
UPDATE products SET stock = 4 WHERE id = 10;  -- app computed 5-1
COMMIT;
```
**Why it's wrong:** two transactions read the same snapshot value `5`; the second UPDATE overwrites the first's committed change instead of building on it. At the execution level, neither read placed an `xmax` lock, so nothing serialized them.

```sql
-- RIGHT: lock at read time, or do the arithmetic in one atomic UPDATE
BEGIN;
SELECT stock FROM products WHERE id = 10 FOR UPDATE;  -- serializes readers
UPDATE products SET stock = stock - 1 WHERE id = 10 AND stock >= 1;
COMMIT;
-- Even better when no app-side check is needed: skip the SELECT entirely and use
-- a single atomic UPDATE ... SET stock = stock - 1 WHERE id=10 AND stock>=1,
-- which takes the row lock implicitly and avoids a round trip.
```

### Wrong 2: FOR UPDATE without SKIP LOCKED in a worker queue

```sql
-- WRONG: every worker blocks on the same top-priority row
BEGIN;
SELECT id FROM jobs WHERE status='queued'
ORDER BY priority DESC LIMIT 1
FOR UPDATE;     -- 20 workers all target the SAME row and queue up behind it
```
**Why it's wrong:** all workers pick the identical highest-priority row; worker 1 locks it, workers 2–20 *block* waiting for that one row. Throughput collapses to serial, and the connection pool fills with blocked sessions. The `LockRows` node waits instead of skipping.

```sql
-- RIGHT: SKIP LOCKED so each worker grabs a different available row
BEGIN;
SELECT id FROM jobs WHERE status='queued'
ORDER BY priority DESC
FOR UPDATE SKIP LOCKED
LIMIT 1;         -- worker N locks the Nth available row; nobody blocks
```

### Wrong 3: FOR UPDATE on a JOIN locking unintended tables

```sql
-- WRONG: locks users rows too, blocking unrelated user updates
BEGIN;
SELECT o.id, u.email
FROM orders o JOIN users u ON u.id = o.user_id
WHERE o.id = $1
FOR UPDATE;     -- locks BOTH the order AND the user row
-- A concurrent profile update on that user now blocks on YOUR order-processing txn.
```
**Why it's wrong:** without `OF`, PostgreSQL locks every table contributing a row. You only meant to lock the order; you accidentally serialized all writes to that user.

```sql
-- RIGHT: restrict the lock to the order
BEGIN;
SELECT o.id, u.email
FROM orders o JOIN users u ON u.id = o.user_id
WHERE o.id = $1
FOR UPDATE OF o;   -- only orders is locked; users is read without a lock
```

### Wrong 4: Long transaction holding the lock across external I/O

```sql
-- WRONG: lock held while calling a payment gateway over HTTP
BEGIN;
SELECT * FROM orders WHERE id=$1 FOR UPDATE;
-- app now does: await stripe.charge(...)   ← 800ms network call
UPDATE orders SET status='paid' WHERE id=$1;
COMMIT;   -- the row was locked for ~800ms, blocking every writer to it
```
**Why it's wrong:** the row lock lives for the whole transaction. Holding it across a slow external call creates a lock convoy and can exhaust the pool. External side effects also can't be rolled back.

```sql
-- RIGHT: do external I/O outside the lock; lock only for the DB state change
-- 1) Outside any txn: call the gateway, get a result token.
-- 2) Short txn to record the outcome:
BEGIN;
SELECT status FROM orders WHERE id=$1 FOR UPDATE;   -- brief lock
UPDATE orders SET status='paid', charge_id=$2 WHERE id=$1;
COMMIT;
-- If the process crashes between step 1 and 2, reconcile via the gateway's
-- idempotency key rather than holding a DB lock across the network.
```

### Wrong 5: Expecting FOR UPDATE to give a strict global order with SKIP LOCKED

```sql
-- WRONG assumption: "SKIP LOCKED + ORDER BY priority guarantees jobs run in priority order"
SELECT id FROM jobs WHERE status='queued'
ORDER BY priority DESC
FOR UPDATE SKIP LOCKED LIMIT 1;
-- Worker B skips the priority-10 job locked by A and runs a priority-9 job FIRST.
```
**Why it's wrong:** `SKIP LOCKED` deliberately ignores locked rows, so a lower-priority job can start before a higher-priority one that's currently held. You get "best available," not strict global ordering.

```sql
-- RIGHT: if you truly need strict ordering, you cannot use SKIP LOCKED —
-- you need a single consumer (LIMIT 1, plain FOR UPDATE) or partition the queue
-- so each partition has one worker and ordering holds within it.
-- For most systems, "best available" is the correct and desired trade-off.
```

---

## 10. Performance Profile

### Cost model of a row lock

Acquiring a `FOR UPDATE` lock is not a pure in-memory operation: it writes the transaction/MultiXact ID into the tuple's `xmax`, sets infomask bits, dirties the heap page, and emits a (small) WAL record. So the per-row cost is higher than a read but far lower than a full UPDATE. The dominant cost at scale is not the lock write itself but **contention** — how long transactions block waiting for each other.

### Scaling behavior

| Scenario | 1M rows | 10M rows | 100M rows |
|---|---|---|---|
| `FOR UPDATE` single row by PK, indexed | <0.1ms | <0.1ms | <0.1ms (index depth ~3–4) |
| Queue poll `SKIP LOCKED LIMIT 1`, partial index | <0.2ms | <0.2ms | <0.3ms |
| Queue poll, **no** supporting index (Seq Scan+Sort) | ~30ms | ~300ms | multi-second — unusable |
| `FOR SHARE` on a hot row with 50 concurrent sharers | fine | MultiXact pressure possible | MultiXact SLRU contention |
| `FOR UPDATE` many rows, inconsistent order | deadlock risk | deadlock risk | deadlock risk |

The row *count* of the table barely matters when you lock by index; what matters is **how many concurrent transactions contend for the same rows** and **how long each holds the lock**.

### Contention math

If a hot row is held under `FOR UPDATE` for an average of `T` ms per transaction, the maximum throughput on that single row is `1000/T` transactions per second, *regardless of hardware* — the lock serializes them. A 50ms transaction caps that row at 20 TPS. This is why "keep locked transactions short" is the number-one performance rule: it directly sets the throughput ceiling on any contended row.

### Optimization techniques specific to locking

1. **Partial indexes for queue polls:** `CREATE INDEX ... WHERE status='queued'` keeps the index tiny (only pending rows) and lets the poll skip `Sort`. As jobs complete, they leave the index.
2. **Reduce lock duration:** move validation, external calls, and heavy computation outside the locked window.
3. **Lock ordering** to prevent deadlocks: `ORDER BY id FOR UPDATE` so all transactions acquire in the same order.
4. **Prefer atomic UPDATE over SELECT FOR UPDATE + UPDATE** when no app-side decision is needed — one statement, one round trip, implicit lock.
5. **Batch with `LIMIT` + `SKIP LOCKED`** so each worker grabs several rows per transaction, amortizing round-trips (but not so many that one worker hoards work or holds locks too long).
6. **Watch `pg_locks` and `pg_stat_activity.wait_event_type='Lock'`** to find blocking chains; `pg_blocking_pids()` reveals who blocks whom.
7. **Tune `deadlock_timeout`** cautiously — lowering it makes the detector run more often (CPU), raising it delays detection.

### FOR SHARE / MultiXact caveat

Many concurrent `FOR SHARE`/`FOR KEY SHARE` locks on the same few rows create MultiXacts. At very high scale this shifts the bottleneck to the `pg_multixact` SLRU and can trigger multixact-wraparound autovacuum. If you see `MultiXactOffset`/`MultiXactMember` wait events, reduce shared-locking of hot rows or restructure so the invariant is enforced differently (e.g., a summary row updated with `FOR NO KEY UPDATE`).

---

## 11. Node.js Integration

### 11.1 Basic FOR UPDATE inside a transaction (pg)

A locking clause is meaningless outside an explicit transaction — with autocommit, the lock is released the instant the SELECT returns. You must `BEGIN` … `COMMIT` on the **same** pooled client.

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function debitAccount(accountId, amount) {
  const client = await pool.connect();   // dedicate ONE connection to the txn
  try {
    await client.query('BEGIN');

    const { rows } = await client.query(
      `SELECT balance
         FROM accounts
        WHERE id = $1
        FOR UPDATE`,               // row locked until COMMIT/ROLLBACK
      [accountId]
    );
    if (rows.length === 0) throw new Error('account not found');

    const balance = Number(rows[0].balance);
    if (balance < amount) {
      await client.query('ROLLBACK');     // releases the lock
      return { ok: false, reason: 'insufficient_funds' };
    }

    await client.query(
      `UPDATE accounts SET balance = balance - $2 WHERE id = $1`,
      [accountId, amount]
    );
    await client.query('COMMIT');         // releases the lock
    return { ok: true, newBalance: balance - amount };
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();                     // return connection to the pool
  }
}
```

### 11.2 Background worker: SKIP LOCKED job queue

```javascript
// A single worker loop. Run N processes/containers of this concurrently —
// SKIP LOCKED guarantees they never grab the same job.
async function processOneBatch(workerId, batchSize = 5) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    const { rows } = await client.query(
      `WITH next_jobs AS (
         SELECT id
           FROM jobs
          WHERE status = 'queued'
            AND run_after <= now()
          ORDER BY priority DESC, run_after ASC, created_at ASC
          FOR UPDATE SKIP LOCKED
          LIMIT $2
       )
       UPDATE jobs j
          SET status = 'processing', locked_by = $1,
              attempts = j.attempts + 1, started_at = now()
         FROM next_jobs n
        WHERE j.id = n.id
       RETURNING j.id, j.payload`,
      [workerId, batchSize]
    );

    await client.query('COMMIT');   // rows are 'processing', locks released

    // Process OUTSIDE the transaction so no lock is held during work:
    for (const job of rows) {
      try {
        await handle(job.payload);
        await pool.query(`UPDATE jobs SET status='done', finished_at=now() WHERE id=$1`, [job.id]);
      } catch (e) {
        await pool.query(
          `UPDATE jobs SET status = CASE WHEN attempts >= 5 THEN 'failed' ELSE 'queued' END,
                          run_after = now() + interval '30 seconds'
            WHERE id = $1`,
          [job.id]
        );
      }
    }
    return rows.length;
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release();
  }
}
```

### 11.3 NOWAIT with fail-fast error handling

```javascript
async function reserveSeat(seatId, userId) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const { rows } = await client.query(
      `SELECT status FROM seats WHERE id = $1 FOR UPDATE NOWAIT`,
      [seatId]
    );
    if (rows[0].status !== 'free') {
      await client.query('ROLLBACK');
      return { ok: false, code: 409, reason: 'not_available' };
    }
    await client.query(
      `UPDATE seats SET status='reserved', reserved_by=$2 WHERE id=$1`,
      [seatId, userId]
    );
    await client.query('COMMIT');
    return { ok: true };
  } catch (err) {
    await client.query('ROLLBACK');
    if (err.code === '55P03') {                 // lock_not_available
      return { ok: false, code: 409, reason: 'busy_retry' };
    }
    throw err;
  } finally {
    client.release();
  }
}
```

### 11.4 Retry loop for serialization failures (REPEATABLE READ / SERIALIZABLE)

```javascript
// Under higher isolation, FOR UPDATE on a concurrently-updated row raises 40001.
async function withRetry(fn, maxAttempts = 5) {
  for (let attempt = 1; ; attempt++) {
    const client = await pool.connect();
    try {
      await client.query('BEGIN ISOLATION LEVEL REPEATABLE READ');
      const result = await fn(client);
      await client.query('COMMIT');
      return result;
    } catch (err) {
      await client.query('ROLLBACK');
      if (err.code === '40001' && attempt < maxAttempts) {   // serialization_failure
        await new Promise(r => setTimeout(r, 10 * attempt)); // small backoff
        continue;                                            // retry whole txn
      }
      throw err;
    } finally {
      client.release();
    }
  }
}
```

**ORM note:** Every serious ORM can express `FOR UPDATE` (see Section 12), but the *transaction boundary* is the part people get wrong — the SELECT and the follow-up UPDATE must run on the same connection inside one transaction. With the raw `pg` pool you must `pool.connect()` and reuse that client; a bare `pool.query('SELECT ... FOR UPDATE')` locks and immediately releases, doing nothing useful.

---

## 12. ORM Comparison

### Prisma

**Can Prisma do FOR UPDATE?** — Not in the type-safe query API for arbitrary strengths; you use `$queryRaw` inside `$transaction`. Prisma added no first-class `.forUpdate()` on `findMany`.

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

await prisma.$transaction(async (tx) => {
  const rows = await tx.$queryRaw<{ id: number; balance: number }[]>(
    Prisma.sql`SELECT id, balance FROM accounts WHERE id = ${accountId} FOR UPDATE`
  );
  if (rows[0].balance < amount) throw new Error('insufficient');
  await tx.$executeRaw(
    Prisma.sql`UPDATE accounts SET balance = balance - ${amount} WHERE id = ${accountId}`
  );
});

// Queue poll:
const jobs = await prisma.$queryRaw<{ id: bigint; payload: unknown }[]>(
  Prisma.sql`SELECT id, payload FROM jobs WHERE status='queued'
             ORDER BY priority DESC FOR UPDATE SKIP LOCKED LIMIT 10`
);
```

**Where it breaks:** no structured API for `FOR UPDATE`/`SKIP LOCKED`/`NOWAIT`; you drop to `$queryRaw` and lose type inference on the shape. You must run it inside `$transaction` or the lock releases immediately. **Verdict:** works via raw SQL only; fine for the well-known patterns, no ergonomic support.

---

### Drizzle ORM

**Can Drizzle do FOR UPDATE?** — Yes, first-class `.for()` with modifiers.

```typescript
import { db } from './db';
import { jobs, accounts } from './schema';
import { eq, and, lte, desc } from 'drizzle-orm';

await db.transaction(async (tx) => {
  const [acct] = await tx.select().from(accounts)
    .where(eq(accounts.id, accountId))
    .for('update');                       // FOR UPDATE
  if (acct.balance < amount) throw new Error('insufficient');
  await tx.update(accounts)
    .set({ balance: acct.balance - amount })
    .where(eq(accounts.id, accountId));
});

// Queue poll with SKIP LOCKED:
const rows = await db.select({ id: jobs.id, payload: jobs.payload })
  .from(jobs)
  .where(and(eq(jobs.status, 'queued'), lte(jobs.runAfter, new Date())))
  .orderBy(desc(jobs.priority))
  .for('update', { skipLocked: true })    // FOR UPDATE SKIP LOCKED
  .limit(10);

// NOWAIT and OF:
await tx.select().from(jobs).where(eq(jobs.id, id)).for('update', { noWait: true });
```

**Where it breaks:** `.for('update', { of: ... })` support varies by version; `FOR NO KEY UPDATE` / `FOR KEY SHARE` may need `sql` fragments on older versions. **Verdict:** best structured support — `skipLocked` and `noWait` are options, close to SQL.

---

### Sequelize

**Can Sequelize do FOR UPDATE?** — Yes, via `lock` (and `skipLocked`) options on `findAll`/`findOne`, inside a managed transaction.

```javascript
const { Transaction } = require('sequelize');

await sequelize.transaction(async (t) => {
  const acct = await Account.findOne({
    where: { id: accountId },
    lock: t.LOCK.UPDATE,          // FOR UPDATE
    transaction: t,
  });
  if (acct.balance < amount) throw new Error('insufficient');
  await acct.decrement('balance', { by: amount, transaction: t });
});

// Queue poll:
await sequelize.transaction(async (t) => {
  const jobs = await Job.findAll({
    where: { status: 'queued' },
    order: [['priority', 'DESC']],
    limit: 10,
    lock: t.LOCK.UPDATE,
    skipLocked: true,             // FOR UPDATE SKIP LOCKED
    transaction: t,
  });
  // ...
});
```

`t.LOCK` values: `UPDATE`, `SHARE`, `KEY_SHARE`, `NO_KEY_UPDATE`. Lock a specific table in a join with `lock: { level: t.LOCK.UPDATE, of: Model }`.

**Where it breaks:** `NOWAIT` is a separate `nowait: true` option and isn't combinable with `skipLocked`. Forgetting `transaction: t` silently runs outside the txn — the lock does nothing. **Verdict:** full support but easy to misuse; always pass the transaction.

---

### TypeORM

**Can TypeORM do FOR UPDATE?** — Yes, via `setLock` on the QueryBuilder, or `lock` option on repository finds, within a transaction.

```typescript
await dataSource.transaction(async (manager) => {
  const acct = await manager.createQueryBuilder(Account, 'a')
    .setLock('pessimistic_write')          // FOR UPDATE
    .where('a.id = :id', { id: accountId })
    .getOne();
  // ...
});

// SKIP LOCKED (TypeORM 0.3+):
const jobs = await manager.createQueryBuilder(Job, 'j')
  .setLock('pessimistic_write')
  .setOnLocked('skip_locked')              // FOR UPDATE SKIP LOCKED
  .where('j.status = :s', { s: 'queued' })
  .orderBy('j.priority', 'DESC')
  .limit(10)
  .getMany();
```

Lock modes: `pessimistic_read` (FOR SHARE), `pessimistic_write` (FOR UPDATE), `pessimistic_partial_write` (older SKIP LOCKED alias), plus `.setOnLocked('nowait' | 'skip_locked')`.

**Where it breaks:** the mapping of lock-mode names to SQL is not obvious (`pessimistic_write` ≠ literal "write"); `FOR NO KEY UPDATE` / `FOR KEY SHARE` aren't directly exposed. **Verdict:** solid support with slightly opaque naming.

---

### Knex.js

**Can Knex do FOR UPDATE?** — Yes: `.forUpdate()`, `.forShare()`, `.skipLocked()`, `.noWait()`, chained inside a `knex.transaction`.

```javascript
await knex.transaction(async (trx) => {
  const [acct] = await trx('accounts').where({ id: accountId }).forUpdate();
  if (acct.balance < amount) throw new Error('insufficient');
  await trx('accounts').where({ id: accountId }).decrement('balance', amount);
});

// Queue poll:
const jobs = await knex('jobs')
  .where('status', 'queued')
  .orderBy('priority', 'desc')
  .limit(10)
  .forUpdate()
  .skipLocked();                 // FOR UPDATE SKIP LOCKED

// Fail fast:
await trx('seats').where({ id }).forUpdate().noWait();

// Lock only one table in a join:
await trx('orders as o').join('users as u','u.id','o.user_id')
  .where('o.id', id).forUpdate('o');   // FOR UPDATE OF o
```

**Where it breaks:** `FOR NO KEY UPDATE` / `FOR KEY SHARE` need `knex.raw`; combining `.skipLocked()` and `.noWait()` is invalid (as in SQL). **Verdict:** most transparent — every wait policy has a builder method mapping 1:1 to SQL.

---

### ORM Summary Table

| ORM | FOR UPDATE | SKIP LOCKED | NOWAIT | FOR SHARE | Lock specific table | Verdict |
|-----|-----------|-------------|--------|-----------|--------------------|---------|
| Prisma | `$queryRaw` only | raw | raw | raw | raw | Raw SQL only |
| Drizzle | `.for('update')` | `{skipLocked:true}` | `{noWait:true}` | `.for('share')` | `{of:...}` (ver-dependent) | Best structured |
| Sequelize | `lock: t.LOCK.UPDATE` | `skipLocked:true` | `nowait:true` | `t.LOCK.SHARE` | `lock:{of:Model}` | Full, easy to misuse |
| TypeORM | `setLock('pessimistic_write')` | `setOnLocked('skip_locked')` | `setOnLocked('nowait')` | `pessimistic_read` | limited | Solid, opaque names |
| Knex | `.forUpdate()` | `.skipLocked()` | `.noWait()` | `.forShare()` | `.forUpdate('alias')` | Most transparent |

---

## 13. Practice Exercises

### Exercise 1 — Basic

Given `accounts(id, owner, balance)`, write a transaction that safely withdraws 50 from account 7 only if the balance is at least 50, using row locking so two concurrent withdrawals cannot both succeed on an insufficient balance. Show the `BEGIN`, the locking `SELECT`, the conditional `UPDATE`, and the `COMMIT`/`ROLLBACK`.

```sql
-- Write your query here
```

Then answer: why does a plain `SELECT balance ... ` (without `FOR UPDATE`) followed by an `UPDATE` allow a lost update under READ COMMITTED?

---

### Exercise 2 — Medium (combines Topic 53 Isolation Levels)

Given `inventory(product_id, qty)` and two concurrent transactions each reserving stock, write the reservation query using `FOR UPDATE`. Then explain what changes if you run the same logic under `REPEATABLE READ` instead of `READ COMMITTED`: which error code can appear, and what must your Node.js code do about it?

```sql
-- Write your query here (READ COMMITTED version)
```

---

### Exercise 3 — Hard (production simulation; naive answer is wrong)

You are building a job queue on `jobs(id, status, priority, run_after, attempts, payload)`. A junior engineer writes this worker poll:

```sql
BEGIN;
SELECT id, payload FROM jobs
WHERE status = 'queued'
ORDER BY priority DESC
LIMIT 1
FOR UPDATE;
-- ... process ...
UPDATE jobs SET status='done' WHERE id = $1;
COMMIT;
```

They run 20 workers and observe near-zero throughput and a pool full of blocked connections.

1. Explain, at the `LockRows` / execution level, exactly why throughput collapsed.
2. Rewrite the poll so 20 workers pull disjoint jobs without blocking, respecting `run_after <= now()` and incrementing `attempts`.
3. Add the exact index that makes the poll sub-millisecond, and explain why it removes the `Sort` node.
4. What happens to a job if the worker process crashes after `COMMIT` of the "processing" state but before finishing? Propose a recovery mechanism.

```sql
-- Write your queries here
```

---

### Exercise 4 — Interview Simulation

Design the locking strategy for a "transfer money between two accounts" operation that must (a) never lose an update, (b) never deadlock under concurrency, and (c) fail fast if either account is currently locked by another transfer. Provide the SQL, name every lock mode and wait policy you use, and justify the ordering of the locks.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Junior Level

**Q: What does `SELECT ... FOR UPDATE` do, and when do the locks release?**

A: It acquires a row-level write-intent lock on every row the SELECT returns, so other transactions cannot update, delete, or `FOR UPDATE`-lock those rows until you finish. The locks are held for the entire transaction and release automatically on `COMMIT` or `ROLLBACK` — there is no manual unlock. It only makes sense inside an explicit transaction; with autocommit the lock is released immediately.

---

**Q: What's the difference between `FOR UPDATE` and `FOR SHARE`?**

A: `FOR UPDATE` is an exclusive intent-to-write lock — only one transaction can hold it on a row, and it blocks all other locks and writes. `FOR SHARE` is a shared read-lock — multiple transactions can hold it on the same row at once, and it blocks writers but not other sharers. Use `FOR UPDATE` when you'll modify the row; use `FOR SHARE` when you only need to guarantee the row won't change while you read it.

---

**Q: What does `SKIP LOCKED` do and what's it good for?**

A: `SKIP LOCKED` tells the query to silently ignore rows that are currently locked by another transaction instead of waiting for them. It's the foundation of concurrent job queues: many workers run the same `SELECT ... FOR UPDATE SKIP LOCKED LIMIT n` and each grabs a different set of available rows without ever blocking, so the same job is never processed twice.

---

### Principal Level

**Q: You have a payment-processing endpoint that does `SELECT ... FOR UPDATE` on the order row, then calls an external gateway over HTTP, then updates the order. Under load the connection pool exhausts and latency spikes. Diagnose and fix.**

A: The row lock is held for the entire transaction, which now spans an ~800ms external HTTP call. Every concurrent request for the same order (or any request that contends on that pooled connection) blocks behind it, and because each blocked request holds a pool connection, the pool drains and unrelated queries starve — a classic lock convoy plus pool exhaustion. The fix is to never hold a database lock across network I/O. Restructure so the external call happens outside any transaction, using the gateway's idempotency key for safety, and use a short transaction only to persist the outcome (`SELECT ... FOR UPDATE` immediately followed by the `UPDATE` and `COMMIT`). Optionally add `NOWAIT` so a second concurrent attempt on the same order fails fast with 409 rather than queuing. The throughput ceiling on a contended row is `1000/T` TPS where `T` is the lock hold time, so cutting `T` from 800ms to 2ms raises that ceiling by ~400×.

---

**Q: Explain how `FOR UPDATE` behaves differently under READ COMMITTED versus REPEATABLE READ when a concurrent transaction has already modified the target row.**

A: Under READ COMMITTED, a blocked `FOR UPDATE` uses EvalPlanQual when it unblocks: it fetches the *latest committed* version of the row, re-applies the WHERE clause to that new version, and either locks it (if it still matches) or drops it from the result (if it no longer matches). So you get fresh data and no error. Under REPEATABLE READ or SERIALIZABLE, the transaction has a fixed snapshot from its start; silently switching to a newer row version would violate snapshot isolation. So instead PostgreSQL aborts the transaction with `SQLSTATE 40001`, `could not serialize access due to concurrent update`. The application must catch `40001` and retry the entire transaction from `BEGIN`. This is why any code using `FOR UPDATE` under higher isolation needs a retry loop with backoff, whereas under READ COMMITTED it does not.

---

**Q: A team reports intermittent deadlocks between transactions that "only insert into the child table." No one is updating the parent. How is that possible, and how do the key-share lock modes factor in?**

A: Inserting a child row that has a foreign key triggers a validation that takes a `FOR KEY SHARE` lock on the referenced parent row to ensure the parent isn't deleted or its key changed mid-insert. If two transactions insert children referencing the *same* parent, both take `FOR KEY SHARE` — which are compatible, so that alone doesn't deadlock. The deadlock arises when the transactions also lock *other* rows in inconsistent orders, or when one of them additionally takes a stronger lock on the parent (e.g., updates a key column or deletes it, needing `FOR UPDATE`, which conflicts with the other's `FOR KEY SHARE`). Before PostgreSQL 9.3 the situation was far worse because child updates took a full `FOR SHARE` on the parent, so plain concurrent child writes to a shared parent deadlocked routinely; the 9.3 introduction of `FOR KEY SHARE`/`FOR NO KEY UPDATE` made ordinary FK activity non-conflicting. The fix is to enforce a consistent lock-acquisition order across all transactions and avoid strengthening the parent lock inside these paths.

**Follow-up the interviewer asks:** *"How would you actually find the blocking chain in production?"* — Query `pg_stat_activity` for sessions with `wait_event_type = 'Lock'`, use `pg_blocking_pids(pid)` to get the blockers, and join to `pg_locks` to see the exact relation and lock modes; enable `log_lock_waits` and set `deadlock_timeout` so the server logs the full deadlock graph when the detector fires.

---

## 15. Mental Model Checkpoint

1. Two workers run `SELECT ... FROM jobs WHERE status='queued' ORDER BY priority DESC FOR UPDATE SKIP LOCKED LIMIT 1` at the same instant. Both see the same highest-priority row as the top candidate. Which job does each worker end up processing, and why do they not collide?

2. You run `SELECT balance FROM accounts WHERE id=5 FOR UPDATE` in transaction A and hold it. Transaction B runs a plain `SELECT balance FROM accounts WHERE id=5` (no locking clause). Does B block? Why or why not?

3. Under REPEATABLE READ, transaction A's `SELECT ... FOR UPDATE` targets a row that transaction B updated and committed after A's snapshot began. What does PostgreSQL do, and what must the application code be prepared to handle?

4. You write `SELECT o.*, u.* FROM orders o JOIN users u ON u.id=o.user_id WHERE o.id=$1 FOR UPDATE`. A teammate complains that unrelated updates to that user's profile are now blocking. What did you lock that you didn't intend to, and how do you fix it with one keyword?

5. Explain why `SELECT count(*) FROM orders FOR UPDATE` is rejected by the parser, in terms of the relationship between output rows and base-table rows.

6. A hot row is held under `FOR UPDATE` for an average of 100ms per transaction. What is the theoretical maximum throughput of operations that touch that row, and why can't more CPUs raise it?

7. When does `FOR UPDATE` create a MultiXact ID rather than just stamping a plain transaction ID into `xmax`, and why might heavy `FOR SHARE` traffic on a few rows eventually cause autovacuum pressure?

---

## 16. Quick Reference Card

```sql
-- LOCK STRENGTHS (weakest → strongest)
FOR KEY SHARE       -- blocks only key change / delete (used by FK checks)
FOR SHARE           -- shared read-lock; many holders; blocks writers
FOR NO KEY UPDATE   -- write intent, non-key; allows concurrent FK child inserts
FOR UPDATE          -- exclusive write intent; blocks everything else

-- CONFLICT TABLE (A holds row / B wants):  ✗ = B blocks
--                KEY SHARE  SHARE  NO KEY UPD  UPDATE
-- KEY SHARE          ✓        ✓         ✓         ✗
-- SHARE              ✓        ✓         ✗         ✗
-- NO KEY UPDATE      ✓        ✗         ✗         ✗
-- UPDATE             ✗        ✗         ✗         ✗

-- WAIT POLICIES (mutually exclusive)
FOR UPDATE                -- (default) WAIT: block until lock is free
FOR UPDATE NOWAIT         -- error 55P03 immediately if any row is locked
FOR UPDATE SKIP LOCKED    -- silently omit already-locked rows, return the rest

-- LOCK ONLY SPECIFIC TABLES IN A JOIN
SELECT ... FROM a JOIN b ... FOR UPDATE OF a;      -- lock a only
SELECT ... FOR UPDATE OF a FOR SHARE OF b;         -- mixed strengths

-- CANONICAL JOB-QUEUE POLL (N workers, no collisions, no blocking)
SELECT id FROM jobs WHERE status='queued' AND run_after<=now()
ORDER BY priority DESC, created_at
FOR UPDATE SKIP LOCKED LIMIT 10;
-- Index: CREATE INDEX ON jobs (priority DESC, run_after, created_at)
--        WHERE status='queued';   -- partial + pre-sorted, no Sort node

-- SAFE READ-MODIFY-WRITE
BEGIN;
SELECT balance FROM accounts WHERE id=$1 FOR UPDATE;
UPDATE accounts SET balance=balance-$2 WHERE id=$1;
COMMIT;   -- lock held only between BEGIN and COMMIT

-- NOT ALLOWED WITH: aggregates, GROUP BY, HAVING, DISTINCT,
--                   UNION/INTERSECT/EXCEPT, window functions
--   → lock base rows in a CTE, aggregate outside the CTE.

-- PERF / SAFETY RULES OF THUMB
-- • Locks live until COMMIT/ROLLBACK — keep locked txns SHORT.
-- • Never hold a row lock across external I/O (HTTP, email).
-- • Lock rows in a CONSISTENT order (ORDER BY id) to avoid deadlocks.
-- • Prefer a single atomic UPDATE ... WHERE over SELECT FOR UPDATE + UPDATE
--   when no app-side decision is needed.
-- • Under REPEATABLE READ/SERIALIZABLE, catch 40001 and RETRY the txn.
-- • NOWAIT → 55P03 (fail fast, 409);  SKIP LOCKED → skip (queues).
-- • FOR UPDATE writes xmax → dirties heap + WAL; not a free read.

-- INTERVIEW ONE-LINERS
-- "FOR UPDATE is pessimistic locking: reserve the row now, hold to commit."
-- "SKIP LOCKED is how you build a queue on Postgres without a broker."
-- "A blocked FOR UPDATE re-reads fresh data under READ COMMITTED (EPQ),
--  but raises 40001 under REPEATABLE READ."
-- "The key/no-key split exists to keep FK checks from blocking updates."
-- "Max TPS on a contended row = 1000 / lock_hold_ms — shorten the lock."
```

---

## Connected Topics

- **Topic 52 — MVCC and Snapshots**: `xmin`/`xmax`/infomask are where row locks physically live; the tuple-header mechanics behind every `FOR UPDATE`.
- **Topic 53 — Isolation Levels (previous)**: Determines whether a blocked `FOR UPDATE` re-reads fresh data (READ COMMITTED) or aborts with `40001` (REPEATABLE READ / SERIALIZABLE). Read this first.
- **Topic 55 — Deadlocks in Practice (next)**: `FOR UPDATE` on multiple rows in inconsistent order is the primary cause; consistent lock ordering is the fix.
- **Topic on Optimistic Concurrency (`version` columns)**: The alternative to pessimistic `FOR UPDATE` — no lock held, detect conflicts at write time. Know when each wins.
- **Foreign Keys and Referential Integrity**: The `FOR KEY SHARE` / `FOR NO KEY UPDATE` split exists entirely to make FK enforcement concurrent-friendly.
- **Topic 20 — GROUP BY Fundamentals**: Explains why locking clauses are incompatible with aggregation — output rows no longer map 1:1 to base rows.
- **Connection Pooling and Transactions**: Why a locking `SELECT` must run on the same pooled connection inside `BEGIN`…`COMMIT`, and how long-held locks exhaust the pool.
