# 42 — Crash Recovery and Checkpoints
## Phase: Transactions & Concurrency

---

## ELI5 — The Simple Analogy

Back to the kitchen with the order book.

The book is now four thousand pages long. The restaurant burns down. You rebuild it, and you must work out what state every dish was in.

**The naive answer:** read the book from page one. Four thousand pages. You'd be reading for a week before you served anything.

**The real answer:** every so often, the head chef stops, makes sure every started dish is actually plated and stored, and writes a line in the book: *"as of here, everything before this point is done and safe."* That line is a **checkpoint**.

Now recovery is easy. Find the last such line, and replay only what came after it. If checkpoints happen every hour, you replay at most an hour.

And here is the tension that runs through the whole topic: **the more often the chef stops to reconcile, the faster the rebuild — and the more work he does during normal service.** Frequent checkpoints mean fast recovery and slow steady state. Rare checkpoints mean fast steady state and slow recovery.

You are choosing your downtime in advance.

---

## Where this fits in the big picture

```
   41 the WAL (what is written, and why)
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 42 CRASH RECOVERY & CHECKPOINTS          │ ← YOU ARE HERE
        │ the WAL, read back                       │
        └────────────────────┬─────────────────────┘
                             ▼
              62 replication (a replica is a machine
                 permanently doing crash recovery)
              64 PITR (recovery, but stopped at a chosen point)
              07 the buffer pool (whose dirty pages
                 checkpoints flush)
```

Topic 41 was the **format**. This is the **algorithm** — and the reason they are separate topics is that most people can half-explain durability precisely because they've conflated the two.

---

## What is this?

**A checkpoint** is a point in the WAL at which PostgreSQL guarantees that every change before it has been written to the data files. It writes all dirty buffers to disk and records a checkpoint record in the WAL.

**Crash recovery** is what happens on startup after an unclean shutdown: find the last checkpoint, replay every WAL record from there forward, then mark any transaction without a commit record as aborted.

The relationship: **the checkpoint is the starting line for recovery.** Everything about the trade-off follows from that.

---

## Why does it matter for a backend developer?

Because this is the topic that turns "how long will we be down?" from a guess into a number you chose:

```
 ① RECOVERY TIME IS A CONFIGURATION DECISION YOU MAKE IN ADVANCE
    checkpoint_timeout = 5 min  →  ~30 s of recovery
    checkpoint_timeout = 30 min →  ~4 min of recovery
    ⇒ ★ you are picking your RTO at config time, not at incident time.

 ② AND IT PULLS AGAINST WAL VOLUME (Topic 41)
    the same setting that shortens recovery lengthens your WAL by 3–5×.
    ⇒ ★ these two topics disagree, deliberately. You must decide.

 ③ CHECKPOINT I/O SPIKES ARE A REAL LATENCY SOURCE
    "our p99 spikes every 5 minutes and nobody knows why" is almost
    always the checkpointer dumping dirty buffers.

 ④ ★ A REPLICA IS A MACHINE PERPETUALLY IN RECOVERY
    Everything here applies to replication. Replay speed, conflicts,
    and lag are all crash recovery running continuously (Topic 62).

 ⑤ AND YOU CANNOT TEST IT UNLESS YOU DO IT
    Most teams have never measured their recovery time. The number
    they quote in their DR plan is invented.
```

---

## The physical reality

### `pg_control` — the 8 KB file everything starts from

```
 $PGDATA/global/pg_control      (8,192 bytes, checksummed)
 ┌──────────────────────────────────────────────────────────────────┐
 │ state              DB_IN_PRODUCTION | DB_SHUTDOWNED | …          │
 │ checkPoint         ★ LSN of the last completed checkpoint RECORD │
 │ prior checkPoint   (removed in PG11+; only one is kept now)      │
 │ redo location      ★★★ WHERE REPLAY MUST START                   │
 │ next XID, oldest XID, next OID                                   │
 │ time of last checkpoint                                          │
 │ full_page_writes, wal_level, data checksum version               │
 │ system identifier  (must match between primary and replica)      │
 └──────────────────────────────────────────────────────────────────┘

 ★ THE TWO LSNs ARE DIFFERENT, AND THE DIFFERENCE IS THE POINT:
   REDO location  — where the checkpoint STARTED
   checkPoint     — where the checkpoint record was WRITTEN (it finished)

   A checkpoint takes minutes. Changes happen during it. So replay must
   begin at the moment the checkpoint BEGAN, not where it ended.
   ⇒ recovery starts at `redo location`, always.
```

```bash
pg_controldata /var/lib/postgresql/data | head -20
```
```
pg_control version number:            1300
Database cluster state:               in production
Latest checkpoint location:           0/1A2B3C8
Latest checkpoint's REDO location:    0/18F4200      ★ replay starts here
Latest checkpoint's REDO WAL file:    000000010000000000000018
Time of latest checkpoint:            Tue 12 Aug 2026 09:14:22
```

### The checkpoint, step by step

```
 A CHECKPOINT IS TRIGGERED BY (in practice, one of):
   • checkpoint_timeout elapsing              → "timed"
   • max_wal_size being exceeded              → "requested"  ⚠ (Topic 41)
   • an explicit CHECKPOINT command
   • a clean shutdown
   • the start of a base backup

 WHAT IT DOES:
  ① Record the current WAL insert position as the REDO POINT.
     ★ Note: BEFORE doing any work. This is what makes it safe.
  ② Scan the entire buffer pool and mark every currently-dirty buffer
     with BM_CHECKPOINT_NEEDED.
  ③ Write those buffers to disk, SPREAD OVER
     checkpoint_completion_target × checkpoint_timeout.
     ⇒ at 0.9 and 30 min, the writes are spread over 27 minutes.
     ⇒ ★ this is what stops the I/O spike.
  ④ fsync every file that was touched.
  ⑤ Write an XLOG_CHECKPOINT_ONLINE record containing the redo point.
  ⑥ Update pg_control.
  ⑦ Recycle or remove WAL segments older than the redo point —
     subject to replication slots and archiving (Topic 41).

 ⇒ ★ A CHECKPOINT IS NOT A PAUSE. The database serves traffic
   throughout. Its cost is I/O, not a stall.
```

