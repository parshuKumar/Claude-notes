# 07 — The Buffer Pool
## Phase: Storage Internals

---

## ELI5 — The Simple Analogy

A mechanic's workshop.

The parts warehouse is across town — a 40-minute round trip. So the mechanic keeps a **shelf by the workbench** with the parts he uses most. When he needs a bolt, he checks the shelf first (2 seconds). Only if it's not there does he drive to the warehouse (40 minutes).

The shelf is small, so it fills up. When he needs space, he throws out something he hasn't touched in a while — not something he used five minutes ago. And critically: **if he has modified a part on the shelf — filed it down, repainted it — he cannot just throw it away.** He must drive it back to the warehouse first. A modified part he hasn't returned yet is a **dirty page**.

The shelf is the **buffer pool**. The 40-minute drive is a disk read: **1,000× slower** than the shelf. Everything about database performance is "did we find it on the shelf?"

---

## Where this fits in the big picture

```
   02 Engine architecture ─────┐
   04 Pages on disk ───────────┤
                               ▼
                    ┌─────────────────────┐
                    │ 07 THE BUFFER POOL  │  ← YOU ARE HERE
                    │  (pages in RAM)     │
                    └──────────┬──────────┘
                               │
        ┌──────────────────────┼──────────────────────┐
        ▼                      ▼                      ▼
   18 planner cost      41 WAL & dirty        67 performance
   model (random_page   page flushing         investigation
   _cost assumes cache)                       (hit ratio)
```

Every query cost in Phase 2 is measured in **pages read**. This topic is where you learn that not all page reads cost the same — and that the difference is three orders of magnitude.

---

## What is this?

The **buffer pool** (in PostgreSQL: *shared buffers*) is a fixed-size region of shared memory holding copies of 8 KB disk pages. Every read and every write goes through it — no backend ever touches the heap file directly.

Its job is to make the 1,000× gap between RAM and disk invisible for the pages you actually use.

---

## Why does it matter for a backend developer?

Because the same query, on the same data, with the same plan, can take 0.2 ms or 200 ms depending on nothing but whether the pages were cached.

- **This is why "it's fast in staging and slow in production."** Staging has 200 MB of data that fits entirely in RAM. Production has 400 GB and a 16 GB pool.
- **This is why the first run of a query is slow and the second is fast** — and why benchmarking the second run tells you nothing.
- **This is why one badly-scoped analytics query causes an unrelated API endpoint to slow down** — it evicted that endpoint's working set.
- **This is why adding RAM sometimes gives a 10× improvement and sometimes gives zero.** If your working set already fits, more RAM does nothing. If it's just over the line, more RAM is the single highest-leverage change available.

And it decides a planner constant you will meet constantly: `random_page_cost`. The planner's willingness to use an index depends on its assumption about how cached your data is.

---

## The physical reality

### The memory layout

```
POSTGRESQL SHARED MEMORY
┌─────────────────────────────────────────────────────────────────────┐
│ BUFFER DESCRIPTORS ARRAY          (one per buffer, ~64 bytes each)  │
│  ┌──────────────────────────────────────────────────────────────┐   │
│  │ buf_id │ tag(rel,fork,block) │ state │ usage_count │ wait_lock│   │
│  │   0    │ (16390, main, 1204) │ DIRTY │      3      │    -     │   │
│  │   1    │ (16393, main,   12) │ VALID │      5      │    -     │   │
│  │   2    │ (16390, main, 8891) │ VALID │      1      │    -     │   │
│  └──────────────────────────────────────────────────────────────┘   │
│                                                                     │
│ BUFFER HASH TABLE   (rel,fork,block) ──▶ buf_id                     │
│   the lookup structure: "is page (16390,main,1204) in memory?"      │
│                                                                     │
│ BUFFER BLOCKS                     shared_buffers bytes              │
│  ┌────────┬────────┬────────┬────────┬────────┬─────────────────┐   │
│  │  8 KB  │  8 KB  │  8 KB  │  8 KB  │  8 KB  │  ... N buffers  │   │
│  └────────┴────────┴────────┴────────┴────────┴─────────────────┘   │
│     buf 0    buf 1    buf 2    buf 3    buf 4                       │
└─────────────────────────────────────────────────────────────────────┘

  N = shared_buffers / 8 KB.   16 GB → 2,097,152 buffers.
```

### The buffer descriptor state, field by field

```
 state (a packed 32-bit word, updated atomically):
   ┌─────────────────────────────────────────────────────────────┐
   │ refcount (18 bits) │ usage_count (4 bits) │ flags (10 bits) │
   └─────────────────────────────────────────────────────────────┘

   refcount     how many backends currently have this buffer PINNED.
                A pinned buffer cannot be evicted. Pinned during any read
                or write of its contents; released immediately after.

   usage_count  0–5. Incremented on each access (capped at 5), decremented
                by the clock sweep. THIS IS THE EVICTION PRIORITY.

   flags        BM_DIRTY       modified since read from disk — MUST be
                               written before eviction
                BM_VALID       contains valid data
                BM_IO_IN_PROGRESS  a read/write is happening now
                BM_JUST_DIRTIED
```

