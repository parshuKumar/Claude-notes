# 09 — The Life of a Query
## Phase: Storage Internals

---

## ELI5 — The Simple Analogy

You post a letter in Mumbai to an address in Chennai.

It doesn't teleport. It goes into a **postbox**, gets collected, arrives at a **sorting office** where a machine reads the pincode and decides which route it takes, is loaded onto a **truck or a plane** depending on which is cheaper today, gets sorted again in Chennai, handed to a postman who knows that street, and finally dropped through a door. Every stage has a person or a machine responsible for exactly one decision.

If the letter is late, "the post office is slow" is useless. You need to know: did it sit in the postbox? Was it mis-sorted? Did the plane get cancelled? Was the postman waiting for a locked gate?

A query is that letter. This topic is the complete route map — so that when a query is slow, you can say *which stage* rather than "the database is slow."

---

## Where this fits in the big picture

```
   01 What is a DB
   02 Engine architecture ─── the components
   04 Pages ────────────────┐
   05 Tuples                ├── the storage
   06 Row vs column         │
   07 Buffer pool ──────────┘
   08 B-tree vs LSM ─── how writes land
              │
              ▼
   ┌──────────────────────────────────────┐
   │ 09 THE LIFE OF A QUERY  ← YOU ARE HERE│
   │ everything above, in one trace        │
   └──────────────────┬───────────────────┘
                      │
                      ▼
   PHASE 2 — Indexes. Every topic from here is
   "how do we make one of these stages cheaper?"
```

**This is the capstone of Phase 1.** If you can narrate this trace from memory, Phase 1 is done.

---

## What is this?

The complete path a SQL statement takes from your Node.js `pool.query()` call to the rows arriving back — through every process, every memory structure, and every file, with the reads and writes at each step named.

Two traces: one SELECT (the read path) and one INSERT inside a transaction (the write path). Everything in the curriculum after this is a zoom-in on one stage.

---

## Why does it matter for a backend developer?

Because "the query is slow" has at least nine distinct causes, and they have nine distinct fixes:

| Stage | Symptom when it's the bottleneck | Fix |
|---|---|---|
| Connection acquisition | latency spikes under concurrency, pool timeouts | pooling (65) |
| Parse | only with enormous statements or 10k-item `IN` lists | rewrite / `= ANY($1)` |
| Plan | high `Planning Time`, many partitions or joins | prepared statements (18) |
| Lock acquisition | query hangs then completes | lock analysis (45) |
| Snapshot / visibility | slow only when other txns are open | long-transaction hygiene (46) |
| Access-path choice | `Seq Scan` where an index exists | statistics, index design (13–18) |
| Buffer I/O | `Buffers: read` ≫ `hit` | RAM, narrower rows, covering index (07) |
| Executor CPU | high time, low I/O, big `Rows Removed by Filter` | better predicate/index (15) |
| Result transfer | fast in psql, slow in Node.js | fetch fewer columns/rows |

Being able to point at the stage is the whole skill. Everything else is looking it up.

---

## The physical reality

Everything one query touches, in one picture:

```
 ┌─ YOUR NODE.JS PROCESS ───────────────────────────────────────────────┐
 │  pg.Pool → a TCP socket to PgBouncer → to PostgreSQL port 5432       │
 └──────────────────────────────┬───────────────────────────────────────┘
                                │ wire protocol messages
 ┌──────────────────────────────▼───────────────────────────────────────┐
 │ BACKEND PROCESS (one OS process, ~10 MB private memory)              │
 │   ├── parse tree, query tree, plan tree      (private)               │
 │   ├── catalog cache, relcache, plan cache    (private)               │
 │   ├── work_mem allocations per Sort/Hash node(private)  ⚠            │
 │   └── executor state                          (private)              │
 └──────────────────────────────┬───────────────────────────────────────┘
                                │
 ┌──────────────────────────────▼───────────────────────────────────────┐
 │ SHARED MEMORY                                                        │
 │  ┌───────────────┬─────────────┬──────────────┬──────────────────┐   │
 │  │ BUFFER POOL   │ LOCK TABLE  │ WAL BUFFERS  │ PROC ARRAY       │   │
 │  │ 8KB pages     │ rel/page/   │              │ (snapshots:      │   │
 │  │ + descriptors │ tuple locks │              │  who is running) │   │
 │  └───────┬───────┴─────────────┴──────┬───────┴──────────────────┘   │
 └──────────┼──────────────────────────── ┼──────────────────────────────┘
            │                             │
 ┌──────────▼─────────────────────────────▼──────────────────────────────┐
 │ DISK                                                                  │
 │  base/16388/16390        heap (rows)                                  │
 │  base/16388/16390_vm     visibility map                               │
 │  base/16388/16393        index                                        │
 │  pg_wal/00000001...      the log                                      │
 │  pg_xact/0000            commit status                                │
 └───────────────────────────────────────────────────────────────────────┘
```

---

## How it works — step by step

### TRACE A — the read path

```js
const { rows } = await pool.query(
  `SELECT o.id, o.total_paise, o.created_at
     FROM orders o
    WHERE o.user_id = $1
      AND o.created_at > now() - interval '30 days'
    ORDER BY o.created_at DESC
    LIMIT 20`, [4471]);
```