### Crash recovery, step by step

```
 ON STARTUP, PostgreSQL reads pg_control:
   state = DB_SHUTDOWNED       → clean. Start normally. No recovery.
   state = DB_IN_PRODUCTION    → ★ UNCLEAN SHUTDOWN. Recover.

 THE RECOVERY ALGORITHM:

  ① Read the redo location from pg_control.
  ② Open the WAL segment containing it.
  ③ REDO PHASE — for each record, in order:
       a) verify the CRC. ★ A failure means we've reached the end of
          valid WAL — that is where the crash interrupted the write.
          Recovery stops here. This is normal, not an error.
       b) read the target page into the buffer pool
       c) ★ COMPARE page.pd_lsn WITH the record's LSN:
            page.pd_lsn >= record LSN → the change is ALREADY on the
              page. SKIP IT.
            page.pd_lsn <  record LSN → apply the change.
          ⇒ ★★★ THIS IS WHAT MAKES REDO IDEMPOTENT. Recovery can be
            interrupted and restarted any number of times and reach the
            same state. Crucial, because a crash during recovery is a
            real possibility.
       d) if the record carries a FULL PAGE IMAGE, overwrite the whole
          page rather than applying a delta — this is how torn pages
          are repaired (Topic 41).
  ④ At the end of replay: any transaction with no COMMIT record is
     implicitly ABORTED. ★ NO UNDO PASS IS NEEDED — its tuples exist
     on pages but pg_xact never says COMMITTED, so nothing can see
     them. (Topic 39.)
  ⑤ Perform a checkpoint, update pg_control, open for connections.

 ⇒ ★ RECOVERY IS REDO-ONLY. PostgreSQL has no undo log, so there is
   no undo phase. This is the direct consequence of the MVCC design.
```

### ARIES, and where PostgreSQL differs

```
 ARIES (Mohan et al., 1992) is the canonical recovery algorithm.
 Three phases:
   ① ANALYSIS — scan from the checkpoint to find which transactions
                were in flight and which pages were dirty
   ② REDO     — replay ALL changes, committed or not ("repeating history")
   ③ UNDO     — roll back the transactions that never committed,
                using the undo log, writing compensation records

 POSTGRESQL:
   ① ANALYSIS — ★ mostly unnecessary. The checkpoint record itself
                lists the running transactions.
   ② REDO     — ✓ identical in spirit. Idempotent via page LSN.
   ③ UNDO     — ★ DOES NOT EXIST.

 ⇒ WHY: in ARIES-style engines (Oracle, InnoDB) an UPDATE overwrites
   the row in place and the old value goes to an undo/rollback segment.
   An aborted transaction must be actively undone.
   In PostgreSQL an UPDATE writes a NEW tuple and leaves the old one.
   An aborted transaction's tuples are simply never visible.

 ★ THE TRADE, ONE MORE TIME:
   no undo pass  → recovery is faster and simpler, rollback is O(1)
   the cost      → dead tuples accumulate ⇒ VACUUM (Topic 47)
   ⇒ this single design decision explains crash recovery, rollback
     cost, MVCC, and VACUUM — all four.
```

---

## How it works — step by step

### The trade, quantified

```
 RECOVERY TIME ≈ (WAL generated since the redo point) ÷ (replay rate)

 REPLAY RATE, measured on typical hardware:
   ~50–150 MB/s of WAL
   ⇒ ★ SINGLE-THREADED. Recovery does not parallelise.
     A 32-core machine recovers at the same speed as a 4-core one.
     This surprises people planning for fast failover.

 WORST-CASE WAL SINCE THE REDO POINT ≈ max_wal_size
   (because exceeding it forces a checkpoint)

 ┌──────────────────┬───────────────┬──────────────┬─────────────────┐
 │ checkpoint_      │ max_wal_size  │ WAL to       │ recovery time   │
 │ timeout          │               │ replay       │ @ 100 MB/s      │
 ├──────────────────┼───────────────┼──────────────┼─────────────────┤
 │ 5 min            │ 1 GB          │ ≤ 1 GB       │ ~10 s           │
 │ 15 min           │ 8 GB          │ ≤ 8 GB       │ ~80 s           │
 │ 30 min           │ 32 GB         │ ≤ 32 GB      │ ~5 min          │
 │ 60 min           │ 64 GB         │ ≤ 64 GB      │ ~11 min         │
 └──────────────────┴───────────────┴──────────────┴─────────────────┘

 ★ AND THE OPPOSING COLUMN (Topic 41):
 ┌──────────────────┬───────────────────┬────────────────────────────┐
 │ checkpoint_      │ full-page images  │ WAL volume                 │
 │ timeout          │ per hour          │ (relative)                 │
 ├──────────────────┼───────────────────┼────────────────────────────┤
 │ 5 min            │ 12 × per page     │ 4.7×                       │
 │ 30 min           │ 2 × per page      │ 1.0×  ★ baseline           │
 └──────────────────┴───────────────────┴────────────────────────────┘

 ⇒ ★★★ THE DECISION:
   You are trading RECOVERY TIME (your RTO) against WAL VOLUME
   (your replication bandwidth, archive storage, and steady-state I/O).
   ⇒ There is no setting that is good at both. Pick by which one is
     your actual constraint.
```

