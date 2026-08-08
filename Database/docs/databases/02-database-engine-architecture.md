# 02 — Database Engine Architecture
## Phase: Storage Internals

---

## ELI5 — The Simple Analogy

A busy restaurant kitchen.

A waiter takes your order in English ("something spicy, no dairy") — that's the **parser**, turning words into a ticket the kitchen understands. The head chef looks at the ticket and decides *how* to make it: fry it or bake it, use the big oven or the small one, which is faster given what's already cooking — that's the **query planner**. The line cooks actually cook it — the **executor**. The pantry keeps frequently-used ingredients on the counter instead of running to the cold store every time — the **buffer pool**. There is a rule that only one cook may touch the one remaining chicken — the **lock manager**. And a runner writes every order into a paper logbook *before* cooking starts, so if the kitchen catches fire you know exactly what was in progress — the **WAL writer**.

The kitchen is not one thing. It is seven specialists with strict handoffs. Debugging a slow restaurant means knowing *which specialist* is the bottleneck. Same for a database.

---

## Where this fits in the big picture

```
   01 What is a DB
   (four problems)
         │
         ▼
   02 ENGINE ARCHITECTURE  ← YOU ARE HERE
   (which component owns which problem)
         │
         ├── buffer pool details      → 07
         ├── storage layer details    → 04, 05, 06, 08
         ├── planner details          → 18, 19
         ├── lock manager details     → 45, 48
         ├── transaction manager      → 39, 40, 46
         └── WAL writer               → 41, 42
         │
         ▼
   09 The life of a query
   (all components, one trace)
```

This topic is the **map**. Every later phase is a zoom-in on one box of this map.

---

## What is this?

A database engine is not a single program loop — it is a set of cooperating subsystems, each owning one responsibility, communicating through shared memory. Understanding the boundaries between them tells you where any given symptom comes from.

The critical structural fact for PostgreSQL: **one OS process per client connection** (the *backend*), plus a fixed set of shared **background processes**, all coordinating through one region of **shared memory**.

---

## Why does it matter for a backend developer?

Because "the database is slow" is not a diagnosis, and every real fix belongs to exactly one component:

| Symptom | Component at fault | The fix lives in |
|---|---|---|
| Query does a sequential scan on 4M rows | Planner / missing access path | Phase 2 |
| Query is fast the second time, slow the first | Buffer pool (cold cache) | Topic 07 |
| Query hangs for 30 seconds then succeeds | Lock manager (blocked on another txn) | Topic 45 |
| Commits are slow, CPU idle, disk busy | WAL writer / fsync | Topic 41 |
| Table is 8GB but holds 400MB of live rows | Transaction manager + vacuum (MVCC debris) | Topic 47 |
| `too many clients already` | Process model (one process per connection) | Topic 65 |

If you can name the box, you can name the fix. If you can't, you add a cache and hope.

---

## The physical reality

Here is what a running PostgreSQL actually looks like to the operating system:

```bash
$ ps -ef | grep postgres
postgres  1  postmaster                                    ← the supervisor
postgres 21  postgres: checkpointer                        ← background: flushes dirty pages
postgres 22  postgres: background writer                   ← background: trickles pages out
postgres 23  postgres: walwriter                           ← background: flushes WAL buffer
postgres 24  postgres: autovacuum launcher                 ← background: cleans dead tuples
postgres 25  postgres: logical replication launcher
postgres 88  postgres: app shop 10.0.4.7(51422) idle       ← BACKEND for connection 1
postgres 89  postgres: app shop 10.0.4.9(51430) SELECT     ← BACKEND for connection 2
postgres 90  postgres: app shop 10.0.4.9(51431) idle in transaction   ← ⚠ see mistakes
```

And the shared memory region they all attach to:

```
SHARED MEMORY (size = shared_buffers + overhead, e.g. 4GB)
┌──────────────────────────────────────────────────────────────┐
│ SHARED BUFFER POOL                          ~ shared_buffers │
│  ┌────────┬────────┬────────┬────────┐                       │
│  │ 8KB pg │ 8KB pg │ 8KB pg │  ...   │  + one descriptor per │
│  └────────┴────────┴────────┴────────┘    buffer (dirty flag,│
│                                            pin count, usage) │
├──────────────────────────────────────────────────────────────┤
│ WAL BUFFERS                                  ~ 16MB          │
├──────────────────────────────────────────────────────────────┤
│ LOCK TABLE          (hash: relation/page/tuple → lock modes) │
├──────────────────────────────────────────────────────────────┤
│ PROC ARRAY          (every live backend: pid, xid, snapshot) │
├──────────────────────────────────────────────────────────────┤
│ CLOG / SLRU BUFFERS (transaction commit status cache)        │
└──────────────────────────────────────────────────────────────┘
                              │
                              ▼
                    DISK: base/*, pg_wal/*, pg_xact/*
```

Per-backend **private** memory (not shared) — this is why `work_mem` is dangerous:

```
EACH BACKEND PROCESS (×N connections)
┌─────────────────────────────────────────────┐
│ work_mem        per sort / hash node!        │  4MB default
│ maintenance_work_mem   (VACUUM, CREATE INDEX)│  64MB default
│ temp_buffers    (temp tables)                │
│ catalog cache, plan cache, parse trees       │
└─────────────────────────────────────────────┘

⚠ 200 connections × 3 sort nodes × work_mem 64MB = 38 GB. This is a real outage.
```

---

## How it works — step by step

The full path of `SELECT * FROM orders WHERE user_id = 42;`

```
 1. LIBPQ / WIRE PROTOCOL
    Your `pg` client sends a Query message over TCP. The postmaster already
    forked a backend for this connection at connect time.
    OWNER: the backend process.

 2. PARSER
    SQL text → raw parse tree. Pure grammar; `orders` is still just a string.
    READS: nothing. FAILS ON: syntax errors.

 3. ANALYSER / REWRITER
    Names resolved against the catalog: orders → OID 16391, user_id → attnum 3.
    Views expanded, row-level security policies injected, rules applied.
    READS: pg_class, pg_attribute, pg_rewrite (via catalog cache, usually RAM).
    FAILS ON: "relation does not exist", "column does not exist", permissions.

 4. PLANNER / OPTIMISER
    Enumerates access paths:
       Seq Scan on orders            cost=0..21000  rows=4,000,000
       Index Scan using idx_user_id  cost=0..8.4    rows=12
    Picks the cheapest using statistics from pg_statistic.
    READS: pg_statistic, pg_class.reltuples/relpages, pg_index.
    OUTPUT: a plan tree of executor nodes.

 5. EXECUTOR
    Walks the plan tree, pulling one tuple at a time (the "volcano" model):
       Index Scan node asks the storage layer for the next matching heap tuple.
    OWNER of: joins, sorts, aggregates, and the per-node work_mem allocations.

 6. TRANSACTION MANAGER
    Supplies the snapshot: "which transaction IDs were committed when I started?"
    Every tuple the executor touches is checked against this snapshot.
    READS: proc array (shared memory), pg_xact.

 7. LOCK MANAGER
    AccessShareLock taken on `orders` — blocks only DDL, not other readers/writers.
    READS/WRITES: the shared lock table.

 8. BUFFER MANAGER
    Executor asks for page 1,204 of relation 16391.
      HIT  → return the pointer, bump usage count.       ~100 ns
      MISS → find a victim buffer, flush it if dirty, read 8KB from disk. ~100 µs (SSD)
    READS: shared buffer pool, then base/16388/16391.

 9. STORAGE / ACCESS METHOD (heapam, nbtree)
    Interprets the raw 8KB page: item pointers → tuples → columns.

10. RESULT
    Tuples formatted per the wire protocol, streamed back to your Node.js client.

── MEANWHILE, INDEPENDENTLY (writes only) ────────────────────────────────
    WAL WRITER      flushes WAL buffers to disk periodically and at commit.
    CHECKPOINTER    every checkpoint_timeout (5 min), writes ALL dirty buffers
                    to disk and records a checkpoint record in the WAL.
    BGWRITER        continuously trickles a few dirty buffers out so backends
                    rarely have to flush one themselves to make room.
    AUTOVACUUM      wakes every minute, finds tables with enough dead tuples,
                    reclaims them, and refreshes planner statistics.
```