```
════════════ CLIENT SIDE ════════════

 0. POOL ACQUISITION                                    [Node.js]
    pg.Pool checks for an idle client. If none and max is reached,
    the request QUEUES. ⚠ Under load this is often the largest single
    component of latency, and it is invisible in every database metric.
    Instrument it: pool.totalCount, pool.idleCount, pool.waitingCount.

 1. EXTENDED QUERY PROTOCOL                             [wire]
    node-postgres sends Parse → Bind → Describe → Execute → Sync.
    Parameters are sent SEPARATELY from the SQL text — this is why
    parameterised queries cannot be SQL-injected: $1 never becomes
    part of the parsed statement. (Topic 69.)

════════════ BACKEND: QUERY PROCESSING ════════════

 2. PARSE                                               [~20 µs]
    SQL text → raw parse tree (grammar only).
    `orders` is still just a string here.
    READS: nothing.  FAILS ON: syntax errors.

 3. ANALYSE / TRANSFORM                                 [~40 µs]
    Names resolved: orders → OID 16390; user_id → attnum 3;
    types inferred; permissions checked.
    READS: pg_class, pg_attribute, pg_type — via the RELCACHE and
           CATCACHE (per-backend RAM). A cold backend does real reads
           here; a warm one does none. ⚠ This is part of why a brand-new
           connection is slower than a reused one.
    FAILS ON: relation does not exist, permission denied.

 4. REWRITE                                             [~5 µs]
    Views expanded to their definitions; RULES applied; ROW LEVEL
    SECURITY policies injected as extra WHERE clauses.
    ⚠ A query on a view of a view of a view expands here — this is why
      deeply nested views produce enormous plans.

 5. PLAN / OPTIMISE                                     [~200 µs]
    Enumerate access paths for `orders`:
      a) Seq Scan + filter        cost 0.00..71204.00  rows=38
      b) Index Scan idx_user_id   cost 0.43..842.10    rows=38
      c) Index Scan idx_user_created (user_id, created_at DESC)
                                  cost 0.43..38.20     rows=38
    Estimate rows from pg_statistic (histogram + MCV for user_id).
    Choose (c) — it satisfies the filter AND the ORDER BY, so no Sort
    node is needed, and LIMIT 20 can stop early.
    READS: pg_statistic, pg_class.reltuples/relpages, pg_index.
    OUTPUT: a plan tree:
              Limit
                └── Index Scan Backward using idx_user_created

 ⚠ If this were a PREPARED statement executed 6+ times, PostgreSQL might
   switch to a GENERIC PLAN — planned once without knowing $1. Great for
   planning time, occasionally catastrophic for plan quality. Control with
   plan_cache_mode. (Topic 18.)

════════════ BACKEND: EXECUTION ════════════

 6. SNAPSHOT ACQUISITION                                [~2 µs]
    Ask the PROC ARRAY: which XIDs are in progress right now?
      snapshot = { xmin: 91004, xmax: 91011, xip: [91007, 91009] }
    Meaning: everything < 91004 is settled; 91007 and 91009 are still
    running and their changes are invisible to me.
    THIS SNAPSHOT IS FIXED for the whole statement (READ COMMITTED) or
    the whole transaction (REPEATABLE READ). (Topics 44, 46.)
    READS: proc array (shared memory).

 7. LOCK ACQUISITION                                    [~3 µs]
    AccessShareLock on orders and on idx_user_created.
    Conflicts ONLY with AccessExclusiveLock (DDL). So this query never
    blocks another SELECT, INSERT, UPDATE or DELETE — only a concurrent
    ALTER TABLE / DROP INDEX / VACUUM FULL.
    WRITES: an entry in the shared LOCK TABLE.
    ⚠ If someone is running `ALTER TABLE orders ADD COLUMN`, this step
      is where your read hangs — and it will hang behind them even
      though ADD COLUMN itself is instant, because lock queues are FIFO.

 8. EXECUTOR STARTUP
    Plan tree → executor state tree. work_mem allocated per node that
    needs it (none here — no sort, no hash).

 9. INDEX SCAN — descend the B-tree                     [3 page accesses]
    ReadBuffer(idx_user_created, root)      → HIT  (~100 ns)
    ReadBuffer(idx_user_created, internal)  → HIT
    ReadBuffer(idx_user_created, leaf 8891) → MISS → clock sweep evicts
        a victim, read() 8192 B from disk   (~100 µs)
    Position at (user_id=4471, created_at=MAX), scan BACKWARD.
    Leaf entries give us TIDs: (1204,3) (2891,17) (4102,5) ...

10. HEAP FETCH + VISIBILITY CHECK — per matching row    [1 page each]
    For each TID:
      a) VISIBILITY MAP check: is page 1204 marked all-visible?
           YES → and if this were an INDEX-ONLY scan with all needed
                 columns in the index, we could SKIP the heap entirely.
                 Here we need total_paise, which isn't in the index,
                 so we fetch anyway.
      b) ReadBuffer(orders, 1204) — HIT or MISS
      c) Read the tuple at line pointer 3.
      d) MVCC VISIBILITY TEST against the snapshot from step 6:
            xmin committed and < snapshot.xmin?      → visible so far
            xmin in snapshot.xip (still running)?    → INVISIBLE
            xmax set and committed and < xmin?       → INVISIBLE (deleted)
         ⚠ Checking "was xmin committed?" may require reading pg_xact.
           The answer is CACHED in the tuple's HINT BITS on first read —
           which DIRTIES THE PAGE. This is why the first SELECT after a
           big INSERT can generate write I/O. It surprises everyone.
      e) Visible → apply the residual filter (created_at > ...) → emit.

11. LIMIT
    After 20 tuples the Limit node stops pulling. The index scan
    ABANDONS the rest of the leaf. This is why an index that matches
    the ORDER BY is so valuable: without it, the executor would have
    to fetch ALL matching rows, sort them, and discard all but 20.

12. RESULT ENCODING
    Each tuple → DataRow message in the text or binary wire format.
    ⚠ Text format means bigint → "249900" (6 bytes) and timestamptz →
      "2026-08-01 09:12:44.123+00" (29 bytes). Wide result sets cost
      real network time. node-postgres then parses these strings in JS —
      which is why returning 50,000 rows to Node.js is slow even when
      the query itself was fast.

13. COMMAND COMPLETE → ReadyForQuery. Backend returns to idle.

14. POOL RELEASE                                         [Node.js]
    Client returned to the pool. If PgBouncer is in TRANSACTION mode,
    the server connection is returned to ITS pool here too.

 TOTAL for a warm cache:  ~0.4 ms.  Cold:  ~3 ms.
 Under lock contention:   unbounded.
```

