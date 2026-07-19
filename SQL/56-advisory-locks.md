# Topic 56 — Advisory Locks
### SQL Mastery Curriculum — Phase 8: Transactions and Concurrency in SQL

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine a small shared office with a single conference room, but the room has no lock on its door. Instead, there is a whiteboard in the hallway. The office rule is simple: **before you walk into the conference room, you write your name on the whiteboard next to the room's number. When you leave, you erase your name.** If you walk up and someone else's name is already written there, you know the room is taken — you either wait, or you go do something else.

Here is the important part: **the whiteboard does not physically stop anyone from entering the room.** A new intern who never learned the rule can barge straight in. The whiteboard only works because everyone who shares the office has *agreed* to check it and honour it. It is a convention enforced by cooperation, not by a physical barrier.

That whiteboard is a PostgreSQL **advisory lock**. The "conference room" is not a table, not a row, not any database object at all — it is whatever *you*, the application developer, decide it means. It might mean "the nightly billing job," or "processing user 4471's account," or "being the leader in this cluster." Postgres does not know or care what the number means. It just guarantees that **at most one session can hold the whiteboard entry for a given number at a time**, and it tells everyone else "taken" when they ask.

Every other lock in Postgres is about protecting a *thing the database owns* — a row, a table, a page. Row locks (Topic 51), table locks, `SELECT ... FOR UPDATE` (Topic 54) — the database picks the lock for you based on what you touch. Advisory locks are the one kind of lock where **you invent the meaning**. Postgres hands you a fast, reliable, cluster-wide "is this number taken?" primitive and steps back. What the number *protects* is entirely a contract between the pieces of your application. That is why they are called *advisory*: the database advises, it does not enforce.

---

## 2. Connection to SQL Internals

Advisory locks live in the same place every other heavyweight lock in PostgreSQL lives: the **shared lock manager**, backed by an in-memory hash table in shared memory called the **lock table** (`pg_locks` is the view over it). This is the same subsystem that manages table locks, row-level lock *conflicts*, and the lock queue you learned about in the deadlock topic (Topic 55).

Concretely:

- Every lock in Postgres is identified by a **`LOCKTAG`** — a small struct that names *what* is being locked. For a table lock the tag encodes `(database oid, relation oid)`. For an advisory lock the tag has `locktag_type = ADVISORY` and the "object" fields hold **your 64-bit key** (either one `bigint`, or two `int4` values packed together). There is no heap page, no tuple, no relation behind it — the key *is* the lock identity.
- Advisory locks are **not** stored in `work_mem` or the buffer pool. They occupy an entry in the fixed-size shared lock table, sized by `max_locks_per_transaction × (max_connections + max_prepared_transactions)` at server start. Holding hundreds of thousands of distinct advisory locks simultaneously can exhaust this table and cause `ERROR: out of shared memory`.
- They are **completely divorced from MVCC**. A row lock is ultimately recorded in the tuple header (the `xmax` / infomask machinery from Topic 51) and is durable across the WAL. An advisory lock touches **no heap tuple, writes no WAL, and produces no dead rows**. It is pure in-memory bookkeeping in the lock manager. This is exactly why it is cheap — acquiring one is a hash-table insert under a partition lightweight lock (LWLock), not a page write.
- Because they write no WAL, advisory locks are **invisible to physical replication and do not survive a crash or failover**. On a streaming replica you cannot even take a session/transaction advisory lock in the normal way during recovery — the primitive is a property of one running instance's shared memory, not of the durable data.

The two scopes map to two different owners in the lock manager:

- **Session-scoped** advisory locks are owned by the **backend process (session)**. They are held until you explicitly unlock them or the session disconnects. They are *reference-counted*: locking the same key twice in one session requires two unlocks.
- **Transaction-scoped** advisory locks (`pg_advisory_xact_lock`) are owned by the **current transaction** and are released automatically at `COMMIT` or `ROLLBACK` — using the exact same resource-owner cleanup path that releases row and table locks at end of transaction. They cannot be unlocked manually and are **not** reference-counted in a way you manage.

So when you reason about advisory locks, reason about them as **entries in the same shared lock table as table locks, tagged by your integer instead of by a relation OID, and subject to the same deadlock detector** (Topic 55) — but with zero MVCC, zero WAL, and zero heap footprint.

---

## 3. Logical Execution Order Context

Advisory lock functions are **ordinary function calls**, so they slot into the logical query pipeline wherever the function is evaluated — but their *effect* (acquiring or releasing a lock) is a **side effect** that happens at execution time, not something the planner reorders freely like a join.

```
FROM ...
JOIN ...
WHERE ...          ← a pg_try_advisory_lock() in WHERE runs per-row, as a filter
GROUP BY ...
HAVING ...
SELECT ...          ← pg_advisory_lock() in the target list runs once per output row
ORDER BY ...
LIMIT ...           ← ⚠️ LIMIT can stop execution before all lock calls run
```

The subtleties that bite people:

1. **A lock function in `SELECT` runs once per row *produced*.** If you write `SELECT pg_advisory_lock(id) FROM jobs` you acquire a *separate* lock for every job row — usually not what you meant, and with session scope you now hold all of them.

2. **A lock function in `WHERE` runs as a per-row filter**, and crucially, **it runs in the physical row order the plan chooses**, which is *not* your `ORDER BY`. `WHERE pg_try_advisory_lock(id)` on a `Seq Scan` attempts locks in heap order; add an index scan and the order changes. This is the classic "skip-locked queue" pattern's footgun.

3. **`LIMIT` short-circuits.** `SELECT id FROM jobs WHERE pg_try_advisory_lock(id) LIMIT 1` will stop *as soon as it finds one row that returns TRUE* — so it acquires exactly one lock (good), but it may have *attempted and acquired* locks on rows it then discards if the plan evaluates the filter before the limit in an unexpected order. Always pair this pattern with an explicit *release of the ones you don't keep*, or use `FOR UPDATE SKIP LOCKED` (Topic 54) which the executor integrates cleanly.

4. **Relative to transactions:** `pg_advisory_xact_lock` binds to the *transaction*, not to the statement. The lock appears the instant the function executes and vanishes at end-of-transaction regardless of where in the statement pipeline it was called. Session locks ignore transaction boundaries entirely — a session lock acquired inside a transaction that then `ROLLBACK`s **stays held** (this surprises everyone the first time).

The practical rule: **treat advisory lock calls as imperative side effects, and control exactly how many times they execute.** Wrap them in a single-row `SELECT` (e.g. `SELECT pg_advisory_lock($1)`), or call them as their own statement, so their execution count is obvious and equal to one.

---

## 4. What Is an Advisory Lock?

An **advisory lock** is an application-defined lock, identified by a 64-bit integer key that *you* choose, which PostgreSQL tracks in its shared lock manager but does not associate with any table, row, or database object. It provides mutual exclusion **only among sessions that cooperatively check for it** — Postgres never blocks a query just because an advisory lock exists.

There are two orthogonal choices — **scope** (session vs transaction) and **behaviour** (blocking vs non-blocking) — plus **share vs exclusive** mode, giving a family of functions:

```sql
-- Exclusive, session-scoped, BLOCKING (waits until acquired)
SELECT pg_advisory_lock(key bigint);
--     │                 │
--     │                 └── your application-defined 64-bit identifier
--     └── blocks the session until the lock is granted; returns void

-- Exclusive, session-scoped, NON-BLOCKING (returns immediately)
SELECT pg_try_advisory_lock(key bigint);
--     │
--     └── returns boolean: TRUE if acquired, FALSE if someone else holds it
--         NEVER waits — this is the primitive for "am I the one who got it?"

-- Exclusive, TRANSACTION-scoped, BLOCKING
SELECT pg_advisory_xact_lock(key bigint);
--                        │
--                        └── "xact" = released automatically at COMMIT/ROLLBACK
--                            cannot be released manually; no unlock function

-- Exclusive, TRANSACTION-scoped, NON-BLOCKING
SELECT pg_try_advisory_xact_lock(key bigint);

-- Explicit release (session-scoped locks ONLY)
SELECT pg_advisory_unlock(key bigint);      -- returns boolean: TRUE if a lock was released
SELECT pg_advisory_unlock_all();            -- release ALL session advisory locks held now
```

### The two key shapes

Every function above comes in **two overloads** — one `bigint` key, or two `int4` keys:

```sql
pg_advisory_lock(key bigint)              -- single 64-bit key
pg_advisory_lock(key1 int4, key2 int4)    -- two 32-bit keys, treated as ONE lock

-- These are DIFFERENT lock spaces. The lock manager stores a "classifier"
-- bit that distinguishes the 1-key form from the 2-key form, so:
pg_advisory_lock(1)        -- and
pg_advisory_lock(0, 1)     -- are DISTINCT locks that never conflict,
--                            even though the bits could look "equal".
```

The two-int form is a convenience for namespacing: use `key1` as a "class" (e.g. `42` = "user account") and `key2` as an id (e.g. the user id). The single-`bigint` form gives you the full 64-bit space in one number.

### Shared mode

Each also has a **shared** variant that behaves like a reader lock — many sessions can hold the *shared* lock on a key simultaneously, but a shared lock conflicts with an *exclusive* lock on the same key:

```sql
SELECT pg_advisory_lock_shared(key);          -- session, shared, blocking
SELECT pg_try_advisory_lock_shared(key);      -- session, shared, non-blocking
SELECT pg_advisory_xact_lock_shared(key);     -- xact, shared, blocking
SELECT pg_advisory_unlock_shared(key);        -- release a session shared lock
```

### The complete function matrix

| Function | Scope | Blocking? | Mode | Manual unlock? |
|----------|-------|-----------|------|----------------|
| `pg_advisory_lock` | session | blocks | exclusive | yes (`pg_advisory_unlock`) |
| `pg_try_advisory_lock` | session | no | exclusive | yes |
| `pg_advisory_lock_shared` | session | blocks | shared | yes (`_unlock_shared`) |
| `pg_try_advisory_lock_shared` | session | no | shared | yes |
| `pg_advisory_xact_lock` | transaction | blocks | exclusive | no (auto at commit) |
| `pg_try_advisory_xact_lock` | transaction | no | exclusive | no |
| `pg_advisory_xact_lock_shared` | transaction | blocks | shared | no |
| `pg_try_advisory_xact_lock_shared` | transaction | no | shared | no |
| `pg_advisory_unlock` | — | — | exclusive | releases one session lock |
| `pg_advisory_unlock_shared` | — | — | shared | releases one session shared lock |
| `pg_advisory_unlock_all` | — | — | both | releases all session advisory locks |

That is the entire surface area. Two scopes × blocking/non-blocking × shared/exclusive, plus the unlock helpers, and each in a 1-key or 2-key overload.

---

## 5. Why Advisory Lock Mastery Matters in Production

1. **They are the only clean cross-session mutex you get "for free" with your database.** If your app already talks to Postgres, you do not need to add Redis, ZooKeeper, or etcd just to guarantee "only one worker runs the nightly report." Advisory locks give you a distributed mutex whose source of truth is the same database as your data, with the same connection, auth, and monitoring you already have.

2. **They prevent duplicate work in a horizontally scaled fleet.** The single most common production use: you run 10 copies of your Node.js service, all with the same cron config, and *all ten* wake up at 02:00 to "send the daily digest." Without coordination you send every email ten times. A `pg_try_advisory_lock` at the top of the job means exactly one instance proceeds and nine bow out instantly.

3. **They avoid holding row/table locks for long logical operations.** Suppose "recomputing user 4471's loyalty tier" involves reading many tables, calling an external API, and writing back. Doing that under `SELECT ... FOR UPDATE` (Topic 54) holds a row lock — and a transaction — open for the whole slow operation, bloating locks and blocking writers. An advisory lock keyed on the user id serialises the *logical operation* without pinning a transaction to a row, and without the MVCC/bloat cost.

4. **They fail in silent, dangerous ways if you get scope or pooling wrong.** This is the flip side. A session-scoped lock acquired on a pooled connection that is then returned to the pool **stays held by whichever connection object the pool hands out next** — you have leaked a lock and can deadlock your own app against itself. Knowing session-vs-transaction scope and the connection-pooling pitfall (Section 6.7) is what separates "advisory locks solved our problem" from "advisory locks caused a 3am incident."

5. **They participate in the deadlock detector, so misuse can deadlock real queries.** Because advisory locks sit in the same lock table as table locks (Section 2), two sessions that each hold one advisory lock and wait for the other's — or that mix advisory locks with row locks in inconsistent order — will be detected and one will be aborted with `deadlock detected` (Topic 55). They are not a magic escape from deadlocks; they are one more lock to order consistently.

6. **They are the standard primitive for one-off coordination that has no natural database row.** Leader election, "run this schema migration exactly once across the fleet," "only one process may rebuild the search index" — these have no row to lock `FOR UPDATE`. The thing being protected is *an action*, not *a datum*. Advisory locks are purpose-built for exactly this.

---

## 6. Deep Technical Content

### 6.1 Session Scope vs Transaction Scope — the Core Distinction

This is the single most important thing to internalise.

**Transaction-scoped** (`pg_advisory_xact_lock`): the lock is tied to the transaction. It is released automatically at `COMMIT` or `ROLLBACK`. You **cannot** release it early, and there is no unlock function for it. This is the *safe default* because there is no way to leak it — the transaction ending always cleans it up.

```sql
BEGIN;
SELECT pg_advisory_xact_lock(42);   -- acquired
-- ... do work ...
COMMIT;                             -- lock released automatically here
-- Even if you had done ROLLBACK, or the statement errored and the tx aborted,
-- the lock would be released. There is no leak path.
```

**Session-scoped** (`pg_advisory_lock`): the lock is tied to the *backend session* (the connection), and it **ignores transaction boundaries completely**. It survives `COMMIT`, survives `ROLLBACK`, and is only released by an explicit `pg_advisory_unlock` (with the matching key and mode), by `pg_advisory_unlock_all()`, or by the session disconnecting.

```sql
BEGIN;
SELECT pg_advisory_lock(42);        -- acquired
ROLLBACK;                           -- transaction gone...
-- ...but the advisory lock is STILL HELD. It is a property of the session,
-- not the transaction. You must:
SELECT pg_advisory_unlock(42);      -- returns TRUE, now released
```

The consequence: **session-scoped advisory locks require disciplined release in a `finally` block.** Transaction-scoped locks are self-cleaning. If you can express your critical section inside a single transaction, prefer the `xact` variant — it removes the entire class of "leaked lock" bugs.

### 6.2 Reference Counting of Session Locks

Session-scoped advisory locks are **reference-counted**. Acquiring the same lock (same key, same mode) N times in one session requires N unlocks to fully release it.

```sql
SELECT pg_advisory_lock(42);   -- refcount 1
SELECT pg_advisory_lock(42);   -- refcount 2 (does NOT block — same session already holds it)
SELECT pg_advisory_unlock(42); -- refcount 1, returns TRUE, still held
SELECT pg_advisory_unlock(42); -- refcount 0, returns TRUE, now actually released
SELECT pg_advisory_unlock(42); -- returns FALSE + WARNING: you unlocked something not held
```

A session can **always re-acquire a lock it already holds** — advisory locks are not re-entrant *blocking*; the same session stacks the count and proceeds immediately. Different sessions conflict as expected. Transaction-scoped locks are *not* something you unlock, so ref-counting is not a concern you manage — nested `xact` locks in the same transaction simply all release together at commit.

`pg_advisory_unlock` returns `boolean`: `TRUE` if it decremented a held lock, `FALSE` (plus a `WARNING`) if the session did not hold that lock in that mode. Always check the return in code where correctness depends on it.

### 6.3 The 64-Bit Key Space and the Two-Int Trap

The key space is a full signed 64-bit integer: `-9223372036854775808 .. 9223372036854775807`. That is the `bigint` overload. The two-`int4` overload packs two 32-bit *signed* integers.

The trap that surprises people: **the one-argument `bigint` form and the two-argument `int4,int4` form occupy separate lock spaces.** The lock manager records a classifier field so it can tell them apart. Therefore:

```sql
-- Are these the same lock? NO.
pg_advisory_lock(4294967296);   -- bigint: 2^32 = 0x1_0000_0000
pg_advisory_lock(1, 0);         -- two int4: key1=1, key2=0
-- Bit-wise, (1<<32 | 0) == 4294967296, so you might think they collide.
-- They DO NOT. The 1-key and 2-key forms are different namespaces by design.
```

So you must pick **one convention and stick to it across your whole codebase.** Mixing "I'll use a single bigint here and two ints there" for what you *think* is the same resource means two different code paths take two different locks and never exclude each other. Pick one:

- **Single `bigint`**: hash a string to a 64-bit value, or compute `class * BIG_CONSTANT + id`. Full range, one number.
- **Two `int4`**: reserve `key1` as a namespace enum (`1`=user, `2`=report, `3`=migration…) and `key2` as the entity id. Human-readable in `pg_locks`.