### The memory hierarchy — the numbers that matter

```
 WHERE THE PAGE MIGHT BE                LATENCY        RELATIVE
 ────────────────────────────────────────────────────────────────
 CPU L1 cache                            ~1 ns              1×
 CPU L3 cache                           ~20 ns             20×
 RAM (buffer pool hit)                 ~100 ns            100×
 ──────────────── the cliff ────────────────────────────────────
 OS page cache hit (a syscall, no I/O)   ~5 µs          5,000×
 NVMe SSD random read                  ~100 µs        100,000×
 SATA SSD random read                  ~200 µs        200,000×
 Network (EBS gp3) random read      ~500–1000 µs      500,000×
 HDD random read (seek + rotate)        ~10 ms     10,000,000×

 ⇒ A buffer pool hit is roughly 1,000× faster than an SSD read.
   A query doing 10,000 page reads:
        all cached   →   1 ms
        all on SSD   →   1,000 ms
   SAME PLAN. SAME DATA. 1000× difference.
```

### The double-caching reality

PostgreSQL does **not** use `O_DIRECT`. Every page it reads from disk also passes through the operating system's page cache.

```
 ┌──────────────────────────────────────────────────────────────┐
 │  POSTGRESQL SHARED BUFFERS         e.g. 16 GB                │
 │   • knows about relations, usage counts, dirty state         │
 │   • knows WAL ordering rules (never flush a page before its  │
 │     WAL record — this is why PG must manage its own cache)   │
 └───────────────────────────┬──────────────────────────────────┘
                             │ read()/write() syscalls
 ┌───────────────────────────▼──────────────────────────────────┐
 │  OS PAGE CACHE                     e.g. 100 GB (the rest)    │
 │   • knows nothing about tables — just files and offsets      │
 │   • plain LRU                                                │
 │   • FREE: no configuration, uses all spare RAM               │
 └───────────────────────────┬──────────────────────────────────┘
                             │ actual block I/O
 ┌───────────────────────────▼──────────────────────────────────┐
 │  DISK                                                        │
 └──────────────────────────────────────────────────────────────┘

 ⚠ A page can be in BOTH caches — wasting RAM. This is exactly why
   shared_buffers = 25% of RAM is the conventional advice, not 90%:
   you deliberately leave room for the OS cache to act as a second,
   cheaper tier. Setting shared_buffers to 80% of RAM usually makes
   things WORSE, not better.
```

---

## How it works — step by step

### Reading a page: `ReadBuffer(relation, blockNum)`

```
 1. Compute the buffer tag: (relfilenode=16390, fork=MAIN, block=1204)

 2. Hash it, look it up in the BUFFER HASH TABLE.

 ─── CASE A: HIT ────────────────────────────────────────────────
 3a. Found → buf_id 8891.
 4a. Atomically increment refcount (PIN it) and usage_count (cap 5).
 5a. Return the pointer. ~100 ns. No syscall. No I/O.

 ─── CASE B: MISS ───────────────────────────────────────────────
 3b. Not found. We need a victim buffer.
 4b. CLOCK SWEEP (see below) → returns buffer 41,203.
 5b. Is buffer 41,203 DIRTY?
        YES → we must write it out first:
              • Ensure its WAL record is already flushed (WAL rule!)
              • write() 8 KB to its relation's file
              • ⚠ this is a WRITE stall inside a READ. This is what
                pg_stat_bgwriter.buffers_backend counts, and a high
                value means your bgwriter is too lazy.
        NO  → proceed.
 6b. Remove the old tag from the hash table, insert the new one.
 7b. Set BM_IO_IN_PROGRESS, release the mapping lock (so other backends
     that want the same page WAIT rather than duplicate the read).
 8b. read() 8192 bytes from base/16388/16390 at offset 1204 × 8192.
 9b. Verify the page checksum (if data checksums are on).
10b. Set BM_VALID, clear BM_IO_IN_PROGRESS, usage_count = 1, refcount = 1.
11b. Return the pointer.  ~100 µs on NVMe.

 12. Caller reads/writes the page contents.
 13. If modified: set BM_DIRTY.
 14. UnpinBuffer → refcount--.  (Now eligible for eviction.)
```

### The clock sweep — PostgreSQL's eviction algorithm

Not true LRU (too expensive — LRU requires a globally-ordered list and a lock on every access). PostgreSQL uses **clock sweep**, an approximation of LRU with an access counter.

```
 A circular buffer array with a moving hand:

              ┌──────────────────────────────────────────┐
              │                                          │
              ▼                                          │
  ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
  │ b0 │ b1 │ b2 │ b3 │ b4 │ b5 │ b6 │ b7 │ b8 │ b9 │... │
  │ u3 │ u0 │ u5 │ u1 │ u2 │ u0 │ u4 │ u1 │ u0 │ u2 │    │
  └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
          ▲
       the HAND
                        u = usage_count

 THE ALGORITHM (each step advances the hand by one):
   if refcount > 0        → PINNED, skip (someone is using it)
   else if usage_count>0  → usage_count--, skip   ← "second chance"
   else                   → EVICT THIS ONE. Return it.

 PROPERTIES:
   • A page accessed 5 times survives 5 full sweeps. Hot pages persist.
   • A page accessed once (usage_count=1) survives one sweep.
   • Cost is O(1) amortised — no sorting, no global list, minimal locking.
   • usage_count caps at 5, so even a page hit a million times can't
     become permanently unevictable.
```

