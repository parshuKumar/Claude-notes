# 41 — The Write-Ahead Log
## Phase: Transactions & Concurrency

---

## ELI5 — The Simple Analogy

A busy kitchen with one order book and forty dishes on the go.

The naive approach: when an order comes in, walk to the right station, find the right pan, and update it immediately. Forty orders means forty walks across the kitchen, in whatever order they arrive. Slow, and if you collapse halfway you have no idea which dishes were started.

The real approach: **write every order into a paper book, in sequence, the instant it arrives.** One book, one pen, always at your elbow — writing takes a second and never involves walking anywhere. The actual cooking happens later, in whatever order is efficient.

Two things follow, and they are the entire idea.

**Writing is fast** because appending one line to a book at your elbow beats walking to forty stations. Sequential beats random, by a factor of hundreds.

**And you can always recover.** If the kitchen catches fire and you restart, you read the book from the last known-good point and redo everything in it. Nothing is lost, because *the order was written down before anyone touched a pan.*

That's the write-ahead rule: **the log record goes to disk before the data page does.** Everything else in this topic follows from that one sentence.

---

## Where this fits in the big picture

```
   39 transactions (the boundary) · 40 ACID (the promises)
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 41 THE WRITE-AHEAD LOG   ← YOU ARE HERE  │
        │ how A and D are actually implemented     │
        └────────────────────┬─────────────────────┘
                             ▼
              42 crash recovery (the WAL, read back)
              62 replication  (the WAL, shipped)
              64 PITR         (the WAL, archived)
              17 write amplification (the WAL, measured)
```

Topic 40 said "durability is a WAL fsync." **This topic is what that log actually contains, why the design is fast rather than slow, and the three other jobs the same log does.**

---

## What is this?

The **write-ahead log** is an append-only sequential file recording every change to the database *before* that change reaches the data files.

The rule, stated formally:

> **A modified data page may not be written to disk until the WAL record describing the modification is durably on disk.**

From that single rule you get atomicity, durability, crash recovery, physical replication, and point-in-time recovery — four features, one mechanism.

---

## Why does it matter for a backend developer?

Because the WAL is the thing your commits actually wait for, and it is the thing that gets shipped, archived, and replayed:

```
 ① COMMIT LATENCY IS WAL FSYNC LATENCY
    Your p99 write latency has a floor set by how long your storage
    takes to confirm a flush. Nothing in your query, your index or
    your schema changes that floor.

 ② WAL VOLUME IS YOUR REPLICATION BUDGET
    Every byte of WAL is shipped to every replica and archived for
    PITR. A schema generating 400 GB/hour of WAL needs 400 GB/hour of
    network to each replica — and that is usually what actually breaks.

 ③ ★ WAL VOLUME IS NOT PROPORTIONAL TO YOUR DATA
    An 88-byte insert can generate 40 KB of WAL, because of full-page
    images. Understanding when that happens explains most "why is our
    WAL so large?" investigations.

 ④ WAL BUILDUP TAKES DOWN THE PRIMARY
    A disconnected replica with a replication slot, or a failing
    archive command, causes WAL to accumulate until pg_wal fills the
    disk — at which point the database shuts down.
    ⇒ ★ this is one of the most common PostgreSQL outages, and it is
      caused by a replica, not by the primary.
```

---

## The physical reality

### The files

```
 $PGDATA/pg_wal/
 ├── 000000010000000000000023      ← 16 MB each, fixed size
 ├── 000000010000000000000024
 ├── 000000010000000000000025
 └── archive_status/
     ├── 000000010000000000000023.ready   ← waiting for archive_command
     └── 000000010000000000000022.done

 THE FILENAME — 24 hex digits, three parts of 8:
   00000001  timeline ID (increments on each PITR recovery — T64)
   00000000  the high 32 bits of the LSN
   00000023  the segment number within that
 ⇒ files are named so that lexicographic order = chronological order.

 ★ WAL FILES ARE RECYCLED, NOT DELETED. After a checkpoint, a segment
   that is no longer needed is RENAMED to the next future name and
   reused. This is why pg_wal has a stable size in steady state, and
   why creating one is free.
```

### The LSN — the address of everything

```
 A LOG SEQUENCE NUMBER is a 64-bit byte offset into the infinite WAL
 stream, printed as two hex halves:

   0/1A2B3C8
   │   └────── low 32 bits: byte offset
   └────────── high 32 bits

 ⇒ EVERY LSN IS A GLOBAL POSITION IN TIME. Comparing two LSNs tells
   you which change happened first, across the whole cluster.

 WHERE LSNs APPEAR:
   pd_lsn      in every 8 KB page header — "the last WAL record that
               touched this page" (Topic 04)
   pg_current_wal_lsn()        the write position
   pg_last_wal_replay_lsn()    a replica's replay position
   ⇒ ★ REPLICATION LAG IS AN LSN SUBTRACTION:
        pg_wal_lsn_diff(sent_lsn, replay_lsn)  → bytes behind
```

### A WAL record, byte by byte

```
 ┌──────────────────────────────────────────────────────────────────┐
 │ XLogRecord HEADER (24 bytes)                                     │
 │   xl_tot_len   (4)  total record length                          │
 │   xl_xid       (4)  the transaction that made this change        │
 │   xl_prev      (8)  LSN of the previous record ★ forms a chain   │
 │   xl_info      (1)  record subtype (INSERT / UPDATE / COMMIT …)  │
 │   xl_rmid      (1)  resource manager: Heap, Btree, Transaction…  │
 │   xl_crc       (4)  CRC-32C over the whole record ★              │
 ├──────────────────────────────────────────────────────────────────┤
 │ BLOCK REFERENCES — which pages this record touches               │
 │   rel (dbOid, tblspcOid, relNode) · forkNum · blockNum           │
 │   flags: has-image? has-data?                                    │
 ├──────────────────────────────────────────────────────────────────┤
 │ [ FULL PAGE IMAGE — the entire 8 KB page, if this is the first   │
 │   write to it since the last checkpoint ]              ★★★       │
 ├──────────────────────────────────────────────────────────────────┤
 │ RECORD DATA — the actual tuple bytes, or the delta               │
 └──────────────────────────────────────────────────────────────────┘

 ★ THE CRC IS WHY RECOVERY KNOWS WHERE TO STOP: replay reads records
   until one fails its checksum, which is the point the crash
   interrupted the write.
```

