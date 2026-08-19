# 47 — VACUUM, Dead Tuples and Bloat
## Phase: Transactions & Concurrency

---

## ELI5 — The Simple Analogy

A library where **removing a book from the shelf is forbidden while anyone might still be reading a catalogue that mentions it**.

So when a book is replaced by a new edition, the old one stays on the shelf with a sticky note: *"superseded on Tuesday."* The shelves fill with superseded books.

A librarian walks the aisles periodically. For each sticky-noted book they ask **one question**: *"Is there anybody in the building whose catalogue was printed before Tuesday?"* If no — the book comes off the shelf and **that slot is now free for a new book**.

Three things you must internalise:

1. **The slot is freed, not the shelf.** The library doesn't get smaller. A new book can go in that slot, but the building is the same size. That is `VACUUM`.
2. **To make the building smaller you must move everything and rebuild it.** That's `VACUUM FULL` — and the library closes while you do it.
3. **★ If even one person is sitting in the corner with a catalogue printed in March, the librarian can remove NOTHING newer than March.** Not one book. No matter how often they walk the aisles.

That third fact is the entire operational story of PostgreSQL bloat.

---

## Where this fits in the big picture

```
   46 MVCC — every UPDATE writes a new version; old ones accumulate
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 47 VACUUM & BLOAT ← YOU ARE HERE             │
        │ ★ the bill for MVCC. Not optional.           │
        └────────────────────┬─────────────────────────┘
                             ▼
        59 partitioning (DROP beats DELETE)
        62 replication (slots pin xmin)
        67 performance investigation
```

---

## What is this?

**Dead tuple** — a row version that no current or future snapshot can see. Garbage.

**Bloat** — dead tuples plus unusable free space, occupying disk and being read into the buffer pool on every scan.

**VACUUM** — the process that finds dead tuples and marks their space reusable. It does four separate jobs, and confusing them causes most of the misunderstanding:

| Job | What it does | Why it matters |
|---|---|---|
| **Reclaim** | mark dead tuple space reusable | stops the table growing forever |
| **Freeze** | mark old tuples wraparound-safe | ★ prevents a total write outage |
| **Update the visibility map** | mark all-visible pages | ★ enables index-only scans |
| **Update statistics** | *(that's `ANALYZE`, run alongside)* | the planner needs it (Topic 15) |

**The one thing VACUUM does not do:** return space to the operating system. That requires a rewrite.

---

## Why does it matter for a backend developer?

```
 ★ THE THREE FAILURE MODES, IN ORDER OF SEVERITY

 ① BLOAT — gradual, expensive, easy to miss
    a 12 GB table becomes 96 GB. Every seq scan reads 8× the pages.
    Your buffer pool holds 8× less useful data. Backups take 8×
    longer. Nobody notices until the disk alarm.

 ② VACUUM RUNNING BUT RECLAIMING NOTHING — the confusing one
    the dashboard says "autovacuum: active." n_dead_tup climbs
    anyway. ⇒ something is pinning xmin. Three candidates, and
    ★ ONE OF THEM IS INVISIBLE unless you know to look.

 ③ ★ TRANSACTION ID WRAPAROUND — total outage
    the database REFUSES ALL WRITES and must be started in
    single-user mode. Multi-hour recovery. This has happened to
    Sentry, Mailchimp, and Joyent publicly.
```

And the positive reason: **tuning autovacuum per table is one of the highest-leverage things you can do to a busy PostgreSQL database**, and the defaults are wrong for any table over a few million rows.

---

## The physical reality

### What "dead" actually means

```
 ★ A TUPLE IS DEAD IFF ITS t_xmax IS COMMITTED AND BELOW THE
   GLOBAL OLDEST xmin.

   "global oldest xmin" = the minimum of:
     ① the xmin of every snapshot held by any active backend
     ② the xmin held by every replication slot
     ③ the xmin of every prepared transaction
     ④ (on a replica with hot_standby_feedback) the replica's xmin

 ⇒ ★ THIS IS ONE NUMBER FOR THE WHOLE CLUSTER, AND THE SINGLE
   OLDEST HOLDER DECIDES IT FOR EVERYONE.

 SEE IT:
   SELECT
     (SELECT min(backend_xmin) FROM pg_stat_activity)   AS from_backends,
     (SELECT min(xmin::text::bigint) FROM pg_replication_slots) AS from_slots,
     (SELECT min(transaction::text::bigint) FROM pg_prepared_xacts) AS from_2pc;

 ⇒ ★ ONE 47-MINUTE TRANSACTION MAKES EVERY DEAD TUPLE IN EVERY
   TABLE CREATED IN THOSE 47 MINUTES UNRECLAIMABLE. Autovacuum
   will run, scan the whole table, and remove zero rows.
```

### What VACUUM physically does — the three phases

```
 PHASE 1 — SCAN THE HEAP
   for each page not marked ALL_VISIBLE in the visibility map:
     • read it into shared buffers
     • for each tuple: is it dead? (t_xmax committed < oldest xmin)
     • collect dead TIDs into a memory array
       ★ sized by maintenance_work_mem (PG ≤16) / autovacuum_work_mem
       ★ PG17+ uses a much more compact TID store — the same memory
         holds ~20× more dead TIDs
     • ★ if the array fills, VACUUM must STOP, do phases 2–3, and
       START OVER from where it left off ⇒ indexes get scanned
       MULTIPLE TIMES ⇒ vacuum takes 3–5× longer
     • PRUNE HOT chains in-page (this part is cheap and immediate)
     • FREEZE tuples older than vacuum_freeze_min_age

 PHASE 2 — SCAN EVERY INDEX
   ★ THE EXPENSIVE PART. For each of N indexes, a FULL index scan
     removing entries that point at the dead TIDs.
   ⇒ vacuum cost scales with the NUMBER OF INDEXES, not just table
     size. A table with 9 indexes costs 9 index scans per vacuum.

 PHASE 3 — SECOND HEAP PASS
   for each dead TID: turn its line pointer into LP_UNUSED
   ⇒ the space is now REUSABLE
   ⇒ record it in the FREE SPACE MAP so future INSERTs find it
   ⇒ update the VISIBILITY MAP for now-all-visible pages

 ⇒ ★ AT NO POINT DOES THE FILE SHRINK.
   The only exception: if the LAST pages of the file end up
   completely empty, VACUUM truncates them — and that truncation
   briefly takes an ★ ACCESS EXCLUSIVE lock.
```

### Why the file doesn't shrink, and what does shrink it

```
 THE FILE AFTER VACUUM:
  page 0    page 1    page 2    page 3    page 4
  [L.d.L]   [d.d.d]   [L.L.L]   [d.L.d]   [L.d.L]
   L = live, d = now-free slot
  ⇒ 40% free space, ★ SCATTERED. The file is still 5 pages.
  ⇒ new INSERTs will fill those slots (via the FSM). If your insert
    rate matches your delete rate, THE TABLE STOPS GROWING and
    plain VACUUM is entirely sufficient. ★ This is the normal,
    healthy steady state — bloat is not automatically a problem.

 TO ACTUALLY SHRINK, YOU MUST REWRITE:
   VACUUM FULL table;
     ✓ perfectly compact, rebuilds indexes too
     ✗ ★ ACCESS EXCLUSIVE for the whole duration — reads AND writes
       blocked. 40 minutes on a 100 GB table.
     ✗ needs disk space for a full second copy
   pg_repack -t table
     ✓ ★ online: builds a copy, syncs changes via triggers, swaps
       with a brief ACCESS EXCLUSIVE at the very end
     ✗ needs a full second copy of the table + indexes
     ✗ an external tool
   ★ CLUSTER table USING idx;   (also ACCESS EXCLUSIVE, but orders
                                 the heap by an index — Topic 15's
                                 correlation)
   ★ REINDEX CONCURRENTLY idx;  (for INDEX bloat only, no table lock)

 ⇒ ★ AND THE ONE THAT COSTS NOTHING: DROP PARTITION. (Topic 59.)
```

### Why autovacuum falls behind — the defaults, computed

```
 AUTOVACUUM TRIGGERS WHEN:
   n_dead_tup > autovacuum_vacuum_threshold
             + autovacuum_vacuum_scale_factor × n_live_tup
   defaults:      50                        + 0.2 × n_live_tup

 ★ WORK THAT OUT FOR REAL TABLE SIZES:
   1,000 rows       ⇒     250 dead tuples      reasonable
   1,000,000 rows   ⇒ 200,000 dead tuples      tolerable
   40,000,000 rows  ⇒ ★ 8,000,000 dead tuples  ← ABSURD
   ⇒ on a table taking 5,000 updates/sec, that's 27 minutes of
     accumulation before autovacuum even STARTS — and then it has
     8 million tuples' worth of work to do while 5,000/sec keep
     arriving.

 AND THEN IT'S THROTTLED:
   autovacuum_vacuum_cost_delay  = 2ms   (PG12+; was 20ms)
   autovacuum_vacuum_cost_limit  = 200
   page costs: hit=1, miss=2, dirty=20
   ⇒ ★ EFFECTIVE THROUGHPUT ≈ 200/20 × (1000/2) = 5,000 dirty
     pages/sec = ~39 MB/s ACROSS ALL AUTOVACUUM WORKERS COMBINED.
   ⇒ on NVMe capable of 2 GB/s, autovacuum is using 2% of it.

 AND THERE ARE ONLY THREE WORKERS:
   autovacuum_max_workers = 3
   ⇒ ★ if you have 4 large bloating tables, one waits.
   ⇒ and a single anti-wraparound vacuum on a huge table can
     occupy one worker for HOURS.

 ⇒ ★ THE THREE DEFAULTS THAT ARE WRONG FOR ANY BUSY DATABASE:
     scale_factor (0.2)  · cost_delay (2ms) · max_workers (3)
```

### The escalation ladder to wraparound

```
 age(relfrozenxid) grows with every transaction in the cluster.

  age                              what happens
  ────────────────────────────────  ──────────────────────────────────
  > vacuum_freeze_table_age (150M)  next VACUUM becomes AGGRESSIVE:
                                    it scans ALL pages, ignoring the
                                    visibility map ⇒ ★ suddenly much
                                    slower, on a table that was fine
  > autovacuum_freeze_max_age(200M) ★ ANTI-WRAPAROUND AUTOVACUUM
                                    starts EVEN IF autovacuum = off.
                                    ★ It cannot be cancelled by
                                    ordinary means — cancel it and it
                                    restarts. Holds SHARE UPDATE
                                    EXCLUSIVE (so it blocks DDL).
  > ~2^31 - 40M                     WARNING: database "shop" must be
                                    vacuumed within 39000000
                                    transactions
  > ~2^31 - 3M                      ★ THE DATABASE STOPS ACCEPTING
                                    WRITES.
                                    ERROR: database is not accepting
                                    commands to avoid wraparound data
                                    loss in database "shop"
                                    ⇒ shut down, start single-user,
                                      VACUUM. Hours of downtime.

 ⇒ ★ WRAPAROUND IS ALWAYS CAUSED BY SOMETHING BLOCKING FREEZING.
   Freezing needs the same thing reclamation needs: an advancing
   oldest-xmin. The three culprits are the same three.
```

---

## How it works — step by step

### The complete diagnostic sequence

```sql
-- ① HOW BAD IS IT, AND WHERE?
SELECT relname,
       n_live_tup, n_dead_tup,
       round(100.0*n_dead_tup/nullif(n_live_tup+n_dead_tup,0),1) AS dead_pct,
       pg_size_pretty(pg_total_relation_size(relid)) AS total,
       last_vacuum, last_autovacuum, autovacuum_count
  FROM pg_stat_user_tables
 WHERE n_dead_tup > 100000
 ORDER BY n_dead_tup DESC LIMIT 10;

-- ② IS AUTOVACUUM EVEN RUNNING RIGHT NOW?
SELECT p.pid, p.relid::regclass AS table, p.phase,
       p.heap_blks_scanned, p.heap_blks_total,
       round(100.0*p.heap_blks_scanned/nullif(p.heap_blks_total,0),1) AS pct,
       p.index_vacuum_count,          -- ★ >1 means work_mem was too small
       now()-a.query_start AS running
  FROM pg_stat_progress_vacuum p JOIN pg_stat_activity a USING (pid);

-- ③ ★ IS ANYTHING PINNING xmin? — THE QUESTION THAT SOLVES IT
SELECT 'backend' AS src, pid::text AS who,
       (now()-xact_start)::text AS age, backend_xmin::text AS xmin
  FROM pg_stat_activity WHERE backend_xmin IS NOT NULL
UNION ALL
SELECT 'repl slot', slot_name, active::text, xmin::text
  FROM pg_replication_slots WHERE xmin IS NOT NULL OR NOT active
UNION ALL
SELECT 'prepared 2pc', gid, (now()-prepared)::text, transaction::text
  FROM pg_prepared_xacts
ORDER BY 4;

-- ④ HOW CLOSE TO WRAPAROUND?
SELECT datname, age(datfrozenxid) AS xid_age,
       (2^31)::bigint - age(datfrozenxid) AS remaining
  FROM pg_database ORDER BY xid_age DESC LIMIT 5;

SELECT relname, age(relfrozenxid) AS xid_age
  FROM pg_class WHERE relkind='r' ORDER BY age(relfrozenxid) DESC LIMIT 10;

-- ⑤ WHY DIDN'T VACUUM RECLAIM? — the definitive answer
VACUUM (VERBOSE) sessions;
-- INFO: table "sessions": found 0 removable, 318440129 nonremovable
-- DETAIL: 318440020 dead row versions cannot be removed yet,
--         oldest xmin: 8842119
--   ★ "cannot be removed yet" + the exact blocking xmin
```

### Reading `pg_stat_progress_vacuum`

```
    phase                    what it means
    ─────────────────────    ────────────────────────────────────────
    initializing             starting up
    scanning heap            phase 1 — collecting dead TIDs
    vacuuming indexes        ★ phase 2 — the expensive part
    vacuuming heap           phase 3 — freeing line pointers
    cleaning up indexes      final index maintenance
    truncating heap          ★ takes ACCESS EXCLUSIVE briefly
    performing final cleanup

 ★ index_vacuum_count > 1 IS A RED FLAG:
   it means the dead-TID array filled up and VACUUM had to scan
   every index more than once.
   ⇒ RAISE autovacuum_work_mem (or maintenance_work_mem).
   ⇒ measured: raising it 64MB → 2GB on a 200M-row table took
     vacuum from 4h 20m (7 index passes) to 38m (1 pass).
```

---

## Concept breakdown

```
VACUUM'S FOUR JOBS — most people know only the first
├── ① RECLAIM dead tuple space for REUSE  (not for the OS)
├── ② ★ FREEZE old tuples — prevents wraparound outage
├── ③ ★ UPDATE THE VISIBILITY MAP — enables index-only scans
└── ④ (ANALYZE, alongside) update planner statistics

THE THREE PHASES — and where the cost is
├── scan heap      — bounded by autovacuum_work_mem
├── ★ scan EVERY index — the expensive part; scales with index count
└── second heap pass — free the line pointers, update FSM + VM

★ VACUUM NEVER SHRINKS THE FILE
   (except trailing empty pages, which needs ACCESS EXCLUSIVE)
   to shrink: VACUUM FULL (locks) · pg_repack (online) ·
              CLUSTER (locks) · ★ DROP PARTITION (free)

★ THE THREE THINGS THAT PIN oldest-xmin — memorise these
├── ① a long-running / idle-in-transaction backend
├── ② ★ a replication slot (especially an INACTIVE one)
└── ③ a prepared transaction left behind by a failed 2PC
     ⇒ if ANY is present, VACUUM reclaims NOTHING newer than it,
       no matter how it's tuned.

THE DEFAULTS THAT ARE WRONG ON A BUSY DATABASE
├── autovacuum_vacuum_scale_factor 0.2
│    ⇒ ★ 8 MILLION dead tuples on a 40M-row table before it starts
├── autovacuum_vacuum_cost_delay 2ms + cost_limit 200
│    ⇒ ★ ~39 MB/s total across ALL workers, on 2 GB/s hardware
├── autovacuum_max_workers 3
│    ⇒ a 4th bloating table simply waits
└── autovacuum_work_mem inherits maintenance_work_mem (64MB)
     ⇒ ★ multiple index passes on large tables

THE WRAPAROUND LADder
  150M aggressive vacuum → 200M ★ uncancellable anti-wraparound
  → warnings → ★ ALL WRITES REFUSED, single-user mode

WHEN BLOAT IS *NOT* A PROBLEM
└── ★ if insert rate ≈ delete rate, free space is REUSED and the
     table reaches a stable size. Steady-state bloat of 20–40% on
     a high-churn table is NORMAL and healthy. Don't repack it.
```

---

## Diagrams

**Diagram 1 — big picture: the three phases and where time goes**

```
  VACUUM sessions;   -- 40M rows, 9 indexes, 318M dead tuples

  ┌─ PHASE 1: SCAN HEAP ──────────────────────────────────────────┐
  │ read pages not marked ALL_VISIBLE                             │
  │ collect dead TIDs → memory array (autovacuum_work_mem)        │
  │ prune HOT chains · freeze old tuples                          │
  │                                          ░░░░░ 22% of time    │
  └───────────────────────┬───────────────────────────────────────┘
                          │ array full? ──yes──┐
                          ▼                     │
  ┌─ PHASE 2: SCAN EVERY INDEX ───────────────┐ │
  │ index 1 of 9: full scan, remove dead TIDs │ │
  │ index 2 of 9: full scan …                 │ │
  │ …                                          │ │
  │ index 9 of 9: full scan                   │ │
  │            ★ ████████████ 68% of time     │ │
  └───────────────────────┬───────────────────┘ │
                          ▼                      │
  ┌─ PHASE 3: SECOND HEAP PASS ───────────────┐ │
  │ line pointers → LP_UNUSED                 │ │
  │ update FSM · update visibility map        │ │
  │                          ░░ 10% of time   │ │
  └───────────────────────┬───────────────────┘ │
                          │                      │
                          └──── more TIDs? ──────┘
                               ★ REPEAT PHASES 2–3
                                 ⇒ index_vacuum_count > 1
                                 ⇒ 3–5× total time

  ⇒ ★ TWO LEVERS FALL STRAIGHT OUT:
    • fewer indexes  ⇒ phase 2 is proportionally cheaper
    • more work_mem  ⇒ ONE pass instead of seven
```

**Diagram 2 — data flow: why a long transaction blocks everything**

```
   TIME →
   ├─ 10:00  analytics session: BEGIN; SELECT …   xmin pinned = 8842119
   │         (Metabase, never commits)
   │
   ├─ 10:05  40M updates on `sessions`     ⇒ 40M dead tuples (xmax > 8842119)
   ├─ 10:15  autovacuum starts on sessions
   │         ┌────────────────────────────────────────────────────┐
   │         │ for each dead tuple:                               │
   │         │   is t_xmax < global_oldest_xmin (8842119)?        │
   │         │   ⇒ NO. Every one of them is NEWER.                │
   │         │ ★ removable: 0                                     │
   │         └────────────────────────────────────────────────────┘
   │         ⇒ full table scan, all 9 indexes scanned, ZERO reclaimed
   │         ⇒ dashboard shows "autovacuum: active" ✓
   │         ⇒ n_dead_tup keeps climbing ✗
   │
   ├─ 10:25  autovacuum runs AGAIN. Same result. And again. And again.
   ├─ 11:00  80M dead tuples. Disk climbing. Nothing works.
   │
   └─ 10:47  ★ SELECT pg_terminate_backend(<metabase pid>);
             ⇒ oldest xmin jumps forward
             ⇒ next autovacuum removes 80M tuples in one pass

 ★ NO AMOUNT OF AUTOVACUUM TUNING WOULD HAVE HELPED.
   The question is never "is vacuum running?" — it is
   "★ WHAT IS THE OLDEST xmin, AND WHO OWNS IT?"
```

**Diagram 3 — before/after: default vs tuned autovacuum on a 40M-row table**

```
 ✗ DEFAULTS
 ┌────────────────────────────────────────────────────────────────┐
 │ autovacuum_vacuum_scale_factor = 0.2                           │
 │   ⇒ trigger at 50 + 0.2 × 40,000,000 = ★ 8,000,000 dead tuples │
 │   ⇒ at 5,000 updates/sec that is 27 MINUTES of accumulation    │
 │                                                                 │
 │ autovacuum_vacuum_cost_delay = 2ms, cost_limit = 200           │
 │   ⇒ ★ ~39 MB/s TOTAL, shared across all 3 workers              │
 │   ⇒ on hardware doing 2,000 MB/s                               │
 │                                                                 │
 │ autovacuum_work_mem = 64MB (inherited)                         │
 │   ⇒ holds ~11M dead TIDs (PG16) ⇒ ★ 7 index passes             │
 │                                                                 │
 │ RESULT: vacuum takes 4h 20m; 27 min of new dead tuples arrive  │
 │         every cycle; ★ it never catches up. 14 GB → 96 GB.     │
 └────────────────────────────────────────────────────────────────┘

 ✓ TUNED — per table, not globally
 ┌────────────────────────────────────────────────────────────────┐
 │ ALTER TABLE sessions SET (                                     │
 │   autovacuum_vacuum_scale_factor = 0.01,  -- 400k, not 8M      │
 │   autovacuum_vacuum_threshold    = 5000,                       │
 │   autovacuum_vacuum_cost_delay   = 0,     -- ★ no throttle     │
 │   autovacuum_vacuum_cost_limit   = 10000,                      │
 │   autovacuum_naptime             = 10                          │
 │ );                                                              │
 │ ALTER SYSTEM SET autovacuum_work_mem = '2GB';                  │
 │ ALTER SYSTEM SET autovacuum_max_workers = 6;                   │
 │                                                                 │
 │ RESULT: vacuum takes 38m, ★ ONE index pass, runs every ~20 min │
 │         table stabilises at 16 GB                              │
 └────────────────────────────────────────────────────────────────┘
     ★ cost_delay = 0 means autovacuum competes with queries for
       I/O. On NVMe that is the right trade. On a shared/throttled
       volume, use 1–2ms and raise cost_limit instead.
```

---

## Example 1 — basic

```sql
CREATE TABLE orders (
  id bigserial PRIMARY KEY,
  customer_id bigint NOT NULL,
  status text NOT NULL,
  total_minor bigint NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
INSERT INTO orders (customer_id, status, total_minor)
SELECT (random()*100000)::bigint, 'pending', (random()*500000)::bigint
  FROM generate_series(1, 500000);
VACUUM ANALYZE orders;
SELECT pg_size_pretty(pg_relation_size('orders')) AS size;
```
```
  size
--------
 34 MB
```

**Create bloat and watch it.**
```sql
UPDATE orders SET status = 'paid';
SELECT n_live_tup, n_dead_tup,
       pg_size_pretty(pg_relation_size('orders')) AS size
  FROM pg_stat_user_tables WHERE relname='orders';
```
```
 n_live_tup | n_dead_tup |  size
------------+------------+--------
     500000 |     500000 | ★ 68 MB      — doubled
```

**VACUUM reclaims, but doesn't shrink.**
```sql
VACUUM (VERBOSE) orders;
```
```
INFO:  vacuuming "public.orders"
INFO:  scanned index "orders_pkey" to remove 500000 row versions
INFO:  table "orders": removed 500000 dead item identifiers in 4425 pages
INFO:  table "orders": found 500000 removable, 500000 nonremovable row versions
DETAIL:  0 dead row versions cannot be removed yet, oldest xmin: 8842204
```
```sql
SELECT n_dead_tup, pg_size_pretty(pg_relation_size('orders')) AS size
  FROM pg_stat_user_tables WHERE relname='orders';
```
```
 n_dead_tup |  size
------------+--------
          0 | ★ 68 MB      — dead tuples gone, FILE UNCHANGED
```

**Prove the space is reused.**
```sql
UPDATE orders SET status = 'shipped';
VACUUM orders;
SELECT pg_size_pretty(pg_relation_size('orders')) AS size;
```
```
  size
--------
 68 MB        ★ STILL 68 MB. The freed slots absorbed the new
              versions. This is the healthy steady state.
```

**Actually shrink it.**
```sql
VACUUM FULL orders;
SELECT pg_size_pretty(pg_relation_size('orders')) AS size;
```
```
  size
--------
 34 MB        ★ back to the true data size — but this took an
              ACCESS EXCLUSIVE lock for the whole rewrite.
```

**Prove a long transaction makes VACUUM useless.**
```sql
-- session 1
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT 1 FROM orders LIMIT 1;      -- snapshot taken, xmin pinned

-- session 2
UPDATE orders SET status = 'delivered';
VACUUM (VERBOSE) orders;
```
```
INFO:  table "orders": found 0 removable, 1000000 nonremovable row versions
DETAIL:  ★ 500000 dead row versions cannot be removed yet, oldest xmin: 8842301
```
```sql
-- session 2 — who is it?
SELECT pid, backend_xmin, now()-xact_start AS age, state, application_name
  FROM pg_stat_activity WHERE backend_xmin IS NOT NULL ORDER BY backend_xmin;
```
```
  pid  | backend_xmin |      age       |        state        | application_name
-------+--------------+----------------+---------------------+------------------
 41202 |      8842301 | 00:04:18.229   | idle in transaction | psql
   ★ that's the one. Its xmin (8842301) IS the "oldest xmin" above.
```
```sql
-- session 1
COMMIT;
-- session 2
VACUUM (VERBOSE) orders;
```
```
INFO:  table "orders": found 500000 removable, 500000 nonremovable row versions
DETAIL:  0 dead row versions cannot be removed yet
   ★ same command, opposite result. The only thing that changed was
     the oldest xmin.
```

**Watch a vacuum in progress.**
```sql
-- in one session
VACUUM orders;
-- in another
SELECT relid::regclass AS tbl, phase, heap_blks_scanned, heap_blks_total,
       index_vacuum_count, max_dead_tuple_bytes, dead_tuple_bytes
  FROM pg_stat_progress_vacuum;
```
```
   tbl   |       phase       | heap_blks_scanned | heap_blks_total | index_vacuum_count
---------+-------------------+-------------------+-----------------+--------------------
 orders  | vacuuming indexes |              8850 |            8850 |                  1
   ★ index_vacuum_count = 1 is what you want. >1 ⇒ raise work_mem.
```

**Check wraparound and index bloat.**
```sql
SELECT datname, age(datfrozenxid) AS xid_age FROM pg_database ORDER BY 2 DESC LIMIT 3;

-- index bloat via avg_leaf_density (Topic 11)
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstatindex('orders_pkey');
```
```
 version | tree_level | index_size | root_block_no | internal_pages | leaf_pages |
       4 |          1 |   11239424 |             3 |              6 |       1366 |
 leaf_fragmentation | avg_leaf_density
                4.2 |          ★ 51.2      — half empty ⇒ REINDEX CONCURRENTLY
```
```sql
REINDEX INDEX CONCURRENTLY orders_pkey;   -- ★ no table lock
SELECT avg_leaf_density FROM pgstatindex('orders_pkey');
```
```
 avg_leaf_density
------------------
             89.7
```

**Exact bloat measurement.**
```sql
SELECT * FROM pgstattuple('orders');
```
```
 table_len | tuple_count | tuple_len | tuple_percent | dead_tuple_count |
  71303168 |      500000 |  34500000 |         48.38 |           500000 |
 dead_tuple_len | dead_tuple_percent | free_space | free_percent
       34500000 |            ★ 48.38 |     412008 |         0.58
   ⇒ ★ pgstattuple is EXACT but does a full table scan. Use
     pg_stat_user_tables estimates for monitoring; pgstattuple to
     confirm before an expensive repack.
```

---

## Example 2 — production scenario

**The situation.** A payments platform, 02:14 IST on a Sunday. PagerDuty:

```
 CRITICAL  postgres-primary: database "payments" is not accepting commands
 CRITICAL  all API writes failing
 CRITICAL  error rate 100% on 34 endpoints
```

```
ERROR:  database is not accepting commands to avoid wraparound data loss
        in database "payments"
HINT:  Stop the postmaster and vacuum that database in single-user mode.
       You might also need to commit or roll back old prepared transactions,
       or drop stale replication slots.
```

```
 ★ THE HINT LITERALLY NAMES THE THREE CAUSES. Read it.
```

**Step 1 — assess. Reads still work.**

```sql
SELECT datname, age(datfrozenxid) AS xid_age,
       (2^31)::bigint - age(datfrozenxid) AS remaining
  FROM pg_database ORDER BY xid_age DESC;
```
```
  datname  |  xid_age   | remaining
-----------+------------+-----------
 payments  | 2145482106 | ★ 1,001,542
 postgres  |      88421 | 2147395227
```

```
 ★ ONE MILLION TRANSACTIONS LEFT out of 2.1 billion. The database
   stopped itself deliberately, at the last safe moment, rather
   than corrupt data.
```

**Step 2 — which table?**

```sql
SELECT c.relname, age(c.relfrozenxid) AS xid_age,
       pg_size_pretty(pg_total_relation_size(c.oid)) AS size,
       s.last_autovacuum, s.autovacuum_count
  FROM pg_class c
  LEFT JOIN pg_stat_user_tables s ON s.relid = c.oid
 WHERE c.relkind IN ('r','m','t')
 ORDER BY age(c.relfrozenxid) DESC LIMIT 5;
```
```
      relname       |  xid_age   |  size   |    last_autovacuum     | autovacuum_count
--------------------+------------+---------+------------------------+------------------
 payment_attempts   | 2145482106 | ★ 412 GB| 2026-05-02 04:11:22+05:30 |             1841
 payment_ledger     |   88204118 |  180 GB | 2026-08-16 22:04:11+05:30 |            22104
 webhooks_inbox     |    4028841 |   22 GB | 2026-08-17 01:58:02+05:30 |            88420
```

```
 ★ payment_attempts: last successful autovacuum ★ 3.5 MONTHS AGO,
   despite autovacuum_count = 1,841 attempts.
 ⇒ autovacuum RAN 1,841 TIMES AND ACCOMPLISHED NOTHING.
 ⇒ this is failure mode ②: something is pinning xmin.
```

**Step 3 — the question that solves it. All three candidates.**

```sql
-- ① long-running backends
SELECT pid, backend_xmin, now()-xact_start AS age, state, application_name
  FROM pg_stat_activity WHERE backend_xmin IS NOT NULL
 ORDER BY backend_xmin LIMIT 5;
```
```
  pid  | backend_xmin |     age      | state  | application_name
-------+--------------+--------------+--------+------------------
 88204 |   2147440118 | 00:00:00.882 | active | api-server-2
   ⇒ nothing old. Not this.
```

```sql
-- ② replication slots
SELECT slot_name, slot_type, active, xmin, catalog_xmin,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS wal_retained
  FROM pg_replication_slots;
```
```
    slot_name     | slot_type | active |   xmin   | catalog_xmin | wal_retained
------------------+-----------+--------+----------+--------------+--------------
 replica_mumbai   | physical  | t      | 2147440002 |            | 240 MB
 debezium_cdc     | logical   | ★ f    |          |   ★ 1958104 | ★ 1.9 TB
```

```sql
-- ③ prepared transactions
SELECT gid, prepared, owner, database FROM pg_prepared_xacts ORDER BY prepared;
```
```
              gid               |         prepared          |  owner   | database
--------------------------------+---------------------------+----------+----------
 recon_batch_2026_05_02_0411    | ★ 2026-05-02 04:11:19+05:30 | recon_svc| payments
   ★ A PREPARED TRANSACTION FROM 2 MAY. 107 DAYS OLD.
```

```
 ★ TWO INDEPENDENT CAUSES, BOTH DATING TO 2 MAY:
   ① a prepared transaction from a reconciliation job that crashed
      between PREPARE and COMMIT and was never cleaned up
   ② an INACTIVE logical replication slot from a Debezium CDC
      connector that was decommissioned — retaining 1.9 TB of WAL
      and pinning catalog_xmin

 ⇒ EITHER ALONE WOULD HAVE CAUSED THIS. Both were invisible on
   every dashboard the team had, because both are "idle" —
   no query, no connection, no CPU. (Topic 51 on 2PC; Topic 41
   on slots.)
```

**Step 4 — recover. Order matters.**

```sql
-- ① release the pins FIRST. Vacuuming before this achieves nothing.
ROLLBACK PREPARED 'recon_batch_2026_05_02_0411';
SELECT pg_drop_replication_slot('debezium_cdc');

-- ② confirm the oldest xmin has jumped forward
SELECT min(age(backend_xmin)) FROM pg_stat_activity WHERE backend_xmin IS NOT NULL;
SELECT slot_name, xmin, catalog_xmin FROM pg_replication_slots;
```

```bash
# ③ vacuum the offending table. Reads still work; writes are refused.
#    ★ VACUUM is allowed even in the "not accepting commands" state.
psql -d payments -c "SET vacuum_cost_delay = 0;" \
                  -c "SET maintenance_work_mem = '8GB';" \
                  -c "VACUUM (FREEZE, VERBOSE, PARALLEL 4) payment_attempts;"
```
```
INFO:  aggressively vacuuming "public.payment_attempts"
INFO:  launched 4 parallel vacuum workers for index cleanup
INFO:  table "payment_attempts": found 1841204882 removable,
       402118 nonremovable row versions in 51204882 pages
DETAIL:  0 dead row versions cannot be removed yet
       new relfrozenxid: 2147441002, which is 2145482106 XIDs ahead
       of the previous value
   ★ 1.84 BILLION dead tuples removed. 71 minutes.
```

```
 ★ WRITES RESUMED at 03:41. Total outage: 1h 27m.
 ★ IF THE PINS HAD NOT BEEN RELEASED FIRST, this VACUUM would have
   run for 71 minutes and removed ZERO rows — and the team would
   have concluded "vacuum doesn't work."
```

**Step 5 — reclaim the space.**

```bash
# 412 GB, of which ~400 GB was dead. VACUUM freed it for reuse but
# the file is still 412 GB. VACUUM FULL would need 412 GB free and
# an ACCESS EXCLUSIVE lock for hours.
pg_repack -d payments -t payment_attempts --jobs 4
# ⇒ 412 GB → 11 GB, online, one brief ACCESS EXCLUSIVE at the swap
```

**Step 6 — prevention. Six changes.**

```sql
-- ① per-table autovacuum on every high-churn table
ALTER TABLE payment_attempts SET (
  autovacuum_vacuum_scale_factor  = 0.01,
  autovacuum_vacuum_threshold     = 10000,
  autovacuum_vacuum_cost_delay    = 0,
  autovacuum_vacuum_cost_limit    = 10000,
  autovacuum_freeze_max_age       = 100000000,   -- ★ half the default
  autovacuum_analyze_scale_factor = 0.02
);

-- ② cluster-wide capacity
ALTER SYSTEM SET autovacuum_max_workers   = 6;
ALTER SYSTEM SET autovacuum_work_mem      = '2GB';
ALTER SYSTEM SET autovacuum_naptime       = '15s';
ALTER SYSTEM SET vacuum_cost_delay        = 0;
ALTER SYSTEM SET maintenance_work_mem     = '4GB';

-- ③ ★ the guards against the two actual causes
ALTER SYSTEM SET max_slot_wal_keep_size   = '128GB';  -- kills runaway slots
ALTER SYSTEM SET idle_in_transaction_session_timeout = '60s';
ALTER SYSTEM SET max_prepared_transactions = 0;
-- ⇒ ★ THE LAST ONE IS THE REAL FIX. This system did not need 2PC.
--   The reconciliation job used it "for safety" and it became the
--   single point of failure. If you don't have a distributed
--   transaction coordinator that reliably resolves prepared
--   transactions, set this to 0. (Topic 51.)

SELECT pg_reload_conf();
```

```sql
-- ④ ★ THE ALERTS THAT WOULD HAVE CAUGHT IT ON DAY ONE
-- a) XID age — warn 500M, page 1B
SELECT max(age(datfrozenxid)) AS max_xid_age FROM pg_database;

-- b) ★ any inactive replication slot at all — page immediately
SELECT count(*) FROM pg_replication_slots WHERE NOT active;

-- c) ★ any prepared transaction older than 5 minutes — page
SELECT count(*) FROM pg_prepared_xacts WHERE prepared < now() - interval '5 min';

-- d) any table whose last_autovacuum is older than 24h while
--    n_dead_tup > 1,000,000  ⇒ "vacuum runs but reclaims nothing"
SELECT relname, n_dead_tup, last_autovacuum FROM pg_stat_user_tables
 WHERE n_dead_tup > 1000000
   AND (last_autovacuum IS NULL OR last_autovacuum < now() - interval '24 hours');

-- e) oldest transaction age
SELECT max(extract(epoch from now()-xact_start)) FROM pg_stat_activity
 WHERE xact_start IS NOT NULL;
```

```sql
-- ⑤ the design fix: payment_attempts is append-only with 90-day
--    retention. It should never have been a single table. (Topic 59.)
--    → range partition by created_at, monthly
--    → retention becomes DROP TABLE payment_attempts_2026_05
--    ⇒ ★ no dead tuples, no vacuum pressure, no freeze pressure,
--      because each partition is dropped long before it ages.
```

**Step 7 — results.**

| | Before | After |
|---|---|---|
| Max XID age | 2,145,482,106 | 41,204,882 |
| `payment_attempts` size | 412 GB | 11 GB |
| Retained WAL | 1.9 TB | 240 MB |
| Inactive replication slots | 1 (107 days) | 0, alerted |
| Prepared transactions | 1 (107 days) | **disabled** |
| Detection | PagerDuty at 100% write failure | 5 alerts, days ahead |
| Retention mechanism | `DELETE` (never ran to completion) | `DROP PARTITION` |

```
 ★ THE LESSON: autovacuum ran 1,841 times over 3.5 months and did
   nothing, because two objects — neither of which held a
   connection, ran a query, or used any CPU — pinned the oldest
   xmin. Every monitoring dashboard showed "autovacuum: healthy."
 ⇒ ★ THE QUESTION IS NEVER "IS VACUUM RUNNING?"
   IT IS "WHAT IS THE OLDEST xmin, AND WHO OWNS IT?"
```

---

## Common mistakes

**1. Asking "is autovacuum running?" instead of "what is the oldest xmin?"**
- *Symptom:* autovacuum active, `n_dead_tup` climbing anyway.
- *Fix:* the three-way union query on `pg_stat_activity` / `pg_replication_slots` / `pg_prepared_xacts`.

**2. Leaving `autovacuum_vacuum_scale_factor` at 0.2 on large tables.**
- *Symptom:* autovacuum on a 40M-row table waits for 8M dead tuples.
- *Fix:* set per table to 0.01 or lower; add a `threshold` floor.

**3. Expecting `VACUUM` to shrink the file.**
- *Symptom:* "vacuum finished and the disk is still full."
- *Engine-level why:* it marks space reusable; only trailing empty pages are truncated.
- *Fix:* `pg_repack` online, or `VACUUM FULL` in a window, or partition and `DROP`.

**4. Running `VACUUM FULL` on a large table in production.**
- *Symptom:* an ACCESS EXCLUSIVE lock for 40+ minutes = a total outage on that table.
- *Fix:* `pg_repack`. And confirm the bloat is real with `pgstattuple` first — steady-state 30% is normal.

**5. Ignoring inactive replication slots.**
- *Symptom:* unreclaimable bloat, unbounded WAL growth, eventually wraparound.
- *Engine-level why:* a slot pins `xmin`/`catalog_xmin` and `restart_lsn` whether or not anything is connected.
- *Fix:* alert on `NOT active`; set `max_slot_wal_keep_size` (Topic 41).

**6. Enabling `max_prepared_transactions` without a coordinator that resolves them.**
- *Symptom:* one abandoned `PREPARE TRANSACTION` blocks vacuum cluster-wide, forever, invisibly.
- *Fix:* set it to 0 unless you genuinely run 2PC with a recovering coordinator; alert on any prepared transaction older than 5 minutes (Topic 51).

**7. `index_vacuum_count > 1` ignored.**
- *Symptom:* vacuum takes 5× longer than it should.
- *Engine-level why:* the dead-TID array filled, forcing repeated full scans of every index.
- *Fix:* raise `autovacuum_work_mem` / `maintenance_work_mem`.

**8. Deleting rows in bulk for retention.**
- *Symptom:* a `DELETE` of 200M rows creates 200M dead tuples, enormous index churn and WAL, frees no space, and may not finish.
- *Fix:* range partition and `DROP PARTITION` (Topic 59).

**9. Vacuuming before releasing the pins.**
- *Symptom:* a 71-minute emergency vacuum that removes zero rows, and a team that concludes vacuum is broken.
- *Fix:* always release the xmin holders **first**, then vacuum.

**10. Treating all bloat as a problem.**
- *Symptom:* repacking a table whose insert and delete rates match, which will simply re-bloat to the same size.
- *Fix:* bloat that is stable and reused is healthy. Repack when the size is *growing* or the free space is unusable.

---

## Hands-on proof

**PROVE IT #1–#8 — Example 1** (bloat created, VACUUM reclaiming without shrinking, space reuse, `VACUUM FULL` shrinking, a long transaction blocking reclamation and the query that names the culprit, `pg_stat_progress_vacuum`, index bloat via `pgstatindex`, exact bloat via `pgstattuple`).

**PROVE IT #9 — a replication slot pinning xmin, with no connection.**
```sql
SELECT pg_create_logical_replication_slot('test_slot', 'pgoutput');
-- nothing consumes it. Now:
UPDATE orders SET status='x';
VACUUM (VERBOSE) orders;
```
```
DETAIL:  ★ 500000 dead row versions cannot be removed yet, oldest xmin: …
```
```sql
SELECT slot_name, active, xmin, catalog_xmin FROM pg_replication_slots;
SELECT pg_drop_replication_slot('test_slot');
VACUUM (VERBOSE) orders;    -- ★ now they are removable
```

**PROVE IT #10 — a prepared transaction doing the same.**
```sql
-- requires max_prepared_transactions > 0
BEGIN;
UPDATE orders SET status='y' WHERE id=1;
PREPARE TRANSACTION 'stuck';
-- the session is now free, but:
SELECT gid, prepared FROM pg_prepared_xacts;
UPDATE orders SET status='z';
VACUUM (VERBOSE) orders;    -- ★ cannot be removed yet
ROLLBACK PREPARED 'stuck';
VACUUM (VERBOSE) orders;    -- ★ removable
```

**PROVE IT #11 — `work_mem` and index passes.**
```sql
SET maintenance_work_mem = '1MB';
UPDATE orders SET status='a';
VACUUM (VERBOSE) orders;
-- INFO: index scan needed: … ★ multiple "scanned index" lines

SET maintenance_work_mem = '1GB';
UPDATE orders SET status='b';
VACUUM (VERBOSE) orders;
-- ★ one "scanned index" line per index
```

**PROVE IT #12 — measure the cost of index count.**
```sql
CREATE TABLE t1 (id bigint PRIMARY KEY, a int, b int, c int, d int);
CREATE TABLE t9 (LIKE t1 INCLUDING ALL);
CREATE INDEX ON t9(a); CREATE INDEX ON t9(b);
CREATE INDEX ON t9(c); CREATE INDEX ON t9(d);
CREATE INDEX ON t9(a,b); CREATE INDEX ON t9(c,d);
CREATE INDEX ON t9(b,c); CREATE INDEX ON t9(a,d);

INSERT INTO t1 SELECT g,g,g,g,g FROM generate_series(1,1000000) g;
INSERT INTO t9 SELECT g,g,g,g,g FROM generate_series(1,1000000) g;
UPDATE t1 SET a = a+1;  UPDATE t9 SET a = a+1;

\timing on
VACUUM t1;    -- Time: 412 ms
VACUUM t9;    -- Time: ★ 4,880 ms   — 11.8× for 9 indexes vs 1
```

---

## The design decision framework

```
★★★ VACUUM IS NOT A MAINTENANCE TASK. IT IS A CAPACITY CONSTRAINT. ★★★

 ① THE FIRST QUESTION IN EVERY BLOAT INVESTIGATION
    NOT "is autovacuum running?"
    ★ BUT "what is the oldest xmin, and who owns it?"
      ① a long / idle-in-transaction backend
      ② ★ a replication slot (especially inactive)
      ③ a prepared transaction
    ⇒ if one is present, NO amount of tuning helps. Fix it first.

 ② TUNE PER TABLE, NOT GLOBALLY
    any table over ~5M rows with meaningful churn:
      autovacuum_vacuum_scale_factor = 0.01–0.02
      autovacuum_vacuum_threshold    = 5000–50000
      autovacuum_vacuum_cost_delay   = 0     (on NVMe)
      autovacuum_vacuum_cost_limit   = 5000–10000
    ⇒ globally: max_workers 4–8, autovacuum_work_mem 1–2GB

 ③ REDUCE THE WORK RATHER THAN SPEEDING UP THE CLEANUP
    ★ fewer dead tuples beats faster vacuum, every time.
      • HOT updates — don't index columns you update (Topic 46)
      • fillfactor 70–90 on update-heavy tables
      • narrow, hot tables split from wide, cold ones
      • ★ retention via DROP PARTITION, never DELETE (Topic 59)
      • fewer indexes ⇒ phase 2 is proportionally cheaper

 ④ CHOOSING HOW TO RECLAIM SPACE
    growing and unusable?    → ★ pg_repack (online)
    a maintenance window?    → VACUUM FULL (ACCESS EXCLUSIVE)
    index bloat only?        → ★ REINDEX CONCURRENTLY
    time-series retention?   → ★ DROP PARTITION — costs nothing
    ★ stable and being reused? → DO NOTHING. That's healthy.

 ⑤ THE FIVE ALERTS EVERY PRODUCTION POSTGRES NEEDS
    ✓ max(age(datfrozenxid))              warn 500M, page 1B
    ✓ ★ any inactive replication slot     page immediately
    ✓ ★ any prepared transaction > 5 min  page immediately
    ✓ max(now() - xact_start)             warn 5 min
    ✓ n_dead_tup > 1M AND last_autovacuum > 24h ago
       ⇒ ★ "vacuum runs but reclaims nothing" — the exact signature

 ⑥ DEFAULTS TO CHANGE ON DAY ONE OF ANY BUSY DATABASE
    idle_in_transaction_session_timeout = '60s'
    max_slot_wal_keep_size              = a real value
    max_prepared_transactions           = 0  ★ unless you truly use 2PC
    autovacuum_max_workers              = 4–8
    autovacuum_work_mem                 = 1–2GB

 ⑦ NEVER TURN AUTOVACUUM OFF
    ⇒ ★ it does not stop anti-wraparound vacuums, so you get all the
      cost and none of the benefit — plus guaranteed wraparound.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Create a 500,000-row table. `UPDATE` every row, then: (a) show the size doubled; (b) `VACUUM` and show `n_dead_tup` → 0 with the size unchanged; (c) `UPDATE` again, `VACUUM`, and show the size *still* unchanged — explain why; (d) `VACUUM FULL` and show it shrink. State what lock (d) took.

### Exercise 2 — medium (apply it)
Reproduce all three xmin-pinning causes and prove each one blocks reclamation:
(a) an idle-in-transaction backend at REPEATABLE READ · (b) an unconsumed logical replication slot · (c) a prepared transaction

For each: show `VACUUM VERBOSE` reporting "cannot be removed yet," write the query that identifies the culprit, release it, and show the same `VACUUM` succeeding. Then write one query that finds all three at once.

### Exercise 3 — hard (production simulation)
At 02:14 a payments database refuses all writes: *"not accepting commands to avoid wraparound data loss."* `age(datfrozenxid)` is 2,145,482,106 — about a million transactions from the limit. `payment_attempts` is 412 GB, its `last_autovacuum` is 3.5 months old, and `autovacuum_count` is 1,841.

(a) Explain how autovacuum can run 1,841 times over 3.5 months and reclaim nothing.
(b) Write the three queries that find the cause, and say what each rules out.
(c) Two independent causes are present, both dating to the same day. Name them and explain how each pins `xmin`, including why neither appears on a connection or CPU dashboard.
(d) Give the recovery sequence in the correct order. Explain what happens if you vacuum before releasing the pins.
(e) Why is `VACUUM` still permitted while the database refuses other writes?
(f) After vacuuming, the table is still 412 GB. Give three ways to reclaim it and pick one, with the lock each takes and the disk each needs.
(g) Write the five alerts that would have caught this, with thresholds and the exact SQL.
(h) Give the per-table and cluster-wide autovacuum settings, and justify each number against this table's size and churn.
(i) `max_prepared_transactions` was set to 8 "for safety." Argue for setting it to 0, and say what you'd need in place before ever raising it.
(j) `payment_attempts` is append-only with 90-day retention. Redesign it so this class of failure becomes structurally impossible.

---

## Mental model checkpoint

1. Name VACUUM's four jobs. Which one prevents a total outage?
2. Why doesn't `VACUUM` return space to the OS? What does?
3. Define "dead tuple" precisely, in terms of the global oldest xmin.
4. Name the three things that pin the oldest xmin. Which two are invisible on a normal connections dashboard?
5. Which of VACUUM's three phases is the expensive one, and what does its cost scale with?
6. What does `index_vacuum_count > 1` mean and what do you change?
7. Compute the default autovacuum trigger threshold for a 40-million-row table. Why is that wrong?
8. Walk the wraparound escalation ladder from 150M to write refusal.
9. When is bloat *not* a problem?

---

## Quick reference card

**Diagnose — in this order**
```sql
-- ① how bad, and where
SELECT relname, n_live_tup, n_dead_tup,
       round(100.0*n_dead_tup/nullif(n_live_tup+n_dead_tup,0),1) AS dead_pct,
       pg_size_pretty(pg_total_relation_size(relid)) AS total, last_autovacuum
  FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;

-- ② ★ WHO PINS xmin — the question that solves it
SELECT 'backend' src, pid::text who, (now()-xact_start)::text age FROM pg_stat_activity
  WHERE backend_xmin IS NOT NULL
UNION ALL SELECT 'slot', slot_name, active::text FROM pg_replication_slots
UNION ALL SELECT '2pc', gid, (now()-prepared)::text FROM pg_prepared_xacts;

-- ③ wraparound
SELECT datname, age(datfrozenxid) FROM pg_database ORDER BY 2 DESC;

-- ④ in-progress vacuum
SELECT relid::regclass, phase, heap_blks_scanned, heap_blks_total, index_vacuum_count
  FROM pg_stat_progress_vacuum;

-- ⑤ the definitive answer
VACUUM (VERBOSE) tbl;    -- "cannot be removed yet, oldest xmin: N"
```

**Reclaim**

| Need | Command | Lock | Extra disk |
|---|---|---|---|
| mark space reusable | `VACUUM` | SHARE UPDATE EXCL | none |
| shrink the file | `VACUUM FULL` | ★ ACCESS EXCLUSIVE | full copy |
| shrink online | `pg_repack` | brief ACCESS EXCL | full copy |
| index bloat only | `REINDEX CONCURRENTLY` | SHARE UPDATE EXCL | index copy |
| **retention** | ★ **`DROP PARTITION`** | brief | **none** |

**Tune per table**
```sql
ALTER TABLE t SET (
  autovacuum_vacuum_scale_factor = 0.01,
  autovacuum_vacuum_threshold    = 10000,
  autovacuum_vacuum_cost_delay   = 0,
  autovacuum_vacuum_cost_limit   = 10000);
```

**Five alerts:** XID age > 500M · any inactive slot · any prepared txn > 5 min · oldest txn > 5 min · `n_dead_tup > 1M AND last_autovacuum > 24h`.

**Day-one settings:** `idle_in_transaction_session_timeout='60s'` · `max_slot_wal_keep_size` set · `max_prepared_transactions=0` · `autovacuum_max_workers=4–8` · `autovacuum_work_mem=1–2GB`. **Never turn autovacuum off.**

---

## When would I use this at work?

1. **The disk-space alert.** Four queries separate "we have more data" from "we have bloat" from "something is pinning xmin" — and the three have completely different fixes.

2. **Before proposing a `VACUUM FULL` or a repack.** Confirm with `pgstattuple` that the bloat is real and *growing*. Repacking a table at healthy steady state accomplishes nothing and costs a lock.

3. **Day one on any PostgreSQL system you inherit.** Check XID age, inactive slots, prepared transactions and per-table autovacuum settings. This finds latent outages that are months away and completely invisible.

4. **Designing any high-churn or time-series table.** The choice between `DELETE`-based retention and partition-based retention determines whether vacuum is a background detail or your primary operational burden.

---

## Connected topics

**Understand before this:** 04 (heap files, FSM, visibility map), 05 (tuple headers), 46 (MVCC — why dead tuples exist at all), 44 (isolation levels — why long snapshots pin xmin), 45 (locks — what `VACUUM FULL` takes).

**This unlocks:**
- **59** — partitioning: `DROP PARTITION` as the retention answer
- **62** — replication: slots, `hot_standby_feedback`, and vacuum conflicts on replicas
- **64** — backup and PITR: why bloat multiplies backup cost
- **67** — performance investigation: bloat as a cause of unexplained slow scans
- **41** — WAL: `max_slot_wal_keep_size`, the other half of the slot problem
- **51** — distributed transactions: why `max_prepared_transactions = 0` is usually right