---

## Concept breakdown

```
QUERY PROCESSING LAYER  — turns intent into a plan
│
├── Parser        text → syntax tree.  Knows grammar, not your schema.
├── Analyser      syntax tree → query tree.  Resolves names, checks types & perms.
├── Rewriter      applies views, rules, row-level security.
├── Planner       query tree → plan tree.  THE ONLY component that makes
│                 cost-based *choices*. Everything else follows orders.
└── Executor      plan tree → tuples.  Pull-based, one node at a time.

STORAGE LAYER — turns pages into tuples and back
│
├── Buffer manager   the RAM cache of 8KB pages. Owns hit/miss, dirty, eviction.
├── Access methods   heapam (tables), nbtree/hash/gin/gist/brin (indexes).
├── Free space map   "which page has room for a 240-byte tuple?"
└── Visibility map   "is every tuple on this page visible to everyone?"

TRANSACTIONAL LAYER — makes concurrency and crashes safe
│
├── Transaction manager  assigns XIDs, builds snapshots, records commit status.
├── Lock manager         heavyweight locks (tables, rows) + lightweight latches.
├── WAL manager          appends redo records; fsyncs at commit.
└── Recovery manager     on startup, replays WAL from the last checkpoint.

PROCESS MODEL — who runs what
│
├── postmaster    supervisor; forks a backend per connection; restarts crashes.
├── backend       ONE PER CONNECTION. Does parse→plan→execute for that client.
└── background    checkpointer, bgwriter, walwriter, autovacuum, archiver, stats.
```

---

## Diagrams

**Diagram 1 — the big picture: the seven boxes**

```
 ┌──────────────────────────── POSTGRESQL INSTANCE ────────────────────────────┐
 │                                                                             │
 │  CLIENT ──TCP──▶ ┌──────────── BACKEND PROCESS ────────────┐                │
 │                  │ Parser → Analyser → Planner → Executor  │                │
 │                  └───────────────────┬─────────────────────┘                │
 │                                      │                                      │
 │  ┌───────────────────────────────────▼───────────────────────────────────┐  │
 │  │                          SHARED MEMORY                                │  │
 │  │  ┌────────────┐ ┌────────────┐ ┌────────────┐ ┌──────────────────┐    │  │
 │  │  │BUFFER POOL │ │LOCK MANAGER│ │ WAL BUFFER │ │ TXN MGR / PROCS  │    │  │
 │  │  └─────┬──────┘ └────────────┘ └─────┬──────┘ └──────────────────┘    │  │
 │  └────────┼──────────────────────────────┼───────────────────────────────┘  │
 │           │                              │                                  │
 │  ┌────────▼──────────┐   ┌───────────────▼────────┐   ┌──────────────────┐  │
 │  │ checkpointer      │   │ walwriter              │   │ autovacuum       │  │
 │  │ bgwriter          │   │ archiver               │   │ stats collector  │  │
 │  └────────┬──────────┘   └───────────────┬────────┘   └────────┬─────────┘  │
 └───────────┼──────────────────────────────┼─────────────────────┼────────────┘
             ▼                              ▼                     ▼
        base/*/16391                    pg_wal/*              pg_stat/*
        (heap + indexes)                (the log)
```

**Diagram 2 — the data flow of a read vs a write**

```
  READ PATH                              WRITE PATH
  ─────────────────────────              ────────────────────────────
  SQL                                    SQL
   │                                      │
  parse/plan                             parse/plan
   │                                      │
  executor ──ask for page──┐             executor
                           │              │
              ┌────────────▼───┐         WAL record → WAL buffer  ①
              │  BUFFER POOL   │          │
              └──┬──────────┬──┘         modify page in BUFFER POOL ② (dirty)
             HIT │          │ MISS        │
        (~100ns) │          ▼            COMMIT → fsync WAL to disk ③  ◀── DURABLE
                 │      read 8KB          │
                 │      from disk        return to client
                 │      (~100µs)          │
                 ▼                       (later) checkpointer writes
              tuples                      dirty page to heap file  ④
                 │
              client
```