### ★ Full-page images — the thing that explains WAL volume

```
 THE PROBLEM THEY SOLVE — THE TORN PAGE:
   PostgreSQL writes 8 KB pages. Disks write 512 B or 4 KB sectors.
   A power cut mid-write can leave a page half-old and half-new.
   ⇒ WAL replay assumes the page is intact and applies a DELTA to it.
     A torn page + a delta = garbage.

 THE SOLUTION:
   The FIRST time a page is modified after a checkpoint, the ENTIRE
   8 KB page is copied into the WAL. Replay then RESTORES the whole
   page rather than patching it.
   ⇒ subsequent modifications to the same page before the next
     checkpoint write only the small delta.

 ★★★ THE CONSEQUENCE THAT SURPRISES EVERYONE:

   CHECKPOINT;
   UPDATE orders SET status='x' WHERE id < 50000;   → 402 MB of WAL
   UPDATE orders SET status='y' WHERE id < 50000;   →  11 MB of WAL
                                                       ↑ 36× less

   THE SAME STATEMENT. The first one paid for full-page images of
   every page it touched; the second wrote deltas.

 ⇒ WAL VOLUME SPIKES IMMEDIATELY AFTER EVERY CHECKPOINT, then decays.
 ⇒ MORE FREQUENT CHECKPOINTS = MORE FULL-PAGE IMAGES = MORE WAL.
   ⇒ ★ this is the single biggest lever on WAL volume, and it points
     the opposite way to most people's intuition about checkpoints.

 ⇒ wal_compression = on|lz4|zstd compresses full-page images.
   Typically 40–60% smaller WAL for ~2% CPU. Almost always worth it.
```

### The WAL buffer and the write path

```
 SHARED MEMORY                                   DISK
 ┌────────────────────────┐
 │ WAL BUFFERS (16 MB)    │
 │ ┌────┬────┬────┬────┐  │
 │ │rec │rec │rec │rec │  │  ── walwriter, every 200 ms ──▶  pg_wal/
 │ └────┴────┴────┴────┘  │  ── OR at COMMIT (fsync)  ─────▶  ★ durable
 │        ▲               │
 └────────┼───────────────┘
          │ WALInsertLock — a set of 8 locks so multiple backends can
          │ reserve space concurrently, then copy their bytes in
   backends inserting records

 ★ WAL INSERTION IS DESIGNED FOR CONCURRENCY:
   ① reserve a byte range (a short atomic operation)
   ② copy your record into that range (no lock held)
   ③ mark it complete
   ⇒ many backends write into the buffer simultaneously.

 ★ GROUP COMMIT — why durability is affordable:
   When backend A calls fsync, backends B…Z whose records are already
   in the buffer get flushed by the SAME fsync.
     1 client:   1 fsync per commit  →  1,204 tps
   200 clients:  ~90 commits per fsync → 18,400 tps
   ⇒ commit_delay / commit_siblings can deliberately wait a few
     microseconds to batch more — usually unnecessary, since natural
     concurrency does it.
```

---

## How it works — step by step

### One INSERT, traced through the WAL

```
 INSERT INTO orders (user_id, total_minor) VALUES (7, 249900);

 1. Build the tuple in the backend's private memory. (Topic 05.)

 2. Pin the target heap page (Topic 07). Take an exclusive page latch.

 3. ★ BUILD THE WAL RECORD — BEFORE touching the page:
      xl_rmid = RM_HEAP, xl_info = XLOG_HEAP_INSERT
      block ref: (relNode 16390, fork MAIN, block 1204)
      IF this is the first write to block 1204 since the last
         checkpoint → include the FULL 8 KB PAGE
      ELSE → include only the ~88-byte tuple
      data: the tuple bytes

 4. Reserve space in the WAL buffer, copy the record in, get an LSN.

 5. ★ SET pd_lsn ON THE PAGE = that LSN.
      This is how the buffer manager later knows: "this page cannot be
      written to disk until the WAL is flushed to at least pd_lsn."
      ⇒ THE WRITE-AHEAD RULE, MECHANISED.

 6. Modify the page in the buffer pool. Mark it dirty. Release the latch.

 7. Repeat 3–6 for every index entry (Topic 17's write amplification).

 8. COMMIT:
      a) build an XLOG_XACT_COMMIT record
      b) ★ XLogFlush(commit_lsn) — write and fsync the WAL up to here
      c) set pg_xact[xid] = COMMITTED
      d) return "INSERT 0 1" to the client

 ⇒ AT STEP 8d: the change is DURABLE (it is in pg_wal) and the HEAP
   FILE ON DISK IS STILL UNCHANGED. The checkpointer writes it minutes
   later. (Topic 42.)
```

### The four jobs of one log

```
 ① DURABILITY (Topic 40)
    commit = flush the WAL. The heap follows lazily.

 ② CRASH RECOVERY (Topic 42)
    replay from the last checkpoint forward. Transactions with no
    commit record are implicitly aborted.

 ③ ★ PHYSICAL REPLICATION (Topic 62)
    the SAME byte stream is sent to replicas over the network. A
    replica is a machine continuously performing crash recovery.
    ⇒ this is why physical replication is byte-identical and cannot
      replicate between different major versions or architectures.

 ④ ★ POINT-IN-TIME RECOVERY (Topic 64)
    base backup + archived WAL = the database as of any moment.
    archive_command copies each completed 16 MB segment somewhere safe.

 ⇒ ONE MECHANISM, FOUR FEATURES. And they interact: a replication
   slot or a failing archive_command prevents WAL from being
   recycled, which is how pg_wal fills the disk.
```

