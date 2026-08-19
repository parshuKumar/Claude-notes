# 45 — Locks in Depth
## Phase: Transactions & Concurrency

---

## ELI5 — The Simple Analogy

A shared office with meeting rooms and a whiteboard.

**Row locks are booking a specific desk.** You put your name on desk 14. Anyone else who wants desk 14 waits. Nobody who wants desk 15 cares.

**Table locks are booking the whole floor** — and they come in *strengths*. "I'm reading in here, others may read too" is compatible with another reader. "I'm repainting the walls" is compatible with nobody.

**The key insight most people miss:** two people both saying *"I'm just reading"* never conflict. But *"I'm repainting"* conflicts with everyone, including the readers. So the interesting question is never "is this locked?" — it's **"which two lock strengths are incompatible, and did I take a stronger one than I needed?"**

And the trap: **the queue is FIFO.** If a painter is waiting for the readers to finish, every new reader queues *behind the painter*. One `ALTER TABLE` waiting on one long `SELECT` can stall an entire application — not because the `ALTER` is slow, but because everyone lines up behind it.

---

## Where this fits in the big picture

```
   43 anomalies · 44 isolation levels (the WHAT)
                          │
          ┌───────────────┴────────────────┐
          ▼                                ▼
 ┌──────────────────────────┐   ┌────────────────────────┐
 │ 45 LOCKS ← YOU ARE HERE  │   │ 46 MVCC                │
 │ how WRITES are serialised│   │ how READS avoid locking│
 └────────────┬─────────────┘   └───────────┬────────────┘
              └───────────┬──────────────────┘
                          ▼
          48 deadlocks · 49 optimistic vs pessimistic
          28 zero-downtime migrations (the lock queue)
```

Topic 44 told you *which anomalies* each level prevents. **This topic is one of the two mechanisms that do the preventing** — the other is MVCC (Topic 46). Locks handle writers. MVCC handles readers.

---

## What is this?

A lock is a **reservation on an object, held for the duration of a transaction**, that makes some other operations wait.

PostgreSQL has three families, and confusing them is the source of most lock confusion:

| Family | Granularity | Held until | Conflicts with |
|---|---|---|---|
| **Table locks** | a relation | end of transaction | other table locks, by a compatibility matrix |
| **Row locks** | one tuple | end of transaction | other row locks on *the same row* |
| **Advisory locks** | an arbitrary `bigint` you choose | session or transaction | whatever you decide it means |

Plus internal ones you'll see in wait events but never take yourself: **page locks**, **`LWLock`s** (buffer pool internals), and **`SIReadLock`s** (SSI predicate markers — not really locks, Topic 44).

**The one rule that makes all of it make sense:**

> **Readers do not block writers. Writers do not block readers.** That is MVCC's guarantee (Topic 46). Locks exist for **writer-vs-writer** and for **DDL-vs-everything**.

---

## Why does it matter for a backend developer?

Three reasons, in increasing order of how much damage they cause:

```
 ① YOU CANNOT FIX A "THE APP IS FROZEN" INCIDENT WITHOUT THEM
    Every stall of the form "everything is slow but CPU is idle" is a
    lock queue. The diagnosis is one query — if you know it.

 ② ★ THE LOCK QUEUE IS FIFO, AND THIS SURPRISES EVERYONE
    A blocked ALTER TABLE does not just wait. It BLOCKS EVERY
    SUBSEQUENT QUERY on that table, including plain SELECTs that
    would never have conflicted with the running transaction.
    ⇒ one 40-minute analytics SELECT + one ALTER TABLE
      = total outage on that table. (Topic 28.)

 ③ SELECT … FOR UPDATE IS THE MOST OVER-USED TOOL IN THE LANGUAGE
    It is the correct fix for a lost update. It is also how you turn
    a 18,000 tps endpoint into a 400 tps endpoint by serialising
    everything on one row.
```

---

## The physical reality

### Where a lock actually lives

```
 ★ TABLE LOCKS live in SHARED MEMORY — a fixed-size hash table.
     size ≈ max_locks_per_transaction (64)
          × (max_connections + max_prepared_transactions)
     ⇒ exceed it and you get:
       ERROR: out of shared memory
       HINT: You might need to increase max_locks_per_transaction.
     ⇒ ★ the classic cause: a query touching 5,000 partitions.
       Each partition needs its own lock entry. (Topic 59.)

 ★ ROW LOCKS DO NOT LIVE IN SHARED MEMORY. THIS IS THE KEY FACT.
     A row lock is written INTO THE TUPLE ITSELF:
       t_xmax        = the locking transaction's XID
       t_infomask    = flags saying "this is a lock, not a delete"
                       (HEAP_XMAX_LOCK_ONLY, HEAP_XMAX_EXCL_LOCK, …)
     ⇒ CONSEQUENCE ①: you can lock 50 MILLION ROWS with no memory
       cost. There is no row-lock table to overflow.
     ⇒ CONSEQUENCE ②: ★ TAKING A ROW LOCK DIRTIES THE PAGE.
       SELECT … FOR UPDATE writes to the heap. It generates WAL.
       A "read" query that produces WAL and dirty buffers.
     ⇒ CONSEQUENCE ③: to WAIT for a row lock, you must first find
       out who holds it — so a lightweight in-memory "tuple lock"
       is taken only while queuing.
```

### The table-lock compatibility matrix

```
 ★ EIGHT MODES. You need to recognise four of them by name.

  Mode                     Taken by                        Conflicts with
  ──────────────────────── ─────────────────────────────── ───────────────
  ACCESS SHARE             SELECT                          only ACCESS EXCLUSIVE
  ROW SHARE                SELECT FOR UPDATE/SHARE         EXCLUSIVE, ACCESS EXCL
  ROW EXCLUSIVE            INSERT, UPDATE, DELETE          SHARE and above
  SHARE UPDATE EXCLUSIVE   VACUUM, ANALYZE,                itself and above
                           CREATE INDEX CONCURRENTLY,
                           ★ ALTER TABLE … ADD COLUMN (PG11+)
  SHARE                    CREATE INDEX (non-concurrent)   ROW EXCL and above
  SHARE ROW EXCLUSIVE      CREATE TRIGGER                  ROW EXCL and above
  EXCLUSIVE                REFRESH MATVIEW CONCURRENTLY    everything but ACCESS SHARE
  ACCESS EXCLUSIVE         ★ ALTER TABLE (most forms),     ★ EVERYTHING
                           DROP, TRUNCATE, VACUUM FULL,
                           REINDEX (non-concurrent)

 ⇒ ★ THE ONLY TWO ROWS THAT MATTER DAY TO DAY:
   • ACCESS SHARE (every SELECT) conflicts with EXACTLY ONE mode:
     ACCESS EXCLUSIVE.
   • ACCESS EXCLUSIVE conflicts with EVERYTHING, including SELECT.
 ⇒ THEREFORE: any DDL taking ACCESS EXCLUSIVE stops all reads.
   The whole art of zero-downtime migration (Topic 28) is
   ① avoiding ACCESS EXCLUSIVE, or
   ② holding it for microseconds and never queuing behind anything.
```