**Diagram 3 — before / after a checkpoint (state change)**

```
BEFORE CHECKPOINT                        AFTER CHECKPOINT
────────────────────────────────         ─────────────────────────────────
buffer pool: 4,000 dirty pages           buffer pool: 0 dirty pages
pg_wal:      120 MB since last ckpt      pg_wal:      checkpoint record written
heap files:  stale by up to 5 min        heap files:  match the buffer pool
recovery:    would replay 120 MB         recovery:    would replay ~0

 ⇒ checkpoints trade steady-state I/O for shorter crash recovery.
   Frequent checkpoints = slow steady state, fast recovery.
   Rare checkpoints     = fast steady state, long recovery. (Topic 42)
```

---

## Example 1 — basic

See every component fire on one query.

```sql
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
CREATE TABLE orders (
  id bigserial PRIMARY KEY,
  user_id bigint NOT NULL,
  total_paise bigint NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
INSERT INTO orders (user_id, total_paise)
SELECT (random()*100000)::bigint, (random()*500000)::bigint
FROM generate_series(1, 2000000);

ANALYZE orders;                    -- refresh planner statistics (component: planner input)

EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE user_id = 42;
```
```
Seq Scan on orders  (cost=0.00..38471.00 rows=20 width=32)
                    (actual time=0.412..184.9 rows=17 loops=1)
  Filter: (user_id = 42)
  Rows Removed by Filter: 1999983
  Buffers: shared hit=1250 read=11221          ← BUFFER MANAGER speaking
Planning Time: 0.104 ms                        ← PLANNER
Execution Time: 185.2 ms                       ← EXECUTOR
```

Read it component by component:
- `Planning Time` — the planner considered its options.
- `Seq Scan` — the planner *chose* to read every page, because there is no index.
- `Buffers: shared hit=1250 read=11221` — the buffer manager found 1,250 pages in RAM and had to fetch 11,221 from disk.
- `Rows Removed by Filter: 1999983` — the executor evaluated the predicate 2 million times.

Now give the planner a better option:

```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = 42;
```
```
Index Scan using idx_orders_user_id on orders (cost=0.43..37.2 rows=20 width=32)
                                              (actual time=0.031..0.062 rows=17 loops=1)
  Index Cond: (user_id = 42)
  Buffers: shared hit=20                       ← 12,471 page reads → 20
Execution Time: 0.081 ms                       ← 185 ms → 0.08 ms
```

Nothing about the data changed. The **planner** got a new access path, and the **buffer manager** stopped doing 100MB of I/O.

---

## Example 2 — production scenario

**The situation.** Node.js checkout service, PostgreSQL on a `db.r6g.2xlarge` (8 vCPU, 64GB). At 6pm the API's p99 goes from 120ms to 14s. CPU is at 30%. Disk IOPS are flat. Nothing is obviously "busy."

**Wrong instinct:** scale up the instance. CPU is idle — it will not help.

**Component-by-component triage:**

```sql
-- 1. Is it the LOCK MANAGER? (waiting, not working)
SELECT pid, wait_event_type, wait_event, state,
       now() - query_start AS duration, left(query, 60)
FROM pg_stat_activity
WHERE state <> 'idle' ORDER BY duration DESC LIMIT 10;
```
```
 pid  | wait_event_type | wait_event | state  | duration | query
------+-----------------+------------+--------+----------+---------------------------
 8891 | Lock            | transactionid | active | 00:00:13 | UPDATE products SET stock...
 8892 | Lock            | transactionid | active | 00:00:13 | UPDATE products SET stock...
 8893 | Lock            | transactionid | active | 00:00:12 | UPDATE products SET stock...
 8455 |                 |            | idle in transaction | 00:14:22 | SELECT * FROM products WHERE id=99
```

There it is. `wait_event_type = Lock` means the backends are not computing — they are **queued**. And pid 8455 is `idle in transaction` for 14 minutes: a Node.js handler opened a transaction, called an external payment API, and is waiting on the network while holding a row lock on the hot product.