### `wal_level` — what the log must contain

```
  minimal   only what crash recovery needs.
            ⇒ some operations (CREATE TABLE AS, CLUSTER, COPY into a
              table created in the same transaction) skip WAL entirely
              and just fsync the file.
            ✗ NO replication, NO PITR.

  replica   (default) + enough for a physical standby and for archiving.

  logical   + enough to decode row-level changes into a logical stream
            ⇒ needed for logical replication and CDC (Topic 76).
            ⇒ costs ~10–20% more WAL, and requires REPLICA IDENTITY
              on tables without a primary key.

 ★ CHANGING IT REQUIRES A RESTART. Set it to `replica` from day one,
   and to `logical` if you might ever want CDC — the cost is small and
   the restart is not.
```

---

## Concept breakdown

```
THE WRITE-AHEAD RULE
└── a data page may not be written to disk before the WAL record
    describing its change is durable.
    ★ mechanised by pd_lsn on every page + XLogFlush before eviction.

LSN — log sequence number
├── a 64-bit byte offset into the infinite WAL stream, "0/1A2B3C8"
├── stored in every page header as pd_lsn
└── ★ replication lag = an LSN subtraction

WAL RECORD
├── 24-byte header: xid, prev-LSN, type, resource manager, ★ CRC
├── block references
├── ★ FULL PAGE IMAGE — the whole 8 KB, on first touch after a checkpoint
└── the delta / tuple bytes

★ FULL-PAGE IMAGES — the key to understanding WAL volume
├── exist to survive TORN PAGES (partial 8 KB writes)
├── written once per page per checkpoint interval
├── ⇒ WAL spikes right after a checkpoint, then decays
├── ⇒ MORE FREQUENT CHECKPOINTS = MORE WAL   ★ counter-intuitive
└── wal_compression = lz4 → typically 40–60% less WAL for ~2% CPU

WAL BUFFERS + GROUP COMMIT
├── 16 MB ring in shared memory; 8 insert locks for concurrency
└── ★ one fsync serves every commit whose record is already buffered
    ⇒ 1 client: 1,204 tps · 200 clients: 18,400 tps

FOUR JOBS, ONE LOG
├── durability · crash recovery · physical replication · PITR
└── ★ and their interaction is the danger: a slot or a failing
    archive_command stops recycling → pg_wal fills → PANIC

wal_level
└── minimal (no replication) · replica (default) · logical (CDC, +10–20%)
```

---

## Diagrams

**Diagram 1 — big picture: why the log makes writes fast**

```
 WITHOUT A LOG — write the pages directly
 ┌──────────────────────────────────────────────────────────────────┐
 │  commit ──▶ page 1204 ──▶ page 8891 ──▶ page 412 ──▶ page 77     │
 │              │              │             │            │         │
 │              ▼              ▼             ▼            ▼         │
 │           RANDOM         RANDOM        RANDOM       RANDOM       │
 │           ~100 µs        ~100 µs       ~100 µs      ~100 µs      │
 │  ⇒ 4 random writes + 4 fsyncs per commit                         │
 └──────────────────────────────────────────────────────────────────┘

 WITH A LOG
 ┌──────────────────────────────────────────────────────────────────┐
 │  commit ──▶ pg_wal ──────────────────────────────────▶           │
 │              │  ONE SEQUENTIAL APPEND, ONE fsync                 │
 │              │  ★ and shared with every other concurrent commit  │
 │              ▼                                                    │
 │           ~830 µs for the fsync, amortised across ~90 commits    │
 │                                                                   │
 │  (minutes later) checkpointer ──▶ pages 1204, 8891, 412, 77      │
 │                                    ★ batched, sorted, off the    │
 │                                      critical path               │
 └──────────────────────────────────────────────────────────────────┘
```

**Diagram 2 — data flow: the write-ahead rule, mechanised**

```
   backend modifies page 1204
            │
            ├──① build the WAL record  ─────▶ WAL buffer (RAM)
            │                                      │
            │                                  returns LSN 0/1A2B3C8
            │                                      │
            ├──② page.pd_lsn = 0/1A2B3C8  ◀────────┘
            │
            ├──③ modify the page bytes, mark DIRTY
            │
            ▼
   ─────────────────────────────────────────────────────────────
   LATER, the checkpointer wants to write page 1204 to disk:
            │
            ├──▶ is WAL flushed to at least page.pd_lsn?
            │         NO  → ★ XLogFlush(page.pd_lsn) FIRST
            │         YES → proceed
            │
            └──▶ write the 8 KB page to base/16388/16390

   ★ THE RULE CANNOT BE VIOLATED, because the page carries the LSN
     that must be durable before it may be written.
```

**Diagram 3 — before/after: full-page images**

```
 IMMEDIATELY AFTER A CHECKPOINT
 ┌──────────────────────────────────────────────────────────────────┐
 │ UPDATE orders SET status='x' WHERE id < 50000;                   │
 │                                                                   │
 │ touches 4,900 distinct pages                                     │
 │ each is the FIRST touch since the checkpoint                     │
 │ ⇒ 4,900 × 8 KB full-page images  = 39 MB                         │
 │   + 4,900 × ~80 B deltas                                         │
 │   + index pages, same treatment                                  │
 │ ⇒ ★ 402 MB of WAL                                                │
 └──────────────────────────────────────────────────────────────────┘

 THE SAME STATEMENT, RUN AGAIN (before the next checkpoint)
 ┌──────────────────────────────────────────────────────────────────┐
 │ every page has already been imaged this checkpoint interval      │
 │ ⇒ deltas only                                                    │
 │ ⇒ ★ 11 MB of WAL                                                 │
 └──────────────────────────────────────────────────────────────────┘
                              36× less

 ⇒ THE LEVER: checkpoint_timeout 5 min → 30 min
   fewer checkpoints ⇒ fewer full-page images ⇒ far less WAL
   ⚠ AT THE COST OF: longer crash recovery (Topic 42) and more dirty
     pages to flush at each checkpoint.
```