### The FIFO queue — the thing that causes outages

```
 t=0    txn A: BEGIN; SELECT … FROM orders;      ← ACCESS SHARE, granted
                (an analytics query, runs 40 minutes)

 t=10s  txn B: ALTER TABLE orders ADD CONSTRAINT … ;
                ← wants ACCESS EXCLUSIVE
                ← ACCESS EXCLUSIVE conflicts with ACCESS SHARE
                ← ⏸ B WAITS

 t=11s  txn C: SELECT * FROM orders WHERE id=1;
                ← wants ACCESS SHARE
                ← ★ ACCESS SHARE is COMPATIBLE with A's ACCESS SHARE!
                ← ★ BUT B IS AHEAD OF C IN THE QUEUE, AND B CONFLICTS
                    WITH C.
                ← ⏸ ★ C WAITS. FOR 40 MINUTES.

 t=12s… every subsequent query on `orders` queues behind B.

 ⇒ ★ THE OUTAGE IS NOT CAUSED BY THE ALTER. It is caused by the
   ALTER *WAITING*. PostgreSQL does not let later-arriving
   compatible requests jump the queue, because that would starve
   the waiter forever.

 ⇒ THE FIX, ALWAYS:
     SET lock_timeout = '2s';
     ALTER TABLE orders ADD CONSTRAINT … ;
   ⇒ if it can't get the lock in 2 seconds, it FAILS instead of
     building a queue. Retry in a loop. (Topic 28.)
```

### The four row-lock strengths

```
 ★ FOUR STRENGTHS, WEAKEST TO STRONGEST:

   FOR KEY SHARE      "I depend on this row's key existing."
                      ⇒ ★ THIS IS WHAT A FOREIGN KEY TAKES on the
                        PARENT row when you INSERT a child.
                      ⇒ conflicts only with FOR UPDATE.
   FOR SHARE          "Nobody may change this row while I read it."
                      ⇒ conflicts with FOR NO KEY UPDATE and above.
   FOR NO KEY UPDATE  "I will UPDATE non-key columns."
                      ⇒ ★ WHAT A PLAIN UPDATE TAKES if it doesn't
                        touch any column in a unique index.
   FOR UPDATE         "I will UPDATE the key, or DELETE this row."
                      ⇒ conflicts with everything, including KEY SHARE.

 COMPATIBILITY:
                       KEY SHARE  SHARE  NO KEY UPD  UPDATE
   FOR KEY SHARE          ✓         ✓        ✓         ✗
   FOR SHARE              ✓         ✓        ✗         ✗
   FOR NO KEY UPDATE      ✓         ✗        ✗         ✗
   FOR UPDATE             ✗         ✗        ✗         ✗

 ★ WHY THIS EXISTS — THE FK STORY:
   Before PostgreSQL 9.3, inserting an order took FOR SHARE on the
   customer row. Two concurrent orders for the same customer? Fine.
   But an UPDATE of that customer's email blocked behind them.
   ⇒ KEY SHARE fixed it: the FK only cares that the key EXISTS, so
     it conflicts only with FOR UPDATE (which might delete or
     re-key the row).
   ⇒ ★ THIS IS WHY `SELECT … FOR UPDATE` ON A PARENT ROW IS
     EXPENSIVE: it blocks every child INSERT. Use FOR NO KEY UPDATE
     if you're only updating non-key columns.
```

---

## How it works — step by step

### Taking a row lock

```sql
BEGIN;
SELECT balance_minor FROM accounts WHERE id = 1 FOR UPDATE;
```

```
 ① plan the query, find the tuple (via the PK index)
 ② read the heap page into shared buffers
 ③ check t_xmax:
      • 0 or aborted        ⇒ free, proceed
      • a live transaction  ⇒ ⏸ take a lightweight tuple lock,
                              XactLockTableWait(that xid), then retry
 ④ write into the tuple, in place:
      t_xmax     ← my XID
      t_infomask ← HEAP_XMAX_EXCL_LOCK | HEAP_XMAX_LOCK_ONLY
 ⑤ ★ mark the buffer DIRTY and emit a WAL record (XLOG_HEAP_LOCK)
 ⑥ the lock is now held until COMMIT or ROLLBACK — ★ there is no
   UNLOCK statement. Locks are released only at transaction end.
```

```
 ★ MULTIPLE LOCKERS ON ONE ROW: t_xmax is ONE field. How do two
   transactions both hold FOR KEY SHARE on the same row?
   ⇒ A MULTIXACT. t_xmax stores a MultiXactId instead of an XID,
     and the flag HEAP_XMAX_IS_MULTI is set. The MultiXactId indexes
     into pg_multixact, which lists the member XIDs and their modes.
   ⇒ COST: multixacts are a separate SLRU that must also be vacuumed
     and frozen. A workload with heavy FK-parent contention can hit
     ★ "multixact members limit exceeded" — a real, rare, and very
     confusing production failure. (Topic 47.)
```

### The five variants of `FOR UPDATE`