**The engine-level explanation:** the lock manager has a FIFO queue per lock. One holder that never commits stalls every waiter behind it. CPU is idle *because* nobody can run. No amount of vCPU fixes a queue.

**The fixes, by component:**

| Fix | Component | Effect |
|---|---|---|
| Never call an external API inside a transaction | app / txn manager | removes the 14-minute holder |
| `SET idle_in_transaction_session_timeout = '30s'` | txn manager | bounds the blast radius |
| `SET lock_timeout = '3s'` in the checkout path | lock manager | fail fast instead of piling up |
| Move stock decrement to the end of the txn | app | shortens lock hold time |

Full treatment in Topics 45 and 49. The point here: **the symptom (slow API) lived in the lock manager, not the planner, not the buffer pool, not the disk** — and only a component map tells you that in 60 seconds.

---

## Common mistakes

**1. Treating "one connection per request" as free.**
- *Symptom:* memory exhaustion, `too many clients`, throughput collapsing as concurrency rises.
- *Engine-level why:* the postmaster **forks a whole OS process** per connection, each with private catalog caches, plan caches, and up to `work_mem` per sort node. This is a fundamentally different cost model from thread-per-connection engines.
- *Diagnose:* `SELECT count(*), state FROM pg_stat_activity GROUP BY state;`
- *Fix:* a pooler (Topic 65). Rule of thumb: `max_connections` should be small (100–200) and a pooler should multiplex thousands of app connections onto it.

**2. Setting `work_mem` globally to a large value.**
- *Symptom:* OOM killer takes down the database under load; works fine in staging.
- *Engine-level why:* `work_mem` is **per node, per backend** — not per query, not global. A query with 4 hash joins and 2 sorts can allocate `6 × work_mem` in one backend.
- *Diagnose:* `EXPLAIN (ANALYZE)` and count `Sort`/`Hash` nodes; look for `Memory Usage:` lines.
- *Fix:* keep the global small (4–16MB); raise it per-session for known-heavy reporting queries only.

**3. Assuming background processes are optional noise.**
- *Symptom:* "we disabled autovacuum because it was using I/O." Three weeks later, tables are 10× their live size and every query is slow.
- *Engine-level why:* autovacuum is the *only* thing that reclaims dead tuples created by MVCC (Topic 46/47), and the only thing that refreshes planner statistics. Without it, both the storage layer and the planner degrade.
- *Fix:* tune it (make it more aggressive), never disable it.

**4. Blaming the planner for a buffer-pool problem (or vice versa).**
- *Symptom:* "the query is slow but EXPLAIN says the plan is fine."
- *Engine-level why:* a perfect plan still has to read pages. `Buffers: read=200000` with a good plan means cold cache or a working set larger than `shared_buffers`.
- *Diagnose:* always run `EXPLAIN (ANALYZE, BUFFERS)`, never bare `EXPLAIN`. Compare `hit` vs `read`.
- *Fix:* more RAM, a narrower index, or a covering index — depending on which.

**5. `idle in transaction` connections.**
- *Symptom:* rising bloat, blocked writers, replication lag, "why is my 10-row table 4GB?"
- *Engine-level why:* an open transaction holds a snapshot. Vacuum cannot remove any dead tuple newer than the **oldest live snapshot in the whole cluster** — one forgotten transaction freezes cleanup database-wide.
- *Diagnose:* `SELECT max(now() - xact_start) FROM pg_stat_activity WHERE state = 'idle in transaction';`
- *Fix:* `idle_in_transaction_session_timeout`, and never `await` I/O you don't control inside a transaction.

---

## Hands-on proof

**PROVE IT #1 — one process per connection.**
```bash
docker exec pg-lab ps -ef | grep "postgres:" | wc -l     # note the number
# open 3 psql sessions, then:
docker exec pg-lab ps -ef | grep "postgres:" | wc -l     # up by exactly 3
```