---

## Example 1 — basic

**Step 1 — watch the LSN advance.**

```sql
SELECT pg_current_wal_lsn() AS before \gset
CREATE TABLE wal_demo (id bigserial PRIMARY KEY, v text);
INSERT INTO wal_demo (v) VALUES ('hello');
SELECT pg_current_wal_lsn() AS after,
       pg_size_pretty(pg_current_wal_lsn() - :'before'::pg_lsn) AS wal_generated;
```
```
   after    | wal_generated
------------+---------------
 0/1A2C4F8  | 8464 bytes
```
**8.4 KB for one row** — because `CREATE TABLE` plus the first insert into a fresh page produced a full-page image.

**Step 2 — the full-page-image effect, measured.**

```sql
CREATE TABLE fpi_demo (id bigserial PRIMARY KEY, n int, pad text);
INSERT INTO fpi_demo (n, pad) SELECT i, repeat('x',100) FROM generate_series(1,200000) i;
VACUUM ANALYZE fpi_demo;

CHECKPOINT;
SELECT pg_current_wal_lsn() AS l1 \gset
UPDATE fpi_demo SET n = n + 1 WHERE id < 50000;
SELECT pg_size_pretty(pg_current_wal_lsn() - :'l1'::pg_lsn) AS first_touch;

SELECT pg_current_wal_lsn() AS l2 \gset
UPDATE fpi_demo SET n = n + 1 WHERE id < 50000;
SELECT pg_size_pretty(pg_current_wal_lsn() - :'l2'::pg_lsn) AS second_touch;
```
```
 first_touch
-------------
 402 MB

 second_touch
--------------
 11 MB              ★ 36× less, for the identical statement
```

**Step 3 — turn on WAL compression and re-measure.**

```sql
SHOW wal_compression;   -- off
SET wal_compression = 'lz4';   -- (superuser; ALTER SYSTEM for persistence)

CHECKPOINT;
SELECT pg_current_wal_lsn() AS l3 \gset
UPDATE fpi_demo SET n = n + 1 WHERE id < 50000;
SELECT pg_size_pretty(pg_current_wal_lsn() - :'l3'::pg_lsn) AS compressed_fpi;
```
```
 compressed_fpi
----------------
 164 MB          ★ 402 → 164 MB, 59% smaller
```

**Step 4 — inspect real WAL records.**

```bash
docker exec -it pg-lab pg_waldump -p /var/lib/postgresql/data/pg_wal \
  --start 0/1A2C000 --limit 8
```
```
rmgr: Heap    len (rec/tot): 59/  8523, tx: 91014, lsn: 0/1A2C4F8,
  desc: INSERT off 12 flags 0x00, blkref #0: rel 1663/16388/16390 blk 1204 FPW
                                                                        ↑★
rmgr: Btree   len (rec/tot): 64/    64, tx: 91014, lsn: 0/1A2E650,
  desc: INSERT_LEAF off 41, blkref #0: rel 1663/16388/16393 blk 8
rmgr: Transaction len (rec/tot): 34/ 34, tx: 91014, lsn: 0/1A2E690,
  desc: COMMIT 2026-08-12 09:14:22.104881+00
```
```
 ★ READ THE FIRST LINE: rec 59 bytes, tot 8,523 bytes.
   The difference is the FULL PAGE IMAGE (marked FPW).
   The second line has no FPW, so rec = tot = 64 bytes.
```

**Step 5 — group commit, demonstrated.**

```bash
pgbench -i -s 10 shop
pgbench -c 1   -j 1 -T 20 -N shop
pgbench -c 50  -j 4 -T 20 -N shop
pgbench -c 200 -j 8 -T 20 -N shop
```
```
  -c 1  : tps = 1,188      ← 1 fsync per commit
  -c 50 : tps = 9,204      ← ~8 commits per fsync
  -c 200: tps = 18,412     ← ~90 commits per fsync
```
```sql
-- see the fsync count directly
SELECT wal_records, wal_bytes, wal_fpi, wal_sync, wal_sync_time
FROM pg_stat_wal;
```
```
 wal_records | wal_bytes  | wal_fpi | wal_sync | wal_sync_time
-------------+------------+---------+----------+---------------
     8412004 | 4102884120 |  204882 |    91204 |     78412.204
```
**8.4M records, 91k fsyncs — 92 records per fsync.** And `wal_fpi` = 204,882 full-page images, which you can now attribute to your checkpoint frequency.

**Step 6 — the checkpoint/WAL trade, both directions.**

```sql
SHOW checkpoint_timeout;   -- 5min
SELECT checkpoints_timed, checkpoints_req, buffers_checkpoint FROM pg_stat_bgwriter;

-- generate load for 5 minutes, record WAL generated
SELECT pg_current_wal_lsn() AS a \gset
-- …run pgbench for 300 s…
SELECT pg_size_pretty(pg_current_wal_lsn() - :'a'::pg_lsn) AS wal_5min_ckpt;

ALTER SYSTEM SET checkpoint_timeout = '30min';
SELECT pg_reload_conf();
-- …repeat…
```
```
 checkpoint_timeout = 5min  :  WAL = 41 GB / 5 min
 checkpoint_timeout = 30min :  WAL = 12 GB / 5 min     ★ 3.4× less
```
⚠ **The cost:** crash recovery must now replay up to 30 minutes of WAL instead of 5. That is Topic 42's trade, and it is a deliberate one.

**Step 7 — the outage nobody expects.**