```sql
-- ① plain — wait indefinitely
SELECT … FOR UPDATE;

-- ② NOWAIT — fail immediately if locked
SELECT … FOR UPDATE NOWAIT;
-- ERROR: could not obtain lock on row in relation "accounts"  (55P03)
-- ⇒ use when "someone else is doing this" is a valid answer.

-- ③ ★ SKIP LOCKED — skip rows others hold. THE QUEUE PATTERN.
SELECT … FROM job_queue WHERE status='pending'
 ORDER BY created_at LIMIT 10 FOR UPDATE SKIP LOCKED;
-- ⇒ N workers each get a DIFFERENT 10 jobs, with zero contention
--   and zero coordination. (Case study 09.)

-- ④ FOR NO KEY UPDATE — weaker; doesn't block child INSERTs
SELECT … FOR NO KEY UPDATE;
-- ⇒ ★ use this instead of FOR UPDATE when you will not change a
--   key column. It's strictly cheaper.

-- ⑤ FOR SHARE — "nobody may change this while I read it"
SELECT … FOR SHARE;
-- ⇒ rarely correct. Two readers both take FOR SHARE, both then try
--   to UPDATE ⇒ ★ GUARANTEED DEADLOCK. (Topic 48.)
```

### Advisory locks — locking things that aren't rows

```sql
-- transaction-scoped (★ prefer this — auto-released at COMMIT)
SELECT pg_advisory_xact_lock(hashtext('import:catalogue:2026-08'));
SELECT pg_try_advisory_xact_lock(12345);        -- non-blocking, returns bool

-- session-scoped (★ leaks across a connection pool — avoid)
SELECT pg_advisory_lock(12345);
SELECT pg_advisory_unlock(12345);

SELECT * FROM pg_locks WHERE locktype='advisory';
```

```
 ★ WHAT THEY'RE FOR: mutual exclusion over something with no row.
   • "only one instance of this cron job runs at a time"
   • "serialise the nightly import"
   • "one leader among N app servers"
 ★ WHAT THEY'RE NOT FOR: replacing row locks or constraints. An
   advisory lock is a convention — nothing enforces that everyone
   who touches the data takes it. A UNIQUE constraint cannot be
   forgotten.
```

---

## Concept breakdown

```
THREE FAMILIES
├── TABLE LOCKS       shared memory, 8 modes, compatibility matrix
│    ★ ACCESS SHARE (SELECT) conflicts with ONLY ACCESS EXCLUSIVE
│    ★ ACCESS EXCLUSIVE (most DDL) conflicts with EVERYTHING
├── ROW LOCKS         ★ stored IN THE TUPLE (t_xmax + infomask)
│    ⇒ unlimited count, no memory cost
│    ⇒ ★ but they DIRTY THE PAGE and generate WAL
│    4 strengths: KEY SHARE < SHARE < NO KEY UPDATE < UPDATE
└── ADVISORY LOCKS    an arbitrary bigint; meaning is yours
     ★ use the _xact_ variants; session ones leak across a pool

★ THE FIFO QUEUE — the #1 cause of "the app froze"
   a waiter BLOCKS everything behind it, even compatible requests
   ⇒ ALWAYS: SET lock_timeout before DDL

WHAT MVCC MEANS FOR LOCKS (Topic 46)
   readers never block writers · writers never block readers
   ⇒ locks exist for WRITER-vs-WRITER and DDL-vs-EVERYTHING

THE FIVE FOR UPDATE VARIANTS
├── FOR UPDATE                blocks; strongest; blocks child INSERTs
├── FOR UPDATE NOWAIT         fail fast (55P03)
├── ★ FOR UPDATE SKIP LOCKED  the job-queue pattern; zero contention
├── ★ FOR NO KEY UPDATE       weaker; use when not changing a key
└── FOR SHARE                 ★ usually a deadlock waiting to happen

THE THREE DIAGNOSTIC FACTS
├── pg_blocking_pids(pid)     who is blocking whom  ← start here
├── pg_locks + pg_stat_activity   what mode, on what, for how long
└── ★ granted=false in pg_locks   is the queue

LIMITS YOU CAN HIT
├── max_locks_per_transaction (64) × connections  ⇒ table-lock slots
│    ★ blown by queries touching thousands of partitions
└── multixact members  ⇒ heavy FK-parent contention
```

---

## Diagrams

**Diagram 1 — big picture: what conflicts with what**

```
                    THE ONLY MATRIX YOU MUST MEMORISE

                      │ SELECT │ INSERT/  │ VACUUM/ │  ALTER  │
                      │        │ UPDATE   │ CREATE  │  TABLE  │
                      │(ACCESS │ (ROW     │ INDEX   │ (ACCESS │
                      │ SHARE) │ EXCL)    │ CONCUR. │  EXCL)  │
                      │        │          │ (SHARE  │         │
                      │        │          │ UPD EX) │         │
 ─────────────────────┼────────┼──────────┼─────────┼─────────┤
  SELECT              │   ✓    │    ✓     │    ✓    │  ★ ✗    │
  INSERT/UPDATE/DELETE│   ✓    │    ✓     │    ✓    │  ★ ✗    │
  VACUUM / CIC        │   ✓    │    ✓     │    ✗    │  ★ ✗    │
  ALTER TABLE         │ ★ ✗    │  ★ ✗     │  ★ ✗    │  ★ ✗    │
 ─────────────────────┴────────┴──────────┴─────────┴─────────┘
   ✓ = compatible, both proceed    ✗ = one waits

 ★ READ THE LAST ROW AND THE LAST COLUMN. Everything else is
   compatible. Almost all lock pain comes from ACCESS EXCLUSIVE.

 ⇒ AND THE ROW LOCKS, SEPARATELY (same row only):
      KEY SHARE ✓ SHARE ✓ NO KEY UPDATE  |  FOR UPDATE ✗ all
      ★ a plain UPDATE takes FOR NO KEY UPDATE when it doesn't
        touch a unique-index column ⇒ child INSERTs still proceed
```

**Diagram 2 — data flow: the FIFO queue outage**

```
 TIME  QUEUE ON TABLE `orders`                       STATE
 ────  ─────────────────────────────────────────────  ──────────────
 t=0   [A: ACCESS SHARE ✓granted]                     A running
       ← a 40-minute analytics SELECT

 t=10s [A: ACCESS SHARE ✓] [B: ACCESS EXCLUSIVE ⏸]    B waiting
       ← ALTER TABLE. Conflicts with A. Queues.

 t=11s [A ✓] [B ⏸] [C: ACCESS SHARE ⏸]                ★ C waiting
       ← a plain SELECT.
       ★ C IS COMPATIBLE WITH A. It is NOT compatible with B.
       ★ PostgreSQL will NOT let C jump ahead of B —
         that would starve B forever.
       ⇒ C WAITS 40 MINUTES FOR A QUERY IT DOESN'T CONFLICT WITH.

 t=12s [A ✓] [B ⏸] [C ⏸] [D ⏸] [E ⏸] … [×2,000 ⏸]
       ⇒ ★ connection pool exhausted. Total outage on `orders`.

 ─────────────────────────────────────────────────────────────────
 ✓ THE FIX
       SET lock_timeout = '2s';
       ALTER TABLE orders …;
 t=10s [A ✓] [B ⏸]
 t=12s [A ✓]                    ← B fails: 55P03 lock_not_available
       ⇒ ★ NO QUEUE EVER FORMS. Retry B in a loop until A finishes.
```