### TRACE B — the write path

```js
await client.query('BEGIN');
await client.query(
  'UPDATE products SET stock = stock - 1 WHERE id = $1 AND stock > 0', [88]);
await client.query(
  'INSERT INTO orders (user_id, total_paise) VALUES ($1, $2)', [4471, 249900]);
await client.query('COMMIT');
```

```
 1. BEGIN
    No XID is assigned yet! PostgreSQL is lazy: read-only transactions
    never consume an XID. A "virtual transaction id" is used until the
    first write.
    ⚠ This is why an idle-in-transaction SELECT-only session still holds
      a SNAPSHOT (blocking vacuum) even without an XID.

 2. UPDATE — planning as in Trace A, then:

 3. XID ASSIGNMENT                                       [first write]
    GetNewTransactionId() → 91012. Recorded in the proc array.

 4. LOCK ACQUISITION
    RowExclusiveLock on products (table level — conflicts only with DDL
    and with SHARE-mode locks, NOT with other writers).
    Then the actual row lock, which is NOT in the lock table:

 5. TUPLE LOCK — the clever bit
    PostgreSQL does NOT store row locks in shared memory (that would
    require a lock table entry per locked row — unbounded memory).
    Instead it writes the locking XID into the TUPLE ITSELF:
        t_xmax = 91012, infomask |= HEAP_XMAX_EXCL_LOCK
    ⇒ Row-level locking is FREE in memory and unlimited in count.
    ⇒ A waiter reads t_xmax, sees 91012, and waits on THAT
      transaction ID in the lock table (one entry per transaction,
      not per row). Elegant.
    If another transaction already holds it, we block here. This is
    the "wait_event_type = Lock, wait_event = transactionid" you see
    in pg_stat_activity. (Topic 45.)

 6. VISIBILITY + PREDICATE
    Read the current tuple, check it's visible to our snapshot, evaluate
    `stock > 0`. Fails → 0 rows updated, no write at all.

 7. WAL RECORD for the update                            [WAL buffer, RAM]
    XLOG_HEAP_UPDATE: old TID, new tuple bytes, LSN assigned.
    ⚠ If this page hasn't been written since the last checkpoint, the
      ENTIRE 8 KB page goes into the WAL as a full-page image (Topic 08).

 8. HEAP MODIFICATION                                    [buffer pool, RAM]
    Old tuple: t_xmax = 91012, t_ctid → new location.
    New tuple written (same page if room + fillfactor allows → HOT update).
    Page marked DIRTY, pd_lsn = the LSN from step 7.
    *** THE DISK FILE IS STILL UNCHANGED. ***

 9. INDEX MAINTENANCE
    HOT update (same page, no indexed column changed)? → NO index writes.
    Otherwise → a new entry in EVERY index, each with its own WAL record.

10. INSERT INTO orders — same sequence: FSM lookup, page pin, WAL record,
    tuple written to a dirty buffer, index entries.

11. COMMIT
    a) A commit WAL record is written to the WAL buffer.
    b) *** fsync() the WAL up to this LSN. ***
       THIS IS THE DURABILITY POINT. ~0.3–1.0 ms on NVMe; ~5–10 ms on
       network storage. Under load, GROUP COMMIT batches many
       transactions into one fsync, so per-transaction cost drops.
    c) pg_xact updated: transaction 91012 = COMMITTED.
    d) XID removed from the proc array → the changes become visible to
       new snapshots.
    e) All locks released. Waiters wake up.
    WRITES: pg_wal (DISK, fsync'd), pg_xact.

12. ACK to the client.
    ⚠ At this instant: the change is DURABLE (it's in the WAL) but the
      heap file on disk still holds the OLD data. If the machine loses
      power now, recovery replays the WAL and the change survives.

13. ASYNCHRONOUS, LATER
    bgwriter / checkpointer write the dirty heap and index pages to
    base/16388/*.  Autovacuum eventually removes the dead old tuple.
```

---

## Concept breakdown