### The ring buffer — why one big scan doesn't destroy your cache

If a sequential scan of a 200 GB table used the normal path, it would evict every useful page. PostgreSQL prevents this:

```
 When a scan's relation is larger than shared_buffers / 4:
   → the backend allocates a small private RING BUFFER and reuses it

   BufferAccessStrategy:
     BAS_BULKREAD    256 KB (32 buffers)   large seq scans
     BAS_BULKWRITE    16 MB                COPY, CREATE TABLE AS
     BAS_VACUUM      256 KB                VACUUM

 ┌────────────────────────────────────────────────────────────┐
 │  shared_buffers (16 GB) — UNTOUCHED by the big scan        │
 │  ┌──────────────────────────────────────────────────┐      │
 │  │ hot OLTP pages stay resident                     │      │
 │  └──────────────────────────────────────────────────┘      │
 │  ┌────┐  ← the 256 KB ring the seq scan cycles through     │
 │  └────┘                                                    │
 └────────────────────────────────────────────────────────────┘

 ⚠ BUT: this protects shared_buffers, NOT the OS page cache — the scan
   still pulls 200 GB through it, evicting the OS-cached copies of your
   hot pages. And index scans, sorts, and hash joins in the same query
   do NOT use a ring. So a big analytics query still hurts. (Topic 06.)
```

---

## Concept breakdown

```
BUFFER
│  └── An 8 KB slot in shared memory + its descriptor. Holds one disk page.
│
├── PIN (refcount)     "I am using this right now; do not evict it."
│                       Held for microseconds — the duration of one access.
│                       NOT a lock; many backends can pin the same buffer.
│
├── BUFFER LOCK        A lightweight latch (LWLock) — shared for read,
│  (content lock)      exclusive for write. Microseconds. Completely
│                       different from the row/table locks of Topic 45.
│
├── usage_count (0–5)  Eviction priority. Incremented on access,
│                       decremented by the clock sweep.
│
└── DIRTY              Modified in memory, not yet written to disk.
                       ⚠ A dirty page can NEVER be written before its WAL
                         record is on disk. This is the write-ahead rule,
                         and it is why PostgreSQL cannot just use mmap.

HIT RATIO
│   hits / (hits + reads)
│   • 99%+ on OLTP → healthy
│   • < 95% → your working set exceeds the pool (or a scan is thrashing it)
│   ⚠ MISLEADING: PostgreSQL's "read" counts pages fetched via read(),
│     many of which are served by the OS page cache in ~5 µs, not 100 µs.
│     So a "miss" is not necessarily a disk seek. Use pg_stat_kcache or
│     I/O metrics for the truth.

WORKING SET
│   The pages actually touched by your live queries over a window.
│   NOT the database size. A 2 TB database with a 40 GB working set runs
│   beautifully on 64 GB of RAM. This distinction is the whole game.

WHO WRITES DIRTY PAGES OUT  (three actors, in order of desirability)
│
├── CHECKPOINTER   every checkpoint_timeout: writes ALL dirty pages,
│                  spread over checkpoint_completion_target (0.9) to
│                  avoid an I/O spike.
├── BGWRITER       continuously trickles out the least-recently-used
│                  dirty pages so backends find clean victims.
└── BACKEND        ✗ WORST CASE: a backend needing a buffer finds only
                   dirty victims and must write one itself — a write
                   stall inside a read. Monitor buffers_backend.
```

---

## Diagrams

**Diagram 1 — big picture: the path of one page request**

```
                        backend needs page (16390, 1204)
                                    │
                                    ▼
                       ┌────────────────────────┐
                       │  BUFFER HASH TABLE     │
                       │  lookup by tag         │
                       └────────┬───────────────┘
                     found ─────┴───── not found
                       │                  │
                       ▼                  ▼
               ┌───────────────┐   ┌──────────────────┐
               │ PIN + usage++ │   │  CLOCK SWEEP     │
               │  ~100 ns  ✓   │   │  find a victim   │
               └───────────────┘   └────────┬─────────┘
                                      dirty?│
                              ┌─────yes─────┴─────no─────┐
                              ▼                          │
                    ┌────────────────────┐               │
                    │ flush WAL to LSN   │               │
                    │ write 8KB to disk  │               │
                    │  (stall!)          │               │
                    └─────────┬──────────┘               │
                              └────────────┬─────────────┘
                                           ▼
                                 ┌──────────────────┐
                                 │ read 8KB, verify │
                                 │ checksum ~100 µs │
                                 └──────────────────┘
```