```sql
-- create a replication slot and never consume it
SELECT pg_create_physical_replication_slot('orphan_slot');

-- generate WAL
INSERT INTO fpi_demo (n,pad) SELECT i, repeat('x',100) FROM generate_series(1,500000) i;
CHECKPOINT;

SELECT slot_name, active,
       pg_size_pretty(pg_current_wal_lsn() - restart_lsn) AS wal_retained
FROM pg_replication_slots;
```
```
  slot_name  | active | wal_retained
-------------+--------+--------------
 orphan_slot | f      | 1204 MB        ★ and growing forever
```
```sql
SELECT pg_size_pretty(sum(size)) AS pg_wal_size FROM pg_ls_waldir();
-- pg_wal keeps growing. When the volume fills:
--   PANIC: could not write to file "pg_wal/…": No space left on device
--   ⇒ the database shuts down.

SELECT pg_drop_replication_slot('orphan_slot');   -- ★ the fix
```

---

## Example 2 — production scenario

**The situation.** A SaaS platform. Three WAL-related incidents in six weeks.

```
 WEEK 1: replicas fall 40 minutes behind every night at 02:00.
 WEEK 3: pg_wal fills the volume at 04:12; the primary PANICs and shuts down.
 WEEK 6: p99 write latency doubles after a "performance tuning" change.
```

**Step 1 — measure WAL volume and attribute it.**

```sql
SELECT
  pg_size_pretty(pg_current_wal_lsn() - '0/0'::pg_lsn) AS total_wal_ever,
  wal_records, wal_fpi,
  pg_size_pretty(wal_bytes) AS wal_bytes,
  round(100.0 * wal_fpi / nullif(wal_records,0), 1) AS pct_records_with_fpi,
  wal_sync, round((wal_sync_time/1000)::numeric,1) AS sync_seconds
FROM pg_stat_wal;
```
```
 wal_records | wal_fpi  | wal_bytes | pct_records_with_fpi | sync_seconds
-------------+----------+-----------+----------------------+--------------
   412008841 | 88204112 | 2.1 TB    |                 21.4 |      41208.4
```

**21.4% of WAL records carry a full 8 KB page image.** That is very high, and it points directly at checkpoint frequency.

```sql
SELECT checkpoints_timed, checkpoints_req,
       round(checkpoints_req::numeric*100/(checkpoints_timed+checkpoints_req),1)
         AS pct_forced
FROM pg_stat_bgwriter;
SHOW checkpoint_timeout;
SHOW max_wal_size;
```
```
 checkpoints_timed | checkpoints_req | pct_forced
-------------------+-----------------+------------
              2880 |           41208 |       93.5   ⚠⚠

 checkpoint_timeout = 5min
 max_wal_size       = 1GB          ★ THE PROBLEM
```

```
 ★ 93.5% OF CHECKPOINTS ARE *REQUESTED*, NOT TIMED.
   A requested checkpoint means max_wal_size was exceeded — the
   database is checkpointing because it ran out of WAL space, not
   because the timer fired.

 ⇒ THE VICIOUS CYCLE:
     max_wal_size too small
       → frequent forced checkpoints
         → more full-page images
           → MORE WAL
             → max_wal_size exceeded sooner
               → even more frequent checkpoints …

 ⇒ ★ THIS IS THE SINGLE MOST COMMON WAL MISCONFIGURATION, and it is
   self-reinforcing, which is why it looks like a mystery.
```

**Step 2 — fix the checkpoint configuration.**

```sql
ALTER SYSTEM SET max_wal_size = '32GB';           -- was 1GB
ALTER SYSTEM SET min_wal_size = '4GB';
ALTER SYSTEM SET checkpoint_timeout = '30min';    -- was 5min
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
ALTER SYSTEM SET wal_compression = 'lz4';
SELECT pg_reload_conf();
```

```
 SIZING max_wal_size — the reasoning, not a guess:
   it must hold the WAL generated between two checkpoints, with
   headroom.
     WAL rate × checkpoint_timeout × 2
   here: 24 MB/s × 1800 s × 2 ≈ 86 GB… but the WAL rate will DROP once
   full-page images fall. Start at 32 GB, measure, adjust.

 checkpoint_completion_target = 0.9
   spreads the checkpoint's page writes over 90% of the interval
   instead of dumping them at once — smoothing the I/O spike.
```

**After 24 hours:**
```
 pct_forced           : 93.5%  →  4.2%
 pct_records_with_fpi : 21.4%  →   3.1%
 WAL generated/hour   : 410 GB →   88 GB      ★ 4.7× less
 replica lag at 02:00 : 40 min →    8 s
```

**Step 3 — the pg_wal disk-full incident.**

```sql
-- what was retaining WAL?
SELECT slot_name, slot_type, active, wal_status,
       pg_size_pretty(pg_current_wal_lsn() - restart_lsn) AS retained,
       pg_size_pretty(safe_wal_size) AS headroom
FROM pg_replication_slots ORDER BY 5 DESC;
```
```
   slot_name    | slot_type | active | wal_status |  retained  | headroom
----------------+-----------+--------+------------+------------+----------
 replica_dr     | physical  | f      | extended   | 412 GB     | -12 GB   ⚠⚠
 replica_read_1 | physical  | t      | reserved   | 84 MB      | 31 GB
 cdc_analytics  | logical   | f      | lost       | 0 bytes    |          ⚠
```

```
 ★ replica_dr: a DR standby decommissioned three months ago. Its slot
   was never dropped. It retained 412 GB of WAL, and `safe_wal_size`
   is NEGATIVE — meaning the primary is past the point where it can
   guarantee the slot's WAL is still available.

 ★ cdc_analytics: wal_status = 'lost'. This slot already exceeded
   max_slot_wal_keep_size and its WAL was removed. The CDC consumer
   CANNOT resume — it must be rebuilt from a snapshot. (Topic 76.)
```

```sql
-- the immediate fix
SELECT pg_drop_replication_slot('replica_dr');
SELECT pg_drop_replication_slot('cdc_analytics');

-- the permanent guard
ALTER SYSTEM SET max_slot_wal_keep_size = '64GB';
-- ⇒ a slot that falls further behind than this is INVALIDATED
--   rather than being allowed to fill the disk.
--   ★ the consumer breaks; the primary survives. That is the right
--     trade — a broken replica is recoverable, a full disk is an outage.
SELECT pg_reload_conf();
```