### `checkpoint_completion_target` — the setting that removes the spike

```
 completion_target = 0.5 (the old default)
 ┌──────────────────────────────────────────────────────────────────┐
 │ write I/O                                                        │
 │  ████████████                        ████████████                │
 │  ████████████                        ████████████                │
 │ ─────────────────────────────────────────────────────▶ time      │
 │  0        2.5min      5min          5min      7.5min             │
 │  ★ all dirty pages written in the first half, then idle.         │
 │    ⇒ a periodic I/O spike, and a periodic p99 spike.             │
 └──────────────────────────────────────────────────────────────────┘

 completion_target = 0.9 (the default since PG14)
 ┌──────────────────────────────────────────────────────────────────┐
 │ write I/O                                                        │
 │  ████████████████████████████████████████████                    │
 │ ─────────────────────────────────────────────────────▶ time      │
 │  0                                    4.5min   5min              │
 │  ★ spread over 90% of the interval. Smooth.                      │
 └──────────────────────────────────────────────────────────────────┘

 ⇒ ★ IF YOUR p99 SPIKES ON A REGULAR PERIOD MATCHING
   checkpoint_timeout, this is why.
```

### Restart points — recovery on a replica

```
 A REPLICA IS PERMANENTLY IN RECOVERY. It cannot create checkpoints
 (only the primary generates WAL), so it creates RESTART POINTS:

   when the replica replays a checkpoint record from the primary AND
   at least checkpoint_timeout has elapsed since its last restart
   point, it flushes its own dirty buffers and updates its pg_control.

 ⇒ ★ IF A REPLICA CRASHES, IT RECOVERS FROM ITS LAST RESTART POINT —
   which follows the PRIMARY's checkpoint frequency.
 ⇒ SO: raising checkpoint_timeout on the primary also lengthens
   recovery on every replica.

 AND THE CONFLICT PROBLEM (Topic 62):
   replay may need to remove a row version that a query on the replica
   is still reading.
     max_standby_streaming_delay = 30s (default)
       → replay WAITS up to 30 s, then CANCELS the query
     hot_standby_feedback = on
       → the replica tells the primary its oldest snapshot, so the
         primary doesn't vacuum those rows
       → ★ queries survive, but the PRIMARY bloats (Topic 06)
   ⇒ you must choose which side pays.
```

---

## Concept breakdown

```
CHECKPOINT
├── a WAL position at which all prior changes are on disk
├── REDO POINT recorded BEFORE the work starts  ★ this is what makes
│   it safe, and it is where recovery begins
├── writes dirty buffers spread over completion_target × timeout
├── triggered by: timeout · max_wal_size · explicit · shutdown · backup
└── ★ not a pause — the database serves traffic throughout

pg_control (8 KB)
├── cluster state (clean vs unclean shutdown)
├── ★ REDO location — where replay starts
└── next XID, wal_level, checksum version, system identifier

CRASH RECOVERY
├── ① read pg_control → redo location
├── ② REDO phase: replay forward, ★ skipping records where
│      page.pd_lsn >= record LSN  ⇒ IDEMPOTENT, restartable
├── ③ CRC failure = the end of valid WAL = where the crash happened
├── ④ full-page images repair torn pages
├── ⑤ ★ NO UNDO PHASE — uncommitted tuples are simply never visible
└── ⑥ checkpoint, open for connections

ARIES vs POSTGRESQL
├── analysis · redo · undo   →   (analysis) · redo · ★ no undo
└── because MVCC writes new tuples instead of overwriting
    ⇒ cheap rollback, cheap recovery, ★ paid for with VACUUM

★ THE CENTRAL TRADE
  recovery time ≈ WAL since the redo point ÷ replay rate (~100 MB/s,
  SINGLE-THREADED)
    longer checkpoint_timeout → less WAL (Topic 41) · slower recovery
    shorter                   → more WAL · faster recovery
  ⇒ you are choosing your RTO in advance. There is no free setting.

RESTART POINTS
└── a replica's equivalent of a checkpoint; follows the primary's
    frequency. Raising the primary's timeout slows every replica's
    recovery too.
```

---

## Diagrams

**Diagram 1 — big picture: what recovery replays**

```
  WAL stream ──────────────────────────────────────────────────▶

  ┌────────┬────────┬─────────┬────────┬────────┬────────┬──────┐
  │ …      │ CKPT   │ txn 91  │ txn 92 │ txn 93 │ txn 94 │ ✗    │
  │        │ record │ COMMIT  │ COMMIT │ COMMIT │ (no    │ CRASH│
  │        │        │         │        │        │ commit)│      │
  └────────┴───┬────┴─────────┴────────┴────────┴────────┴──────┘
               │                                          │
        REDO POINT ◀─────── replay this range ───────────▶│
               │                                          │
               ▼                                          ▼
   ┌──────────────────────────────────────────────────────────┐
   │ ① redo every record, skipping any whose change is already │
   │   on the page (page.pd_lsn >= record LSN)                 │
   │ ② stop at the first CRC failure — that is the crash point │
   │ ③ txn 94 has no COMMIT record ⇒ implicitly ABORTED        │
   │   ★ nothing to undo. Its tuples exist but pg_xact never   │
   │     says COMMITTED, so no snapshot can see them.          │
   └──────────────────────────────────────────────────────────┘
```

**Diagram 2 — data flow: why redo is idempotent**