**Diagram 2 — data flow: the three tiers and who fills them**

```
  QUERY
    │ ReadBuffer()
    ▼
 ┌─────────────────────────────────────────────────┐
 │ SHARED BUFFERS  16 GB · 2,097,152 buffers        │  hit: 100 ns
 │  managed by PostgreSQL · knows about WAL & dirty │
 └────────┬────────────────────────────▲────────────┘
          │ read()                     │ write()
          ▼                            │
 ┌─────────────────────────────────────┴────────────┐
 │ OS PAGE CACHE   ~100 GB · plain LRU              │  hit: ~5 µs
 └────────┬────────────────────────────▲────────────┘
          │                            │
          ▼                            │ fsync() at COMMIT (WAL only)
 ┌─────────────────────────────────────┴────────────┐
 │ DISK                                             │  hit: ~100 µs
 │   base/16388/16390 (heap)   pg_wal/ (log)        │
 └──────────────────────────────────────────────────┘

  ⚠ Dirty heap pages go DOWN lazily (checkpointer/bgwriter).
    WAL goes down EAGERLY (fsync at every commit).
    That asymmetry IS durability. (Topic 41.)
```

**Diagram 3 — before/after: the working set fitting vs not fitting**

```
 CASE A — working set (12 GB) FITS in shared_buffers (16 GB)
 ┌───────────────── shared_buffers 16 GB ─────────────────┐
 │ ████████████████████████████████████░░░░░░░░░░░░░░░░░░ │
 │ ← 12 GB hot working set, fully resident →   4 GB spare │
 └────────────────────────────────────────────────────────┘
   hit ratio 99.7%  ·  p99 = 8 ms  ·  disk IOPS ~200

 CASE B — working set (20 GB) EXCEEDS shared_buffers (16 GB)
 ┌───────────────── shared_buffers 16 GB ─────────────────┐
 │ ████████████████████████████████████████████████████████│
 │ ← thrashing: pages evicted, immediately needed again → │
 └────────────────────────────────────────────────────────┘
   hit ratio 91%  ·  p99 = 340 ms  ·  disk IOPS ~28,000

   ⚠ NOT a linear degradation. 25% over capacity → 40× worse latency.
     This is a CLIFF, and you fall off it overnight when a table grows.

 CASE C — same as A, but an analytics query runs
 ┌───────────────── shared_buffers 16 GB ─────────────────┐
 │ ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓█████████│
 │ ← cold analytics pages evicted the hot set →   hot 4 GB│
 └────────────────────────────────────────────────────────┘
   hit ratio 71%  ·  p99 = 900 ms  ·  recovers over ~10 min
   (the ring buffer mitigates seq scans; sorts and index scans don't use it)
```

---

## Example 1 — basic

Watch the hit/miss difference directly.

```sql
CREATE EXTENSION IF NOT EXISTS pg_buffercache;
CREATE EXTENSION IF NOT EXISTS pg_prewarm;

CREATE TABLE orders AS
SELECT i AS id, (random()*100000)::bigint AS user_id,
       (random()*500000)::bigint AS total_paise,
       now() - (random()*365)::int * interval '1 day' AS created_at
FROM generate_series(1, 3000000) i;
CREATE INDEX ON orders(user_id);
VACUUM ANALYZE orders;

-- Evict everything (restart is the reliable way in a container)
-- docker restart pg-lab ; then reconnect

EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM orders;
```
```
Aggregate  (actual time=1204.8..1204.8 rows=1 loops=1)
  Buffers: shared hit=32 read=19608          ← COLD: 19,608 disk reads
Execution Time: 1206.2 ms
```
```sql
-- run it again immediately
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM orders;
```
```
Aggregate  (actual time=118.4..118.4 rows=1 loops=1)
  Buffers: shared hit=19640 read=0           ← WARM: 0 disk reads
Execution Time: 119.1 ms
```

**Same plan. Same data. 10× faster.** (Not 1000× — because the "cold" run was largely served by the OS page cache, and because 19,608 sequential reads benefit from readahead. This is exactly why "hit ratio" overstates your problem: a miss is often 5 µs, not 100 µs.)

Now see what's actually resident:

```sql
SELECT c.relname, count(*) AS buffers,
       pg_size_pretty(count(*) * 8192) AS ram,
       round(100.0 * count(*) FILTER (WHERE b.isdirty) / count(*), 1) AS pct_dirty,
       round(avg(b.usagecount), 2) AS avg_usage
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY c.relname ORDER BY buffers DESC LIMIT 8;
```
```
     relname      | buffers |   ram   | pct_dirty | avg_usage
------------------+---------+---------+-----------+-----------
 orders           |   19640 | 153 MB  |       0.0 |      1.02
 orders_user_id_idx|   8232 | 64 MB   |       0.0 |      2.41
 pg_attribute     |      44 | 352 kB  |       0.0 |      5.00
```

`avg_usage = 1.02` on `orders` means the seq scan touched each page once — those pages are the *first* to be evicted. `pg_attribute` at 5.00 is catalog data touched constantly; it will survive everything.