```
THE FIVE PHASES  (memorise these five words)
│
├── 1. PARSE      text → tree.        Knows grammar, not your schema.
├── 2. ANALYSE    resolve + check.    Catalog lookups, types, permissions.
├── 3. REWRITE    views, rules, RLS.  Query tree → query tree.
├── 4. PLAN       choose the path.    THE ONLY COST-BASED DECISION.
└── 5. EXECUTE    produce tuples.     Pull-based, node by node.

THE EXECUTOR IS A PULL PIPELINE ("volcano" / iterator model)
│
│   Limit                ← calls next() on its child 20 times, then stops
│    └── Sort            ← MUST consume its ENTIRE child before emitting
│         └── Hash Join  ← builds a hash of one side, probes with the other
│              └── Seq Scan
│
├── BLOCKING nodes must consume everything first: Sort, Hash (build side),
│   Materialize, aggregate without a group key.
├── STREAMING nodes emit as they go: Seq Scan, Index Scan, Nested Loop,
│   Append, most Filters.
└── ⚠ THIS IS WHY `LIMIT 20` IS SOMETIMES FREE AND SOMETIMES NOT:
     Limit over an Index Scan → stops after 20.       FAST
     Limit over a Sort        → sorts ALL rows first. SLOW
     An index matching your ORDER BY converts the second into the first.
     That single fact explains most pagination performance problems.

TWO KINDS OF "LOCK" THAT SHARE A NAME
│
├── HEAVYWEIGHT LOCKS (lock table)   relations, transactions, advisory.
│    Visible in pg_locks. Held to end of transaction. Deadlock-detected.
└── LIGHTWEIGHT LOCKS / LATCHES      buffer content, WAL insert, hash
     partitions. Microseconds. Not in pg_locks. Not deadlock-checked
     (ordering is enforced by code). Visible as wait_event_type = LWLock.

WHERE TIME ACTUALLY GOES  (typical warm OLTP query, 0.4 ms total)
   pool acquire      ~0–∞ µs   ⚠ often the biggest, and invisible to the DB
   parse+analyse      ~60 µs
   plan              ~200 µs   ⚠ can dominate for simple queries!
   snapshot+locks      ~5 µs
   execute (warm)    ~100 µs
   result encoding    ~30 µs
   ⇒ For a fast query, PLANNING is frequently the largest server-side cost.
     This is the entire argument for prepared statements.
```

---

## Diagrams

**Diagram 1 — big picture: the five phases**

```
   SQL TEXT
      │
      ▼
 ┌──────────┐   raw parse tree    ┌──────────┐   query tree
 │  PARSER  │ ──────────────────▶ │ ANALYSER │ ──────────────┐
 └──────────┘                     └──────────┘               │
   grammar only                    names, types, perms       │
                                                             ▼
 ┌──────────┐   plan tree    ┌──────────┐   query tree  ┌──────────┐
 │ EXECUTOR │ ◀───────────── │ PLANNER  │ ◀──────────── │ REWRITER │
 └────┬─────┘                └──────────┘               └──────────┘
      │                       cost-based                views, RLS
      │                       ↑ pg_statistic
      ▼
  ┌────────────────────────────────────────────┐
  │  buffer pool ◀──▶ disk   ·   lock table    │
  │  snapshot (proc array)   ·   WAL           │
  └────────────────────────────────────────────┘
      │
      ▼
   ROWS → wire → Node.js
```

**Diagram 2 — data flow: read path vs write path**

```
  READ                                  WRITE
  ────────────────────────────          ────────────────────────────────
  parse/analyse/rewrite/plan            parse/analyse/rewrite/plan
      │                                     │
  SNAPSHOT ◀── proc array               XID assigned
      │                                     │
  AccessShareLock                       RowExclusiveLock (table)
      │                                     │
  index descend ──┐                     TUPLE LOCK (t_xmax in the row!)
                  ▼                         │
            BUFFER POOL                 WAL record → WAL buffer (RAM)
             hit/miss                       │
                  │                     heap page modified (RAM, dirty)
            heap fetch                      │
                  │                     index entries (unless HOT)
       MVCC visibility check                │
       (may read pg_xact,                COMMIT ──▶ fsync WAL ──▶ DURABLE
        may DIRTY the page                   │
        writing hint bits)                ack to client
                  │                          │
              rows out                  (later) checkpointer writes
                                        dirty pages to the heap file
```

**Diagram 3 — before/after: the state of the world across one transaction**

```
                    │ BEFORE BEGIN │ AFTER UPDATE │ AFTER COMMIT │ AFTER CKPT
 ───────────────────┼──────────────┼──────────────┼──────────────┼───────────
 proc array         │      —       │  XID 91012   │      —       │     —
 lock table         │      —       │ RowExclusive │      —       │     —
 tuple t_xmax       │      0       │    91012     │    91012     │   91012
 buffer page        │    clean     │    DIRTY     │    DIRTY     │   clean
 WAL buffer (RAM)   │      —       │   1 record   │   flushed    │     —
 pg_wal on disk     │      —       │      —       │  ✓ RECORD    │ ✓ RECORD
 pg_xact            │      —       │   in-prog    │  COMMITTED   │ COMMITTED
 HEAP FILE ON DISK  │     old      │     old      │  ⚠ STILL OLD │   NEW
 visible to others  │      no      │      no      │     YES      │    YES
 survives a crash   │      no      │      no      │     YES      │    YES

 ★ The two starred facts are the whole of durability:
   • "survives a crash" flips to YES at COMMIT (because of the WAL fsync)
   • "HEAP FILE ON DISK" updates LATER (because of the checkpointer)
   Recovery replays the WAL to close that gap. (Topics 41, 42.)
```

---

## Example 1 — basic

Instrument every stage of one query.

```sql
CREATE TABLE orders (
  id bigserial PRIMARY KEY,
  user_id bigint NOT NULL,
  total_paise bigint NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
INSERT INTO orders (user_id, total_paise, created_at)
SELECT (random()*50000)::bigint, (random()*500000)::bigint,
       now() - (random()*365)::int * interval '1 day'
FROM generate_series(1, 3000000);
VACUUM ANALYZE orders;

-- STAGE-BY-STAGE, no index yet:
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, TIMING)
SELECT id, total_paise, created_at FROM orders
WHERE user_id = 4471 AND created_at > now() - interval '30 days'
ORDER BY created_at DESC LIMIT 20;
```
```
Limit  (actual time=284.1..284.1 rows=5 loops=1)
  ->  Sort  (actual time=284.1..284.1 rows=5 loops=1)
        Sort Key: created_at DESC
        Sort Method: quicksort  Memory: 25kB
        ->  Seq Scan on orders  (actual time=0.4..283.9 rows=5 loops=1)
              Filter: ((created_at > ...) AND (user_id = 4471))
              Rows Removed by Filter: 2999995        ← the executor's real work
              Buffers: shared hit=1240 read=18368    ← 153 MB of I/O
Planning Time: 0.181 ms
Execution Time: 284.2 ms
```