**PROVE IT #2 — see the background processes by name.**
```sql
SELECT pid, backend_type, state FROM pg_stat_activity ORDER BY backend_type;
```
```
 pid |         backend_type         | state
-----+------------------------------+--------
  24 | autovacuum launcher          |
  88 | client backend               | active
  21 | checkpointer                 |
  22 | background writer            |
  23 | walwriter                    |
```

**PROVE IT #3 — look inside the buffer pool.**
```sql
CREATE EXTENSION IF NOT EXISTS pg_buffercache;
SELECT c.relname, count(*) AS buffers,
       pg_size_pretty(count(*) * 8192) AS in_ram
FROM pg_buffercache b JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
GROUP BY c.relname ORDER BY buffers DESC LIMIT 5;
```
```
      relname       | buffers |  in_ram
--------------------+---------+---------
 orders             |   12471 | 97 MB
 idx_orders_user_id |    5486 | 43 MB
 products           |     892 | 7 MB
```

**PROVE IT #4 — watch the lock manager.**
```sql
-- session 1
BEGIN; UPDATE products SET stock = stock - 1 WHERE id = 1;   -- do NOT commit
-- session 2
UPDATE products SET stock = stock - 1 WHERE id = 1;          -- hangs
-- session 3
SELECT pid, wait_event_type, wait_event, left(query,40) FROM pg_stat_activity
WHERE wait_event_type = 'Lock';
```
```
 pid | wait_event_type |  wait_event   |           query
-----+-----------------+---------------+---------------------------
 912 | Lock            | transactionid | UPDATE products SET stock
```

**PROVE IT #5 — see the checkpointer's effect.**
```sql
SELECT checkpoints_timed, checkpoints_req, buffers_checkpoint, buffers_clean, buffers_backend
FROM pg_stat_bgwriter;
CHECKPOINT;      -- force one
SELECT checkpoints_req FROM pg_stat_bgwriter;   -- incremented
```
> `buffers_backend` high relative to `buffers_checkpoint` means backends are being forced to flush pages themselves — your bgwriter/checkpointer settings are too lazy for your write rate.

**PROVE IT #6 — planning vs execution time are separate components.**
```sql
EXPLAIN (ANALYZE) SELECT * FROM orders o JOIN orders o2 USING (user_id) LIMIT 1;
-- Planning Time and Execution Time reported independently.
-- A high Planning Time with low Execution Time = planner problem
-- (too many joins, too many partitions) — fix with prepared statements or fewer partitions.
```

---

## The design decision framework

This topic is a map rather than a trade-off, but there is one real decision in it: **where to spend memory.**