**Diagram 3 — before/after: `FOR UPDATE` vs `SKIP LOCKED` on a job queue**

```
 ✗ FOR UPDATE — every worker serialises on the same head row
 ┌───────────────────────────────────────────────────────────────┐
 │ SELECT * FROM job_queue WHERE status='pending'                │
 │  ORDER BY created_at LIMIT 1 FOR UPDATE;                      │
 │                                                                │
 │  worker1 ──► job#1  ✓ locked                                  │
 │  worker2 ──► job#1  ⏸ waits                                   │
 │  worker3 ──► job#1  ⏸ waits                                   │
 │  worker4 ──► job#1  ⏸ waits          ★ 20 workers, 1 working  │
 │                                                                │
 │ MEASURED: 20 workers → 340 jobs/s (≈ single-threaded)         │
 └───────────────────────────────────────────────────────────────┘

 ✓ FOR UPDATE SKIP LOCKED — each worker takes a different row
 ┌───────────────────────────────────────────────────────────────┐
 │ SELECT * FROM job_queue WHERE status='pending'                │
 │  ORDER BY created_at LIMIT 10 FOR UPDATE SKIP LOCKED;         │
 │                                                                │
 │  worker1 ──► jobs #1–10    ✓                                  │
 │  worker2 ──► jobs #11–20   ✓  (skipped 1–10, they're locked)  │
 │  worker3 ──► jobs #21–30   ✓                                  │
 │  worker4 ──► jobs #31–40   ✓  ★ zero contention, zero waiting │
 │                                                                │
 │ MEASURED: 20 workers → 11,200 jobs/s      ★ 33×               │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

```sql
CREATE TABLE accounts (id bigint PRIMARY KEY, balance_minor bigint NOT NULL);
INSERT INTO accounts VALUES (1,100000),(2,100000);
```

**See a row lock block another.**
```sql
-- session 1                          -- session 2
BEGIN;
SELECT * FROM accounts WHERE id=1
  FOR UPDATE;
                                      BEGIN;
                                      SELECT * FROM accounts WHERE id=1
                                        FOR UPDATE;   -- ⏸ hangs
```
```sql
-- session 3 — diagnose it
SELECT pid, pg_blocking_pids(pid) AS blocked_by, state,
       now()-query_start AS waited, substring(query,1,50) AS query
  FROM pg_stat_activity
 WHERE cardinality(pg_blocking_pids(pid)) > 0;
```
```
  pid  | blocked_by | state  |     waited      |              query
-------+------------+--------+-----------------+-------------------------------
 41288 | {41202}    | active | 00:00:14.882    | SELECT * FROM accounts WHERE id
   ★ one query. This is the first thing to run in any lock incident.
```
```sql
-- session 1
COMMIT;    -- session 2 immediately proceeds
```

**Prove readers do not block.**
```sql
-- session 1                          -- session 2
BEGIN;
UPDATE accounts SET balance_minor=1
  WHERE id=1;
                                      SELECT balance_minor FROM accounts
                                        WHERE id=1;      -- ★ returns INSTANTLY
```
```
 balance_minor
---------------
        100000       ★ the pre-UPDATE value, via MVCC. No waiting.
```
```sql
ROLLBACK;
```

**Prove `FOR UPDATE` dirties the page and writes WAL.**
```sql
SELECT pg_current_wal_lsn() AS before \gset
BEGIN;
SELECT * FROM accounts WHERE id=1 FOR UPDATE;
COMMIT;
SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), :'before') AS wal_bytes;
```
```
 wal_bytes
-----------
       154        ★ a "read-only" query produced 154 bytes of WAL.
```

**See the four row-lock strengths interact.**
```sql
CREATE TABLE orders (
  id bigserial PRIMARY KEY,
  customer_id bigint NOT NULL REFERENCES accounts(id)
);

-- session 1                          -- session 2
BEGIN;
SELECT * FROM accounts WHERE id=1
  FOR UPDATE;                         -- strongest
                                      INSERT INTO orders (customer_id)
                                        VALUES (1);
                                      -- ⏸ ★ BLOCKS. The FK takes
                                      --   FOR KEY SHARE on accounts(1),
                                      --   which conflicts with FOR UPDATE.
ROLLBACK;

BEGIN;
SELECT * FROM accounts WHERE id=1
  FOR NO KEY UPDATE;                  -- ★ weaker
                                      INSERT INTO orders (customer_id)
                                        VALUES (1);
                                      -- ★ SUCCEEDS IMMEDIATELY.
ROLLBACK;
```
```
 ★ THE LESSON: if you are not changing a key column, FOR NO KEY
   UPDATE is strictly better. FOR UPDATE on a parent row blocks
   every child INSERT.
```

**See the table-lock queue form, and break it with `lock_timeout`.**
```sql
-- session 1
BEGIN; SELECT count(*) FROM accounts;   -- holds ACCESS SHARE, stays open

-- session 2
ALTER TABLE accounts ADD COLUMN nickname text;    -- ⏸ waits (ACCESS EXCL)

-- session 3
SELECT id FROM accounts WHERE id=1;               -- ⏸ ★ ALSO WAITS

-- session 4 — see the queue
SELECT a.pid, l.mode, l.granted, substring(a.query,1,40) AS query
  FROM pg_locks l JOIN pg_stat_activity a USING (pid)
 WHERE l.relation = 'accounts'::regclass ORDER BY l.granted DESC, a.query_start;
```
```
  pid  |        mode         | granted |            query
-------+---------------------+---------+------------------------------
 41202 | AccessShareLock     | t       | SELECT count(*) FROM accounts
 41290 | AccessExclusiveLock | f       | ALTER TABLE accounts ADD COL
 41305 | AccessShareLock     | f       | SELECT id FROM accounts WHER
   ★ granted=false is the queue. Two waiters, one of which
     (41305) does not conflict with the holder at all.