Read it stage by stage:
- **Planning 0.181 ms** — the planner did its job fast; not the problem.
- **Seq Scan** — the planner had no better option.
- **`Rows Removed by Filter: 2999995`** — the executor evaluated the predicate 3 million times. That's CPU.
- **`Buffers: read=18368`** — 143 MB pulled through the buffer pool, evicting whatever was there.
- **`Sort`** — a *blocking* node. `LIMIT 20` saved nothing, because the sort had to see every matching row first.

Now change the access path:

```sql
CREATE INDEX idx_orders_user_created ON orders (user_id, created_at DESC);
VACUUM ANALYZE orders;
EXPLAIN (ANALYZE, BUFFERS) SELECT id, total_paise, created_at FROM orders
WHERE user_id = 4471 AND created_at > now() - interval '30 days'
ORDER BY created_at DESC LIMIT 20;
```
```
Limit  (actual time=0.031..0.037 rows=5 loops=1)
  ->  Index Scan using idx_orders_user_created on orders
        Index Cond: ((user_id = 4471) AND (created_at > ...))
        Buffers: shared hit=8
Planning Time: 0.214 ms
Execution Time: 0.051 ms
```

**Every stage changed:**
- The **Sort node is gone** — the index is already in `created_at DESC` order, so the plan satisfies `ORDER BY` for free. The blocking node became a streaming one.
- `Buffers: 18,368 → 8` — the buffer pool stopped being disturbed.
- `Rows Removed by Filter` is gone — the executor evaluates nothing; the index condition does the filtering during the descent.
- **`Planning Time` (0.214 ms) is now 4× the `Execution Time` (0.051 ms).** For queries this fast, planning is the dominant server cost — which is the empirical argument for prepared statements.

Prove that last point:

```sql
PREPARE q(bigint) AS SELECT id, total_paise, created_at FROM orders
  WHERE user_id = $1 AND created_at > now() - interval '30 days'
  ORDER BY created_at DESC LIMIT 20;
EXPLAIN (ANALYZE) EXECUTE q(4471);   -- run 6+ times
```
```
Planning Time: 0.008 ms      ← plan reused; 26× cheaper
Execution Time: 0.049 ms
```

And prove the hint-bit surprise:

```sql
CREATE TABLE hb AS SELECT i FROM generate_series(1,500000) i;
CHECKPOINT;
SELECT pg_current_wal_lsn() AS b \gset
SELECT count(*) FROM hb;                                    -- a READ...
SELECT pg_size_pretty(pg_current_wal_lsn() - :'b'::pg_lsn); -- ...that generated WAL
```
```
 pg_size_pretty
----------------
 3400 kB           ← a SELECT wrote 3.4 MB. Hint bits, dirtying pages.
```

---

## Example 2 — production scenario

**The situation.** `POST /checkout` p99 is 3.2 seconds. The DBA says "the database looks fine — no slow queries." Your APM says the DB call takes 40 ms. Both are telling the truth, and the endpoint is still slow.

**Walk the trace, stage by stage. This is the actual method.**

**Stage 0 — pool acquisition (before the database sees anything).**

```js
setInterval(() => {
  logger.info({ total: pool.totalCount, idle: pool.idleCount,
                waiting: pool.waitingCount });
}, 1000);
```
```
{ total: 20, idle: 0, waiting: 47 }
```

**There it is.** 47 requests queued for a connection. The database is idle; the *pool* is the bottleneck. `pool.max = 20` across 40 pods = 800 connections attempted against PgBouncer, but each pod only allows 20 concurrent DB operations, and the handler holds a connection for 3 seconds.

**Why 3 seconds?** Look at the handler:

```js
// THE BUG
const client = await pool.connect();
await client.query('BEGIN');
await client.query('UPDATE products SET stock = stock-1 WHERE id=$1', [pid]);
const payment = await razorpay.orders.create({ amount });   // ⚠ 200–2800 ms
await client.query('INSERT INTO orders (...) VALUES (...)', [...]);
await client.query('COMMIT');
client.release();
```

**The external HTTP call is inside the transaction.** So each request holds:
1. a Node.js pool slot,
2. a PgBouncer server connection,
3. a **row lock** on `products.id = pid` (written into `t_xmax` — step 5 of Trace B),
4. an **open snapshot** blocking vacuum cluster-wide,

for the entire 2.8-second round trip to Razorpay.

**The cascade, mapped to the trace stages:**

```
 STAGE                    WHAT'S WRONG
 ─────────────────────────────────────────────────────────────────────
 0  pool acquire          47 waiting — connections held 3 s each
 5  tuple lock            other buyers of the same product block on
                          wait_event = transactionid
 6  snapshot held open    autovacuum can't reclaim ANY dead tuple newer
                          than the oldest of these snapshots → bloat →
                          working set grows → hit ratio falls (Topic 07)
 11 COMMIT                fine — but reached 3 s too late
```

Confirm each on the server:

```sql
-- stage 5: lock waits
SELECT count(*) FROM pg_stat_activity WHERE wait_event_type='Lock';
-- 34

-- stage 6: snapshots pinning vacuum
SELECT max(now() - xact_start) AS longest_txn,
       count(*) FILTER (WHERE state='idle in transaction') AS idle_in_txn
FROM pg_stat_activity;
-- longest_txn: 00:00:02.9 | idle_in_txn: 18

-- the consequence
SELECT relname, n_dead_tup, last_autovacuum FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC LIMIT 3;
-- products | 8402118 | 2026-07-29 03:12   ← 9 days ago
```

**The fix, restructured around the trace:**

```js
// 1. RESERVE — a short transaction. Lock held ~2 ms, not 2800 ms.
const { rows } = await pool.query(
  `INSERT INTO stock_reservations (product_id, user_id, expires_at, idempotency_key)
   SELECT $1, $2, now() + interval '15 minutes', $4
     FROM products WHERE id = $1 AND stock > 0
   ON CONFLICT (idempotency_key) DO NOTHING
   RETURNING id`, [pid, userId, null, idemKey]);
if (!rows.length) return { status: 409, error: 'OUT_OF_STOCK' };

// 2. EXTERNAL CALL — NO transaction, NO connection held.
const payment = await razorpay.orders.create({ amount });

// 3. CONFIRM — another short transaction.
await withTransaction(async (tx) => {
  await tx.query('UPDATE products SET stock = stock-1 WHERE id=$1 AND stock>0', [pid]);
  await tx.query('UPDATE stock_reservations SET state=$2 WHERE id=$1', [rows[0].id, 'confirmed']);
  await tx.query('INSERT INTO orders (...) VALUES (...)');
  await tx.query(`INSERT INTO outbox (topic,payload) VALUES ('order.created',$1)`, [payload]);
});
```

Plus two guardrails that would have contained the blast radius from day one:

```sql
ALTER ROLE app SET idle_in_transaction_session_timeout = '5s';
ALTER ROLE app SET lock_timeout = '3s';
ALTER ROLE app SET statement_timeout = '15s';
```

**Result:** p99 3.2 s → 90 ms. Connection hold time 2,900 ms → 4 ms. `waiting` → 0. Bloat stops growing. **Not one query was optimised.** The fix was entirely about *which stages of the trace held which resources for how long* — which you can only see if you know the stages.

---

## Common mistakes

**1. Measuring the query but not the connection.**
- *Symptom:* "the DB call is 40 ms" but the endpoint is 3 s.
- *Why:* pool acquisition happens before any database metric exists. It is invisible to `pg_stat_statements`, `EXPLAIN`, and every DB dashboard.
- *Diagnose:* log `pool.waitingCount`; instrument the time between "handler start" and "query sent."
- *Fix:* shorter transactions first, then pool sizing (Topic 65).

**2. Using bare `EXPLAIN` instead of `EXPLAIN (ANALYZE, BUFFERS)`.**
- *Symptom:* "the plan looks fine" while the query is slow.
- *Why:* bare `EXPLAIN` shows *estimates* only — no actual rows, no timing, no I/O. The gap between estimated and actual rows is the single most diagnostic number available, and bare `EXPLAIN` cannot show it.
- *Fix:* always `EXPLAIN (ANALYZE, BUFFERS)`. On production writes, wrap in `BEGIN; ... ROLLBACK;` — `ANALYZE` actually executes the statement.

**3. Not knowing which plan nodes block.**
- *Symptom:* `LIMIT 10` is slow; adding `LIMIT` didn't help.
- *Why:* `Limit` over `Sort` must materialise and sort everything first. `Limit` over `Index Scan` stops early.
- *Diagnose:* look for a `Sort` node under your `Limit` in `EXPLAIN`.
- *Fix:* an index whose order matches your `ORDER BY` — this converts a blocking pipeline into a streaming one, and it is the fix for nearly all slow pagination.

**4. Holding a transaction across an external call.**
- *Symptom:* lock waits, `idle in transaction`, unbounded bloat, connection exhaustion — all at once.
- *Why:* row locks (`t_xmax`) and snapshots are both held to end of transaction. A 3-second HTTP call is a 3-second lock.
- *Diagnose:* `SELECT max(now()-xact_start) FROM pg_stat_activity WHERE state='idle in transaction';`
- *Fix:* restructure into reserve → external call → confirm. Set `idle_in_transaction_session_timeout`.

**5. Ignoring planning time on fast queries.**
- *Symptom:* a query with `Execution Time: 0.05 ms` and `Planning Time: 4 ms`, called 20,000 times/sec.
- *Why:* planning is real work — enumerating paths, consulting statistics. On partitioned tables with 200 partitions, or 8-way joins, it can take tens of milliseconds.
- *Fix:* prepared statements (node-postgres: pass a `name` to `query()`), `plan_cache_mode`, and fewer partitions.

**6. Being surprised that a SELECT wrote to disk.**
- *Symptom:* read-only workload generating WAL and dirty pages.
- *Why:* hint bits. The first read of a tuple after its inserting transaction commits caches the commit status in the tuple header, dirtying the page. Also: index-only scans set visibility-map bits; `SELECT` can trigger HOT pruning.
- *Fix:* nothing to fix — but `VACUUM` after bulk loads sets hint bits and the visibility map in one pass, so the first user query doesn't pay for it.

---

## Hands-on proof

**PROVE IT #1 — see all five phases separately.**
```sql
EXPLAIN (ANALYZE, BUFFERS, VERBOSE, SETTINGS)
SELECT * FROM orders WHERE user_id = 4471 ORDER BY created_at DESC LIMIT 20;
```
`Planning Time` = phases 1–4. `Execution Time` = phase 5. `Buffers` = the storage layer.