### 6.4 Deriving Keys from Strings (hashtext / hashtextextended)

Most real resources are named by strings — `"nightly-billing"`, `"user:4471"`, a UUID. You need a stable mapping from string to a 64-bit (or 32-bit) integer. Postgres ships hash functions:

```sql
-- 32-bit signed hash (stable across sessions, same DB version family)
SELECT hashtext('nightly-billing');           -- e.g. -1547923004  (int4)

-- 64-bit hash with a seed (preferred for the bigint key form)
SELECT hashtextextended('nightly-billing', 0);  -- e.g. 8135927461023847  (int8)

-- Typical usage: one namespace int + a hashed id
SELECT pg_advisory_lock(1, hashtext('user:4471'));   -- key1=1 (namespace), key2=hash
-- or the single-key form:
SELECT pg_advisory_lock(hashtextextended('billing:daily:2026-07-19', 0));
```

Caveats:
- `hashtext` returns a **signed int4** — negative values are normal and fine as keys.
- Two different strings can collide to the same hash (birthday paradox at ~2^16 strings for 32-bit). A collision means two unrelated resources *accidentally* serialise against each other — a correctness-neutral but performance-harming false conflict. For the 32-bit form and high-cardinality keys, prefer `hashtextextended` (64-bit) to make collisions astronomically unlikely.
- The hash algorithm is stable within a major-version lineage but is **not** a documented cross-version contract. Do not persist a hashed key and assume it matches after a major upgrade. Recompute from the string; never store the hash as the source of truth.

### 6.5 Blocking vs Non-Blocking — Choosing the Right Behaviour

```sql
-- BLOCKING: I MUST do this work; wait my turn.
SELECT pg_advisory_lock(key);
-- Use when serialization is required and every caller must eventually run.
-- Danger: unbounded wait. Pair with statement_timeout / lock_timeout.

-- NON-BLOCKING: I want to do this IF no one else is; otherwise skip.
SELECT pg_try_advisory_lock(key);   -- boolean
-- Use for "only one instance runs the cron" — losers should exit, not queue.
```

Guard blocking acquisitions with a timeout so a stuck holder cannot hang callers forever:

```sql
SET lock_timeout = '5s';            -- error out if not granted within 5s
SELECT pg_advisory_lock(key);       -- raises: canceling statement due to lock timeout
```

Note `lock_timeout` applies to advisory lock waits exactly as it does to table/row lock waits — same code path (Section 2).

### 6.6 Shared vs Exclusive Advisory Locks

The `_shared` variants implement a reader/writer pattern **entirely by convention**:

| Held mode → / Requested ↓ | (none) | Shared | Exclusive |
|---------------------------|--------|--------|-----------|
| **Shared** | granted | granted | conflict (waits/FALSE) |
| **Exclusive** | granted | conflict | conflict |

Use case: many sessions may "read/use resource X concurrently" (take shared), but a maintenance task that "rebuilds X" needs everyone out (takes exclusive, which waits for all shared holders to release and blocks new shared holders). This mirrors a `RWLock`. In practice, most advisory-lock use is plain exclusive mutual exclusion; shared mode is for the occasional read-mostly-with-rare-exclusive-maintenance pattern.

### 6.7 The Connection-Pooling Pitfall (the big one)

This is the failure mode that causes real incidents. **A session-scoped advisory lock belongs to the physical database connection, not to your logical request.** Connection pools (both app-side like `pg.Pool`, and especially external poolers like **PgBouncer**) hand the same physical connection to different logical requests over time.

Two distinct disasters:

**(a) Leaked session lock via app pool.** You check out a connection, `pg_advisory_lock(42)`, do work, and return the connection to the pool **without unlocking** (e.g. an exception skipped your unlock). The pool later hands that same connection to an unrelated request. That request's session *still holds lock 42*. Now:
- The original logical owner thinks it released (it didn't).
- A different request holds a lock it never asked for and never releases.
- Any waiter on key 42 blocks forever until that connection happens to disconnect.

**(b) PgBouncer in `transaction` pooling mode breaks session locks entirely.** In transaction pooling, PgBouncer multiplexes many clients over few server connections, reassigning the server connection **at every transaction boundary**. A session-scoped advisory lock taken in one transaction is on a server connection that PgBouncer may give to a *different client* for the next transaction. Your `pg_advisory_unlock` may run on a *different* server connection that never held the lock (returns FALSE + WARNING), while the lock leaks on the original one. **Session-scoped advisory locks are fundamentally incompatible with PgBouncer transaction/statement pooling.**

The defences, in order of preference:

1. **Prefer transaction-scoped locks** (`pg_advisory_xact_lock`). They are released at `COMMIT`/`ROLLBACK`, so even PgBouncer transaction pooling is safe: the lock's lifetime is exactly one transaction, which is exactly PgBouncer's unit of multiplexing. This is the single best mitigation.
2. If you must use session locks, **acquire and release on the *same* checked-out client** and put the unlock in a `finally` (Section 11). Never let a request end with a session lock still held.
3. With PgBouncer, use **session pooling mode** for any connection that takes session advisory locks, or route lock operations to a dedicated non-pooled connection.
4. **Monitor `pg_locks`** for `locktype = 'advisory'` entries whose backing PID is idle — a lock held by a long-idle connection is a probable leak.

### 6.8 Inspecting Advisory Locks in `pg_locks`

Advisory locks show up in the `pg_locks` view with `locktype = 'advisory'`:

```sql
SELECT
  l.locktype,
  l.mode,                 -- ExclusiveLock / ShareLock
  l.granted,              -- TRUE = held, FALSE = waiting in queue
  l.pid,                  -- backend holding/waiting
  l.classid,              -- for 2-int keys: key1; for bigint: high 32 bits
  l.objid,                -- for 2-int keys: key2; for bigint: low 32 bits
  l.objsubid,             -- 1 = two-int form, 2 = single-bigint form
  a.state,
  a.query
FROM pg_locks l
JOIN pg_stat_activity a ON a.pid = l.pid
WHERE l.locktype = 'advisory'
ORDER BY l.granted, l.pid;
```

Reading it:
- `objsubid = 1` → the lock was taken with the **two-int** form; `classid` is `key1`, `objid` is `key2`.
- `objsubid = 2` → the lock was taken with the **single bigint** form; the 64-bit key is `(classid::bigint << 32) | objid`.
- `granted = false` rows are sessions **waiting** for the lock — a queue. If you see many, you have contention on one key.
- A `granted = true` row whose `pid` maps to a session in `state = 'idle'` (not `idle in transaction`, just idle) for a long time is the fingerprint of a **leaked session lock** (Section 6.7).

### 6.9 Advisory Locks and the Deadlock Detector

Advisory locks are ordinary lockable objects to the deadlock detector (Topic 55). Classic advisory-lock deadlock:

```sql
-- Session A                          -- Session B
SELECT pg_advisory_lock(1);          SELECT pg_advisory_lock(2);
SELECT pg_advisory_lock(2);  -- waits SELECT pg_advisory_lock(1);  -- waits
-- ERROR: deadlock detected -- one of the two is aborted after deadlock_timeout
```

The same lock-ordering discipline from Topic 55 applies: **always acquire multiple advisory locks in a consistent global order** (e.g. ascending key). And beware mixing advisory locks with row locks — if session A holds advisory(1) then waits on a row that session B has locked while B waits on advisory(1), that is a cross-type deadlock the detector will resolve by aborting one session.

### 6.10 What Advisory Locks Do NOT Do

- **They do not protect data.** Nothing stops a session that never called the lock function from reading or writing the "protected" rows. Enforcement is 100% cooperative.
- **They do not survive crashes or failover.** No WAL, no durability. On failover to a replica, all advisory locks are gone. Do not rely on them as a durable "this ran once, ever" record — pair them with a durable marker row if you need exactly-once *durability* (Section 8, Example 3).
- **They are not fair queues with priorities.** Blocking waiters form a FIFO-ish queue at the lock manager, but you get no priority control.
- **They are node-local to one Postgres instance.** They coordinate all sessions on *one* primary. They do not coordinate across two independent Postgres clusters (there is no shared lock table between them).

---

## 7. EXPLAIN — Advisory Locks in the Plan

Advisory lock functions are `VOLATILE` functions returning scalar values; in a plan they appear as function evaluations, and the *lock acquisition* is a runtime side effect, not a scan node. The interesting thing EXPLAIN shows is **how many times the function will be invoked** — which equals how many locks you attempt.

### A single explicit acquisition (the correct pattern)

```sql
EXPLAIN (ANALYZE, VERBOSE)
SELECT pg_try_advisory_lock(1, 4471);
```

```
Result  (cost=0.00..0.01 rows=1 width=1) (actual time=0.031..0.032 rows=1 loops=1)
  Output: pg_try_advisory_lock(1, 4471)
Planning Time: 0.020 ms
Execution Time: 0.048 ms
```

**Reading it**: a `Result` node evaluates the function exactly **once** (`rows=1 loops=1`). One lock attempt. This is what you want — an unambiguous single side effect. Sub-millisecond: acquiring an advisory lock is an in-memory hash-table operation.

### The dangerous per-row pattern

```sql
EXPLAIN (ANALYZE, VERBOSE)
SELECT id, pg_try_advisory_lock(id)
FROM jobs
WHERE status = 'pending';
```

```
Seq Scan on public.jobs  (cost=0.00..189.00 rows=1200 width=5)
                         (actual time=0.028..3.412 rows=1200 loops=1)
  Output: id, pg_try_advisory_lock(id)
  Filter: (jobs.status = 'pending'::text)
  Rows Removed by Filter: 8800
Planning Time: 0.11 ms
Execution Time: 3.9 ms
```

**Reading it**: the `Output` line shows `pg_try_advisory_lock(id)` is evaluated **per output row** — `rows=1200` means **1200 separate advisory locks attempted and (mostly) acquired**, all held by this session until unlocked. On a session scope this is a lock-table flood: 1200 entries in the shared lock table from one statement. Note the function runs *after* the `Filter`, but *for every surviving row*. This is almost never intended.

### The queue pattern with LIMIT — watch the evaluation order

```sql
EXPLAIN (ANALYZE, VERBOSE)
SELECT id
FROM jobs
WHERE status = 'pending'
  AND pg_try_advisory_xact_lock(id)
ORDER BY created_at
LIMIT 1;
```

```
Limit  (cost=8.41..8.42 rows=1 width=12) (actual time=0.210..0.211 rows=1 loops=1)
  Output: id, created_at
  ->  Sort  (cost=8.41..8.44 rows=6 width=12) (actual time=0.209..0.209 rows=1 loops=1)
        Output: id, created_at
        Sort Key: jobs.created_at
        Sort Method: top-N heapsort  Memory: 25kB
        ->  Seq Scan on public.jobs
              (cost=0.00..8.38 rows=6 width=12) (actual time=0.05..0.19 rows=42 loops=1)
              Output: id, created_at
              Filter: ((status = 'pending') AND pg_try_advisory_xact_lock(id))
              Rows Removed by Filter: 158
```

**Reading it — the trap**: the `Sort` sits **above** the `Seq Scan`, so `pg_try_advisory_xact_lock(id)` in the `Filter` runs **during the scan, before the sort/limit**. The scan attempted the lock on **42 rows** (the ones passing `status='pending'`) even though `LIMIT 1` keeps one. With transaction scope those 41 extra locks release at commit — acceptable. With **session scope** you would have leaked 41 locks. This is exactly why the SKIP LOCKED pattern (Topic 54) is usually preferable for queue draining: `FOR UPDATE ... SKIP LOCKED` integrates locking into the scan node and does not over-acquire. The lesson EXPLAIN teaches: **the plan shape decides how many rows your lock function touches — always check it.**

### What to look for in EXPLAIN

| Symptom in plan | Meaning | Action |
|-----------------|---------|--------|
| Lock function in `Output` of a multi-row node | One lock **per row** attempted | Wrap in single-row `SELECT`; expect a flood |
| Lock function in `Filter` below a `Sort`/`Limit` | Locks taken on **pre-limit** row count | Use `xact` scope or `SKIP LOCKED` instead |
| `Result` node, `rows=1 loops=1` | Exactly one lock attempt | Correct — the intended pattern |
| Function under `loops=N` (nested loop) | N attempts | Ensure key is constant per intended critical section |

---

## 8. Query Examples

### Example 1 — Basic: A Distributed Mutex for a Cron Job

```sql
-- Only one instance in the fleet should run the nightly report.
-- Each instance runs this at 02:00; exactly one wins.

-- Transaction-scoped, non-blocking: if we don't get it, we skip cleanly.
BEGIN;
SELECT pg_try_advisory_xact_lock(hashtextextended('cron:nightly-report', 0)) AS got_it;
--   returns TRUE  -> proceed to run the report inside this transaction
--   returns FALSE -> another instance is already running it; ROLLBACK and exit
-- ... run the report work here, only if got_it was TRUE ...
COMMIT;   -- lock auto-released; safe even if we crash mid-report
```

### Example 2 — Intermediate: Serialising Work Per Entity (Blocking, With Timeout)

```sql
-- Recompute a single user's loyalty tier. Two requests for the SAME user must
-- not run concurrently, but different users run in parallel freely.
-- We serialise per-user with a two-int key: (namespace=1 "user", user_id).

SET lock_timeout = '3s';   -- don't wait forever if another recompute is stuck

BEGIN;
-- Blocking xact lock: queue behind any in-flight recompute for THIS user.
SELECT pg_advisory_xact_lock(1, 4471);   -- key1=1 (user namespace), key2=user_id
-- We now hold exclusive right to recompute user 4471 for the rest of this tx.

UPDATE users u
SET loyalty_tier = t.tier
FROM (
  SELECT CASE
           WHEN SUM(o.total_amount) >= 10000 THEN 'platinum'
           WHEN SUM(o.total_amount) >=  2000 THEN 'gold'
           ELSE 'standard'
         END AS tier
  FROM orders o
  WHERE o.customer_id = 4471
    AND o.status = 'completed'
) t
WHERE u.id = 4471;

COMMIT;   -- releases the advisory lock automatically
-- If lock_timeout fired, we got: ERROR canceling statement due to lock timeout,
-- the tx aborted, no lock leaked.
```

### Example 3 — Production Grade: Exactly-Once Job Runner With Durable Marker

```sql
-- CONTEXT:
--   job_runs table: ~5M rows historically, indexed on (job_key, run_date).
--   fleet of 12 Node.js workers, all fire the same schedule.
--   Requirement: each (job_key, run_date) executes AT MOST once, AND we keep a
--   DURABLE record that it ran (advisory locks alone are not durable — Section 6.10).
--
-- STRATEGY: advisory lock guarantees single-runner AT THIS MOMENT (fast, no row
--   contention); the durable marker row guarantees it never runs AGAIN on a
--   future day / after failover. Belt and suspenders.
--
-- INDEX AVAILABILITY:
--   CREATE UNIQUE INDEX job_runs_key_date_uq ON job_runs(job_key, run_date);
-- PERF EXPECTATION: advisory acquisition < 1ms; the unique-index probe is a
--   single-row index lookup; total well under 5ms in the "already ran" fast path.

BEGIN;

-- 1) Grab the transaction-scoped advisory lock for this job today.
--    key1 = hashtext(job_key), key2 = run_date as days-since-epoch (fits int4).
SELECT CASE
  WHEN pg_try_advisory_xact_lock(
         hashtext('daily-digest'),
         (CURRENT_DATE - DATE '1970-01-01')  -- int: days since epoch, unique per day
       )
  THEN TRUE ELSE FALSE
END AS acquired;
-- If acquired = FALSE: another worker holds it RIGHT NOW. ROLLBACK and exit.

-- 2) Durable idempotency check + claim, guarded by the unique index.
--    ON CONFLICT DO NOTHING makes the INSERT the atomic "claim" of the run.
INSERT INTO job_runs (job_key, run_date, started_at, worker_id, status)
VALUES ('daily-digest', CURRENT_DATE, NOW(), 'worker-07', 'running')
ON CONFLICT (job_key, run_date) DO NOTHING
RETURNING id;
-- 0 rows returned -> a prior run already claimed today (durably). ROLLBACK, exit.
-- 1 row returned  -> we own today's run; proceed.

-- 3) ... perform the actual digest work here (still inside the tx or a follow-up) ...

-- 4) Mark complete.
UPDATE job_runs
SET status = 'complete', finished_at = NOW()
WHERE job_key = 'daily-digest' AND run_date = CURRENT_DATE;

COMMIT;   -- advisory lock auto-released; durable marker persists forever.
```

```sql
-- EXPLAIN of the critical idempotent-claim INSERT:
EXPLAIN (ANALYZE, BUFFERS)
INSERT INTO job_runs (job_key, run_date, started_at, worker_id, status)
VALUES ('daily-digest', CURRENT_DATE, NOW(), 'worker-07', 'running')
ON CONFLICT (job_key, run_date) DO NOTHING;
```

```
Insert on job_runs  (cost=0.00..0.01 rows=0 width=0)
                    (actual time=0.089..0.089 rows=0 loops=1)
  Conflict Resolution: NOTHING
  Conflict Arbiter Indexes: job_runs_key_date_uq
  Tuples Inserted: 0
  Conflicting Tuples: 1
  Buffers: shared hit=7
  ->  Result  (cost=0.00..0.01 rows=1 width=64) (actual time=0.010..0.011 rows=1 loops=1)
Planning Time: 0.14 ms
Execution Time: 0.16 ms
```

**Reading it**: `Conflict Arbiter Indexes: job_runs_key_date_uq` confirms the unique index arbitrates the claim; `Conflicting Tuples: 1` / `Tuples Inserted: 0` is the "already ran today" fast path — 7 buffer hits, 0.16ms, no work done twice. The advisory lock handled the *concurrent-right-now* race; the unique index handled the *durable ever-again* guarantee.

---

## 9. Wrong → Right Patterns

### Wrong 1: Session lock on a pooled connection, never released

```sql
-- WRONG (application uses pg.Pool; simplified to SQL the app runs):
SELECT pg_advisory_lock(4471);   -- session-scoped, on a checked-out pool connection
-- ... do work, then an exception is thrown before the unlock runs ...
-- The connection is returned to the pool WITH LOCK 4471 STILL HELD.
```

**Exact failure**: the next request that checks out this physical connection *inherits* a lock it never asked for. Any session calling `pg_advisory_lock(4471)` (blocking) hangs forever; any `pg_try_advisory_lock(4471)` returns FALSE forever. `pg_locks` shows `granted=true` on an `idle` backend. **Why at the execution level**: session-scoped advisory locks are owned by the *backend process*, and pooling reuses backend processes across logical requests (Section 6.7). The lock outlives the request that took it.

```sql
-- RIGHT: use transaction scope so the lock cannot outlive the transaction.
BEGIN;
SELECT pg_advisory_xact_lock(4471);
-- ... do work; if it throws, the tx aborts ...
COMMIT;   -- or ROLLBACK on error — EITHER releases the lock. No leak possible.
```

If you truly need session scope, the release **must** be in a `finally` on the *same* client (Section 11), never spanning a pool return.

### Wrong 2: Per-row lock explosion from putting the function in SELECT

```sql
-- WRONG: intended "lock the one job I'm about to process"
SELECT id, pg_advisory_lock(id)
FROM jobs
WHERE status = 'pending';
-- ERROR? No — worse: it SUCCEEDS and silently acquires a session lock on
-- EVERY pending row (e.g. 1,200 locks), all held until unlock/disconnect.
```

**Exact result**: 1,200 entries appear in the shared lock table under one session (see the EXPLAIN in Section 7). At scale this raises `ERROR: out of shared memory` / `You might need to increase max_locks_per_transaction`, taking down *other* sessions too. **Why**: a volatile function in the target list is evaluated once per output row (Section 3).

```sql
-- RIGHT: claim exactly one job with a single, scoped lock attempt.
-- Prefer SKIP LOCKED (Topic 54) which integrates with the scan:
SELECT id
FROM jobs
WHERE status = 'pending'
ORDER BY created_at
FOR UPDATE SKIP LOCKED
LIMIT 1;
-- Or, if using advisory locks, one explicit call for the chosen id:
-- SELECT pg_try_advisory_xact_lock(id) for a single, already-selected id.
```

### Wrong 3: Assuming ROLLBACK releases a session lock

```sql
-- WRONG: expecting the lock to be gone after rollback
BEGIN;
SELECT pg_advisory_lock(42);   -- session-scoped
-- ... error occurs ...
ROLLBACK;
-- Developer believes lock 42 is now free. IT IS NOT.
SELECT pg_try_advisory_lock(42) FROM another_session;  -- returns FALSE, still held!
```

**Exact result**: the lock survives the rollback because it is session-scoped, not transaction-scoped (Section 6.1). The rolled-back session silently holds 42 until it disconnects or explicitly unlocks. **Why**: session advisory locks are owned by the backend and ignore transaction boundaries by design.

```sql
-- RIGHT: use the xact variant so rollback DOES release it.
BEGIN;
SELECT pg_advisory_xact_lock(42);
-- ... error occurs ...
ROLLBACK;   -- lock 42 released here, automatically.
```

### Wrong 4: Mixing the 1-key and 2-key forms for the same resource

```sql
-- Module A guards "user 4471" like this:
SELECT pg_advisory_lock(4471);           -- single-bigint form
-- Module B guards "user 4471" like this:
SELECT pg_advisory_lock(1, 4471);        -- two-int form (namespace=1, id=4471)
```

**Exact result**: **no mutual exclusion between the two modules.** They take locks in *different namespaces* (`objsubid=2` vs `objsubid=1` in `pg_locks`) and never conflict, so both run their critical section concurrently — the exact bug the lock was meant to prevent. **Why**: the single-`bigint` and `int4,int4` overloads are distinct lock spaces (Section 6.3).

```sql
-- RIGHT: one convention, enforced in a shared helper. Pick, e.g., two-int:
--   key1 = LOCK_NS_USER (=1), key2 = user_id
SELECT pg_advisory_lock(1, 4471);   -- used identically EVERYWHERE for users.
```

### Wrong 5: Session advisory lock through PgBouncer transaction pooling

```sql
-- WRONG (client connected via PgBouncer in transaction pooling mode):
BEGIN; SELECT pg_advisory_lock(99); COMMIT;   -- taken on server conn X
-- ... later ...
BEGIN; SELECT pg_advisory_unlock(99); COMMIT; -- may run on server conn Y (≠ X)
```

**Exact result**: the unlock frequently lands on a *different* server connection that never held 99 → returns FALSE + `WARNING: you don't own a lock of type ExclusiveLock`, while the lock **leaks** on connection X. **Why**: PgBouncer reassigns the server connection at each transaction boundary; session locks are pinned to a server connection, so acquire and release drift onto different backends (Section 6.7).

```sql
-- RIGHT: transaction-scoped lock — lifetime == one transaction == PgBouncer's unit.
BEGIN;
SELECT pg_advisory_xact_lock(99);
-- ... critical section entirely within this transaction ...
COMMIT;   -- released on the SAME server connection that took it. Pooling-safe.
```

---

## 10. Performance Profile

### Cost of a single acquisition

Acquiring or releasing an advisory lock is an **in-memory hash-table operation** in the shared lock manager, guarded by a partition LWLock. There is **no disk I/O, no WAL write, no tuple visibility check**. Typical uncontended acquisition is **single-digit microseconds** of server work (sub-millisecond end to end including the round trip). This is dramatically cheaper than a `SELECT ... FOR UPDATE`, which must locate the tuple, follow the update chain, and set lock bits on the heap page (a WAL-logged change).

| Operation | Server-side cost | I/O | WAL |
|-----------|-----------------|-----|-----|
| `pg_try_advisory_lock` (uncontended) | ~microseconds, hash insert | none | none |
| `pg_advisory_unlock` | ~microseconds, hash delete | none | none |
| `SELECT ... FOR UPDATE` on 1 row | index probe + heap fetch + lock-bit set | maybe | yes (row lock) |
| Row `UPDATE` | tuple version + index maintenance | yes | yes |

### Scaling with number of DISTINCT keys held

The constraint is the **shared lock table size**, fixed at server start:

```
total lock slots ≈ max_locks_per_transaction × (max_connections + max_prepared_transactions)
-- defaults: 64 × (100 + 0) = 6400 lock slots shared across the whole cluster
```

Advisory locks compete with table/row locks for these slots. Holding many distinct advisory keys at once (e.g. the "per-row lock explosion," Wrong 2) can exhaust the table:

```
ERROR: out of shared memory
HINT:  You might need to increase max_locks_per_transaction.
```

| Distinct advisory keys held simultaneously (fleet-wide) | Concern |
|--------------------------------------------------------|---------|
| 1 – 100 | Negligible. The overwhelming common case (one mutex per job/entity). |
| 100 – few thousand | Fine, but budget it against `max_locks_per_transaction`. |
| 10K+ | Risk of exhausting the shared lock table; redesign (batch, or use SKIP LOCKED). |
| Millions (one per row) | Anti-pattern. Never lock per-row with advisory locks. |

Note it is the count of **distinct keys held at one instant**, not the total ever taken, that matters. A job that takes one key, releases it, and takes another uses one slot at a time.

### Contention behaviour

- **Non-blocking (`try`)**: zero waiting — losers return FALSE instantly and consume no queue. This scales beautifully for the "one winner, many losers" cron pattern: 12 workers hit the same key, 1 gets TRUE in microseconds, 11 get FALSE in microseconds. No thundering herd, no queue buildup.
- **Blocking**: waiters queue in the lock manager (like any lock). A slow holder serialises everyone; a stuck holder hangs them until `lock_timeout` or disconnect. Always set `lock_timeout` for blocking acquisitions in production.

### At 1M / 10M / 100M rows

Advisory locks are **independent of table size** — they never touch the heap. Whether your `orders` table has 1M or 100M rows, a `pg_advisory_xact_lock(user_id)` costs the same microseconds. This is their headline performance advantage over `FOR UPDATE` for coordinating *logical operations*: the cost is a function of *how many locks you hold*, never *how big your tables are*. The one place table size matters is the *queue-draining* pattern (Section 7's LIMIT example), where the *scan* under the lock filter grows with the table — there, `FOR UPDATE SKIP LOCKED` is the size-independent choice.

### Optimization techniques specific to advisory locks

1. **Prefer `try` + `xact` for cron/leader patterns** — no queue, no leak, no timeout tuning.
2. **Namespace with two-int keys** so `pg_locks` is human-diagnosable and collisions across resource classes are impossible.
3. **Keep critical sections short** — the lock cost is trivial; the *work under the lock* is what serialises. Do slow external calls *outside* the lock where correctness allows.
4. **Never lock per-row**; if you're tempted, you want `FOR UPDATE SKIP LOCKED` (Topic 54).
5. **Alert on long-held locks** by joining `pg_locks` to `pg_stat_activity` and flagging `granted=true` advisory locks on backends idle for minutes.

---

## 11. Node.js Integration

### 11.1 The golden rule: one logical lock = one dedicated client

Because session-scoped advisory locks live on a *physical connection*, you must acquire and release on the **same** `client` — never on the pool directly. `pool.query()` may use a different connection each call, so a session lock taken via `pool.query()` and released via `pool.query()` can land on two different backends (the same class of bug as the PgBouncer pitfall, Section 6.7).

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// SESSION-scoped lock done correctly: check out ONE client, unlock in finally.
async function withSessionAdvisoryLock(key, fn) {
  const client = await pool.connect();          // dedicate a physical connection
  let acquired = false;
  try {
    const { rows } = await client.query(
      'SELECT pg_try_advisory_lock($1) AS ok',   // $1 param binding, never string-concat
      [key]
    );
    acquired = rows[0].ok;                        // boolean from pg
    if (!acquired) return { ran: false };         // someone else holds it — bow out
    const result = await fn(client);              // do work ON THE SAME client
    return { ran: true, result };
  } finally {
    if (acquired) {
      // Release on the SAME client that acquired it. Must run even on error.
      await client.query('SELECT pg_advisory_unlock($1)', [key]);
    }
    client.release();                             // return connection to pool AFTER unlock
  }
}
```

### 11.2 The safer default: transaction-scoped lock (no manual unlock, pooling-safe)

```javascript
// TRANSACTION-scoped: lock auto-releases at COMMIT/ROLLBACK. No leak path,
// safe even under PgBouncer transaction pooling.
async function withXactAdvisoryLock(key, fn) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    const { rows } = await client.query(
      'SELECT pg_try_advisory_xact_lock($1) AS ok',
      [key]
    );
    if (!rows[0].ok) {
      await client.query('ROLLBACK');             // releases nothing to leak
      return { ran: false };
    }
    const result = await fn(client);
    await client.query('COMMIT');                 // <-- lock released HERE
    return { ran: true, result };
  } catch (err) {
    await client.query('ROLLBACK');               // <-- or released HERE, on error
    throw err;
  } finally {
    client.release();
  }
}
```

### 11.3 Real cron-guard usage

```javascript
// Every one of the 12 workers calls this at 02:00. Exactly one runs the digest.
import crypto from 'node:crypto';