Prove the ring buffer exists:

```sql
SELECT setting FROM pg_settings WHERE name='shared_buffers';   -- e.g. 16384 (128MB)
-- orders is 153 MB > 128MB/4 = 32MB, so a seq scan uses BAS_BULKREAD.
-- After a fresh restart + one seq scan:
SELECT count(*) FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid) WHERE c.relname='orders';
```
```
 count
-------
    32          ← THIRTY-TWO buffers, not 19,640. The ring buffer at work.
```
This is why the second run above was fast only because of the *OS* cache. Change `shared_buffers` to 1 GB and re-run: now the scan fully populates shared buffers.

---

## Example 2 — production scenario

**The situation.** `db.r6g.4xlarge`: 16 vCPU, 128 GB RAM, `shared_buffers = 32 GB`. Database is 480 GB. For eight months, p99 on `GET /orders/:id` has been 12 ms. On a Tuesday it becomes 210 ms and stays there. No deploy. No traffic change.

**Step 1 — hit ratio.**

```sql
SELECT datname,
       blks_hit, blks_read,
       round(100.0 * blks_hit / nullif(blks_hit + blks_read, 0), 2) AS hit_pct
FROM pg_stat_database WHERE datname = 'shop';
```
```
 datname |   blks_hit    |  blks_read   | hit_pct
---------+---------------+--------------+---------
 shop    | 1892400000000 | 141200000000 |   93.06
```
93% — but this is cumulative since the stats were reset. Reset and re-measure over 5 minutes:

```sql
SELECT pg_stat_reset();
-- wait 300 seconds
SELECT round(100.0*blks_hit/nullif(blks_hit+blks_read,0),2) AS hit_pct FROM pg_stat_database WHERE datname='shop';
```
```
 hit_pct
---------
   88.40         ← was 99.4% last month
```

**Step 2 — what changed? Find the working set.**

```sql
SELECT relname, pg_size_pretty(pg_relation_size(oid)) AS size,
       pg_size_pretty(pg_total_relation_size(oid)) AS total
FROM pg_class WHERE relkind='r' AND relnamespace='public'::regnamespace
ORDER BY pg_total_relation_size(oid) DESC LIMIT 6;
```
```
      relname       |  size   |  total
--------------------+---------+---------
 orders             | 180 GB  | 244 GB
 order_items        |  92 GB  | 131 GB
 sessions           |  38 GB  |  61 GB    ← ⚠ was 4 GB three months ago
 users              |  12 GB  |  19 GB
```

`sessions` grew 10×. It's read on **every authenticated request** — so it is 100% working set. It went from 4 GB (fits comfortably) to 38 GB, pushing the total working set from ~28 GB to ~62 GB against a 32 GB pool.

**Step 3 — why did `sessions` grow?**

```sql
SELECT * FROM pgstattuple('sessions');
```
```
 table_len          | 40802189312
 tuple_count        | 2100000
 tuple_len          | 1050000000     -- 1 GB of live data
 dead_tuple_count   | 68000000       -- ⚠
 dead_tuple_percent | 71.2
 free_percent       | 26.1
```

**1 GB of live sessions inside a 38 GB table.** Sessions are updated on every request (`last_seen_at`), each update creating a dead tuple (Topic 46), and autovacuum's default `autovacuum_vacuum_scale_factor = 0.2` means it waits for 20% of the table to be dead — which on a growing table is a moving target it never catches.

**Step 4 — the fix, in order of speed to deploy.**

```sql
-- (1) IMMEDIATE: reclaim the space (needs a window, or use pg_repack online)
VACUUM (FULL, ANALYZE, VERBOSE) sessions;
-- 38 GB → 1.4 GB. Working set drops back under 32 GB. p99 recovers same day.

-- (2) SAME DAY: stop it recurring
ALTER TABLE sessions SET (
  autovacuum_vacuum_scale_factor  = 0.01,   -- vacuum at 1% dead, not 20%
  autovacuum_vacuum_cost_delay    = 2,      -- let it work faster
  fillfactor                      = 80      -- headroom for HOT updates
);

-- (3) THIS SPRINT: stop generating the churn at all
--     `last_seen_at` doesn't belong in PostgreSQL. Move it to Redis with a
--     TTL, and flush to PostgreSQL once every 5 minutes per user.
--     Write rate on `sessions`: 8,000/s → ~30/s.
```

**Step 5 — the monitoring that would have caught this in week one.**

```sql
-- Working set vs pool. Run hourly; alert when it crosses 80%.
SELECT
  pg_size_pretty(sum(pg_total_relation_size(relid))) AS hot_tables_total,
  current_setting('shared_buffers') AS pool
FROM pg_stat_user_tables
WHERE seq_scan + idx_scan > 1000;      -- "tables anyone actually reads"

-- Bloat leaderboard. Run daily.
SELECT relname, n_live_tup, n_dead_tup,
       round(100.0*n_dead_tup/nullif(n_live_tup+n_dead_tup,0),1) AS dead_pct,
       last_autovacuum
FROM pg_stat_user_tables WHERE n_dead_tup > 100000
ORDER BY dead_pct DESC LIMIT 10;
```