```
   WAL record: "set tuple 12 on page 1204", LSN 0/1A2B3C8
                            │
                            ▼
              read page 1204 into the buffer pool
                            │
                  page.pd_lsn = ?
                            │
        ┌───────────────────┴────────────────────┐
        ▼                                        ▼
   0/1A2B3C8 or higher                     0/18F4200 (lower)
        │                                        │
   ★ ALREADY APPLIED                       APPLY THE CHANGE
     — the page was flushed                 set page.pd_lsn = 0/1A2B3C8
       after this change                            │
     SKIP                                           ▼
                                              continue

   ⇒ ★ RECOVERY CAN CRASH AND RESTART ANY NUMBER OF TIMES.
     Each run reaches exactly the same state. This is not an
     optimisation — it is a correctness requirement, because a
     crash during recovery is entirely possible.
```

**Diagram 3 — before/after: the checkpoint trade**

```
 checkpoint_timeout = 5 min
 ┌──────────────────────────────────────────────────────────────────┐
 │ RECOVERY:  ≤ 1 GB to replay  ⇒  ~10 seconds        ★ fast        │
 │ STEADY:    12 full-page images per page per hour                 │
 │            WAL volume 4.7× baseline                              │
 │            ⇒ 410 GB/hour shipped to every replica                │
 │            ⇒ p99 spike every 5 minutes                           │
 └──────────────────────────────────────────────────────────────────┘

 checkpoint_timeout = 30 min, max_wal_size = 32 GB, completion 0.9
 ┌──────────────────────────────────────────────────────────────────┐
 │ RECOVERY:  ≤ 32 GB to replay  ⇒  ~5 minutes        ⚠ slower      │
 │ STEADY:    2 full-page images per page per hour                  │
 │            WAL volume 1.0× baseline                              │
 │            ⇒ 88 GB/hour                           ★ 4.7× less    │
 │            ⇒ writes spread over 27 min — no spike                │
 └──────────────────────────────────────────────────────────────────┘

 ★ THE QUESTION IS NOT "WHICH IS BETTER" BUT:
   "is 5 minutes of recovery acceptable, given that we have a
    replica we can fail over to in 30 seconds?"
   ⇒ if yes — and it usually is — take the 4.7× WAL saving.
   ⇒ if you have NO replica, recovery time IS your RTO, and the
     answer flips.
```

---

## Example 1 — basic

**Step 1 — read `pg_control`.**

```bash
docker exec -it pg-lab pg_controldata /var/lib/postgresql/data | \
  grep -E 'cluster state|checkpoint location|REDO location|REDO WAL file|Time of latest'
```
```
Database cluster state:               in production
Latest checkpoint location:           0/1A2B3C8
Latest checkpoint's REDO location:    0/18F4200
Latest checkpoint's REDO WAL file:    000000010000000000000018
Time of latest checkpoint:            Tue 12 Aug 2026 09:14:22
```
**The two LSNs differ by ~28 MB** — that's the WAL written *during* the checkpoint. Replay starts at the REDO location.

**Step 2 — cause a real crash and watch recovery.**

```sql
CREATE TABLE crash_demo (id bigserial PRIMARY KEY, v text);
INSERT INTO crash_demo (v) SELECT repeat('x',200) FROM generate_series(1,500000);
CHECKPOINT;

-- now generate WAL that will need replaying
INSERT INTO crash_demo (v) SELECT repeat('y',200) FROM generate_series(1,500000);
SELECT count(*) FROM crash_demo;   -- 1000000
SELECT pg_current_wal_lsn();       -- 0/2C4A118
```
```bash
# an UNCLEAN shutdown — SIGKILL, not a graceful stop
docker kill -s SIGKILL pg-lab
docker start pg-lab
docker logs pg-lab --tail 20
```
```
LOG:  database system was interrupted; last known up at 2026-08-12 09:14:22
LOG:  database system was not properly shut down; automatic recovery in progress
LOG:  redo starts at 0/18F4200                      ★ the REDO location
LOG:  invalid record length at 0/2C4A118: wanted 24, got 0
                                          ★ the CRC/length check —
                                            this is where the crash hit
LOG:  redo done at 0/2C4A0E0, elapsed time: 1.84 s   ★ recovery time
LOG:  checkpoint starting: end-of-recovery immediate
LOG:  database system is ready to accept connections
```
```sql
SELECT count(*) FROM crash_demo;   -- 1000000   ★ nothing lost
```

**Step 3 — an uncommitted transaction is simply invisible.**

```sql
-- session 1
BEGIN;
INSERT INTO crash_demo (v) SELECT 'UNCOMMITTED' FROM generate_series(1,100000);
-- do NOT commit
```
```bash
docker kill -s SIGKILL pg-lab && docker start pg-lab
```
```sql
SELECT count(*) FROM crash_demo WHERE v='UNCOMMITTED';
```
```
 count
-------
     0        ★ invisible
```
```sql
-- but the tuples are physically there
SELECT pg_size_pretty(pg_relation_size('crash_demo'));   -- larger than before
VACUUM VERBOSE crash_demo;
-- INFO: … 100000 dead row versions removed
--   ★ NO UNDO PASS HAPPENED. VACUUM cleaned them up later.
```

**Step 4 — measure recovery time as a function of WAL.**

```sql
ALTER SYSTEM SET checkpoint_timeout = '30min';
ALTER SYSTEM SET max_wal_size = '8GB';
SELECT pg_reload_conf();
CHECKPOINT;

SELECT pg_current_wal_lsn() AS start \gset
-- generate ~4 GB of WAL
INSERT INTO crash_demo (v) SELECT repeat('z',200) FROM generate_series(1,8000000);
SELECT pg_size_pretty(pg_current_wal_lsn() - :'start'::pg_lsn) AS wal_to_replay;
```
```
 wal_to_replay
---------------
 4102 MB
```
```bash
docker kill -s SIGKILL pg-lab && docker start pg-lab
docker logs pg-lab 2>&1 | grep 'redo done'
```
```
LOG:  redo done at 0/12A4B008, elapsed time: 41.28 s
```
```
 ⇒ 4,102 MB ÷ 41.28 s = ★ 99.4 MB/s replay rate
 ⇒ NOW YOU CAN PREDICT: with max_wal_size = 32 GB, worst-case
   recovery ≈ 32,768 ÷ 99.4 ≈ 330 s ≈ 5.5 minutes.
 ★ MEASURE THIS ON YOUR OWN HARDWARE. It is the only way to know
   your real RTO, and almost nobody does it.
```