// Derive a stable bigint key from the job name (JS-side, avoids relying on
// server hashtext stability across versions). Use a signed 64-bit that fits int8.
function jobKey(name) {
  const h = crypto.createHash('sha1').update(name).digest();
  // take 8 bytes, interpret as signed BigInt in the int8 range
  return h.readBigInt64BE(0);
}

async function runNightlyDigest() {
  const key = jobKey('cron:nightly-digest:' + new Date().toISOString().slice(0, 10));
  const { ran } = await withXactAdvisoryLock(key.toString(), async (client) => {
    // ... all digest work here, using `client` for queries in the same tx ...
    await client.query('UPDATE ...');
  });
  if (!ran) {
    console.log('Another worker is running the digest; skipping.');
  }
}
```

Note the key is passed as a **string** (`key.toString()`) because `node-postgres` sends JS `BigInt` most safely as a text-encoded `bigint` parameter; Postgres coerces it to `int8` for the `pg_advisory_xact_lock(bigint)` overload.

### 11.4 Guarding a blocking acquisition with a timeout

```javascript
async function withBlockingLockTimeout(key, ms, fn) {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');
    await client.query(`SET LOCAL lock_timeout = '${Number(ms)}ms'`); // SET LOCAL = tx-scoped
    await client.query('SELECT pg_advisory_xact_lock($1)', [key]);    // may throw on timeout
    const result = await fn(client);
    await client.query('COMMIT');
    return result;
  } catch (err) {
    await client.query('ROLLBACK');
    if (err.code === '55P03') {          // lock_not_available
      throw new Error('Could not acquire advisory lock within timeout');
    }
    throw err;
  } finally {
    client.release();
  }
}
```

### 11.5 Do ORMs handle advisory locks?

No ORM models advisory locks as first-class entities — there is no table to map. Every ORM exposes them only through its **raw query** escape hatch (covered in Section 12). Because the correctness hinges on *which physical connection* runs the acquire and release, you must use the ORM's mechanism for pinning to a single connection/transaction (a transaction callback, or a manually-managed connection), exactly as with raw `pg`.

---

## 12. ORM Comparison

The universal truth: **no ORM has a native "advisory lock" abstraction.** They all reach it via raw SQL. What differs is how easy it is to pin acquire+release to one connection/transaction — which is the whole game for correctness.

### Prisma

**Can Prisma do advisory locks?** — Only via `$queryRaw` / `$executeRaw`. There is no model, no `.lock()`.

```typescript
import { PrismaClient } from '@prisma/client';
const prisma = new PrismaClient();