**The lesson:** the buffer pool didn't shrink and traffic didn't grow. The *working set* grew — silently, through bloat, on a table nobody was watching. Hit ratio is a lagging indicator; **working-set size vs pool size is the leading one.**

---

## Common mistakes

**1. Setting `shared_buffers` to 80% of RAM.**
- *Symptom:* performance no better, sometimes worse; checkpoint I/O spikes.
- *Engine-level why:* PostgreSQL double-buffers through the OS page cache. Giving 80% to shared buffers starves the OS cache without removing the read path through it. You also make checkpoints write far more dirty data at once.
- *Fix:* 25% of RAM as a starting point (up to ~40% if the workload is write-light and the whole DB fits). Measure, don't guess.

**2. Chasing hit ratio as a KPI.**
- *Symptom:* "our hit ratio is 99.2%, so cache is fine" — while p99 is terrible.
- *Engine-level why:* 0.8% of a billion buffer requests is 8 million disk reads. And PostgreSQL's "read" includes OS-cache hits, so the number is optimistic. Meanwhile, a query doing 50,000 *hits* is still slow — hits aren't free (~100 ns each, plus lock traffic).
- *Fix:* measure per-query `shared_blks_read` from `pg_stat_statements`, and actual disk IOPS from the OS. Optimise the queries reading the most, not the global ratio.

**3. Benchmarking only warm runs.**
- *Symptom:* "the query takes 8 ms" in testing; 400 ms in production.
- *Engine-level why:* your test ran twice and measured the cached run. Production's first-touch-of-the-day is always cold.
- *Fix:* report both. `EXPLAIN (ANALYZE, BUFFERS)` after a restart, and again warm. The gap tells you your exposure.

**4. Assuming a read replica isolates cache pressure.**
- *Symptom:* analytics on a replica still slows the primary.
- *Engine-level why:* buffer pools *are* isolated — that part works. But `hot_standby_feedback` propagates snapshots, blocking vacuum on the primary and causing bloat, which grows the primary's working set. (Topic 06, mistake #2.)
- *Fix:* accept query cancellations on the analytics replica, or move analytics off PostgreSQL.

**5. Ignoring `buffers_backend`.**
- *Symptom:* write latency spikes; reads occasionally stall for tens of milliseconds.
- *Engine-level why:* a backend that needs a free buffer and finds only dirty ones must write one out itself — a synchronous disk write inside a read path.
- *Diagnose:*
  ```sql
  SELECT buffers_checkpoint, buffers_clean, buffers_backend,
         round(100.0*buffers_backend/nullif(buffers_checkpoint+buffers_clean+buffers_backend,0),1) AS pct_backend
  FROM pg_stat_bgwriter;
  ```
  `pct_backend > 10%` is a problem.
- *Fix:* raise `bgwriter_lru_maxpages`, lower `bgwriter_delay`, and/or increase `checkpoint_timeout` with `checkpoint_completion_target = 0.9`.

**6. Not knowing your working set, only your database size.**
- *Symptom:* "our database is 2 TB, we can't cache it" → team gives up on tuning.
- *Reality:* a 2 TB database with a 30 GB working set runs perfectly on 64 GB. Conversely, a 40 GB database where every query scans everything will thrash a 32 GB pool.
- *Fix:* measure. `pg_buffercache` tells you what's resident; `pg_stat_user_tables` tells you what's touched.

---

## Hands-on proof

**PROVE IT #1 — cold vs warm, isolated.**
```bash
docker restart pg-lab && sleep 5
```
```sql
\timing on
SELECT count(*) FROM orders;   -- cold
SELECT count(*) FROM orders;   -- warm
```

**PROVE IT #2 — what's in the pool right now, with usage counts.**
```sql
SELECT usagecount, count(*), pg_size_pretty(count(*)*8192::bigint) AS ram,
       count(*) FILTER (WHERE isdirty) AS dirty
FROM pg_buffercache WHERE relfilenode IS NOT NULL
GROUP BY usagecount ORDER BY usagecount DESC;
```
```
 usagecount | count | ram     | dirty
------------+-------+---------+-------
          5 |  1204 | 9408 kB |    12
          4 |   882 | 6896 kB |     4
          3 |  2140 | 17 MB   |    88
          2 |  4102 | 32 MB   |   201
          1 | 11212 | 88 MB   |   440
          0 |  2891 | 23 MB   |     0   ← next to be evicted
```

**PROVE IT #3 — force a page into the pool and out again.**
```sql
CREATE EXTENSION IF NOT EXISTS pg_prewarm;
SELECT pg_prewarm('orders');                      -- load the whole table
SELECT count(*) FROM pg_buffercache b JOIN pg_class c
  ON b.relfilenode = pg_relation_filenode(c.oid) WHERE c.relname='orders';
-- now read a big unrelated table and watch orders get evicted
```