**Step 5 — recovery is single-threaded.**

```bash
# during the recovery above, in another terminal:
docker exec pg-lab top -bn1 | head -12
```
```
  PID USER      %CPU  COMMAND
  892 postgres  99.7  postgres: startup recovering 000000010000000000000098
  893 postgres   2.1  postgres: checkpointer
```
**One process at 100% of one core.** More cores do not help — which matters enormously when planning failover time.

**Step 6 — the checkpoint I/O spike, and how to remove it.**

```sql
ALTER SYSTEM SET checkpoint_timeout = '1min';
ALTER SYSTEM SET checkpoint_completion_target = 0.1;   -- ★ the spike setting
SELECT pg_reload_conf();
```
```bash
pgbench -c 50 -j 4 -T 300 -P 5 -N shop
```
```
progress: 5.0 s, 9204.1 tps, lat 5.4 ms
progress: 10.0 s, 9188.4 tps, lat 5.4 ms
progress: 60.0 s, 2104.8 tps, lat 23.8 ms     ★ checkpoint
progress: 65.0 s, 9201.2 tps, lat 5.4 ms
progress: 120.0 s, 2088.1 tps, lat 24.1 ms    ★ checkpoint
```
```sql
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
SELECT pg_reload_conf();
```
```
progress: 60.0 s, 8104.2 tps, lat 6.1 ms      ★ no spike
progress: 120.0 s, 8088.4 tps, lat 6.2 ms
```
**The periodic p99 spike disappears** — the same pages are written, spread over 90% of the interval instead of 10%.

**Step 7 — see checkpoint activity directly.**

```sql
SELECT checkpoints_timed, checkpoints_req,
       round(checkpoint_write_time/1000.0,1) AS write_seconds,
       round(checkpoint_sync_time/1000.0,1)  AS sync_seconds,
       buffers_checkpoint, buffers_backend
FROM pg_stat_bgwriter;
```
```
 checkpoints_timed | checkpoints_req | write_seconds | sync_seconds | buffers_checkpoint
-------------------+-----------------+---------------+--------------+--------------------
              1204 |              88 |        8412.4 |        204.1 |           41208841
```
```
 ★ HOW TO READ THIS:
   checkpoints_req high relative to timed → max_wal_size too small (T41)
   sync_seconds high                      → slow storage; the fsync at
                                            the end of the checkpoint hurts
   buffers_backend high                   → backends are flushing pages
                                            themselves; bgwriter too lazy (T07)
```

---

## Example 2 — production scenario

**The situation.** A payments platform. After a cloud host failure, the primary took **14 minutes** to come back. The DR runbook claims an RTO of 60 seconds. Leadership wants to know which number is real.

**Step 1 — find out what actually happened.**

```bash
grep -E 'redo starts|redo done|not properly shut down' /var/log/postgresql/postgresql.log
```
```
LOG:  database system was not properly shut down; automatic recovery in progress
LOG:  redo starts at 3/A2F41008
LOG:  redo done at 4/8814C220, elapsed time: 812.44 s        ★ 13.5 minutes
```
```sql
-- how much WAL was that?
SELECT pg_size_pretty('4/8814C220'::pg_lsn - '3/A2F41008'::pg_lsn) AS wal_replayed;
```
```
 wal_replayed
--------------
 78 GB
```
```
 ⇒ 78 GB ÷ 812 s = 96 MB/s replay rate.
 ⇒ ★ THE RTO OF 60 SECONDS WAS NEVER ACHIEVABLE. It was invented.
```

**Step 2 — why was there 78 GB to replay?**

```sql
SHOW max_wal_size;         -- 128GB     ★
SHOW checkpoint_timeout;   -- 60min     ★
SELECT checkpoints_timed, checkpoints_req FROM pg_stat_bgwriter;
```
```
 checkpoints_timed | checkpoints_req
-------------------+-----------------
              1042 |              14
```
```
 ★ Someone had followed Topic 41's advice too far. They raised
   max_wal_size to 128 GB and checkpoint_timeout to 60 minutes to cut
   WAL volume — which worked. WAL dropped from 410 GB/hour to 71 GB/hour.

 ⇒ AND NOBODY COMPUTED THE RECOVERY CONSEQUENCE:
     worst case = 128 GB ÷ 96 MB/s = 22 minutes.
   They got 13.5 because the crash happened mid-interval.

 ★ THIS IS THE TENSION BETWEEN TOPICS 41 AND 42, IN PRODUCTION.
   Each setting was optimised in isolation. Nobody owned the trade.
```

**Step 3 — decide the actual RTO requirement.**

```
 THE QUESTIONS THAT SETTLE IT:

 ① IS CRASH RECOVERY EVEN ON THE CRITICAL PATH?
    → There are two streaming replicas. Failover with Patroni takes
      ~25 s. Crash recovery of the failed primary happens AFTER the
      failover, while it rejoins as a replica.
    ⇒ ★ RECOVERY TIME IS NOT THE RTO. Failover time is.

 ② SO WHY DID THE OUTAGE LAST 14 MINUTES?
    → Patroni was configured with `maximum_lag_on_failover: 1048576`
      (1 MB). Both replicas were 40 MB behind at the moment of failure
      (because of the very WAL volume they'd been trying to reduce).
      ⇒ ★ PATRONI REFUSED TO PROMOTE EITHER ONE, and waited for the
        old primary to recover instead.

 ⇒ ★★★ THE REAL FINDING: the 14-minute outage was NOT a recovery
   problem. It was a FAILOVER POLICY problem, and recovery time was
   only the consequence of that policy's refusal to promote.
```