// SAFE pattern: interactive transaction pins all queries to ONE connection,
// and pg_advisory_xact_lock auto-releases when the transaction ends.
await prisma.$transaction(async (tx) => {
  const [{ ok }] = await tx.$queryRaw<{ ok: boolean }[]>`
    SELECT pg_try_advisory_xact_lock(${1}, ${4471}) AS ok
  `;
  if (!ok) return;                 // another holder — the tx will commit as a no-op
  // ... work using tx ...
});
// Lock released at end of $transaction. No leak path.
```

**Where it breaks**: a *session*-scoped lock via `prisma.$executeRaw` (outside `$transaction`) runs on a connection Prisma may not give back to you for the unlock → leak. Prisma's connection pool makes session locks unsafe unless you drop to a dedicated `pg` client. Also, `$transaction` interactive transactions have their own timeout; a long critical section can hit it.

**Verdict**: Use `pg_advisory_xact_lock` inside `prisma.$transaction`. Never use session-scoped advisory locks through Prisma's pool.

### Drizzle ORM

**Can Drizzle do advisory locks?** — Via `sql` raw execution, on a transaction or a dedicated connection.

```typescript
import { sql } from 'drizzle-orm';
import { db } from './db';

await db.transaction(async (tx) => {
  const res = await tx.execute(
    sql`SELECT pg_try_advisory_xact_lock(${1}, ${4471}) AS ok`
  );
  if (!res.rows[0].ok) return;     // bow out; tx-scoped lock, auto-released
  // ... work using tx ...
});
```

**Where it breaks**: same as everyone — session-scoped locks need a pinned connection. Drizzle's `db.transaction` pins correctly, so stick to `xact` scope. Drizzle does not validate that your key is a valid `int4`/`int8`; passing a JS number > 2^31 into the two-int overload silently misbehaves.

**Verdict**: Clean with `pg_advisory_xact_lock` inside `db.transaction`. Drizzle's thin SQL layer makes the intent obvious.

### Sequelize

**Can Sequelize do advisory locks?** — Only via `sequelize.query`. (Sequelize's `transaction({ lock })` option is for `FOR UPDATE`/`FOR SHARE` row locks, *not* advisory locks.)

```javascript
const { QueryTypes } = require('sequelize');