**PROVE IT #4 — the ring buffer.**
```sql
SHOW shared_buffers;                              -- note it
-- restart, then ONE seq scan of a table > shared_buffers/4:
SELECT count(*) FROM orders;
SELECT count(*) FROM pg_buffercache b JOIN pg_class c
  ON b.relfilenode = pg_relation_filenode(c.oid) WHERE c.relname='orders';
-- ~32, not the full table. BAS_BULKREAD.
```

**PROVE IT #5 — dirty pages and who writes them.**
```sql
SELECT count(*) FILTER (WHERE isdirty) AS dirty_now,
       pg_size_pretty(count(*) FILTER (WHERE isdirty) * 8192::bigint) AS dirty_bytes
FROM pg_buffercache;

UPDATE orders SET total_paise = total_paise + 1 WHERE id < 100000;
SELECT count(*) FILTER (WHERE isdirty) FROM pg_buffercache;   -- jumps
CHECKPOINT;
SELECT count(*) FILTER (WHERE isdirty) FROM pg_buffercache;   -- drops to ~0
```

**PROVE IT #6 — the planner's cache assumption.**
```sql
SHOW random_page_cost;      -- 4.0 default (assumes spinning disk, uncached)
SHOW seq_page_cost;         -- 1.0
SHOW effective_cache_size;  -- 4GB default — the planner's ESTIMATE of total
                            -- cache (shared_buffers + OS cache). It does NOT
                            -- allocate anything; it only changes plan choices.

EXPLAIN SELECT * FROM orders WHERE user_id BETWEEN 1 AND 5000;
SET random_page_cost = 1.1;          -- correct for SSD
SET effective_cache_size = '96GB';   -- correct for a 128GB box
EXPLAIN SELECT * FROM orders WHERE user_id BETWEEN 1 AND 5000;
```
The plan often flips from `Seq Scan` to `Index Scan`. **`random_page_cost = 4.0` is a 1990s HDD default.** On SSD it should be 1.1–1.5, and leaving it at 4 is one of the most common causes of "PostgreSQL won't use my index."

---

## The design decision framework