**Step 4 — the fix has three parts, and only one is about checkpoints.**

```sql
-- ① BOUND RECOVERY TIME to something defensible as a FALLBACK
--    (not as the primary RTO — that is failover's job)
ALTER SYSTEM SET max_wal_size = '32GB';        -- was 128GB
ALTER SYSTEM SET checkpoint_timeout = '20min'; -- was 60min
ALTER SYSTEM SET checkpoint_completion_target = 0.9;
ALTER SYSTEM SET wal_compression = 'lz4';      -- ★ recovers the WAL
                                               --   saving without the
                                               --   recovery cost
SELECT pg_reload_conf();
```
```
 ⇒ worst-case recovery: 128 GB → 32 GB ⇒ 22 min → 5.5 min
 ⇒ WAL volume: 71 → 96 GB/hour without compression,
               but 71 → 58 GB/hour WITH lz4.
 ★ wal_compression let them SHORTEN recovery AND reduce WAL —
   the one lever that doesn't trade.
```

```yaml
# ② FIX THE FAILOVER POLICY — the actual cause
patroni:
  maximum_lag_on_failover: 33554432   # 32 MB, was 1 MB
  # ⇒ accept up to ~32 MB (≈0.3 s) of data loss to promote in 25 s
  #   rather than waiting 14 minutes for the old primary.
  # ★ THIS IS AN RPO-vs-RTO DECISION AND IT MUST BE MADE EXPLICITLY.
```

```sql
-- ③ AND ELIMINATE THE DATA LOSS THAT ② ACCEPTS
ALTER SYSTEM SET synchronous_standby_names = 'ANY 1 (r1, r2)';
ALTER SYSTEM SET synchronous_commit = 'remote_write';
SELECT pg_reload_conf();
-- ⇒ at least one replica is guaranteed to have every committed
--   transaction ⇒ RPO = 0 AND fast promotion.
--   Cost: commit latency 0.83 ms → 2.10 ms (Topic 41).
```

**Step 5 — measure the real numbers, and write them down.**

```bash
# a scheduled DR test, run monthly — not a document, an EXERCISE
#!/usr/bin/env bash
set -e
psql -c "CHECKPOINT;"
LSN_START=$(psql -tAc "SELECT pg_current_wal_lsn()")
pgbench -c 50 -j 4 -T 600 -N shop >/dev/null    # 10 min of realistic load
LSN_END=$(psql -tAc "SELECT pg_current_wal_lsn()")
WAL=$(psql -tAc "SELECT pg_size_pretty('$LSN_END'::pg_lsn - '$LSN_START'::pg_lsn)")

docker kill -s SIGKILL pg-primary
START=$(date +%s.%N)
docker start pg-primary
until pg_isready -q; do sleep 0.1; done
END=$(date +%s.%N)

echo "WAL generated: $WAL"
echo "Recovery time: $(echo "$END - $START" | bc) s"
grep 'redo done' /var/log/postgresql/postgresql.log | tail -1
```
```
 RESULTS, RECORDED IN THE RUNBOOK:
   WAL generated in 10 min of load : 9.8 GB
   Worst-case WAL (max_wal_size)   : 32 GB
   Measured replay rate            : 96 MB/s
   ⇒ WORST-CASE RECOVERY           : 5 min 40 s
   ⇒ MEASURED FAILOVER (Patroni)   : 24 s
   ⇒ ★ RTO = 24 s (failover), with a 5m40s fallback if BOTH
     replicas are unavailable.
   ⇒ ★ RPO = 0 (synchronous_commit = remote_write, ANY 1)
```

**Step 6 — the monitoring that predicts recovery time continuously.**

```sql
CREATE OR REPLACE VIEW recovery_estimate AS
WITH ckpt AS (
  SELECT pg_current_wal_lsn() AS now_lsn,
         (SELECT redo_lsn FROM pg_control_checkpoint()) AS redo_lsn
)
SELECT
  pg_size_pretty(now_lsn - redo_lsn)                       AS wal_since_redo_point,
  round(((now_lsn - redo_lsn) / 1024.0 / 1024.0 / 96)::numeric, 1)
                                                            AS est_recovery_seconds,
  (SELECT setting FROM pg_settings WHERE name='max_wal_size') AS max_wal_size,
  (SELECT round(100.0*checkpoints_req/nullif(checkpoints_timed+checkpoints_req,0),1)
     FROM pg_stat_bgwriter)                                 AS pct_forced_checkpoints
FROM ckpt;
```
```
 wal_since_redo_point | est_recovery_seconds | max_wal_size | pct_forced
----------------------+----------------------+--------------+------------
 8412 MB              |                 87.6 | 32GB         |        1.3
```
```
 ★ ALERT IF est_recovery_seconds EXCEEDS YOUR STATED FALLBACK RTO.
   This is the number your DR document should quote — measured, and
   updated continuously, not invented once.
```

**Step 7 — results.**

| | Before | After |
|---|---|---|
| Stated RTO | 60 s (invented) | **24 s (measured)** |
| Actual outage | 14 min | **24 s** |
| Worst-case recovery | 22 min | **5 min 40 s** (fallback only) |
| Replay rate | unknown | **96 MB/s, measured** |
| WAL per hour | 71 GB | **58 GB** (lz4 compensated) |
| RPO | up to 1 MB | **0** |
| DR test | never run | **monthly, automated** |

**The checkpoint settings were the smallest part of the fix.** The outage was a failover-policy decision that nobody had made explicitly.

---

## Common mistakes