await sequelize.transaction(async (t) => {
  const [{ ok }] = await sequelize.query(
    'SELECT pg_try_advisory_xact_lock(:k1, :k2) AS ok',
    { replacements: { k1: 1, k2: 4471 }, type: QueryTypes.SELECT, transaction: t }
  );
  if (!ok) return;                 // must pass `transaction: t` or it runs on another conn!
  // ... work, each query passing { transaction: t } ...
});
```

**Where it breaks**: the classic Sequelize bug — forgetting `{ transaction: t }` on the `query` call. Without it, the lock query runs on a *different* pooled connection than the transaction, so the `xact` lock is on the wrong connection and releases at that stray query's implicit commit, not your transaction's. Every query in the critical section must carry `transaction: t`.

**Verdict**: Works, but you must thread `transaction: t` through *every* call. Easy to get subtly wrong.

### TypeORM

**Can TypeORM do advisory locks?** — Via `queryRunner.query` on a manually managed `QueryRunner` (which pins one connection), or inside `dataSource.transaction`.

```typescript
const queryRunner = dataSource.createQueryRunner();
await queryRunner.connect();                 // pins ONE physical connection
await queryRunner.startTransaction();
try {
  const [{ ok }] = await queryRunner.query(
    'SELECT pg_try_advisory_xact_lock($1, $2) AS ok', [1, 4471]
  );
  if (!ok) { await queryRunner.rollbackTransaction(); return; }
  // ... work via queryRunner.query ...
  await queryRunner.commitTransaction();     // releases the xact lock
} catch (e) {
  await queryRunner.rollbackTransaction();
  throw e;
} finally {
  await queryRunner.release();               // return connection to pool AFTER unlock
}
```

**Where it breaks**: using `dataSource.query()` (not the query runner) for a session lock runs on an arbitrary pool connection → leak. TypeORM's `setLock('pessimistic_write')` is row-level `FOR UPDATE`, unrelated to advisory locks.

**Verdict**: Use a `QueryRunner` (pins the connection) with `pg_advisory_xact_lock`. It is the most explicit ORM here about connection pinning, which suits advisory locks well.

### Knex.js

**Can Knex do advisory locks?** — Via `knex.raw`, ideally inside `knex.transaction` (which pins a connection).

```javascript
await knex.transaction(async (trx) => {
  const { rows } = await trx.raw(
    'SELECT pg_try_advisory_xact_lock(?, ?) AS ok', [1, 4471]
  );
  if (!rows[0].ok) return;          // tx-scoped, auto-released at trx end
  // ... work via trx ...
});

// Session-scoped variant needs a dedicated connection:
const conn = await knex.client.acquireConnection();
try {
  await knex.raw('SELECT pg_advisory_lock(?)', [4471]).connection(conn);
  // ... work, each query .connection(conn) ...
} finally {
  await knex.raw('SELECT pg_advisory_unlock(?)', [4471]).connection(conn);
  await knex.client.releaseConnection(conn);
}
```

**Where it breaks**: a bare `knex.raw('SELECT pg_advisory_lock(...)')` uses a pool connection you don't control → leak. `.connection(conn)` is the escape hatch to pin it.

**Verdict**: Clean with `knex.transaction` + `pg_advisory_xact_lock`. For session locks, `.connection(conn)` pinning is mandatory.

### ORM Summary Table

| ORM | Advisory lock via | Connection pinning mechanism | Safe pattern | Main footgun |
|-----|-------------------|------------------------------|--------------|--------------|
| Prisma | `$queryRaw` | `$transaction` interactive tx | `pg_advisory_xact_lock` in `$transaction` | Session lock on pool → leak; tx timeout |
| Drizzle | `sql` + `.execute` | `db.transaction` | `xact` lock in `db.transaction` | No key-range validation |
| Sequelize | `sequelize.query` | pass `{ transaction: t }` everywhere | `xact` + `transaction: t` on every query | Forgetting `transaction: t` |
| TypeORM | `queryRunner.query` | `QueryRunner` pins connection | `xact` lock via `QueryRunner` | `dataSource.query()` session lock → leak |
| Knex | `knex.raw` | `knex.transaction` / `.connection()` | `xact` lock in `knex.transaction` | Bare `raw` session lock → leak |

**The one-line takeaway for every ORM**: use the **transaction-scoped** advisory lock inside the ORM's **transaction** helper. That single choice sidesteps connection-pinning and leak bugs across all of them.

---

## 13. Practice Exercises

### Exercise 1 — Basic

You run 6 identical worker processes. Each, on startup, should attempt to become the sole "metrics flusher." Only one should win; the rest should continue without flushing. Write the single SQL statement each worker runs to attempt acquisition, using a **session-scoped, non-blocking** exclusive advisory lock on the key derived from the string `'role:metrics-flusher'`. Then write the statement the winner runs at shutdown to release it.

```sql
-- Write your query here
```

### Exercise 2 — Medium (combines Topic 54 and Topic 55)

You are draining a `jobs(id, status, created_at)` queue with 12 workers. You want each worker to pick up the **oldest pending job that no other worker is currently processing**, process it, and mark it done — with no two workers ever processing the same job. 

1. Write the claim query using `FOR UPDATE SKIP LOCKED` (Topic 54).
2. Then explain, in comments, why an advisory-lock-in-`WHERE` approach (`WHERE pg_try_advisory_xact_lock(id)`) would attempt locks on more rows than SKIP LOCKED does, referencing the evaluation-order issue from Section 3.
3. Note what lock-ordering rule (Topic 55) you would need if a worker had to lock *two* jobs at once by advisory key.

```sql
-- Write your query here
```

### Exercise 3 — Hard (production simulation; naive answer is wrong)

A fleet of 8 Node.js instances runs behind PgBouncer configured in **transaction pooling** mode. A developer wrote this to ensure "reindex-search-catalog" runs on only one instance:

```sql
SELECT pg_advisory_lock(hashtext('reindex-search-catalog'));   -- at job start
-- ... long reindex ...
SELECT pg_advisory_unlock(hashtext('reindex-search-catalog')); -- at job end
```

In production, sometimes the reindex runs on two instances at once, and `pg_locks` accumulates stuck advisory locks on idle backends.

1. Explain the two independent bugs (scope choice, and PgBouncer mode) that cause this.
2. Rewrite it correctly so it is safe under PgBouncer transaction pooling.
3. The reindex takes 20 minutes and must not be re-run the same day even after a failover. Advisory locks are not durable — add the durable guard. What table and index do you add, and what is the acquire logic?

```sql
-- Write your query here
```

### Exercise 4 — Interview Simulation

You are asked live: "We have `users` and a `recompute_credit_score(user_id)` operation that reads 4 tables, calls an external bureau API (up to 8 seconds), and writes one row back. Two concurrent requests for the same user must never run it simultaneously, but different users must run fully in parallel, and we run 20 app instances. Design the locking. Justify why you did *not* use `SELECT ... FOR UPDATE` on the user row."

Write the SQL and a one-paragraph justification.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1: What is an advisory lock, and how does it differ from a row lock?

**Junior answer**: "It's a lock you create manually. You lock some number and other people can't use it."

**Principal answer**: An advisory lock is an application-defined lock identified by a 64-bit key that *you* choose, tracked in Postgres's shared lock manager but **not tied to any table, row, or database object**. Unlike a row lock — which the engine takes automatically when you `UPDATE` or `SELECT ... FOR UPDATE`, records in the tuple header via MVCC, WAL-logs, and *enforces* (a conflicting writer physically blocks) — an advisory lock touches no heap tuple, writes no WAL, and is **purely cooperative**: nothing stops a session that never called the lock function from touching the "protected" data. The database *advises*; it does not *enforce*. That makes them ideal for serialising *logical operations or actions* that have no natural row to lock (cron jobs, leader election, per-entity workflows), and much cheaper than a row lock (an in-memory hash insert vs a WAL-logged heap change).

**Interviewer follow-up**: "If they're not enforced, why are they useful — couldn't a rogue query ignore them?" → Yes, correctness depends on every participant honouring the convention; they coordinate *your own* cooperating processes, not defend against arbitrary clients. Their value is a fast, cluster-wide mutex over a namespace *you* define, sharing the DB you already trust as the source of truth.

### Q2: Session vs transaction scope — when do you pick which, and what goes wrong?

**Junior answer**: "Session lasts longer than transaction. I'd use session so it doesn't disappear."

**Principal answer**: Transaction-scoped (`pg_advisory_xact_lock`) auto-releases at `COMMIT`/`ROLLBACK` and cannot be released manually — there is **no leak path**, which is why it's the safe default. Session-scoped (`pg_advisory_lock`) is owned by the backend, **ignores transaction boundaries** (survives rollback!), is reference-counted, and *must* be explicitly unlocked or it lives until disconnect. I default to **transaction scope** and only reach for session scope when the critical section genuinely must span multiple transactions on one connection. The thing that "goes wrong" with session scope is leaks: an exception skipping the unlock, or — the killer — a **connection pool** returning the still-locked physical connection to another request, which now silently holds a lock it never took.

**Interviewer follow-up**: "You said session survives rollback — walk me through what a developer expects vs what happens." → They `BEGIN; pg_advisory_lock(42); ROLLBACK;` and assume 42 is free; it is not — it's owned by the session, so it stays held until an explicit `pg_advisory_unlock(42)` or disconnect. The `xact` variant is the one that respects rollback.

### Q3: You deploy behind PgBouncer and advisory locks start behaving erratically — sometimes two workers run the "single-instance" job, and `pg_locks` fills with stuck locks. What's happening?

**Junior answer**: "Maybe PgBouncer has a bug, or the lock timeout is too short."

**Principal answer**: This is the connection-pooling pitfall. In PgBouncer **transaction pooling** mode, the server connection is reassigned at every transaction boundary. A **session-scoped** advisory lock is pinned to a *server connection*, so the acquire lands on connection X, and the later unlock — running in a different transaction — may execute on connection Y, which never held the lock (returns FALSE + WARNING) while the lock **leaks** on X. Meanwhile a *different* client that inherits X now silently holds the lock. The fix is to use **transaction-scoped** advisory locks: their lifetime is exactly one transaction, which is exactly PgBouncer's multiplexing unit, so acquire and release always happen on the same server connection and the lock cannot outlive its transaction. Alternatively, use PgBouncer *session* pooling for connections that take session locks, or a dedicated non-pooled connection.

**Interviewer follow-up**: "And with just `pg.Pool` app-side, no PgBouncer — are session locks safe?" → Only if you check out one `client` and do acquire+work+release+`finally` on that *same* client, never via `pool.query()` which can pick a different connection per call. Even then, an unlock skipped by an error path leaks the lock into the pool. Transaction scope removes the whole class of bug.

### Q4: How do advisory locks interact with the deadlock detector?

**Junior answer**: "Advisory locks avoid deadlocks because you control them."

**Principal answer**: The opposite — they *participate* in deadlock detection because they live in the same shared lock table as table and row locks. Two sessions that each hold one advisory key and block waiting for the other's will be detected after `deadlock_timeout` and one is aborted with `deadlock detected`. You can also deadlock *across* lock types: session A holds advisory(1) and waits on a row B locked, while B waits on advisory(1). The mitigation is identical to Topic 55: acquire multiple advisory locks in a **consistent global order** (e.g. ascending key), and be careful mixing advisory and row locks. Advisory locks are not an escape from deadlocks; they are one more lock to order.

**Interviewer follow-up**: "How would you make a multi-key advisory acquisition deadlock-free?" → Sort the keys and always acquire in ascending order across all code paths, or acquire them non-blocking (`try`) and back off/retry if any fails, releasing what you got.

---

## 15. Mental Model Checkpoint

1. A session runs `BEGIN; SELECT pg_advisory_lock(7); ROLLBACK;`. Is lock 7 still held afterward? Why? What if it had used `pg_advisory_xact_lock(7)`?

2. You call `pg_advisory_lock(0, 1)` in one session and `pg_advisory_lock(1)` in another. Do they conflict? Explain in terms of the lock namespaces.

3. A worker does `SELECT id, pg_advisory_lock(id) FROM jobs WHERE status='pending'` and there are 900 pending jobs. How many advisory locks does it now hold, and what shared-memory risk does that create?

4. Under PgBouncer transaction pooling, why is `pg_advisory_xact_lock` safe while `pg_advisory_lock` is not? Trace which server connection the acquire and the release land on in each case.

5. Two sessions each hold one advisory lock and each waits for the other's. What does Postgres do, and roughly how long until it does it? What config controls that delay?

6. You need "run this migration exactly once, ever, across the fleet, and it must stay run even after a failover." Are advisory locks alone sufficient? If not, what do you add and why?

7. A session-scoped lock was acquired twice on key 42 in the same session. How many `pg_advisory_unlock(42)` calls are required to fully release it, and what does the third call return?

---

## 16. Quick Reference Card

```sql
-- ACQUIRE (exclusive)
SELECT pg_advisory_lock(key);           -- session, blocking, returns void
SELECT pg_try_advisory_lock(key);       -- session, non-blocking, returns bool
SELECT pg_advisory_xact_lock(key);      -- transaction, blocking (auto-release @ commit)
SELECT pg_try_advisory_xact_lock(key);  -- transaction, non-blocking, returns bool