```sql
-- and the archive side, which had also been failing
SELECT archived_count, failed_count, last_failed_wal, last_failed_time
FROM pg_stat_archiver;
```
```
 archived_count | failed_count |    last_failed_wal       |  last_failed_time
----------------+--------------+--------------------------+--------------------
        4102884 |         8412 | 00000001000000A200000041 | 2026-08-12 04:08:11
```
**8,412 archive failures.** The S3 credentials had rotated. Every `.ready` file accumulated in `archive_status/`, and **WAL cannot be recycled until it is archived** — a second, independent cause of the same disk-full outage.

```bash
# make archive failures loud
# archive_command must return non-zero on failure and must be idempotent
archive_command = 'test ! -f /archive/%f && aws s3 cp %p s3://wal-archive/%f'
```

**Step 4 — the p99 regression.**

```sql
SELECT name, setting, source FROM pg_settings
WHERE name IN ('synchronous_commit','synchronous_standby_names','wal_sync_method');
```
```
          name             |    setting     |  source
---------------------------+----------------+------------
 synchronous_commit        | remote_apply   | ★ config file
 synchronous_standby_names | ANY 2 (r1,r2)  | ★ config file
 wal_sync_method           | fdatasync      | default
```

```
 ★ SOMEONE SET synchronous_commit = remote_apply WITH 2 SYNCHRONOUS
   REPLICAS. Every commit now waits for:
     local fsync (0.8 ms) + network round trip (1.2 ms)
     + the replica's fsync (0.8 ms) + the replica's REPLAY (0.4 ms)
     × the slower of two replicas
   ⇒ ~4.1 ms per commit, vs 0.8 ms before.
```

```sql
-- measure what each level actually costs, on this hardware
\timing on
SET synchronous_commit = off;          -- 0.09 ms/commit
SET synchronous_commit = local;        -- 0.81 ms
SET synchronous_commit = on;           -- 0.83 ms  (= local, no sync standbys… )
SET synchronous_commit = remote_write; -- 2.10 ms
SET synchronous_commit = remote_apply; -- 4.12 ms
RESET synchronous_commit;
```

**The decision, made explicitly:**

```sql
-- ONE synchronous replica is enough for zero-RPO failover.
-- remote_write, not remote_apply — we do not require read-your-writes
-- on the standby.
ALTER SYSTEM SET synchronous_standby_names = 'ANY 1 (r1, r2)';
ALTER SYSTEM SET synchronous_commit = 'remote_write';
SELECT pg_reload_conf();
-- ⇒ 4.12 ms → 2.10 ms, and RPO is still zero.
```

**Step 5 — the monitoring that would have caught all three.**

```sql
CREATE OR REPLACE VIEW wal_health AS
SELECT
  (SELECT round(100.0*checkpoints_req/nullif(checkpoints_timed+checkpoints_req,0),1)
     FROM pg_stat_bgwriter)                              AS pct_forced_checkpoints,
  (SELECT round(100.0*wal_fpi/nullif(wal_records,0),1)
     FROM pg_stat_wal)                                    AS pct_fpi,
  (SELECT pg_size_pretty(sum(size)) FROM pg_ls_waldir())  AS pg_wal_size,
  (SELECT count(*) FROM pg_replication_slots WHERE NOT active) AS inactive_slots,
  (SELECT pg_size_pretty(max(pg_current_wal_lsn() - restart_lsn))
     FROM pg_replication_slots)                           AS max_slot_retention,
  (SELECT failed_count FROM pg_stat_archiver)             AS archive_failures,
  (SELECT count(*) FROM pg_ls_dir('pg_wal/archive_status')
     WHERE pg_ls_dir LIKE '%.ready')                      AS pending_archives;
```
```
 ALERT ON:
   pct_forced_checkpoints > 20    → max_wal_size too small
   pct_fpi > 10                   → checkpoints too frequent
   inactive_slots > 0             → ★ a slot retaining WAL for nobody
   max_slot_retention > 32 GB     → a replica or CDC consumer is behind
   archive_failures increasing    → ★ WAL cannot be recycled
   pending_archives > 100         → the archive is falling behind
```

**Step 6 — results.**

| | Before | After |
|---|---|---|
| Forced checkpoints | 93.5% | **4.2%** |
| Records with a full-page image | 21.4% | **3.1%** |
| WAL per hour | 410 GB | **88 GB** |
| Replica lag at 02:00 | 40 min | **8 s** |
| pg_wal disk-full incidents | 1 (outage) | **0** — bounded by `max_slot_wal_keep_size` |
| Commit p99 | 4.1 ms | **2.1 ms** |
| RPO | 0 | **0** (unchanged) |

**Not one line of application code changed.**

---

## Common mistakes

**1. `max_wal_size` too small.**
- *Symptom:* 90%+ forced checkpoints, huge WAL volume, replica lag — and it looks self-reinforcing because it is.
- *Engine-level why:* exceeding `max_wal_size` forces a checkpoint; frequent checkpoints mean more full-page images; more full-page images mean more WAL.
- *Diagnose:* `checkpoints_req` vs `checkpoints_timed` in `pg_stat_bgwriter`.
- *Fix:* size it as `WAL rate × checkpoint_timeout × 2`, then re-measure — the rate will drop.

**2. Not knowing full-page images exist.**
- *Symptom:* "our WAL volume makes no sense — an 88-byte insert produced 40 KB."
- *Fix:* `wal_fpi` in `pg_stat_wal`, and `pg_waldump` showing `FPW`. Then `wal_compression = lz4` and a longer `checkpoint_timeout`.