```
```sql
-- session 2, the right way
SET lock_timeout = '2s';
ALTER TABLE accounts ADD COLUMN nickname text;
```
```
ERROR:  canceling statement due to lock timeout
   ★ it failed in 2 seconds instead of building a queue.
```

**The job-queue pattern.**
```sql
CREATE TABLE job_queue (
  id bigserial PRIMARY KEY,
  status text NOT NULL DEFAULT 'pending',
  payload jsonb NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX ON job_queue (created_at) WHERE status='pending';
INSERT INTO job_queue (payload) SELECT jsonb_build_object('n',g)
  FROM generate_series(1,100000) g;

-- run this in several sessions simultaneously
BEGIN;
SELECT id FROM job_queue
 WHERE status='pending'
 ORDER BY created_at
 LIMIT 5
 FOR UPDATE SKIP LOCKED;
-- ★ each session gets a DIFFERENT set of 5. Nobody waits.
UPDATE job_queue SET status='done' WHERE id = ANY($1);
COMMIT;
```

**Advisory locks.**
```sql
-- session 1
SELECT pg_try_advisory_xact_lock(hashtext('nightly-import'));   -- t
-- session 2
SELECT pg_try_advisory_xact_lock(hashtext('nightly-import'));   -- f
-- ★ session 2 knows another instance is running, without blocking.
```

---

## Example 2 — production scenario

**The situation.** A logistics platform. 06:42 IST, a routine migration. Within 90 seconds, every API endpoint touching `shipments` returns 504. CPU is at 4%. Disk is idle. Nothing is "slow."

```
 THE ALERT
   p99 latency        120 ms → timeout
   active connections 40 → 500 (pool max), then connection refusals
   CPU                4%      ← ★ nothing is working
   disk I/O           near zero
   error rate         100% on 11 endpoints
```

**Step 1 — the one query that diagnoses every lock incident.**

```sql
SELECT
  a.pid,
  pg_blocking_pids(a.pid) AS blocked_by,
  a.state,
  now() - a.xact_start   AS xact_age,
  now() - a.query_start  AS wait,
  a.application_name,
  substring(a.query, 1, 70) AS query
FROM pg_stat_activity a
WHERE a.datname = current_database()
  AND (cardinality(pg_blocking_pids(a.pid)) > 0 OR a.wait_event_type = 'Lock')
ORDER BY a.xact_start;
```
```
  pid  | blocked_by |  state  | xact_age |  wait   | application_name |   query
-------+------------+---------+----------+---------+------------------+-------------------
 22104 | {}         | idle in | 00:47:11 |         | metabase         | SELECT s.id, s.st…
       |            | transac.|          |         |                  |
 22890 | {22104}    | active  | 00:01:32 | 00:01:32| flyway           | ALTER TABLE shipm…
 23001 | {22890}    | active  | 00:01:29 | 00:01:29| api-server-3     | SELECT * FROM shi…
 23002 | {22890}    | active  | 00:01:29 | 00:01:29| api-server-1     | SELECT * FROM shi…
 …     | {22890}    | active  | …        | …       | …                | …
 (487 rows)
```

```
 ★ THE ENTIRE INCIDENT, IN ONE READING:
   • 22104 is a Metabase dashboard query, ★ IDLE IN TRANSACTION for
     47 minutes. It holds ACCESS SHARE on shipments.
   • 22890 is the migration. It wants ACCESS EXCLUSIVE. It waits.
   • ★ 485 API queries want ACCESS SHARE — which is COMPATIBLE with
     22104 — but they are BEHIND 22890 in the FIFO queue.
 ⇒ THE MIGRATION IS NOT THE PROBLEM. THE MIGRATION *WAITING* IS
   THE PROBLEM. And the reason it waits is a BI tool that opened a
   transaction and never closed it.
```

**Step 2 — confirm with `pg_locks`.**

```sql
SELECT l.pid, l.mode, l.granted, a.application_name
  FROM pg_locks l JOIN pg_stat_activity a USING (pid)
 WHERE l.relation = 'shipments'::regclass
 ORDER BY l.granted DESC, a.query_start
 LIMIT 6;
```
```
  pid  |        mode         | granted | application_name
-------+---------------------+---------+------------------
 22104 | AccessShareLock     | t       | metabase          ← the holder
 22890 | AccessExclusiveLock | f       | flyway            ← ★ the blocker
 23001 | AccessShareLock     | f       | api-server-3
 23002 | AccessShareLock     | f       | api-server-1
 23003 | AccessShareLock     | f       | api-server-2
 23004 | AccessShareLock     | f       | api-server-4
```

**Step 3 — resolve it. Two options, in order.**

```sql
-- ★ FIRST: cancel the MIGRATION, not the dashboard.
--   Cancelling 22890 dissolves the queue instantly, and the 485
--   waiters proceed — they never conflicted with 22104.
SELECT pg_cancel_backend(22890);
-- ⇒ recovery in ~200 ms. All 11 endpoints green.

-- SECOND, separately: deal with the idle-in-transaction session.
SELECT pg_terminate_backend(22104);
-- ⇒ pg_cancel_backend does nothing to an IDLE session — there is no
--   running statement to cancel. Termination is required.
```

```
 ★ THE COUNTERINTUITIVE PART: the instinct is to kill the 47-minute
   dashboard query. That also works — but it's the wrong first move.
   Cancelling the BLOCKER (the migration) restores service
   immediately and touches nothing anyone depends on.
```

**Step 4 — the four fixes, so it cannot recur.**

```sql
-- ① ★ NEVER RUN DDL WITHOUT lock_timeout. Non-negotiable.
--    In the migration tool's session setup:
SET lock_timeout = '3s';
SET statement_timeout = '30s';
-- ⇒ a migration that cannot get its lock now FAILS in 3 seconds
--   instead of building a 485-deep queue.
```

```sql
-- ② KILL IDLE-IN-TRANSACTION SESSIONS AUTOMATICALLY.
ALTER SYSTEM SET idle_in_transaction_session_timeout = '60s';
SELECT pg_reload_conf();
-- ⇒ ★ this single setting would have prevented the entire incident.
--   A BI tool leaving a transaction open for 47 minutes is a bug
--   the database can enforce against.

-- and give the BI tool its own, stricter budget:
ALTER ROLE metabase SET idle_in_transaction_session_timeout = '10s';
ALTER ROLE metabase SET statement_timeout = '120s';
ALTER ROLE metabase SET default_transaction_read_only = on;
```

```js
// ③ THE MIGRATION RETRY LOOP — how to actually apply DDL safely
async function applyDdlWithRetry(sql, { attempts = 20, waitMs = 5000 } = {}) {
  for (let i = 0; i < attempts; i++) {
    const client = await pool.connect();
    try {
      await client.query('BEGIN');
      await client.query("SET LOCAL lock_timeout = '3s'");
      await client.query(sql);
      await client.query('COMMIT');
      return;
    } catch (e) {
      await client.query('ROLLBACK').catch(() => {});
      if (e.code !== '55P03' /* lock_not_available */) throw e;
      log.warn({ attempt: i + 1 }, 'DDL could not acquire lock; retrying');
      await sleep(waitMs);          // ★ let the long query finish
    } finally { client.release(); }
  }
  throw new Error('DDL could not acquire lock after 20 attempts');
}
```

```sql
-- ④ USE THE LOCK-FREE FORMS WHERE THEY EXIST (Topic 28)
--    ✗ ACCESS EXCLUSIVE for minutes:
CREATE INDEX idx_shipments_status ON shipments (status);
--    ✓ SHARE UPDATE EXCLUSIVE — concurrent reads AND writes:
CREATE INDEX CONCURRENTLY idx_shipments_status ON shipments (status);

--    ✗ full table rewrite under ACCESS EXCLUSIVE:
ALTER TABLE shipments ADD CONSTRAINT ck_weight CHECK (weight_grams > 0);
--    ✓ two steps, each holding the lock for microseconds:
ALTER TABLE shipments ADD CONSTRAINT ck_weight CHECK (weight_grams > 0) NOT VALID;
ALTER TABLE shipments VALIDATE CONSTRAINT ck_weight;   -- SHARE UPDATE EXCLUSIVE
```

**Step 5 — the monitoring that catches it before users do.**

```sql
-- alert: any transaction older than 5 minutes
SELECT max(extract(epoch from now() - xact_start)) AS oldest_xact_seconds
  FROM pg_stat_activity WHERE xact_start IS NOT NULL;

-- alert: anything idle in transaction at all
SELECT count(*) FROM pg_stat_activity WHERE state = 'idle in transaction';

-- ★ alert: any ungranted lock older than 5 seconds — the leading
--   indicator of a queue forming
SELECT count(*) FROM pg_locks l JOIN pg_stat_activity a USING (pid)
 WHERE NOT l.granted AND now() - a.query_start > interval '5 seconds';
```

**Step 6 — results.**

| | Before | After |
|---|---|---|
| Longest lock queue seen | 487 waiters | 0 |
| Migration failure mode | 8-minute outage | fails in 3 s, retries |
| `idle in transaction` max age | 47 min | 60 s (enforced) |
| Time to diagnose a lock incident | 22 min | one query, < 1 min |
| DDL applied during business hours | never (fear) | routinely |

```
 ★ THE LESSON: the migration was correct, the query it waited on was
   harmless, and the two together caused a total outage — purely
   because of FIFO queueing. The fix is not "be careful"; it is
   TWO SETTINGS:
     SET lock_timeout on every DDL session
     idle_in_transaction_session_timeout globally
```

---

## Common mistakes

**1. Running DDL without `lock_timeout`.**
- *Symptom:* every query on the table hangs, including ones that don't conflict.
- *Engine-level why:* the lock queue is FIFO; a blocked ACCESS EXCLUSIVE request blocks everything behind it.
- *Fix:* `SET lock_timeout = '3s'` before every DDL statement, plus a retry loop.

**2. Leaving sessions idle in transaction.**
- *Symptom:* mysterious blocking with no running query; VACUUM stops reclaiming space.
- *Engine-level why:* the transaction holds every lock it took and pins its xmin until COMMIT.
- *Fix:* `idle_in_transaction_session_timeout`; never `await` an HTTP call inside a transaction.

**3. `FOR UPDATE` when `FOR NO KEY UPDATE` would do.**
- *Symptom:* child `INSERT`s block on a parent row you're only updating a non-key column of.
- *Engine-level why:* FK checks take `FOR KEY SHARE` on the parent, which conflicts with `FOR UPDATE` but not with `FOR NO KEY UPDATE`.
- *Fix:* use the weaker mode when you won't change a key.

**4. `FOR UPDATE` on a queue head.**
- *Symptom:* 20 workers, throughput of one.
- *Fix:* `FOR UPDATE SKIP LOCKED` with `LIMIT n`.

**5. `SELECT … FOR SHARE` then `UPDATE`.**
- *Symptom:* reliable deadlocks under concurrency.
- *Engine-level why:* two transactions both hold `FOR SHARE`; each then needs to upgrade to exclusive, and each waits for the other.
- *Fix:* take `FOR UPDATE`/`FOR NO KEY UPDATE` up front. Never upgrade a lock (Topic 48).

**6. Holding a transaction across a network call.**
- *Symptom:* row locks held for the duration of a payment-gateway call; contention scales with the gateway's latency.
- *Fix:* do external work outside the transaction; use the transactional-outbox pattern (Topic 52).

**7. Assuming row locks cost nothing.**
- *Symptom:* WAL volume and dirty-buffer rate rise on a read-heavy workload.
- *Engine-level why:* `FOR UPDATE` writes `t_xmax` into the tuple — it dirties the page and emits WAL.
- *Fix:* only lock rows you will actually write.

**8. Blowing `max_locks_per_transaction` with partitions.**
- *Symptom:* `ERROR: out of shared memory / You might need to increase max_locks_per_transaction`.
- *Engine-level why:* each partition touched needs its own table-lock entry.
- *Fix:* ensure partition pruning works; raise the setting; reduce partition count (Topic 59).

**9. Killing the wrong backend.**
- *Symptom:* you cancel the long query, the migration then acquires the lock and holds it for eight more minutes.
- *Fix:* cancel the **blocker** (the waiter that everyone is queued behind) first — service returns instantly. `pg_cancel_backend` won't touch an idle session; use `pg_terminate_backend`.

---

## Hands-on proof

**PROVE IT #1 — `pg_blocking_pids` finds it in one query.** (Example 1.)

**PROVE IT #2 — readers don't block writers.** (Example 1.)

**PROVE IT #3 — `FOR UPDATE` writes WAL.** (Example 1.)

**PROVE IT #4 — `FOR UPDATE` blocks child inserts, `FOR NO KEY UPDATE` doesn't.** (Example 1.)

**PROVE IT #5 — the FIFO queue, and `lock_timeout` preventing it.** (Example 1.)

**PROVE IT #6 — `SKIP LOCKED` throughput.**
```bash
cat > /tmp/q_forupdate.sql <<'EOF'
BEGIN;
SELECT id FROM job_queue WHERE status='pending' ORDER BY created_at LIMIT 1 FOR UPDATE;
UPDATE job_queue SET status='done' WHERE id = (SELECT id FROM job_queue WHERE status='pending' ORDER BY created_at LIMIT 1);
COMMIT;
EOF
cat > /tmp/q_skiplocked.sql <<'EOF'
BEGIN;
UPDATE job_queue SET status='done' WHERE id IN (
  SELECT id FROM job_queue WHERE status='pending'
   ORDER BY created_at LIMIT 5 FOR UPDATE SKIP LOCKED);
COMMIT;
EOF
pgbench -f /tmp/q_forupdate.sql  -c 20 -j 4 -T 20 shop | grep tps
pgbench -f /tmp/q_skiplocked.sql -c 20 -j 4 -T 20 shop | grep tps
```
```
 FOR UPDATE   : tps =    340.2
 SKIP LOCKED  : tps = 11,204.8      ★ 33×
```

**PROVE IT #7 — see a multixact form.**
```sql
-- three sessions each take FOR KEY SHARE on the same row
-- session 1/2/3: BEGIN; SELECT * FROM accounts WHERE id=1 FOR KEY SHARE;
CREATE EXTENSION IF NOT EXISTS pageinspect;
SELECT t_xmax, t_infomask::bit(16)
  FROM heap_page_items(get_raw_page('accounts', 0)) WHERE lp = 1;
-- ★ t_infomask has HEAP_XMAX_IS_MULTI (0x1000) set;
--   t_xmax is a MultiXactId, not an XID.
SELECT count(*) FROM pg_multixact_members_test;   -- (conceptual)
```

**PROVE IT #8 — see the lock modes DDL actually takes.**
```sql
BEGIN;
ALTER TABLE accounts ADD COLUMN test_col text;
SELECT mode FROM pg_locks WHERE relation='accounts'::regclass AND pid=pg_backend_pid();
ROLLBACK;
```
```
        mode
---------------------
 AccessExclusiveLock      ★ conflicts with SELECT
```
```sql
BEGIN;
CREATE INDEX CONCURRENTLY … ;   -- ⚠ cannot run inside a transaction block
-- run it outside, then check from another session:
SELECT mode FROM pg_locks WHERE relation='accounts'::regclass;
--  ShareUpdateExclusiveLock    ★ compatible with SELECT, INSERT, UPDATE
```

---

## The design decision framework

```
★★★ TAKE THE WEAKEST LOCK THAT IS SUFFICIENT, FOR THE SHORTEST TIME. ★★★

 ① DO YOU NEED A LOCK AT ALL?
    Read-modify-write on one row?
      ⇒ ★ NO. Use one atomic statement:
        UPDATE t SET x = x - $1 WHERE id=$2 AND x >= $1
      ⇒ this is faster AND correct AND takes no explicit lock.
    Enforcing an invariant?
      ⇒ ★ NO. Use UNIQUE / EXCLUDE / CHECK (Topics 24, 43).
    ⇒ MOST "I need FOR UPDATE" SITUATIONS DON'T.

 ② IF YOU DO NEED A ROW LOCK, PICK THE WEAKEST
    will you change a key column or DELETE?   ⇒ FOR UPDATE
    only non-key columns?                     ⇒ ★ FOR NO KEY UPDATE
    is "someone else is doing it" a valid answer? ⇒ NOWAIT
    is it a queue?                            ⇒ ★ SKIP LOCKED
    do you want FOR SHARE?                    ⇒ ★ almost certainly no
                                                 (deadlock, Topic 48)

 ③ LOCK ORDER IS A GLOBAL CONTRACT
    Always acquire in a deterministic order — e.g. ascending PK:
      SELECT … WHERE id = ANY($1) ORDER BY id FOR UPDATE
    ⇒ ★ a consistent order makes deadlock structurally impossible.

 ④ NEVER HOLD A LOCK ACROSS
    ✗ an HTTP call    ✗ a queue publish    ✗ a file write
    ✗ user think-time ✗ a retry with backoff
    ⇒ your lock duration becomes someone else's tail latency.

 ⑤ DDL — THE NON-NEGOTIABLE CHECKLIST
    ✓ SET lock_timeout ('2–5s') before EVERY DDL statement
    ✓ retry in a loop with a wait between attempts
    ✓ prefer the CONCURRENTLY / NOT VALID forms (Topic 28)
    ✓ know the mode: ACCESS EXCLUSIVE stops ALL reads
    ✗ never run DDL from a session without a timeout, ever

 ⑥ SETTINGS EVERY PRODUCTION DATABASE SHOULD HAVE
    idle_in_transaction_session_timeout = '60s'   ★ prevents the
                                                    classic outage
    statement_timeout                    per role
    lock_timeout                         per DDL session
    log_lock_waits = on                  ★ logs any wait exceeding
    deadlock_timeout = '1s'                deadlock_timeout — free
                                           forensic evidence

 ⑦ WHEN AN INCIDENT HAPPENS — THE ORDER
    ① SELECT pid, pg_blocking_pids(pid), … FROM pg_stat_activity
       WHERE cardinality(pg_blocking_pids(pid)) > 0;
    ② find the ROOT: the pid that blocks others and is blocked by
       nobody
    ③ ★ cancel the BLOCKER everyone is queued behind (often the
       waiter, not the holder) — service returns immediately
    ④ then deal with the root holder
    ⑤ pg_cancel_backend for active; ★ pg_terminate_backend for idle
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
With two `psql` sessions: (a) show `FOR UPDATE` blocking `FOR UPDATE`; (b) show a plain `SELECT` *not* blocking on an uncommitted `UPDATE`; (c) show `FOR UPDATE` on a parent blocking a child `INSERT`, and `FOR NO KEY UPDATE` not blocking it. Explain (c) in terms of `FOR KEY SHARE`.

### Exercise 2 — medium (apply it)
For each, name the lock taken, whether it blocks a concurrent `SELECT`, and give a lower-impact alternative:
(a) `CREATE INDEX` · (b) `ALTER TABLE … ADD COLUMN c text` · (c) `ALTER TABLE … ADD COLUMN c text NOT NULL DEFAULT 'x'` · (d) `ALTER TABLE … ADD FOREIGN KEY` · (e) `VACUUM FULL` · (f) `TRUNCATE` · (g) `REFRESH MATERIALIZED VIEW`

Then build the FIFO-queue outage yourself with three sessions, and show `lock_timeout` preventing it.

### Exercise 3 — hard (production simulation)
At 06:42 a migration runs. Within 90 seconds every endpoint touching `shipments` returns 504. CPU is 4%, disk idle. `pg_stat_activity` shows a Metabase session idle in transaction for 47 minutes, a Flyway `ALTER TABLE` waiting on it, and 485 API queries waiting on the Flyway session.

(a) Explain precisely why the 485 API `SELECT`s wait, given that `ACCESS SHARE` is compatible with `ACCESS SHARE`. Why doesn't PostgreSQL let them jump the queue?
(b) Write the single diagnostic query and interpret its output.
(c) Which backend do you cancel *first*, and why is the obvious answer the wrong one? What is the measured recovery time for each choice?
(d) Why does `pg_cancel_backend` not work on the Metabase session?
(e) Give the two settings that would have prevented this entirely, with values and justification.
(f) Write the DDL retry helper: which SQLSTATE, what timeouts, `SET` vs `SET LOCAL`, how many attempts, what wait.
(g) Rewrite three of the migration's statements to use lock-free forms.
(h) Write the three alerts that catch this as a leading indicator, with thresholds.
(i) The BI tool needs read access but must never cause this again. Give the four `ALTER ROLE` settings you'd apply.

---

## Mental model checkpoint

1. Where is a row lock physically stored? Name two consequences of that answer.
2. Which single table-lock mode conflicts with `ACCESS SHARE`? Why does that make DDL dangerous?
3. Explain the FIFO queue and why a compatible query can wait behind an incompatible waiter.
4. Name the four row-lock strengths, weakest first. Which one does a foreign-key check take, and on which row?
5. Why is `FOR NO KEY UPDATE` often strictly better than `FOR UPDATE`?
6. What does `FOR UPDATE SKIP LOCKED` solve, and roughly what throughput difference should you expect?
7. Why does `SELECT … FOR SHARE` followed by `UPDATE` deadlock?
8. Give the one query you run first in any lock incident.

---

## Quick reference card

**Diagnose — memorise this one**
```sql
SELECT pid, pg_blocking_pids(pid) AS blocked_by, state,
       now()-xact_start AS xact_age, now()-query_start AS wait,
       application_name, substring(query,1,80) AS query
  FROM pg_stat_activity
 WHERE cardinality(pg_blocking_pids(pid)) > 0
 ORDER BY xact_start;
```
```sql
SELECT l.pid, l.mode, l.granted, a.application_name
  FROM pg_locks l JOIN pg_stat_activity a USING (pid)
 WHERE l.relation = 'orders'::regclass ORDER BY l.granted DESC;
-- ★ granted = false IS the queue
```

**Resolve**
```sql
SELECT pg_cancel_backend(pid);      -- active queries
SELECT pg_terminate_backend(pid);   -- ★ required for 'idle in transaction'
```

| Statement | Table lock | Blocks `SELECT`? |
|---|---|---|
| `SELECT` | ACCESS SHARE | no |
| `INSERT`/`UPDATE`/`DELETE` | ROW EXCLUSIVE | no |
| `VACUUM`, `ANALYZE`, `CREATE INDEX CONCURRENTLY` | SHARE UPDATE EXCLUSIVE | no |
| `CREATE INDEX` | SHARE | no (blocks writes) |
| **`ALTER TABLE`, `DROP`, `TRUNCATE`, `VACUUM FULL`** | **ACCESS EXCLUSIVE** | ★ **yes** |

| Row lock | Taken by | Blocks child `INSERT`? |
|---|---|---|
| `FOR KEY SHARE` | FK check on the parent | no |
| `FOR SHARE` | explicit | no |
| `FOR NO KEY UPDATE` | plain `UPDATE` of non-key columns | ★ no |
| `FOR UPDATE` | explicit; `UPDATE` of a key; `DELETE` | ★ **yes** |

**Settings every production database needs**
```sql
idle_in_transaction_session_timeout = '60s'   -- ★ prevents the classic outage
log_lock_waits = on                            -- free forensics
deadlock_timeout = '1s'
SET lock_timeout = '3s';                       -- ★ before EVERY DDL
```

**The rule:** weakest lock, shortest time, deterministic order, never across a network call.

---

## When would I use this at work?

1. **Any "everything is slow but the CPU is idle" incident.** That signature is a lock queue with near-certainty. One query names the root, and cancelling the right backend restores service in under a second.

2. **Every migration you ever write.** `SET lock_timeout` plus a retry loop is the difference between a migration that fails safely and one that takes down the product for eight minutes.

3. **Designing a job queue, a worker pool, or anything with contention.** `FOR UPDATE SKIP LOCKED` is 33× faster than the naive version and needs no external queue system for most workloads (case study 09).

4. **Code review.** `SELECT … FOR UPDATE` in a pull request deserves a question: could this be one atomic statement, or a constraint, instead?

---

## Connected topics

**Understand before this:** 39 (transactions), 43 (anomalies — what locks prevent), 44 (isolation levels), 05 (tuple layout — `t_xmax` is where row locks live), 23 (foreign keys — why `FOR KEY SHARE` exists).

**This unlocks:**
- **46** — MVCC: why readers never need these locks at all
- **47** — VACUUM: multixacts, and why held locks block cleanup
- **48** — deadlocks: what happens when two lock waits form a cycle
- **49** — optimistic vs pessimistic: `FOR UPDATE` vs a version column
- **28** — zero-downtime migrations: the lock-mode table, applied
- **Case study 09** — `SKIP LOCKED` as a production queue