-- SHARED (reader) variants: append _shared to any of the above
SELECT pg_advisory_lock_shared(key);
SELECT pg_advisory_xact_lock_shared(key);

-- RELEASE (session-scoped only; xact locks release automatically)
SELECT pg_advisory_unlock(key);         -- returns bool (FALSE+WARNING if not held)
SELECT pg_advisory_unlock_all();        -- release ALL session advisory locks

-- KEY FORMS (DIFFERENT namespaces — never mix for the same resource!)
pg_advisory_lock(k bigint)              -- single 64-bit key
pg_advisory_lock(k1 int4, k2 int4)      -- two 32-bit keys = ONE lock (namespaced)

-- DERIVE KEY FROM STRING
SELECT hashtext('name');                       -- int4 (signed, negatives are fine)
SELECT hashtextextended('name', 0);            -- int8 (prefer for bigint key form)

-- INSPECT
SELECT * FROM pg_locks WHERE locktype = 'advisory';
--   objsubid = 1 -> two-int form (classid=k1, objid=k2)
--   objsubid = 2 -> single-bigint form (key = classid<<32 | objid)
--   granted = false rows are WAITERS (contention on that key)

-- GUARD A BLOCKING ACQUIRE
SET lock_timeout = '5s';  -- applies to advisory waits too

-- ─────────────── RULES OF THUMB ───────────────
-- • Default to pg_advisory_xact_lock: no leak path, pooling-safe.
-- • Use pg_TRY_* for "one winner, many losers" (cron/leader) — losers exit instantly.
-- • Session locks: acquire + release on the SAME client, unlock in finally.
-- • NEVER put a lock function in SELECT/WHERE over many rows -> lock flood.
-- • Pick ONE key convention (1-key OR 2-key) codebase-wide.
-- • Advisory locks are NOT durable — pair with a marker row for exactly-once-ever.
-- • Multi-key acquire: always ascending key order (deadlock-safe, Topic 55).
-- • Behind PgBouncer transaction pooling: xact-scope ONLY.

-- ─────────────── INTERVIEW ONE-LINERS ───────────────
-- "Advisory = a mutex over a number YOU define; the DB advises, doesn't enforce."
-- "xact-scope releases at commit/rollback and can't leak; session-scope can."
-- "Session locks + connection pools = leaked locks; prefer xact-scope."
-- "Rollback does NOT release a session advisory lock — it's session-owned."
-- "The 1-key and 2-key forms are different namespaces; never mix them."
-- "They're not durable — no WAL, gone on failover; add a marker row for once-ever."
-- "SKIP LOCKED for queue draining; advisory for coordinating actions with no row."
```

---

## Connected Topics

**Internals this builds on:**
- **Topic 51 — Row-Level Locking and MVCC**: the contrast that defines advisory locks — row locks live in the tuple header and are WAL-logged; advisory locks touch no heap and no WAL.
- **The shared lock manager / `pg_locks`**: advisory locks share the same fixed-size shared lock table as table and row locks, sized by `max_locks_per_transaction`.

**Prior SQL topics:**
- **Topic 54 — SELECT ... FOR UPDATE / SKIP LOCKED**: the *enforced* row-lock alternative; `SKIP LOCKED` is the preferred primitive for queue draining, where advisory-lock-in-WHERE over-acquires.
- **Topic 55 — Deadlocks**: advisory locks participate in the deadlock detector; the same consistent-ordering discipline applies to multi-key advisory acquisition.
- **Topic 53 — Transaction Isolation Levels**: transaction-scoped advisory locks are bound to the same transaction lifecycle whose isolation you configure there.

**Next SQL topic:**
- **Topic 57 — Transaction Patterns in Node.js**: where the `withXactAdvisoryLock` helper, `finally`-based cleanup, and connection-pinning patterns from Section 11 become the backbone of production transaction management with `pg`, Prisma, and Knex.