```
INCREASE shared_buffers WHEN:
  ✓ Working set > current pool, and RAM is available
  ✓ pg_stat_database hit ratio consistently < 98% on an OLTP workload
  ✓ pg_buffercache shows high-usage_count pages being evicted
  ✓ You are below ~25% of system RAM
  Start: 25% of RAM. Raise in steps, measure p99 each time.

DO NOT INCREASE IT WHEN:
  ✗ Already at 40%+ of RAM (you're starving the OS cache)
  ✗ Hit ratio is 99.9% and p99 is still bad → the problem is elsewhere
    (bad plans, lock waits, CPU, N+1) — Topic 67
  ✗ The working set exceeds total RAM by 10× → no pool size saves you;
    you need partitioning, archiving, or a narrower schema

SHRINK THE WORKING SET INSTEAD WHEN:  (usually the better lever)
  ✓ Bloat > 20% → VACUUM / pg_repack / autovacuum tuning        (Topic 47)
  ✓ Rows are wide with cold columns → vertical split            (Topics 04, 05)
  ✓ Old data is never queried → partition + archive             (Topic 59)
  ✓ Indexes are unused → drop them (they occupy pool too)       (Topic 17)
  ✓ Hot churn belongs in Redis → move it                        (Topic 57)
  → These are usually 5–20× cheaper than buying RAM, and they compound.

TUNE THE PLANNER'S CACHE ASSUMPTIONS WHEN:
  ✓ On SSD/NVMe:  random_page_cost = 1.1
  ✓ effective_cache_size = shared_buffers + OS cache (~75% of RAM)
  → Costs nothing, changes plans immediately, fixes a huge number of
    "why won't it use my index" complaints.

THE SIGNAL TO LOOK FOR:
      -- The single most useful buffer-pool query:
      SELECT c.relname,
             pg_size_pretty(pg_relation_size(c.oid))        AS on_disk,
             pg_size_pretty(count(*) * 8192::bigint)        AS in_ram,
             round(100.0*count(*)*8192/nullif(pg_relation_size(c.oid),0),1) AS pct_cached
      FROM pg_buffercache b
      JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
      GROUP BY c.oid, c.relname
      ORDER BY count(*) DESC LIMIT 15;

  • A hot table at pct_cached < 50% → your pool is too small for it,
    OR the table is bloated, OR it's being scanned rather than sought.
  • A cold table occupying a lot of the pool → something is scanning it
    that shouldn't be. Find that query.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Restart your lab instance. Run `EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM orders;` twice. Record `hit`, `read`, and time for both. Then explain: why is the cold run only ~10× slower and not ~1000×, given that a buffer hit is 1000× faster than an SSD read? Your answer must mention two distinct mechanisms.

### Exercise 2 — medium (apply it)
Using `pg_buffercache`, write a query that reports, for each table: size on disk, bytes resident in the pool, percent cached, and average usage count. Run it, then run a large sequential scan of your biggest table, then run it again. Explain what changed and — specifically — why the scanned table did **not** take over the whole pool. Then set `shared_buffers` large enough that the ring buffer no longer applies, and show the difference.

### Exercise 3 — hard (production simulation)
A 900 GB database on a 128 GB box (`shared_buffers = 32 GB`) has been healthy for a year at 99.5% hit ratio. Over six weeks, p99 climbs from 15 ms to 400 ms. Traffic is flat. No deploys. Disk IOPS have gone from 300 to 22,000.

(a) List, in priority order, the five diagnostic queries you'd run, and state what result at each step would rule a cause in or out.
(b) Give three distinct root causes that produce exactly this signature, and the query that distinguishes each.
(c) For each, give the immediate mitigation and the permanent fix.
(d) The 22,000 IOPS number is important — explain what it tells you that hit ratio alone does not.
(e) Design the alert that would have fired in week one. It must not be "hit ratio < 98%" — explain why that alert is inadequate and what leading indicator you'd use instead.

---

## Mental model checkpoint

1. Describe the clock-sweep algorithm. Why not true LRU?
2. What is a *pin*, and how is it different from a buffer content lock and from a row lock?
3. Why can a dirty page never be written to disk before its WAL record? What would break?
4. Why is `shared_buffers = 80% of RAM` usually worse than 25%?
5. A backend needs a buffer and all candidates are dirty. What happens, what does it cost, and which counter shows it?
6. Explain the ring buffer: when does it apply, what does it protect, and what does it *not* protect?
7. Your database is 2 TB with 64 GB of RAM. Under what circumstances is that completely fine, and under what circumstances is it hopeless? Name the quantity that decides.

---

## Quick reference card

| Concept | One line |
|---|---|
| Buffer | An 8 KB slot in shared memory holding one page |
| Pin (refcount) | "In use right now" — prevents eviction |
| usage_count | 0–5 eviction priority; clock sweep decrements it |
| Dirty | Modified in RAM, not yet on disk |
| Clock sweep | O(1) LRU approximation with second chances |
| Ring buffer | 256 KB private ring for big scans; protects the pool |
| Working set | The pages your live queries actually touch |

**Latency table (memorise)**

| Layer | Latency | vs RAM |
|---|---|---|
| Buffer pool hit | 100 ns | 1× |
| OS page cache | 5 µs | 50× |
| NVMe read | 100 µs | 1,000× |
| EBS gp3 | 500 µs | 5,000× |
| HDD seek | 10 ms | 100,000× |

**Settings**

| Setting | Default | Recommended |
|---|---|---|
| `shared_buffers` | 128 MB | 25% of RAM |
| `effective_cache_size` | 4 GB | ~75% of RAM |
| `random_page_cost` | 4.0 | **1.1 on SSD** |
| `seq_page_cost` | 1.0 | 1.0 |
| `bgwriter_lru_maxpages` | 100 | 300–1000 if `buffers_backend` high |
| `checkpoint_completion_target` | 0.9 | 0.9 |

**The four queries to keep**

```sql
-- hit ratio (reset stats first for a meaningful window)
SELECT round(100.0*blks_hit/nullif(blks_hit+blks_read,0),2) FROM pg_stat_database WHERE datname=current_database();
-- what's resident
SELECT c.relname, count(*) FROM pg_buffercache b JOIN pg_class c ON b.relfilenode=pg_relation_filenode(c.oid) GROUP BY 1 ORDER BY 2 DESC LIMIT 10;
-- who's writing dirty pages
SELECT buffers_checkpoint, buffers_clean, buffers_backend FROM pg_stat_bgwriter;
-- per-query I/O
SELECT shared_blks_read, shared_blks_hit, calls, left(query,60) FROM pg_stat_statements ORDER BY shared_blks_read DESC LIMIT 10;
```

---

## When would I use this at work?

1. **"It's fast on my machine."** A query is 8 ms locally and 400 ms in production. You can immediately explain that your laptop's 200 MB dataset is fully cached while production's working set exceeds the pool, and ask for `EXPLAIN (ANALYZE, BUFFERS)` from production rather than arguing about the query.

2. **Sizing an RDS instance.** Instead of guessing, you measure the working set with `pg_buffercache` and `pg_stat_user_tables`, find it's 44 GB, and specify a 128 GB instance with `shared_buffers = 32 GB` — with the numbers to justify it in the ticket.

3. **The "PostgreSQL won't use my index" ticket.** Before touching the query, you check `random_page_cost`. If it's 4.0 on an NVMe instance, you set it to 1.1 and half the team's index complaints evaporate — because the planner had been assuming every random read costs four sequential reads, which was true in 1998.

---

## Connected topics

**Understand before this:** 02 (the buffer manager component), 04 (what a page is).

**This unlocks:**
- **12** — index scans and why random I/O is the expensive kind
- **17** — when a sequential scan beats an index scan (it's a cache/IO argument)
- **18** — the planner's cost model, built on `seq_page_cost`/`random_page_cost`
- **41** — WAL, and the write-ahead rule that constrains dirty-page flushing
- **47** — VACUUM and bloat: the main cause of an unexpectedly large working set
- **67** — the full performance investigation methodology