**1. Quoting an RTO nobody has measured.**
- *Symptom:* a DR document says 60 seconds; the real outage is 14 minutes.
- *Fix:* measure the replay rate on your hardware (Example 1, step 4) and compute worst case from `max_wal_size`. Then run a real DR test on a schedule.

**2. Optimising `max_wal_size` for WAL volume without computing recovery.**
- *Symptom:* WAL drops 4×, and the next crash takes 22 minutes.
- *Engine-level why:* worst-case replay is bounded by `max_wal_size`, and replay runs at ~100 MB/s single-threaded.
- *Fix:* decide which constraint is real. Use `wal_compression` — the one lever that improves both.

**3. Assuming recovery parallelises.**
- *Symptom:* a plan to "recover faster by using a bigger instance."
- *Engine-level why:* the startup process replays WAL single-threaded.
- *Fix:* faster storage and less WAL to replay. More cores do nothing.

**4. Blaming p99 spikes on the application.**
- *Symptom:* latency spikes on a regular period matching `checkpoint_timeout`.
- *Fix:* `checkpoint_completion_target = 0.9`, and check `buffers_backend` (Topic 07).

**5. Running `CHECKPOINT` manually as a routine.**
- *Symptom:* WAL volume balloons.
- *Engine-level why:* every checkpoint resets the full-page-image state, so the next write to each page is 8 KB (Topic 41).
- *Fix:* it's a diagnostic and a pre-backup step, not a maintenance task.

**6. Forgetting that replicas inherit the checkpoint setting.**
- *Symptom:* a replica takes as long to recover as the primary.
- *Engine-level why:* restart points follow the primary's checkpoint frequency.
- *Fix:* account for it when sizing. And remember a replica is *always* in recovery — Topic 62's conflicts apply.

**7. Treating recovery time as the RTO when a replica exists.**
- *Symptom:* the wrong thing gets optimised.
- *Fix:* with replicas, RTO is **failover** time. Recovery time is the fallback for when failover is impossible — and it needs a *separate*, stated budget.

---

## Hands-on proof

**PROVE IT #1 — read `pg_control`.** (Example 1, step 1.)
**PROVE IT #2 — a real crash and recovery.** (Example 1, step 2.)
**PROVE IT #3 — no undo pass.** (Example 1, step 3.)
**PROVE IT #4 — measure your replay rate.** (Example 1, step 4.)
**PROVE IT #5 — recovery is single-threaded.** (Example 1, step 5.)
**PROVE IT #6 — the checkpoint spike, and removing it.** (Example 1, step 6.)

**PROVE IT #7 — estimate recovery time right now.**
```sql
SELECT
  pg_size_pretty(pg_current_wal_lsn() - (SELECT redo_lsn FROM pg_control_checkpoint()))
    AS wal_since_redo_point,
  (SELECT setting FROM pg_settings WHERE name='max_wal_size') AS worst_case_bound,
  (SELECT checkpoint_time FROM pg_control_checkpoint()) AS last_checkpoint;
-- ⇒ divide by your MEASURED replay rate for the estimate.
```

**PROVE IT #8 — redo idempotency, observed.**
```bash
# kill the server DURING recovery, then restart it again
docker kill -s SIGKILL pg-lab && docker start pg-lab
sleep 2 && docker kill -s SIGKILL pg-lab     # kill mid-recovery
docker start pg-lab
docker logs pg-lab 2>&1 | grep -E 'redo starts|redo done'
```
```
LOG:  redo starts at 0/18F4200
LOG:  redo done at 0/2C4A0E0, elapsed time: 1.91 s
-- ★ identical redo range, identical result. Interrupting recovery is safe.
```

---

## The design decision framework