**PROVE IT #2 — prepared statements collapse planning time.**
```sql
PREPARE p(bigint) AS SELECT * FROM orders WHERE user_id=$1 LIMIT 20;
EXPLAIN (ANALYZE) EXECUTE p(1);     -- custom plan, planning ~0.2 ms
EXPLAIN (ANALYZE) EXECUTE p(2);
EXPLAIN (ANALYZE) EXECUTE p(3);
EXPLAIN (ANALYZE) EXECUTE p(4);
EXPLAIN (ANALYZE) EXECUTE p(5);
EXPLAIN (ANALYZE) EXECUTE p(6);     -- generic plan kicks in, planning ~0.005 ms
```

**PROVE IT #3 — row locks live in the tuple, not the lock table.**
```sql
-- session 1
BEGIN; UPDATE products SET stock = stock WHERE id = 88;
SELECT txid_current();          -- e.g. 91012
-- session 2
CREATE EXTENSION IF NOT EXISTS pageinspect;
SELECT t_xmax, t_infomask FROM heap_page_items(get_raw_page('products',0))
WHERE t_xmax <> 0;
-- t_xmax = 91012  ← the lock IS the row
SELECT count(*) FROM pg_locks WHERE locktype = 'tuple';
-- 0  ← no tuple lock in the lock table at all
SELECT locktype, transactionid FROM pg_locks WHERE locktype='transactionid';
-- ONE entry per transaction, regardless of how many rows it locked
```

**PROVE IT #4 — watch a query wait on each different thing.**
```sql
-- terminal 1: hold a lock
BEGIN; UPDATE products SET stock=stock WHERE id=88;
-- terminal 2: try to update it
UPDATE products SET stock=stock WHERE id=88;    -- hangs
-- terminal 3:
SELECT pid, state, wait_event_type, wait_event, now()-query_start AS waited,
       left(query,50) FROM pg_stat_activity WHERE state='active';
```
```
 pid | state  | wait_event_type |  wait_event   |  waited  | query
-----+--------+-----------------+---------------+----------+---------------
 918 | active | Lock            | transactionid | 00:00:14 | UPDATE products
```

**PROVE IT #5 — the visibility map turns a heap fetch into nothing.**
```sql
CREATE INDEX idx_cover ON orders (user_id) INCLUDE (total_paise);
VACUUM ANALYZE orders;   -- sets the visibility map
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, total_paise FROM orders WHERE user_id = 4471;
```
```
Index Only Scan using idx_cover on orders
  Heap Fetches: 0                    ← step 10 of Trace A skipped entirely
  Buffers: shared hit=5
```
Now make it stale and watch it come back:
```sql
UPDATE orders SET total_paise = total_paise WHERE user_id = 4471;
EXPLAIN (ANALYZE) SELECT user_id, total_paise FROM orders WHERE user_id=4471;
-- Heap Fetches: 5    ← the VM bits were cleared; the heap must be consulted
```

**PROVE IT #6 — see the snapshot itself.**
```sql
BEGIN;
SELECT pg_current_snapshot();      -- e.g. 91004:91011:91007,91009
--                                       xmin :xmax :in-progress list
SELECT txid_current_if_assigned(); -- NULL — read-only, no XID yet
UPDATE products SET stock=stock WHERE id=88;
SELECT txid_current_if_assigned(); -- 91012 — assigned lazily on first write
ROLLBACK;
```

---

## The design decision framework

This topic is a diagnostic method rather than a design choice. The decision it supports is: **where do I spend my next hour?**