```
GIVE MEMORY TO shared_buffers WHEN:
  ✓ Your working set (hot tables + hot indexes) is bigger than current cache
  ✓ pg_stat_database shows blks_read high relative to blks_hit (<99% hit ratio)
  ✓ Workload is read-heavy OLTP with a stable hot set
  Typical: 25% of system RAM. Above ~40% you fight the OS page cache.

GIVE MEMORY TO work_mem WHEN:
  ✓ EXPLAIN ANALYZE shows "Sort Method: external merge  Disk: 240MB"
  ✓ Hash joins are spilling to disk (Batches > 1)
  ✓ ...and you can bound concurrency (set it per-session, not globally)

GIVE MEMORY TO maintenance_work_mem WHEN:
  ✓ CREATE INDEX / VACUUM are the bottleneck (maintenance windows)

THE SIGNAL TO LOOK FOR:
  Run EXPLAIN (ANALYZE, BUFFERS).
    High `read` vs `hit`         → shared_buffers / index design problem
    "external merge Disk:"       → work_mem problem
    High Planning Time           → planner problem (partitions, joins, no prepare)
    wait_event_type = Lock       → concurrency problem, memory won't help
    None of the above, high CPU  → the query itself is doing real work; rewrite it
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
List the background processes on your lab instance using `pg_stat_activity`. For each one, write a single sentence: what does it do, and what breaks if it stops?

### Exercise 2 — medium (apply it)
Run a query that returns 500,000 rows sorted by a non-indexed column. Using `EXPLAIN (ANALYZE, BUFFERS)`, identify (a) which component decided to sort, (b) which component supplied the pages, (c) whether the sort spilled to disk, and (d) the exact setting you'd change and to what value. Then change it in-session and prove the plan changed.

### Exercise 3 — hard (production simulation)
Your API's p99 is 9 seconds. `top` on the DB server shows 12% CPU and near-zero disk utilisation. Write the exact sequence of queries you would run — in order — to determine which engine component is responsible, and state what result at each step would rule that component in or out. Your answer must distinguish between: lock waits, an idle-in-transaction holder, buffer-pool misses, a bad plan, and connection saturation. Then say which of those five is *most likely* given "CPU idle, disk idle" and justify it in one sentence.

---

## Mental model checkpoint

1. Which single component makes cost-based *decisions*? What does every other component do instead?
2. What is the memory-model consequence of PostgreSQL forking one process per connection?
3. Is `work_mem` per query, per connection, or per plan node? What is the worst-case total?
4. Name the four main background processes and the one thing each is responsible for.
5. A query's `Planning Time` is 400ms and `Execution Time` is 2ms. Which component is the problem, and name two causes.
6. Why does a `CHECKPOINT` make crash recovery faster but steady-state throughput worse?
7. One backend sits `idle in transaction` for an hour. Name three separate things that degrade cluster-wide because of it.

---

## Quick reference card

| Component | Owns | Symptom when it's the bottleneck |
|---|---|---|
| Parser/Analyser | syntax, name resolution | errors, not slowness |
| Planner | choosing the access path | high Planning Time; wrong scan type |
| Executor | doing the work | high CPU, big Rows Removed by Filter |
| Buffer manager | RAM page cache | `Buffers: read` ≫ `hit` |
| Lock manager | serialising conflicting access | `wait_event_type = Lock`, idle CPU |
| Transaction manager | XIDs, snapshots, visibility | bloat, long-running snapshots |
| WAL manager | durability | slow COMMITs, disk-bound writes |
| Checkpointer/bgwriter | flushing dirty pages | I/O spikes every 5 min |
| Autovacuum | reclaiming dead tuples, stats | growing tables, degrading plans |

**Numbers to memorise**

| Setting | Default | Sane starting point |
|---|---|---|
| `shared_buffers` | 128 MB | 25% of RAM |
| `work_mem` | 4 MB | 4–16 MB global; raise per session |
| `maintenance_work_mem` | 64 MB | 512 MB–2 GB |
| `max_connections` | 100 | 100–200 + a pooler |
| `checkpoint_timeout` | 5 min | 15 min for write-heavy |
| Buffer hit latency | ~100 ns | |
| SSD page read | ~100 µs | ~1000× slower than a hit |

**Key diagnostic queries**

```sql
pg_stat_activity      -- who is running what, and what are they waiting on
pg_stat_bgwriter      -- checkpoint/bgwriter pressure
pg_buffercache        -- what is actually in RAM
pg_stat_statements    -- which queries cost the most in aggregate
pg_locks              -- the lock table, directly
```

---

## When would I use this at work?

1. **Capacity planning.** Your team wants to raise `max_connections` to 2,000 because the app pool is exhausting. Knowing that each connection is an OS process with private memory turns that into a clear "no — put PgBouncer in front" with a number attached.

2. **On-call triage.** The first query you run in any database incident is `pg_stat_activity` with `wait_event_type`. It tells you in one screen whether you have a lock problem, an I/O problem, or a CPU problem — which is the difference between a 5-minute fix and a 3-hour guess.

3. **Reviewing a config PR.** Someone bumps `work_mem` to 256MB in `postgresql.conf` to fix one slow report. You can explain that with 150 connections and multi-node plans this is a multi-gigabyte exposure, and propose `SET LOCAL work_mem` in that one job instead.

---

## Connected topics

**Understand before this:** 01 — What is a database (the four problems).

**This unlocks:**
- **04–08** — the storage layer boxes in detail
- **07** — the buffer pool box in detail
- **09** — all boxes traced at once on one query
- **18** — the planner box in detail
- **41, 45, 46** — the transactional layer boxes in detail
- **65** — the process model as an operational constraint