```
★★★ THE CENTRAL TRADE
    recovery time ≈ WAL since the redo point ÷ replay rate
    worst case    ≈ max_wal_size ÷ replay rate
    replay rate   ≈ 50–150 MB/s, ★ SINGLE-THREADED — MEASURE YOURS

    longer checkpoint_timeout / larger max_wal_size
      ✓ far less WAL (Topic 41) · smoother I/O
      ✗ slower crash recovery
    ⇒ there is no setting good at both. EXCEPT:

★ THE ONE LEVER THAT DOESN'T TRADE:
    wal_compression = lz4
    ⇒ 40–60% less WAL AND less to replay. Take it.

STEP 1 — IS RECOVERY TIME EVEN YOUR RTO?
  DO YOU HAVE A STREAMING REPLICA AND AUTOMATED FAILOVER?
    YES → ★ RTO = FAILOVER TIME (~20–40 s). Recovery is the FALLBACK
          for when no replica is promotable. Give it its own, looser
          budget — 5–10 minutes is usually fine.
          ⇒ then optimise checkpoints for WAL VOLUME.
    NO  → ★ RECOVERY TIME *IS* YOUR RTO. Optimise for it, and accept
          the WAL cost. And get a replica.

STEP 2 — MEASURE, DON'T GUESS
  ① measure the replay rate (Example 1, step 4)
  ② worst-case recovery = max_wal_size ÷ that rate
  ③ ★ RUN A REAL DR TEST ON A SCHEDULE. A number in a document that
    nobody has produced by killing a server is fiction.

STEP 3 — THE SETTINGS
  checkpoint_timeout           15–30 min (with replicas)
                               5–10 min  (without)
  max_wal_size                 WAL rate × timeout × 2, capped by your
                               recovery budget
  checkpoint_completion_target 0.9    ★ removes the periodic p99 spike
  wal_compression              lz4    ★ always
  ⇒ then VERIFY: checkpoints_req < 20% of total (Topic 41)

STEP 4 — IF FAILOVER IS THE RTO, TUNE FAILOVER
  maximum_lag_on_failover — ★ an RPO-vs-RTO decision. Too strict and
    the cluster refuses to promote and waits for recovery, which is
    the worst of both.
  synchronous_commit = remote_write with ANY 1 standby
    ⇒ RPO = 0 AND fast promotion. Costs ~1.3 ms per commit.

THE SIGNAL TO LOOK FOR:
      SELECT pg_size_pretty(pg_current_wal_lsn() -
             (SELECT redo_lsn FROM pg_control_checkpoint())) AS wal_since_redo;
  ÷ your measured replay rate = your recovery time RIGHT NOW.
  • exceeds your stated fallback RTO → lower max_wal_size
  • and if you cannot state a measured replay rate, that is the
    first thing to fix.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Read `pg_controldata` and explain the difference between "Latest checkpoint location" and "Latest checkpoint's REDO location," and why replay starts at the second. Then cause an unclean shutdown with an uncommitted transaction in flight, and show from the logs and the data that (a) committed work survived, (b) uncommitted work is invisible, and (c) no undo phase ran.

### Exercise 2 — medium (apply it)
Measure your hardware's replay rate: `CHECKPOINT`, record the LSN, generate a known volume of WAL, `SIGKILL`, restart, and read `redo done … elapsed time` from the log. Then:
(a) compute worst-case recovery for `max_wal_size` values of 1 GB, 8 GB and 32 GB,
(b) demonstrate the checkpoint I/O spike with `completion_target = 0.1` and its removal at 0.9,
(c) show that recovery uses one CPU,
(d) state the `checkpoint_timeout` and `max_wal_size` you would choose, and the recovery budget that justifies them.

### Exercise 3 — hard (production simulation)
A payments platform suffers a 14-minute outage after a host failure. The DR document claims a 60-second RTO. Investigation shows: `redo done … elapsed time: 812 s` replaying 78 GB, `max_wal_size = 128GB`, `checkpoint_timeout = 60min`, two streaming replicas, and Patroni with `maximum_lag_on_failover: 1MB`.

(a) Compute the measured replay rate and the worst-case recovery time for the current settings.
(b) Explain why the WAL settings were raised, and what was not computed when they were.
(c) **The 14-minute outage was not primarily a recovery problem.** Identify the real cause and explain the mechanism.
(d) Explain why the replicas were 40 MB behind, and how that connects to the WAL settings.
(e) Give the three-part fix, and say which part addresses the actual cause.
(f) `wal_compression = lz4` is described as "the one lever that doesn't trade." Explain why, in terms of both WAL volume and replay.
(g) Design the monthly automated DR test, including what it measures and what it writes back into the runbook.
(h) Write the monitoring view that continuously estimates recovery time, and state the alert threshold and why.

---

## Mental model checkpoint

1. Why does recovery start at the REDO location rather than the checkpoint record's location?
2. What makes redo idempotent, and why is that a correctness requirement rather than an optimisation?
3. Why does PostgreSQL have no undo phase? What does it pay for that?
4. How does recovery know where the WAL ends?
5. Give the formula for worst-case recovery time. What is the replay rate, and does it parallelise?
6. What does `checkpoint_completion_target` do, and what symptom does the wrong value produce?
7. If you have streaming replicas and automated failover, what is your RTO — and what is recovery time then?

---

## Quick reference card

| Concept | |
|---|---|
| Checkpoint | a WAL position where all prior changes are on disk |
| REDO location | where replay starts — recorded *before* the checkpoint's work |
| `pg_control` | 8 KB; cluster state + redo location |
| Recovery | redo-only, idempotent via `pd_lsn`, **no undo phase** |
| End of WAL | the first CRC/length failure |
| Replay rate | **~50–150 MB/s, single-threaded** — measure yours |

**Worst-case recovery ≈ `max_wal_size` ÷ replay rate**

| `checkpoint_timeout` | `max_wal_size` | Recovery @ 100 MB/s | WAL volume |
|---|---|---|---|
| 5 min | 1 GB | ~10 s | 4.7× |
| 15 min | 8 GB | ~80 s | 1.6× |
| 30 min | 32 GB | ~5.5 min | 1.0× |

**Settings**

| Setting | Guidance |
|---|---|
| `checkpoint_timeout` | 15–30 min with replicas; 5–10 min without |
| `max_wal_size` | WAL rate × timeout × 2, capped by your recovery budget |
| `checkpoint_completion_target` | **0.9** — removes the periodic p99 spike |
| `wal_compression` | **`lz4`** — the one lever that improves both sides |

**With replicas: RTO = failover time. Recovery time is the fallback, with its own budget.**

**Measure it:** `redo done … elapsed time` in the log, ÷ the LSN delta.

---

## When would I use this at work?

1. **Writing or reviewing a DR plan.** Measuring the replay rate and computing worst-case recovery from `max_wal_size` replaces an invented RTO with one you can defend — and usually reveals that failover, not recovery, is the number that matters.

2. **After someone tunes `max_wal_size` for WAL volume.** Asking "and what did that do to recovery time?" catches a trade that was made in one direction only, which is how a 4× WAL saving becomes a 22-minute outage.

3. **p99 spikes on a regular period.** Matching the period to `checkpoint_timeout` identifies the checkpointer in seconds, and `completion_target = 0.9` fixes it without touching anything else.

---

## Connected topics

**Understand before this:** 41 (the WAL format, full-page images, `max_wal_size`), 07 (dirty buffers — what a checkpoint flushes), 39–40 (commit and durability).

**This unlocks:**
- **62** — replication: a replica is permanently in recovery; restart points and conflicts
- **63** — high availability: why failover time, not recovery time, is usually the RTO
- **64** — PITR: recovery stopped at a chosen point, on a new timeline
- **47** — VACUUM: what cleans up the tuples no undo pass removed