**3. An orphaned replication slot.**
- *Symptom:* pg_wal grows without bound until the volume fills and the database PANICs.
- *Engine-level why:* a slot guarantees WAL retention for its consumer. An inactive slot guarantees it forever.
- *Diagnose:* `SELECT * FROM pg_replication_slots WHERE NOT active;`
- *Fix:* drop it, and set `max_slot_wal_keep_size` so the slot is invalidated instead of the disk filling.

**4. A silently failing `archive_command`.**
- *Symptom:* the same disk-full outage, from a different cause.
- *Engine-level why:* WAL cannot be recycled until archived. Failures accumulate `.ready` files.
- *Diagnose:* `pg_stat_archiver.failed_count`, and the `.ready` count.
- *Fix:* alert on it. And make the command idempotent — `test ! -f` first, so a retry doesn't corrupt an existing archive.

**5. `synchronous_commit = remote_apply` without needing it.**
- *Symptom:* commit latency 5× higher than expected.
- *Engine-level why:* every commit waits for a network round trip plus the replica's fsync *and* replay.
- *Fix:* `remote_write` gives zero RPO without waiting for replay. Use `remote_apply` only when you genuinely need read-your-writes on the standby.

**6. `wal_level = minimal` in production.**
- *Symptom:* you cannot add a replica or take a PITR backup without a restart.
- *Fix:* `replica` from day one; `logical` if CDC is remotely plausible. The change needs a restart, so decide early.

**7. Assuming a `CHECKPOINT` is harmless.**
- *Symptom:* running `CHECKPOINT` before a benchmark and getting anomalous WAL numbers.
- *Engine-level why:* it resets the full-page-image state, so the next writes to every page are 8 KB each.
- *Fix:* it's a useful *diagnostic* (Example 1 step 2) but never routine on a busy system.

---

## Hands-on proof

**PROVE IT #1 — the LSN advances.** (Example 1, step 1.)
**PROVE IT #2 — full-page images, 36×.** (Example 1, step 2.)
**PROVE IT #3 — WAL compression.** (Example 1, step 3.)
**PROVE IT #4 — read real records with `pg_waldump`.** (Example 1, step 4.)
**PROVE IT #5 — group commit.** (Example 1, step 5.)
**PROVE IT #6 — the orphaned slot.** (Example 1, step 7.)

**PROVE IT #7 — attribute your WAL volume.**
```sql
SELECT wal_records, wal_fpi,
       round(100.0*wal_fpi/nullif(wal_records,0),1) AS pct_fpi,
       pg_size_pretty(wal_bytes) AS total,
       wal_sync, round((wal_sync_time/1000)::numeric,1) AS fsync_seconds,
       round(wal_records::numeric/nullif(wal_sync,0),1) AS records_per_fsync
FROM pg_stat_wal;
-- ★ pct_fpi > 10 → checkpoints are too frequent
-- ★ records_per_fsync low → little group commit (low concurrency, or
--   synchronous replication)
```

**PROVE IT #8 — the checkpoint diagnosis.**
```sql
SELECT checkpoints_timed, checkpoints_req,
       round(100.0*checkpoints_req/nullif(checkpoints_timed+checkpoints_req,0),1)
         AS pct_forced,
       buffers_checkpoint, buffers_backend,
       round(100.0*buffers_backend/nullif(buffers_checkpoint+buffers_clean+buffers_backend,0),1)
         AS pct_backend_writes
FROM pg_stat_bgwriter;
-- ★ pct_forced > 20 → max_wal_size too small
-- ★ pct_backend_writes > 10 → bgwriter too lazy (Topic 07)
```

**PROVE IT #9 — what is retaining WAL right now.**
```sql
SELECT slot_name, active, wal_status,
       pg_size_pretty(pg_current_wal_lsn() - restart_lsn) AS retained
FROM pg_replication_slots
UNION ALL
SELECT 'ARCHIVE', NULL, NULL,
       (SELECT count(*)::text || ' pending' FROM pg_ls_dir('pg_wal/archive_status')
        WHERE pg_ls_dir LIKE '%.ready');
SELECT pg_size_pretty(sum(size)) AS pg_wal_total FROM pg_ls_waldir();
```

---

## The design decision framework