```
GIVEN: "this endpoint is slow." Walk the trace IN THIS ORDER.
Each step is cheap, and each one rules out everything above it.

 1. IS IT THE DATABASE AT ALL?
      Compare endpoint p99 with DB time from your APM.
      Gap → pool acquisition, serialization, N+1, or JS work.
      → log pool.waitingCount FIRST. It is free and it is often the answer.

 2. IS IT ONE QUERY OR MANY?
      SELECT calls, total_exec_time, mean_exec_time FROM pg_stat_statements
      ORDER BY total_exec_time DESC;
      High `calls`, low `mean` → N+1 (Topic 66). Fix the loop, not the query.
      Low `calls`, high `mean`  → one bad query. Continue to 3.

 3. IS IT WAITING OR WORKING?
      SELECT wait_event_type, count(*) FROM pg_stat_activity
      WHERE state='active' GROUP BY 1;
      `Lock`   → contention. Topics 45, 48, 49. STOP — no plan tuning helps.
      `IO`     → buffer misses. Go to 5.
      (null)   → actually computing. Go to 4.

 4. IS IT PLANNING OR EXECUTING?
      EXPLAIN (ANALYZE) — compare Planning Time vs Execution Time.
      Planning dominates → prepared statements, fewer partitions (Topic 18).
      Execution dominates → go to 5.

 5. IS IT I/O OR CPU?
      EXPLAIN (ANALYZE, BUFFERS).
      `read` ≫ `hit`            → I/O bound. RAM, narrower rows, covering
                                  index, or fewer rows touched (Topics 07, 12).
      `hit` high, `read` low    → CPU bound. Look at `Rows Removed by Filter`
                                  and blocking Sort/Hash nodes.

 6. IS THE PLAN WRONG?
      Compare estimated vs actual rows at EVERY node.
      Off by >10×  → statistics problem. ANALYZE; raise
                     default_statistics_target; add extended statistics
                     for correlated columns (Topic 15).
      Estimates good, plan still bad → cost constants. Check
                     random_page_cost (should be ~1.1 on SSD) (Topic 07).

 THE SIGNAL TO LOOK FOR — one number above all others:
      The ratio of ESTIMATED to ACTUAL rows on the deepest node.
      If the planner thinks 12 rows and gets 400,000, every decision
      above that node is wrong, and no amount of index tuning fixes it.
      Fix the statistics first. Always.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Run one query with `EXPLAIN (ANALYZE, BUFFERS, VERBOSE)`. Map each number in the output to a stage in Trace A. Specifically identify: which stage `Planning Time` covers, which stage `Buffers: hit/read` covers, and which stage `Rows Removed by Filter` covers. Then state which stage produced the *most* time and what you'd do about it.

### Exercise 2 — medium (apply it)
Build a table of 2M rows. Write a paginated query (`ORDER BY created_at DESC LIMIT 20 OFFSET 40000`) and measure it. Then:
(a) explain, using the pull-pipeline model, why `OFFSET 40000` is expensive even with a perfect index,
(b) rewrite it using keyset pagination,
(c) measure both and explain the difference in terms of which nodes block and how many tuples each pulls,
(d) state the one situation where `OFFSET` is still acceptable.

### Exercise 3 — hard (production simulation)
An endpoint's p99 is 4.5 s. You are given: APM says DB time is 60 ms; `pg_stat_statements` shows no query over 20 ms mean; CPU on the DB is 20%; disk is idle; `pg_stat_activity` shows 90 connections, 62 of them `idle in transaction`.

(a) Which trace stage is the bottleneck? Justify from the evidence, and say which stages you have *ruled out* and how.
(b) Name three distinct code patterns that produce exactly this signature.
(c) `idle in transaction` connections cause damage in at least four separate ways. Name all four, each with the mechanism.
(d) Write the three PostgreSQL settings that bound this failure mode, with values, and explain what each one prevents.
(e) The team proposes raising `pool.max` from 20 to 200. Explain precisely why that makes things worse, referencing the process model from Topic 02.
(f) Design the dashboard panel that would have made this obvious in week one, and say why standard DB metrics would not have.

---

## Mental model checkpoint

1. Name the five phases of query processing in order. Which one makes cost-based decisions?
2. At which exact step does a write become durable? What is on disk at that moment, and what is not?
3. Where is a row lock physically stored in PostgreSQL, and why is that design important?
4. Why can a `SELECT` generate WAL and dirty pages?
5. Explain why `LIMIT 20` is free over an `Index Scan` but not over a `Sort`. What plan change fixes it?
6. A query has `Planning Time: 4 ms` and `Execution Time: 0.05 ms`, called 20k/s. What do you do?
7. Walk the six-step diagnostic order from the framework, from memory. What single number matters most, and why?

---

## Quick reference card

**The five phases:** Parse → Analyse → Rewrite → **Plan** → Execute

**Read path:** pool → parse/plan → **snapshot** → lock → index descend → heap fetch → **MVCC visibility** → filter → encode → wire

**Write path:** parse/plan → **XID** → table lock → **tuple lock (t_xmax)** → WAL record → dirty buffer → index entries → **COMMIT = fsync WAL** → ack → (later) checkpoint

| Stage | Typical warm cost | Bottleneck symptom |
|---|---|---|
| Pool acquire | 0 µs | `waitingCount > 0` — invisible to the DB |
| Parse + analyse | ~60 µs | huge statements, big `IN` lists |
| Plan | ~200 µs | high `Planning Time`; many partitions |
| Snapshot | ~2 µs | — |
| Locks | ~3 µs | `wait_event_type = Lock` |
| Index descend | ~3 pages | — |
| Heap fetch | 1 page/row | `Buffers: read` ≫ `hit` |
| Visibility | ~0 (hint bits) | first read after bulk load |
| Encode + wire | ~30 µs | wide rows, many rows |
| COMMIT fsync | 0.3–1 ms | slow commits, disk-bound writes |

**Blocking vs streaming nodes**

| Blocking (must consume all input) | Streaming (emits as it goes) |
|---|---|
| Sort, Hash (build side), Materialize, ungrouped Aggregate | Seq Scan, Index Scan, Nested Loop, Append, Filter, Limit |

**The six diagnostic questions, in order**
1. Is it the database at all? (`pool.waitingCount`)
2. One query or many? (`pg_stat_statements`)
3. Waiting or working? (`wait_event_type`)
4. Planning or executing? (`EXPLAIN ANALYZE`)
5. I/O or CPU? (`BUFFERS`)
6. Is the plan wrong? (**estimated vs actual rows**)

---

## When would I use this at work?

1. **Any latency incident, every time.** The six-question order turns a vague "the DB is slow" into a specific answer in about four minutes, and — just as valuable — it tells you what to *stop* investigating.

2. **Code review of a transaction.** You see an `await` on anything non-database between `BEGIN` and `COMMIT` and can immediately name the four resources being held (pool slot, server connection, row lock, snapshot) and the four downstream failures. That's a comment that saves an outage.

3. **Explaining a fix to your team.** "We added an index" is forgettable. "The `ORDER BY` forced a blocking Sort node that had to materialise 400k rows before `LIMIT 20` could discard them; the composite index makes the scan return rows already ordered, so the pipeline streams and stops after 20" is a lesson the team keeps.

---

## Connected topics

**Understand before this:** 01–08. This topic assembles all of Phase 1.

**This unlocks — all of Phase 2, which optimises individual stages:**
- **10–17** — the index scan stage: making the descent and the heap fetch cheaper
- **12** — index-only scans and the visibility map (step 10a)
- **18** — the planner stage in full, and reading `EXPLAIN` properly
- **19** — the join and sort nodes in the executor
- **41, 42** — the WAL and commit stages
- **45, 46** — the lock and snapshot stages
- **65** — the pool-acquisition stage
- **67** — the full performance methodology, of which the six questions here are the core