```
CHECKPOINT AND WAL SIZING — the trade, stated:
  LONGER checkpoint_timeout / LARGER max_wal_size
    ✓ far fewer full-page images ⇒ much less WAL
    ✓ less checkpoint I/O
    ✗ longer crash recovery (Topic 42)
    ✗ more dirty pages to flush at each checkpoint
  SHORTER
    ✓ faster crash recovery
    ✗ ★ dramatically more WAL, and a self-reinforcing cycle

  ⇒ START HERE:
      checkpoint_timeout = 15–30 min
      max_wal_size = WAL_rate × checkpoint_timeout × 2
      checkpoint_completion_target = 0.9
      wal_compression = lz4            ★ almost always worth it
  ⇒ THEN VERIFY: checkpoints_req should be < 20% of the total.

DURABILITY LEVEL — choose by RPO, not by feel (Topic 40):
  off           regenerable data only. ~600 ms loss window.
  local / on    single-node zero loss.  ★ the default, and usually right
  remote_write  zero RPO on failover, no wait for replay
  remote_apply  ★ only if you need read-your-writes on a standby
                  — it adds a full replay wait to every commit

wal_level
  replica  ★ the default; set it and forget it
  logical  if CDC is even plausible. +10–20% WAL, but changing it
           later needs a RESTART.

★ THE THREE THINGS THAT MAKE pg_wal FILL — all of them external:
  ① an inactive replication slot   → drop it; set max_slot_wal_keep_size
  ② a failing archive_command      → alert on pg_stat_archiver.failed_count
  ③ a very long-running transaction (in some versions)
  ⇒ ★ SET max_slot_wal_keep_size. It converts "the primary dies" into
    "one consumer breaks" — which is always the better failure.

THE SIGNAL TO LOOK FOR:
      SELECT round(100.0*checkpoints_req/
                   nullif(checkpoints_timed+checkpoints_req,0),1) AS pct_forced
      FROM pg_stat_bgwriter;
      SELECT round(100.0*wal_fpi/nullif(wal_records,0),1) AS pct_fpi
      FROM pg_stat_wal;
      SELECT slot_name, active,
             pg_size_pretty(pg_current_wal_lsn()-restart_lsn) FROM pg_replication_slots;

  • pct_forced > 20        → max_wal_size too small ⇒ the vicious cycle
  • pct_fpi > 10           → checkpoints too frequent
  • any inactive slot      → ★ a disk-full outage in waiting
  • archive failed_count rising → the same outage, different cause
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Measure WAL generated by the same `UPDATE` run twice — once immediately after a `CHECKPOINT`, once straight after. Explain the ratio. Then enable `wal_compression = lz4` and re-measure, and use `pg_waldump` to find a record where `tot` is much larger than `rec` and identify why.

### Exercise 2 — medium (apply it)
Demonstrate group commit: run `pgbench` at 1, 10, 50 and 200 clients and record tps and `wal_sync` from `pg_stat_wal` for each. Compute records-per-fsync at each level. Then:
(a) explain why durability is cheaper under load,
(b) set `synchronous_commit = off` and repeat — explain why the *relative* gain shrinks as concurrency rises,
(c) state the exact data-loss window you accepted and how you would justify it.

### Exercise 3 — hard (production simulation)
A platform has: 93.5% forced checkpoints, 21.4% of WAL records carrying full-page images, 410 GB of WAL per hour, replicas 40 minutes behind at 02:00, one pg_wal disk-full PANIC, and a commit p99 of 4.1 ms.

(a) Explain the self-reinforcing cycle that produces both the forced checkpoints and the WAL volume, and give the query that identifies it.
(b) Size `max_wal_size` from first principles for a 24 MB/s WAL rate and a 30-minute checkpoint timeout. Explain why the answer changes after the fix.
(c) The disk-full incident had **two independent causes**. Identify both and give the query for each.
(d) One replication slot shows `wal_status = 'lost'`. Explain what that means for its consumer and why it cannot simply resume.
(e) Explain `max_slot_wal_keep_size` and argue why invalidating a slot is the better failure mode than filling the disk.
(f) The commit p99 doubled after a config change. Identify the likely setting, measure each `synchronous_commit` level, and choose one — justifying it in terms of RPO rather than latency.
(g) Write the `wal_health` monitoring view and the alert threshold for each field.
(h) State the crash-recovery cost you accepted by raising `checkpoint_timeout`, and how you would measure it (forward reference to Topic 42).

---

## Mental model checkpoint

1. State the write-ahead rule. What mechanism on each page enforces it?
2. What is an LSN, and where does it appear besides the WAL?
3. What is a full-page image, what problem does it solve, and when is one written?
4. Why does a *more frequent* checkpoint produce *more* WAL? Why is that counter-intuitive?
5. Explain group commit. Why is durability cheap at high concurrency and expensive when idle?
6. Name the four jobs the WAL performs.
7. Name the two independent ways an external consumer can fill `pg_wal` and take down the primary.

---

## Quick reference card

**The rule:** a data page may not reach disk before the WAL record describing its change is durable. Enforced by `pd_lsn` on every page.

| Concept | |
|---|---|
| WAL segment | 16 MB, recycled not deleted |
| LSN | 64-bit byte offset, `0/1A2B3C8` |
| Full-page image | whole 8 KB, on first touch after a checkpoint |
| Group commit | one fsync serves many commits |
| Four jobs | durability · recovery · replication · PITR |

**Key settings**

| Setting | Default | Guidance |
|---|---|---|
| `checkpoint_timeout` | 5 min | **15–30 min** |
| `max_wal_size` | 1 GB | WAL rate × timeout × 2 |
| `checkpoint_completion_target` | 0.9 | 0.9 |
| `wal_compression` | off | **`lz4`** — 40–60% less WAL |
| `wal_level` | replica | `logical` if CDC is plausible |
| `max_slot_wal_keep_size` | −1 (unlimited) | **set it** — e.g. 64 GB |
| `synchronous_commit` | on | `remote_write` for zero-RPO failover |

**The three diagnostics**

```sql
-- ① the vicious cycle
SELECT round(100.0*checkpoints_req/(checkpoints_timed+checkpoints_req),1) FROM pg_stat_bgwriter;
-- > 20% → max_wal_size too small

-- ② full-page image share
SELECT round(100.0*wal_fpi/wal_records,1) FROM pg_stat_wal;   -- > 10% → checkpoints too frequent

-- ③ what is retaining WAL
SELECT slot_name, active, pg_size_pretty(pg_current_wal_lsn()-restart_lsn)
FROM pg_replication_slots;
SELECT failed_count FROM pg_stat_archiver;
```

---

## When would I use this at work?

1. **"Why is our WAL volume so high?"** Two queries — forced-checkpoint percentage and full-page-image percentage — usually identify an undersized `max_wal_size` and its self-reinforcing cycle in under a minute.

2. **A pg_wal disk-full outage.** The cause is almost always external: an orphaned replication slot or a failing archive command. `max_slot_wal_keep_size` converts that outage into a broken consumer, which is always recoverable.

3. **Commit latency that doubled with no code change.** Checking `synchronous_commit` and `synchronous_standby_names` finds it immediately, and measuring each level turns "how durable do we want to be?" into a concrete RPO-vs-milliseconds decision.

---

## Connected topics

**Understand before this:** 04 (pages and `pd_lsn`), 07 (dirty pages and the buffer pool), 39–40 (the commit sequence and durability).

**This unlocks:**
- **42** — crash recovery: the WAL read back, and the checkpoint trade
- **17** — write amplification: WAL volume per index, measured
- **62** — replication: the same byte stream, over the network
- **64** — PITR: the same log, archived
- **76** — CDC and logical decoding, which needs `wal_level = logical`
