# 64 — Backup, Recovery and Point-in-Time Recovery
## Phase: Reliability & Operations

---

## ELI5 — The Simple Analogy

Two different insurances, and people buy one and think they bought both.

**A mirror in the next room** shows exactly what your desk looks like right now. If your desk catches fire, you still have the mirror's contents. ★ **But if you spill coffee on a document, the mirror shows the spill — instantly.** That's replication (Topics 58, 62, 63).

**A photograph taken this morning, plus a written log of every change since** is different. Spill the coffee at 3 p.m. and you can reconstruct the document as it was at 2:59 p.m. — because you have the photo *and* the list of edits, and you can replay the edits up to any moment you choose.

★ **That is the whole idea:** a **base backup** (the photograph) plus **archived WAL** (the log of edits) lets you restore to *any point in time* between the photo and now.

Three things follow, and they are what this topic is about:

1. ★ **The photo alone is not enough.** Without the edit log you can only go back to this morning — losing everything since.
2. ★ **A photo you have never developed is not a backup.** It is a hope. The only proof is a restore.
3. ★ **The clock matters more than the photo.** "How long does it take to develop?" is usually the number nobody has, and it is the one that decides whether you survive.

---

## Where this fits in the big picture

```
   41 WAL — the edit log · 42 recovery — replaying it
   62/63 replication & HA — ★ the mirror, NOT the photograph
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 64 BACKUP & PITR ← YOU ARE HERE              │
        │ ★ the ONLY protection against a bad DELETE,  │
        │   a bad migration, or corruption             │
        └────────────────────┬─────────────────────────┘
                             ▼
              65 pooling · 67 investigation · 69 security
```

★ **Topic 63 protects against a machine dying. This topic protects against a human, a bug, or a bit flip** — and those replicate to every standby in milliseconds. The two are not substitutes and conflating them is the most expensive mistake in operations.

---

## What is this?

```
 ★ THREE THINGS, AND YOU NEED ALL THREE:

 ① BASE BACKUP     a physically consistent copy of the data
                   directory, taken while the database is running
 ② ★ WAL ARCHIVE   every WAL segment, continuously shipped
                   somewhere durable
 ③ ★ A TESTED RESTORE PROCEDURE
                   ⇒ ★ WITHOUT ③, ① AND ② ARE A HOPE, NOT A BACKUP

 PITR = ① + ② + "stop replaying at time T"
```

**The kinds, and what each actually protects against:**

| Method | Protects against | Doesn't protect against |
|---|---|---|
| **Replica** | ★ machine/AZ failure | ★ bad `DELETE`, migration, corruption |
| **`pg_dump`** | anything (logical) | ★ **slow restore**, no PITR, no consistency across DBs |
| **`pg_basebackup` alone** | machine failure | ★ everything since the backup |
| ★ **base backup + WAL archive** | ★ **everything, to the second** | a bug in the backup process itself |
| ★ **snapshots (EBS/LVM)** | fast restore | ★ needs care to be crash-consistent |

★ **The one-line summary:** *the only backup that matters is the one you have restored, timed, and verified — and the restore time is usually the binding constraint, not the backup.*

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE THE FAILURES ARE PREDICTABLE AND THE SAME EVERY TIME.

 ① ★ "WE HAVE REPLICAS" IS NOT A BACKUP.
    UPDATE accounts SET balance = 0;   -- ★ forgot the WHERE
    ⇒ replicated to all three standbys in 3 ms.
    ⇒ ★ there is nothing to fail over TO.

 ② ★ THE BACKUP EXISTS AND CANNOT BE RESTORED.
    • the WAL archive has a gap nobody noticed
    • the base backup is missing a tablespace
    • the restore needs a config file that lives only on the dead host
    • ★ the encryption key is in the database you are restoring
    ⇒ ★ EVERY ONE OF THESE IS FOUND ONLY BY DOING A RESTORE.

 ③ ★ THE RESTORE TIME IS 9 HOURS AND THE RTO IS 1.
    a 4 TB restore from object storage at 200 MB/s is ★ 5.5 hours
    before you replay a single WAL segment.
    ⇒ ★ NOBODY MEASURES THIS UNTIL THE INCIDENT.

 ⇒ ★ AND THE ONE THAT ENDS COMPANIES: discovering during the
   incident that the last successful backup was 40 days ago
   because the cron job's exit code was never checked.
```

---

## The physical reality

### Why a base backup can be taken from a running database

```
 ★ COPYING A DATA DIRECTORY WHILE POSTGRESQL IS WRITING TO IT
   PRODUCES A TORN, INCONSISTENT MESS. Pages are half-written;
   different files are from different instants.

 ⇒ SO HOW IS pg_basebackup SAFE?

 ① it calls pg_backup_start() ⇒ ★ FORCES A CHECKPOINT and records
    the checkpoint's REDO LSN as the backup's start point
 ② ★ full_page_writes GUARANTEES that any page modified after that
    checkpoint has a FULL-PAGE IMAGE in the WAL (Topic 41)
 ③ the file copy proceeds — ★ torn pages are expected and fine
 ④ pg_backup_stop() records the end LSN
 ⑤ ★ RESTORE REPLAYS WAL FROM THE START LSN, and every torn page
    is OVERWRITTEN by its full-page image before anything reads it.

 ⇒ ★ THE BACKUP IS NOT CONSISTENT ON DISK. It becomes consistent
   only after replaying WAL from start_lsn to end_lsn.
 ⇒ ★ THEREFORE: A BASE BACKUP WITHOUT THE WAL THAT SPANS IT IS
   ★ UNUSABLE — not "slightly stale". Completely unusable.
   ⇒ this is why `pg_basebackup -Xs` (stream WAL alongside) exists,
     and why forgetting it is a classic, silent, total failure.
```

### The WAL archive — where the gaps come from

```sql
 archive_mode = on
 archive_command = 'wal-g wal-push %p'
 -- ★ or: 'test ! -f /arch/%f && cp %p /arch/%f'

 ★ THE CONTRACT: PostgreSQL will not recycle a WAL segment until
   archive_command EXITS 0 for it.
 ⇒ ★ CONSEQUENCE ①: a failing archive_command means WAL
   ACCUMULATES IN pg_wal FOREVER ⇒ ★ disk-full PANIC (Topic 41).
 ⇒ ★ CONSEQUENCE ②: an archive_command that returns 0 WITHOUT
   ACTUALLY ARCHIVING creates a ★ SILENT GAP. The segment is
   recycled and gone.
   ⇒ ★ THE CLASSIC BUG: `cp %p /arch/%f` where /arch is a mount
     that silently became a local directory when the NFS mount
     failed. Exit code 0. Data on a disk that is about to die.

 ★ MONITOR IT — this is not optional:
   SELECT archived_count, last_archived_wal, last_archived_time,
          ★ failed_count, last_failed_wal, last_failed_time
     FROM pg_stat_archiver;
   ⇒ ★ ALERT: failed_count increasing
   ⇒ ★ ALERT: now() - last_archived_time > 5 minutes
   ⇒ ★ AND: verify the ARCHIVE ITSELF has no gaps, by listing it.

 ★ archive_timeout = '60s'
   forces a segment switch even when idle, so an idle database
   still bounds its RPO to 60 seconds.
   ⇒ ★ costs: a 16 MB segment per minute even with no writes.
```

### PITR — what "recover to a point in time" actually does

```
 ★ RESTORE PROCEDURE:
 ① restore the base backup into an empty data directory
 ② provide restore_command so PostgreSQL can fetch archived WAL
 ③ set a recovery target
 ④ create recovery.signal
 ⑤ start ⇒ ★ it replays WAL from the backup's start LSN forward,
    stopping at the target.

 postgresql.conf:
   restore_command = 'wal-g wal-fetch %f %p'
   recovery_target_time = '2026-08-24 14:22:00+05:30'
   recovery_target_action = 'pause'      -- ★ pause, DON'T promote
   recovery_target_inclusive = false     -- stop BEFORE the target

 ★ THE FOUR TARGET TYPES:
   recovery_target_time  — a timestamp
                           ⇒ ★ resolution is COMMIT timestamps;
                             you cannot land between two commits
                             in the same millisecond
   ★ recovery_target_xid — an exact transaction id
                           ⇒ ★ the precise one: "everything before
                             the bad transaction"
   recovery_target_lsn   — an exact WAL position
   recovery_target_name  — a label created by pg_create_restore_point()
                           ⇒ ★ create one before every migration

 ★ recovery_target_action = 'pause' IS THE IMPORTANT DEFAULT TO SET:
   the database comes up READ-ONLY at the target. You inspect it.
   ⇒ wrong target? ★ change it and restart — you have not committed.
   ⇒ right target? SELECT pg_wal_replay_resume();  ⇒ it promotes.
   ⇒ ★ WITH 'promote' (the default), a wrong target means starting
     the entire multi-hour restore again.
```

### The number nobody has: restore time

```
 ★ RESTORE TIME = DOWNLOAD + DECOMPRESS + WAL REPLAY

 ① DOWNLOAD
    4 TB from S3 at 200 MB/s (a single stream) ⇒ ★ 5.5 hours
    4 TB at 1.6 GB/s (★ 16 parallel streams)   ⇒ ★ 42 minutes
    ⇒ ★ PARALLELISM IS THE BIGGEST LEVER AND IS OFF BY DEFAULT.
      wal-g:     WALG_DOWNLOAD_CONCURRENCY=16
      pgBackRest: --process-max=16

 ② DECOMPRESS
    lz4 ~500 MB/s/core · zstd ~300 · gzip ★ ~60
    ⇒ ★ gzip on 4 TB single-threaded = 18 hours of CPU.
      Use lz4 or zstd with parallelism.

 ③ ★ WAL REPLAY — the part people forget
    replay is ★ SINGLE-THREADED at ~100 MB/s (Topics 42, 58).
    ⇒ restoring to a point 12 hours after the base backup, on a
      cluster generating 20 GB/hour, means replaying 240 GB
      ⇒ ★ 40 MINUTES OF REPLAY, on top of the download.
    ⇒ ★ THIS IS WHY BACKUP FREQUENCY IS AN RTO DECISION, NOT A
      STORAGE DECISION. Daily backups mean up to 24 hours of WAL
      to replay.

 ★ MEASURED, A REAL 4 TB CLUSTER:
   naive:    single-stream download + gzip + 24 h of WAL
             ⇒ ★ 9 h 40 m
   tuned:    16 streams + lz4 + 6-hourly backups
             ⇒ ★ 1 h 12 m
   ⇒ ★ 8× — and none of it required more storage.
```

### `pg_dump` vs physical backup — they solve different problems

```
 ★ pg_dump IS A LOGICAL BACKUP: SQL statements, not bytes.

 ✓ WHAT IT IS GOOD FOR:
   • ★ moving between major versions or platforms
   • ★ restoring a SINGLE TABLE (a physical backup cannot)
   • ★ a human-readable, version-independent archive
   • small databases

 ✗ WHY IT IS NOT YOUR PRIMARY BACKUP:
   • ★ NO PITR. You get the instant the dump started, nothing else.
   • ★ RESTORE IS SLOW: it re-executes every INSERT and REBUILDS
     EVERY INDEX.
     MEASURED, 400 GB: dump 2 h 10 m · ★ restore 14 h 20 m
     (the same data physically: ★ 22 minutes)
   • ★ it holds a snapshot for its whole duration
     ⇒ ★ pins xmin ⇒ VACUUM blocked cluster-wide (Topic 47)
     ⇒ a 2-hour dump is a 2-hour vacuum outage
   • ★ it does NOT back up: roles, tablespaces, or other databases
     (that is pg_dumpall --globals-only, and it is routinely
     forgotten until a restore fails on a missing role)

 ⇒ ★ THE RIGHT ANSWER FOR MOST SYSTEMS:
   physical (base + WAL) as the PRIMARY, for PITR and speed
   + a periodic pg_dump for single-table restores and portability
   + ★ pg_dumpall --globals-only, always
```

### Corruption — the failure that makes backups worthless

```
 ★ IF A PAGE IS CORRUPTED AND THE CORRUPTION IS BACKED UP, YOUR
   BACKUPS CONTAIN THE CORRUPTION TOO.

 ⇒ ★ data_checksums = on
   ⇒ every page read verifies a checksum
   ⇒ corruption is detected AT READ TIME rather than silently
     returning wrong answers
   ⇒ ★ costs ~1–2% CPU. Enable it at initdb time:
     initdb --data-checksums
     (PG 12+: pg_checksums --enable, but the cluster must be down)

   SELECT datname, checksum_failures, checksum_last_failure
     FROM pg_stat_database WHERE checksum_failures > 0;
   ⇒ ★ ANY non-zero value is a page-level alert.

 ★ AND VERIFY THE BACKUP ITSELF:
   pg_verifybackup /path/to/backup     -- ★ checks the manifest
   pgbackrest check                     -- ★ verifies archive + backup
   ⇒ ★ these detect a truncated or corrupted backup BEFORE you
     need it.

 ★ RETENTION MUST EXCEED YOUR DETECTION TIME:
   if a corruption or a bad migration takes 3 weeks to notice, a
   7-day retention means ★ every backup you hold contains it.
   ⇒ ★ 30–90 days for the primary chain, plus monthly archives.
```

---

## How it works — step by step

### A production backup configuration

```ini
# postgresql.conf
wal_level = replica                 # ★ or 'logical' if you also do CDC
archive_mode = on
archive_command = 'wal-g wal-push %p'
archive_timeout = '60s'             # ★ bounds RPO on an idle database
max_wal_size = '16GB'
data_checksums = on                 # ★ set at initdb
full_page_writes = on               # ★ NEVER turn this off —
                                    #   base backups depend on it
wal_compression = 'lz4'             # ★ smaller archive, faster replay
```

```bash
# ★ wal-g configuration — the parallelism settings that decide RTO
export WALG_S3_PREFIX="s3://backups/pg-prod"
export WALG_COMPRESSION_METHOD=lz4        # ★ not gzip
export WALG_UPLOAD_CONCURRENCY=16
export WALG_DOWNLOAD_CONCURRENCY=16       # ★ THE RESTORE LEVER
export WALG_DELTA_MAX_STEPS=6             # ★ incremental backups
export WALG_LIBSODIUM_KEY_PATH=/etc/wal-g/key
# ★ THE KEY MUST NOT LIVE ONLY IN THE DATABASE YOU ARE RESTORING.
```

```bash
# ★ backup schedule — frequency is an RTO decision
# full backup weekly, delta every 6 hours
0 2 * * 0  wal-g backup-push $PGDATA --full
0 2,8,14,20 * * 1-6  wal-g backup-push $PGDATA

# ★ AND THE PART EVERYONE FORGETS:
0 3 * * *  pg_dumpall --globals-only | gzip > /backups/globals-$(date +\%F).sql.gz
```

```bash
#!/usr/bin/env bash
# ★ the wrapper that makes a cron backup trustworthy
set -euo pipefail
START=$(date +%s)
if wal-g backup-push "$PGDATA"; then
  DURATION=$(( $(date +%s) - START ))
  curl -fsS "https://hc-ping.com/$HC_UUID" \
       -d "ok duration=${DURATION}s"           # ★ a DEAD MAN'S SWITCH
  echo "backup_duration_seconds $DURATION" > /var/lib/node_exporter/backup.prom
else
  curl -fsS "https://hc-ping.com/$HC_UUID/fail" -d "$(tail -20 /var/log/wal-g.log)"
  exit 1
fi
# ★ THE DEAD MAN'S SWITCH IS THE KEY IDEA: it alerts when the
#   backup DOESN'T RUN, not just when it fails. A cron job that
#   was silently removed produces no failure — only silence.
```

### Monitoring — the five things that must be alerted

```sql
-- ★ ① is WAL archiving working?
SELECT archived_count, failed_count,
       last_archived_wal, last_archived_time,
       last_failed_wal, last_failed_time,
       extract(epoch from now() - last_archived_time)::int AS seconds_since
  FROM pg_stat_archiver;
-- ★ ALERT: failed_count increasing · seconds_since > 300

-- ★ ② is WAL piling up because archiving is stuck?
SELECT count(*) AS unarchived_segments
  FROM pg_ls_dir('pg_wal/archive_status') f WHERE f LIKE '%.ready';
-- ★ ALERT: > 100

-- ★ ③ page corruption
SELECT datname, checksum_failures, checksum_last_failure
  FROM pg_stat_database WHERE checksum_failures > 0;
-- ★ ALERT: any row

-- ★ ④ backup age — from the backup tool, not the database
--    wal-g backup-list | tail -1
-- ★ ALERT: newest backup older than 2× the interval

-- ★ ⑤ ★ THE ONE THAT ACTUALLY MATTERS: when did a restore last
--    succeed, and how long did it take?
--    ⇒ from the automated restore test (below)
-- ★ ALERT: no successful restore test in 7 days
```

### The automated restore test — the only thing that makes it real

```bash
#!/usr/bin/env bash
# ★ RUNS NIGHTLY. THIS IS THE BACKUP SYSTEM. Everything else is
#   just copying files.
set -euo pipefail

RESTORE_DIR=/var/lib/postgresql/restore-test
TARGET_TIME=$(date -d '2 hours ago' -Iseconds)
START=$(date +%s)

rm -rf "$RESTORE_DIR"; mkdir -p "$RESTORE_DIR"; chmod 700 "$RESTORE_DIR"

# ① fetch the base backup — ★ with parallelism
WALG_DOWNLOAD_CONCURRENCY=16 wal-g backup-fetch "$RESTORE_DIR" LATEST
FETCH_DONE=$(date +%s)

# ★ ② verify the manifest BEFORE trying to start
pg_verifybackup "$RESTORE_DIR" || { echo "★ MANIFEST FAILED"; exit 1; }

# ③ configure PITR — ★ pause, do not promote
cat >> "$RESTORE_DIR/postgresql.conf" <<EOF
restore_command = 'wal-g wal-fetch %f %p'
recovery_target_time = '$TARGET_TIME'
recovery_target_action = 'pause'
port = 5499
EOF
touch "$RESTORE_DIR/recovery.signal"

# ④ start and wait for the target
pg_ctl -D "$RESTORE_DIR" -l /tmp/restore.log start
for i in $(seq 1 600); do
  if psql -p 5499 -tAc "SELECT pg_is_in_recovery()" 2>/dev/null | grep -q t; then
    STATE=$(psql -p 5499 -tAc "SELECT pg_get_wal_replay_pause_state()" 2>/dev/null || true)
    [ "$STATE" = "paused" ] && break
  fi
  sleep 1
done
REPLAY_DONE=$(date +%s)

# ★ ⑤ VERIFY THE DATA — not just that it started
ROWS=$(psql -p 5499 -tAc "SELECT count(*) FROM orders WHERE created_at < '$TARGET_TIME'")
SUM=$(psql -p 5499 -tAc "SELECT coalesce(sum(total_minor),0) FROM orders
                          WHERE created_at < '$TARGET_TIME'")
EXPECTED_ROWS=$(psql -h prod -tAc "SELECT count(*) FROM orders
                                    WHERE created_at < '$TARGET_TIME'")
[ "$ROWS" = "$EXPECTED_ROWS" ] || { echo "★ ROW COUNT MISMATCH"; exit 1; }

# ★ ⑥ verify every relation is readable (catches page corruption)
psql -p 5499 -tAc "
  SELECT count(*) FROM (
    SELECT pg_relation_size(c.oid) FROM pg_class c
     WHERE c.relkind IN ('r','i','m')) x" >/dev/null

pg_ctl -D "$RESTORE_DIR" stop -m immediate

# ★ ⑦ RECORD THE TIMES — this is your real RTO
cat > /var/lib/node_exporter/restore_test.prom <<EOF
restore_test_success 1
restore_test_fetch_seconds $(( FETCH_DONE - START ))
restore_test_replay_seconds $(( REPLAY_DONE - FETCH_DONE ))
restore_test_total_seconds $(( $(date +%s) - START ))
restore_test_timestamp $(date +%s)
EOF
```

---

## Concept breakdown

```
★ THREE COMPONENTS — ALL THREE REQUIRED
   ① base backup  ② ★ WAL archive  ③ ★ A TESTED RESTORE
   ⇒ ★ WITHOUT ③, ① AND ② ARE A HOPE.

★ HA ≠ BACKUP
   a bad DELETE replicates in 3 ms to every standby.
   ⇒ ★ replication protects against MACHINES; backups protect
     against HUMANS, BUGS and BITS.

★ WHY A RUNNING BACKUP IS SAFE
   pg_backup_start forces a checkpoint · ★ full_page_writes puts a
   FULL IMAGE of every modified page in the WAL · torn pages are
   OVERWRITTEN during replay
   ⇒ ★ THEREFORE: A BASE BACKUP WITHOUT ITS SPANNING WAL IS
     COMPLETELY UNUSABLE, not merely stale.

★ THE WAL ARCHIVE
   PostgreSQL won't recycle a segment until archive_command exits 0
   ⇒ ★ failing command ⇒ pg_wal fills ⇒ PANIC (Topic 41)
   ⇒ ★ command exits 0 WITHOUT archiving ⇒ a SILENT GAP
   ⇒ ★ archive_timeout bounds RPO on an idle database

★ PITR TARGETS
   time · ★ xid (the precise one) · lsn · ★ name (before migrations)
   ⇒ ★ recovery_target_action = 'pause' — inspect before committing.
     'promote' means a wrong target = restart the whole restore.

★ RESTORE TIME = download + decompress + ★ WAL replay
   ★ download: parallelism is the biggest lever, OFF by default
   ★ decompress: lz4/zstd, never gzip
   ★ replay: SINGLE-THREADED ~100 MB/s
   ⇒ ★ BACKUP FREQUENCY IS AN RTO DECISION: daily backups mean up
     to 24 h of WAL to replay.
   ⇒ measured: 9h40m naive → ★ 1h12m tuned, same storage

pg_dump vs PHYSICAL
   ★ dump: cross-version, single-table restore, ★ NO PITR,
     ★ 14 h restore vs 22 min physical, ★ pins xmin for its duration
   ★ ALWAYS also: pg_dumpall --globals-only (roles!)

★ CORRUPTION
   data_checksums = on ⇒ detected at read time
   pg_verifybackup / pgbackrest check ⇒ the backup itself
   ★ retention must EXCEED your detection time (30–90 days)

★ THE FIVE ALERTS
   archiver failed_count · seconds since last archive · .ready
   pile-up · checksum_failures · ★ NO SUCCESSFUL RESTORE TEST IN 7 DAYS
   ⇒ ★ plus a DEAD MAN'S SWITCH — alert when the backup DOESN'T RUN
```

---

## Diagrams

**Diagram 1 — big picture: what each mechanism protects against**

```
                        THE FAILURE
        ┌──────────────┬─────────────┬──────────────┬─────────────┐
        │ machine dies │ AZ dies     │ ★ bad DELETE │ ★ corruption│
        │              │             │ / migration  │             │
 ───────┼──────────────┼─────────────┼──────────────┼─────────────┤
 replica│      ✓       │  ✓ (multi-  │  ★ ✗ replicates in 3 ms    │
 (58/63)│              │     AZ)     │  ★ ✗ replicates            │
 ───────┼──────────────┼─────────────┼──────────────┼─────────────┤
 pg_dump│      ✓       │      ✓      │  ✓ (to the   │  ✓ (if the  │
        │ ★ slow       │  ★ slow     │  dump time)  │  dump ran   │
        │              │             │              │  before it) │
 ───────┼──────────────┼─────────────┼──────────────┼─────────────┤
 ★ base │      ✓       │      ✓      │  ★ ✓ TO THE  │  ★ ✓ if     │
 + WAL  │              │             │    SECOND    │  retention  │
        │              │             │              │  > detection│
 ───────┴──────────────┴─────────────┴──────────────┴─────────────┘

 ★ THE COLUMN THAT MATTERS: "bad DELETE / migration".
   It is the most common real data-loss event, and ONLY PITR
   addresses it.
 ★ AND THE LAST COLUMN'S CONDITION: if corruption takes 3 weeks to
   notice and you keep 7 days, ★ every backup you hold contains it.
```

**Diagram 2 — data flow: PITR, end to end**

```
  TIME ──────────────────────────────────────────────────────────►
   02:00          08:00                14:22:04          14:30
     │              │                     │                │
     ▼              ▼                     ▼                ▼
  ┌──────┐     ┌──────┐            ★ THE BAD          "we noticed"
  │ BASE │     │ BASE │              DELETE
  │BACKUP│     │BACKUP│
  └──┬───┘     └──┬───┘
     │            │
     └── WAL ─────┴──── WAL ──── WAL ──── WAL ──── WAL ────►
         segments continuously archived to object storage

  ★ RESTORE TO 14:22:00 (4 seconds before the DELETE):
  ┌────────────────────────────────────────────────────────────────┐
  │ ① fetch the 08:00 base backup     ★ 16 parallel streams        │
  │    4 TB / 1.6 GB/s = ★ 42 min                                  │
  │ ② pg_verifybackup                  ★ before wasting hours      │
  │ ③ restore_command fetches WAL from 08:00 → 14:22:00            │
  │    6h22m × 20 GB/h = 127 GB                                    │
  │    ★ replay is SINGLE-THREADED at ~100 MB/s ⇒ 21 min           │
  │ ④ recovery_target_time = '14:22:00'                            │
  │    recovery_target_action = ★ 'pause'                          │
  │ ⑤ ★ INSPECT: does the data look right?                         │
  │    SELECT count(*) FROM accounts WHERE balance > 0;            │
  │ ⑥ ★ wrong target? change it, restart — nothing is committed    │
  │    right target? SELECT pg_wal_replay_resume();  ⇒ promotes    │
  │                                                                 │
  │ ★ TOTAL: 63 minutes. ★ DATA LOSS: 8 minutes (14:22 → 14:30).   │
  └────────────────────────────────────────────────────────────────┘

  ★ IF THE 08:00 BACKUP DIDN'T EXIST (daily at 02:00 only):
    12h22m of WAL = 247 GB ⇒ ★ 41 min of replay instead of 21.
    ⇒ ★ BACKUP FREQUENCY IS AN RTO DECISION.
```

**Diagram 3 — before/after: the restore that took 9 hours**

```
 ✗ THE DEFAULT SETUP — nobody had ever restored it
 ┌───────────────────────────────────────────────────────────────┐
 │ backup:   nightly pg_basebackup to S3, ★ gzip, single stream  │
 │ WAL:      archived, ★ never verified                          │
 │ restore:  ★ never tested                                       │
 │                                                                │
 │ THE INCIDENT — a migration dropped a column at 14:22           │
 │  14:30  noticed                                                │
 │  14:35  "restore from backup"                                  │
 │  14:41  ★ the S3 download runs at 180 MB/s single-stream       │
 │  ★ 20:52  4 TB downloaded                    (6 h 11 m)        │
 │  ★ 21:04  gunzip, single-threaded            (needed 12 m)     │
 │  21:06  start recovery                                         │
 │  ★ 21:14  FATAL: could not locate required checkpoint record   │
 │           ⇒ ★ the WAL segment spanning the backup was MISSING  │
 │           ⇒ the archive had a 40-minute gap from a failed      │
 │             archive_command three weeks earlier                │
 │  21:20   fall back to the PREVIOUS night's backup              │
 │  ★ 03:40  restored — ★ TO THE WRONG DAY                        │
 │                                                                │
 │ ★ TOTAL: 13 hours. ★ DATA LOSS: 26 hours.                     │
 └───────────────────────────────────────────────────────────────┘

 ✓ AFTER — the same 4 TB, the same object storage
 ┌───────────────────────────────────────────────────────────────┐
 │ backup:   ★ full weekly + delta every 6 h, ★ lz4, 16 streams   │
 │ WAL:      ★ archived + pg_verifybackup + archive gap check     │
 │ restore:  ★ TESTED NIGHTLY, with timings recorded              │
 │                                                                │
 │  14:30  noticed                                                │
 │  14:32  ★ run the tested restore script                        │
 │  15:14  4 TB downloaded              ★ 42 m (16 streams, lz4)  │
 │  15:16  pg_verifybackup ✓                                      │
 │  15:37  ★ replayed to 14:22:00       21 m                      │
 │  15:38  ★ PAUSED — inspected — correct                         │
 │  15:39  pg_wal_replay_resume() ⇒ promoted                      │
 │                                                                │
 │ ★ TOTAL: 67 minutes. ★ DATA LOSS: 8 minutes.                  │
 │ ★ AND IT WAS BORING, because it had been done 180 times.      │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**Set up continuous archiving.**
```bash
mkdir -p /var/lib/postgresql/archive
psql -c "ALTER SYSTEM SET archive_mode = on;
         ALTER SYSTEM SET archive_command =
           'test ! -f /var/lib/postgresql/archive/%f && cp %p /var/lib/postgresql/archive/%f';
         ALTER SYSTEM SET archive_timeout = '60s';
         ALTER SYSTEM SET wal_compression = 'lz4';"
pg_ctl restart
```
```sql
SELECT archived_count, failed_count, last_archived_wal, last_archived_time
  FROM pg_stat_archiver;
```
```
 archived_count | failed_count | last_archived_wal        | last_archived_time
----------------+--------------+--------------------------+---------------------------
             12 |            0 | 00000001000000000000000C | 2026-08-24 14:04:22+05:30
```

**Take a base backup.**
```bash
pg_basebackup -D /backups/base-$(date +%F-%H%M) -Ft -z -Xs -P -c fast
#                                                    ▲▲
#                                    ★ -Xs streams the WAL alongside.
#                                    ★ WITHOUT IT THE BACKUP IS UNUSABLE.
```
```
 4194304/4194304 kB (100%), 1/1 tablespace
```
```bash
ls -la /backups/base-2026-08-24-1404/
```
```
 base.tar.gz
 pg_wal.tar.gz        ★ the WAL that spans the backup — from -Xs
 backup_manifest      ★ used by pg_verifybackup
```

**Verify it — before you need it.**
```bash
pg_verifybackup /backups/base-2026-08-24-1404
```
```
 backup successfully verified
```

**Prove a base backup without WAL is useless.**
```bash
pg_basebackup -D /backups/nowal -Ft -Xnone -P     # ★ no WAL
cp -a /backups/nowal /tmp/restore-nowal
cd /tmp/restore-nowal && tar xf base.tar
pg_ctl -D /tmp/restore-nowal start
```
```
 FATAL:  could not locate required checkpoint record
 HINT:  If you are restoring from a backup, touch
        "/tmp/restore-nowal/recovery.signal" …
   ★ COMPLETELY UNUSABLE. Not "slightly stale" — unusable.
```

**Do a PITR.**
```sql
-- ① note the time, then cause a disaster
SELECT now();
```
```
 2026-08-24 14:22:00.412+05:30
```
```sql
SELECT count(*) FROM orders;
```
```
 count
--------
 ★ 412088
```
```sql
DELETE FROM orders WHERE created_at < '2026-01-01';   -- ★ the mistake
SELECT count(*) FROM orders;
```
```
 count
--------
 ★ 88204
```
```bash
# ② restore to just before it
pg_ctl stop -D "$PGDATA" -m fast
rm -rf /tmp/pitr && mkdir -p /tmp/pitr && chmod 700 /tmp/pitr
tar xzf /backups/base-2026-08-24-1404/base.tar.gz -C /tmp/pitr
tar xzf /backups/base-2026-08-24-1404/pg_wal.tar.gz -C /tmp/pitr/pg_wal

cat >> /tmp/pitr/postgresql.conf <<'EOF'
restore_command = 'cp /var/lib/postgresql/archive/%f %p'
recovery_target_time = '2026-08-24 14:22:00+05:30'
recovery_target_action = 'pause'
recovery_target_inclusive = false
port = 5499
EOF
touch /tmp/pitr/recovery.signal
pg_ctl -D /tmp/pitr -l /tmp/pitr.log start
```
```bash
grep -E 'recovery|consistent|pause' /tmp/pitr.log
```
```
 LOG:  starting point-in-time recovery to 2026-08-24 14:22:00+05:30
 LOG:  restored log file "00000001000000000000000C" from archive
 LOG:  redo starts at 0/C000028
 LOG:  consistent recovery state reached at 0/C0000F8
 LOG:  recovery stopping before commit of transaction 8842119, time 2026-08-24 14:22:03
 LOG:  ★ pausing at the end of recovery
 HINT:  Execute pg_wal_replay_resume() to promote.
```
```bash
psql -p 5499 -c "SELECT count(*) FROM orders"
```
```
 count
--------
 ★ 412088        — the DELETE is undone.
```
```sql
-- ③ ★ inspect first. Only then commit to it.
SELECT pg_is_in_recovery(), pg_get_wal_replay_pause_state();
```
```
 pg_is_in_recovery | pg_get_wal_replay_pause_state
-------------------+-------------------------------
 t                 | ★ paused
```
```sql
SELECT pg_wal_replay_resume();   -- ★ now it promotes
SELECT pg_is_in_recovery();
```
```
 f
```

**Recover to an exact transaction instead of a time.**
```bash
# ★ find the offending transaction in the WAL
pg_waldump /var/lib/postgresql/archive/00000001000000000000000C \
  | grep -i 'DELETE\|COMMIT' | head
```
```
 rmgr: Heap  desc: DELETE off 4 flags 0x00 ... xid 8842119
 rmgr: Transaction  desc: COMMIT 2026-08-24 14:22:03.884
```
```ini
recovery_target_xid = '8842119'
recovery_target_inclusive = false     # ★ stop BEFORE it
recovery_target_action = 'pause'
```
```
 ★ MORE PRECISE THAN A TIMESTAMP: if several transactions commit
   in the same millisecond, a time target cannot separate them.
```

**Create a restore point before a migration.**
```sql
SELECT pg_create_restore_point('before_v42_migration');
```
```
 pg_create_restore_point
-------------------------
 4A/8C001220
```
```ini
recovery_target_name = 'before_v42_migration'
recovery_target_action = 'pause'
-- ★ do this before EVERY migration. It costs one query.
```

**Prove a broken `archive_command` fills the disk.**
```sql
ALTER SYSTEM SET archive_command = 'false';   -- ★ always fails
SELECT pg_reload_conf();
```
```bash
pgbench -c 10 -T 60 shop >/dev/null
ls "$PGDATA/pg_wal/archive_status/"*.ready | wc -l
du -sh "$PGDATA/pg_wal"
```
```
 ★ 184
 ★ 2.9G        — and growing. This ends in a PANIC (Topic 41).
```
```sql
SELECT failed_count, last_failed_wal, last_failed_time FROM pg_stat_archiver;
```
```
 failed_count | last_failed_wal          | last_failed_time
--------------+--------------------------+---------------------------
        ★ 184 | 000000010000000000000015 | 2026-08-24 14:31:08+05:30
```

**And the more dangerous version — a command that succeeds without archiving.**
```sql
ALTER SYSTEM SET archive_command = 'true';   -- ★ exits 0, archives nothing
SELECT pg_reload_conf();
```
```bash
pgbench -c 10 -T 60 shop >/dev/null
psql -tAc "SELECT archived_count, failed_count FROM pg_stat_archiver"
```
```
 ★ 296|0        — "296 archived, 0 failed". Everything looks perfect.
```
```bash
ls /var/lib/postgresql/archive/ | wc -l
```
```
 ★ 12        — ★ THE SEGMENTS ARE GONE. A SILENT, PERMANENT GAP.
   ⇒ this is why you must verify the ARCHIVE, not just the counters.
```

**Detect archive gaps.**
```bash
# ★ list what should exist between two segments and diff it
psql -tAc "SELECT pg_walfile_name(pg_current_wal_lsn())"
ls /var/lib/postgresql/archive/ | grep -E '^[0-9A-F]{24}$' | sort > /tmp/have
# generate the expected sequence and compare
comm -13 /tmp/have /tmp/expected | head
```
```
 ★ 000000010000000000000010
 ★ 000000010000000000000011      — a gap. Recovery cannot cross it.
```

**Compare `pg_dump` restore against physical.**
```bash
\timing on
time pg_dump -Fc -j 8 -f /tmp/shop.dump shop
```
```
 real  ★ 4m12s        (400 GB)
```
```bash
time pg_restore -j 8 -d shop_restored /tmp/shop.dump
```
```
 real  ★ 41m08s        — 10× the dump time, because every index rebuilds
```
```bash
time (tar xzf /backups/base.tar.gz -C /tmp/phys && pg_ctl -D /tmp/phys start)
```
```
 real  ★ 3m22s        — ★ 12× faster than pg_restore
```

**And the globals nobody backs up.**
```bash
pg_dumpall --globals-only > /tmp/globals.sql
head -5 /tmp/globals.sql
```
```
 CREATE ROLE app_user;
 ALTER ROLE app_user WITH LOGIN PASSWORD 'SCRAM-SHA-256$...';
 CREATE ROLE readonly;
   ★ NONE OF THIS IS IN pg_dump OR IN A BASE BACKUP OF ONE DATABASE.
     A restore without it fails on "role does not exist".
```

**Enable and check checksums.**
```sql
SHOW data_checksums;
```
```
 on
```
```sql
SELECT datname, checksum_failures, checksum_last_failure
  FROM pg_stat_database WHERE checksum_failures > 0;
```
```
 (0 rows)        ★ any row here is a page-level corruption alert.
```

---

## Example 2 — production scenario

**The situation.** A healthcare records platform. 6 TB, PostgreSQL 16, three-node Patroni cluster, nightly `pg_basebackup` to S3, WAL archived with `wal-g`. Regulatory RPO: 15 minutes. RTO: 4 hours.

```
 THE INCIDENT
   09:14  a data-migration job intended to normalise phone numbers
          runs against production instead of staging.
          ★ 4.1 MILLION patient records have their `phone` column
            overwritten with a truncated value.
   09:31  a clinician reports a wrong number
   09:44  ★ confirmed: 4.1M rows corrupted, replicated to all
          three nodes in 4 ms
   09:45  ★ "fail over to a replica" — no. They all have it.
   09:47  the decision: PITR to 09:13
```

**Step 1 — what they found in the first thirty minutes.**

```bash
wal-g backup-list
```
```
 name                          modified              wal_segment_backup_start
 base_000000010000004A0000008C 2026-08-24T02:00:11Z  000000010000004A0000008C
 base_00000001000000490000001A 2026-08-23T02:00:09Z  00000001000000490000001A
```
```
 ★ FINDING 1: the newest backup is 7 hours 14 minutes old.
   ⇒ 7h14m × 22 GB/h = ★ 159 GB of WAL to replay
   ⇒ replay is single-threaded at ~100 MB/s ⇒ ★ 27 minutes
   ⇒ acceptable — but only just.
```
```bash
env | grep WALG
```
```
 WALG_S3_PREFIX=s3://hc-backups/pg-prod
 WALG_COMPRESSION_METHOD=★ gzip
 (★ WALG_DOWNLOAD_CONCURRENCY is not set — defaults to 10 but
  gzip cannot be decompressed in parallel per file)
```
```
 ★ FINDING 2: gzip. A 6 TB restore would decompress
   single-threaded at ~60 MB/s ⇒ ★ 28 HOURS.
 ⇒ ★ THIS ALONE BLEW THE 4-HOUR RTO, AND NOBODY KNEW BECAUSE
   NO RESTORE HAD EVER BEEN TIMED.
```
```sql
SELECT archived_count, failed_count, last_archived_time FROM pg_stat_archiver;
```
```
 archived_count | failed_count |    last_archived_time
----------------+--------------+---------------------------
       18402118 |         ★ 41 | 2026-08-24 09:44:02+05:30
```
```
 ★ FINDING 3: 41 archive failures, historically. Were any of them
   in the window we need?
```
```bash
# ★ enumerate the WAL the restore will need and check every one exists
wal-g wal-verify integrity
```
```
 [wal-verify] integrity check status: ★ OK
 [wal-verify] timeline 1, segments 000000010000004A0000008C ..
              000000010000004A000000F2 — ★ 103 segments, no gaps
```
```
 ★ RELIEF. The 41 failures were all retried successfully. But
   ★ NOBODY KNEW THAT UNTIL THEY RAN THE CHECK, DURING AN INCIDENT.
```

**Step 2 — the restore, and the decision that saved them.**

```
 ★ THE OPTIONS, WEIGHED IN REAL TIME:

 ① FULL PITR OF THE WHOLE CLUSTER TO 09:13
    ⇒ ★ 28 h decompress. Blows RTO by 7×.
    ⇒ ★ AND it discards 31 minutes of legitimate work from every
      other part of the system — appointments, prescriptions,
      lab results.

 ② ★ RESTORE TO A SIDE CLUSTER, EXTRACT ONLY THE PHONE COLUMN,
    AND REPAIR IN PLACE.
    ⇒ still needs the restore, but ★ the production cluster stays
      up the whole time
    ⇒ ★ no legitimate work is lost
    ⇒ THIS IS ALMOST ALWAYS THE RIGHT SHAPE FOR A LOGICAL DATA
      CORRUPTION, and it is not what people reach for first.

 ⇒ ★ THEY CHOSE ②. The production database was never taken down.
```

```bash
# ★ restore to a side host, with every parallelism lever engaged
export WALG_DOWNLOAD_CONCURRENCY=32
export WALG_S3_PREFIX=s3://hc-backups/pg-prod

wal-g backup-fetch /restore LATEST
# ★ 6 TB, gzip, 32 streams across FILES (each file still serial)
#   ⇒ measured 4 h 51 m — better than 28 h, still far too slow
```
```bash
pg_verifybackup /restore && echo "manifest ok"
```
```
 ★ manifest ok
```
```ini
# /restore/postgresql.conf
restore_command = 'wal-g wal-fetch %f %p'
recovery_target_time = '2026-08-24 09:13:00+05:30'
recovery_target_action = 'pause'          # ★ inspect first
recovery_target_inclusive = false
port = 5499
max_wal_size = 32GB                        # ★ speeds replay
maintenance_work_mem = 8GB
```
```bash
touch /restore/recovery.signal
pg_ctl -D /restore -l /restore/recovery.log start
tail -f /restore/recovery.log
```
```
 LOG:  starting point-in-time recovery to 2026-08-24 09:13:00+05:30
 LOG:  consistent recovery state reached at 4A/8C0F2210
 LOG:  recovery stopping before commit of transaction 88420118
 LOG:  ★ pausing at the end of recovery
```
```
 ★ TOTAL: 5 h 18 m (fetch 4h51m + replay 27m).
   ★ RTO WAS 4 HOURS. THEY MISSED IT — but production never went
     down, so the user-visible impact was 31 minutes of wrong phone
     numbers, not 5 hours of no system.
```

**Step 3 — the repair.**

```sql
-- ★ on the restored side cluster, extract ONLY what was damaged
COPY (
  SELECT id, phone FROM patients
) TO '/tmp/phones_before.csv' WITH (FORMAT csv, HEADER);
```
```bash
scp /tmp/phones_before.csv prod:/tmp/
```
```sql
-- ★ on production, in a transaction, with verification first
BEGIN;
CREATE TEMP TABLE phone_restore (id bigint PRIMARY KEY, phone text);
\copy phone_restore FROM '/tmp/phones_before.csv' WITH (FORMAT csv, HEADER)

-- ★ how many actually differ? (some were legitimately changed
--   between 09:13 and 09:14 — do not clobber those)
SELECT count(*) FROM patients p JOIN phone_restore r ON r.id = p.id
 WHERE p.phone IS DISTINCT FROM r.phone;
```
```
 count
---------
 ★ 4102884
```
```sql
-- ★ and how many were changed by the migration vs legitimately?
SELECT count(*) FROM patients p JOIN phone_restore r ON r.id = p.id
 WHERE p.phone IS DISTINCT FROM r.phone
   AND p.updated_at BETWEEN '2026-08-24 09:13:00' AND '2026-08-24 09:15:00';
```
```
 count
---------
 ★ 4102884        — all of them. No legitimate edits in the window.
```
```sql
-- ★ audit the repair itself
INSERT INTO data_repair_audit (incident, table_name, rows_affected, repaired_at)
VALUES ('INC-2026-0824', 'patients', 4102884, now());

UPDATE patients p SET phone = r.phone, updated_at = now()
  FROM phone_restore r
 WHERE r.id = p.id AND p.phone IS DISTINCT FROM r.phone;

SELECT count(*) FROM patients p JOIN phone_restore r ON r.id = p.id
 WHERE p.phone IS DISTINCT FROM r.phone;
```
```
 count
-------
 ★ 0
```
```sql
COMMIT;
```

**Step 4 — the seven changes that followed.**

```bash
# ★ ① COMPRESSION — the single largest RTO lever
export WALG_COMPRESSION_METHOD=lz4       # ★ was gzip
export WALG_UPLOAD_CONCURRENCY=16
export WALG_DOWNLOAD_CONCURRENCY=32
# ★ MEASURED: 6 TB restore 4h51m → 51m.  5.7×.
```

```bash
# ★ ② BACKUP FREQUENCY — an RTO decision, not a storage decision
# was: full nightly
# now: full weekly + delta every 4 hours
export WALG_DELTA_MAX_STEPS=6
0 2 * * 0        wal-g backup-push $PGDATA --full
0 2,6,10,14,18,22 * * *  wal-g backup-push $PGDATA
# ★ MEASURED: max WAL to replay 7h14m → 4h ⇒ replay 27m → 15m
# ★ storage cost: +18% (deltas are small)
```

```bash
# ★ ③ THE NIGHTLY RESTORE TEST — this is the actual fix
# (the full script from "How it works", running at 03:00)
# ★ It records: fetch time, replay time, total, and a row-count
#   verification against production.
```

```sql
-- ★ ④ THE FIVE ALERTS, ALL OF WHICH WERE MISSING
-- a) archiver failures
SELECT failed_count FROM pg_stat_archiver;              -- ★ alert on increase
-- b) archive staleness
SELECT extract(epoch from now() - last_archived_time)   -- ★ alert > 300
  FROM pg_stat_archiver;
-- c) unarchived pile-up
SELECT count(*) FROM pg_ls_dir('pg_wal/archive_status') f
 WHERE f LIKE '%.ready';                                 -- ★ alert > 100
-- d) checksum failures
SELECT sum(checksum_failures) FROM pg_stat_database;     -- ★ alert > 0
-- e) ★ RESTORE TEST FRESHNESS
--    restore_test_timestamp from the nightly job          ★ alert > 7 days
```

```bash
# ★ ⑤ A DEAD MAN'S SWITCH — alert when the backup DOESN'T RUN
#    (a removed cron job produces no failure, only silence)
wal-g backup-push $PGDATA && curl -fsS "https://hc-ping.com/$UUID"
```

```bash
# ★ ⑥ WAL ARCHIVE INTEGRITY, CONTINUOUSLY — not during an incident
0 */4 * * *  wal-g wal-verify integrity timeline \
               || alert "★ WAL archive gap detected"
```

```sql
-- ★ ⑦ A RESTORE POINT BEFORE EVERY MIGRATION
--    added to the migration runner, unconditionally
SELECT pg_create_restore_point('pre_migration_' || :version);
-- ⇒ ★ turns "restore to a guessed timestamp" into
--   "restore to a named, exact point".
```

**Step 5 — and the process change that mattered most.**

```
 ★ THE MIGRATION RAN AGAINST PRODUCTION BECAUSE THE OPERATOR HAD
   TWO TERMINALS OPEN AND USED THE WRONG ONE.

 ⇒ ★ THE TECHNICAL CONTROLS ADDED:
   ① ★ production psql prompt shows a red PRODUCTION banner
      \set PROMPT1 '%[%033[1;31m%]PRODUCTION%[%033[0m%] %/%R%# '
   ② ★ the migration runner REFUSES to run without an explicit
      --env=production flag AND a typed confirmation of the
      database name
   ③ ★ a transaction-wrapped dry run that reports affected row
      counts and requires confirmation above a threshold:
      BEGIN; UPDATE …; -- ⇒ "4,102,884 rows"
      ⇒ ★ above 10,000 rows, the runner aborts and demands review
   ④ ★ the app role has no direct psql access; migrations run
      through the runner only (Topic 69)

 ⇒ ★ PITR IS THE LAST LINE OF DEFENCE, NOT THE FIRST. The cheapest
   fix in this incident was a red prompt.
```

**Step 6 — results.**

| | Before | After |
|---|---|---|
| Compression | gzip | ★ **lz4** |
| 6 TB restore (fetch) | 4 h 51 m | ★ **51 min** (5.7×) |
| Max WAL to replay | 7 h 14 m (159 GB) | **4 h** (88 GB) |
| Total measured RTO | ★ **5 h 18 m** (unknown before) | ★ **1 h 09 m**, measured nightly |
| Restore tested | ★ never | ★ **nightly, timed, verified** |
| Archive integrity | checked during an incident | every 4 hours |
| Backup alerts | none | 5 + a dead man's switch |
| Restore points | none | ★ before every migration |
| Wrong-environment protection | none | ★ 4 controls |

```
 ★ SIX LESSONS:
 ① ★ "FAIL OVER TO A REPLICA" WAS THE FIRST INSTINCT AND WAS
   USELESS. The corruption was on all three nodes in 4 ms.
 ② ★ THE RTO WAS UNKNOWN BECAUSE NO RESTORE HAD EVER BEEN TIMED.
   gzip made it 28 hours in theory and 4h51m in practice — a
   number nobody could have quoted before the incident.
 ③ ★ RESTORING TO A SIDE CLUSTER AND REPAIRING IN PLACE beat a
   full PITR: production never went down and no legitimate work
   was lost. ★ This is usually the right shape for logical
   corruption, and it is not the first instinct.
 ④ ★ BACKUP FREQUENCY IS AN RTO DECISION. Nightly backups mean up
   to 24 hours of single-threaded WAL replay.
 ⑤ ★ THE ARCHIVE HAD 41 HISTORICAL FAILURES AND NOBODY KNEW
   WHETHER ANY MATTERED. `wal-verify integrity` answered it in 40
   seconds — and should have been running every 4 hours for years.
 ⑥ ★ THE CHEAPEST FIX WAS A RED PROMPT. PITR is the last line of
   defence; preventing the wrong-environment mistake costs nothing.
```

---

## Common mistakes

**1. Treating replication as backup.**
- *Symptom:* a bad `UPDATE` is on every standby in milliseconds and there is nothing to fail over to.
- *Fix:* base backups + WAL archive + a tested restore. They solve different problems (Topic 63).

**2. Never testing a restore.**
- *Symptom:* the first restore attempt is during an incident, and it fails on a missing WAL segment, a missing role, or a missing key.
- *Fix:* an automated nightly restore that verifies row counts and records timings. **This is the backup system.**

**3. `pg_basebackup` without `-Xs`.**
- *Symptom:* `FATAL: could not locate required checkpoint record` — the backup is completely unusable.
- *Fix:* always stream WAL alongside, or guarantee the archive covers the backup window.

**4. An `archive_command` that returns 0 without archiving.**
- *Symptom:* `archived_count` climbs, `failed_count` is 0, and the archive is empty. A silent, permanent gap.
- *Fix:* verify the archive itself (`wal-g wal-verify integrity`), not just the counters.

**5. Not alerting on `failed_count`.**
- *Symptom:* WAL accumulates until `pg_wal` fills and the primary PANICs (Topic 41).
- *Fix:* alert on any increase, and on time since last archive.

**6. gzip compression.**
- *Symptom:* a restore that is 5–20× slower than necessary, discovered during an incident.
- *Fix:* lz4 or zstd, plus download concurrency.

**7. Not setting download concurrency.**
- *Symptom:* a 4 TB single-stream download taking six hours.
- *Fix:* `WALG_DOWNLOAD_CONCURRENCY` / `--process-max`. It is off by default and is the biggest RTO lever.

**8. Choosing backup frequency by storage cost.**
- *Symptom:* nightly backups mean up to 24 hours of single-threaded WAL replay on top of the download.
- *Fix:* frequency is an RTO decision. Deltas are cheap.

**9. `recovery_target_action = 'promote'`.**
- *Symptom:* a wrong target means restarting a multi-hour restore.
- *Fix:* `'pause'`, inspect, then `pg_wal_replay_resume()`.

**10. Forgetting `pg_dumpall --globals-only`.**
- *Symptom:* the restore completes and the application cannot connect — the roles don't exist.
- *Fix:* back up globals separately, every day.

**11. Retention shorter than detection time.**
- *Symptom:* corruption noticed after three weeks; every retained backup contains it.
- *Fix:* 30–90 days for the primary chain, plus monthly archives.

**12. Storing the encryption key in the database being backed up.**
- *Symptom:* the backups are unreadable precisely when you need them.
- *Fix:* an independent key store, with its own tested recovery.

**13. Restoring the whole cluster for logical corruption.**
- *Symptom:* hours of downtime and the loss of every legitimate transaction since the incident.
- *Fix:* restore to a side cluster, extract the damaged data, repair in place.

**14. No dead man's switch.**
- *Symptom:* a cron job removed months ago; no failures, no backups, no alerts.
- *Fix:* alert on the *absence* of a success signal.

---

## Hands-on proof

**PROVE IT #1–#12 — Example 1** (archiving configured and verified, a base backup with `-Xs`, `pg_verifybackup`, a backup without WAL failing to start, a full PITR undoing a `DELETE` with `pause`/`resume`, an xid target, a named restore point, a failing `archive_command` piling up 2.9 GB, an `archive_command` that silently loses segments, gap detection, `pg_dump` restore at 12× physical, and the globals nobody backs up).

**PROVE IT #13 — measure restore time properly.**
```bash
for c in 1 8 32; do
  rm -rf /tmp/rt; mkdir /tmp/rt
  echo -n "concurrency=$c  "
  /usr/bin/time -f '%e s' env WALG_DOWNLOAD_CONCURRENCY=$c \
    wal-g backup-fetch /tmp/rt LATEST 2>&1 | tail -1
done
```
```
 concurrency=1   ★ 8,842 s
 concurrency=8   ★ 1,402 s
 concurrency=32  ★ 512 s        — 17×, from one environment variable
```

**PROVE IT #14 — WAL replay is the other half.**
```bash
# restore to a target 1 hour after the backup, then 12 hours after
for hrs in 1 6 12; do
  T=$(date -d "$(date -d '2026-08-24 02:00' +%s) + $hrs hour" -Iseconds)
  # …configure recovery_target_time=$T, start, time until paused…
done
```
```
 target +1h   replay ★ 84 s
 target +6h   replay ★ 508 s
 target +12h  replay ★ 1,022 s     — ★ linear in WAL volume,
                                     single-threaded
```

**PROVE IT #15 — corruption detection.**
```bash
# ★ deliberately corrupt a page in a scratch cluster
pg_ctl stop -D /tmp/scratch
dd if=/dev/urandom of=/tmp/scratch/base/16384/16400 bs=1 seek=8300 count=64 conv=notrunc
pg_ctl start -D /tmp/scratch
psql -p 5499 -c "SELECT count(*) FROM victim_table"
```
```
 ERROR:  invalid page in block 1 of relation base/16384/16400
   ★ WITH data_checksums = on, this is caught at read time.
   ★ WITHOUT it, the query returns WRONG DATA silently.
```
```sql
SELECT datname, checksum_failures FROM pg_stat_database WHERE checksum_failures > 0;
```

**PROVE IT #16 — the dead man's switch.**
```bash
# disable the backup cron and confirm the alert fires
crontab -l | grep -v backup-push | crontab -
# ★ within the configured grace period, healthchecks.io alerts:
#   "pg-prod-backup has not pinged in 26 hours"
# ⇒ ★ a plain "alert on failure" would NEVER have fired.
```

---

## The design decision framework

```
★★★ THE ONLY BACKUP THAT EXISTS IS ONE YOU HAVE RESTORED,
    TIMED AND VERIFIED. ★★★

 ① ★ START FROM RPO AND RTO, AND WRITE THEM DOWN
    RPO ⇒ ★ archive_timeout bounds it on an idle database;
          otherwise it is "the last archived segment"
    RTO ⇒ ★ download + decompress + WAL replay, MEASURED
    ⇒ ★ if you cannot state your measured RTO, you do not have one.

 ② ★ USE PHYSICAL BACKUPS + WAL ARCHIVE AS THE PRIMARY
    pg_basebackup / wal-g / pgBackRest
    ✓ ★ ALWAYS -Xs (or guarantee the archive covers the window)
    ✓ ★ pg_verifybackup after every backup
    ✓ ★ PLUS pg_dumpall --globals-only, daily
    ✓ PLUS a periodic pg_dump for single-table restores
    ✗ ★ pg_dump alone is not a backup strategy: no PITR, 10× the
      restore time, and ★ it pins xmin for its duration

 ③ ★ THE THREE RTO LEVERS, IN ORDER OF IMPACT
    ① ★ DOWNLOAD CONCURRENCY — off by default. 17× measured.
    ② ★ COMPRESSION — lz4/zstd, never gzip. 5.7× measured.
    ③ ★ BACKUP FREQUENCY — replay is SINGLE-THREADED at ~100 MB/s.
       ⇒ ★ frequency is an RTO decision, not a storage decision.

 ④ ★ THE RESTORE TEST IS THE BACKUP SYSTEM
    nightly, automated, and it must:
    ✓ fetch with production settings
    ✓ ★ pg_verifybackup BEFORE starting
    ✓ recover to a target, with recovery_target_action = 'pause'
    ✓ ★ VERIFY DATA — row counts and checksums against production
    ✓ ★ RECORD TIMINGS as metrics (fetch / replay / total)
    ⇒ ★ ALERT if no successful test in 7 days.

 ⑤ ★ FIVE ALERTS + A DEAD MAN'S SWITCH
    ✓ pg_stat_archiver.failed_count increasing
    ✓ time since last_archived_time > 5 min
    ✓ .ready files > 100
    ✓ checksum_failures > 0
    ✓ ★ restore-test freshness
    ✓ ★ A DEAD MAN'S SWITCH — alert when the backup DOESN'T RUN.
      A removed cron job produces silence, not failure.

 ⑥ ★ VERIFY THE ARCHIVE, NOT JUST THE COUNTERS
    an archive_command that exits 0 without archiving produces a
    ★ silent, permanent gap and perfect-looking statistics.
    ⇒ `wal-g wal-verify integrity` / `pgbackrest check`, every 4 h.

 ⑦ RETENTION MUST EXCEED DETECTION TIME
    ★ if corruption takes 3 weeks to notice and you keep 7 days,
      every backup you hold contains it.
    ⇒ 30–90 days primary chain + monthly archives.
    ⇒ ★ and keep the encryption key somewhere OTHER than the
      database you are backing up.

 ⑧ ★ FOR LOGICAL CORRUPTION, RESTORE TO A SIDE CLUSTER
    extract the damaged data, repair in place, ★ keep production up.
    ⇒ a full PITR discards every legitimate transaction since the
      incident. That is almost never what you want.

 ⑨ ★ MAKE PITR PRECISE
    ✓ pg_create_restore_point() before EVERY migration
    ✓ recovery_target_xid for surgical targets
    ✓ ★ recovery_target_action = 'pause' — inspect before committing

 ⑩ ★ AND PREVENT THE INCIDENT
    PITR is the last line of defence. A red PRODUCTION prompt, a
    migration runner requiring an explicit environment flag, and a
    dry-run row-count threshold cost nothing and prevent the most
    common cause of needing it.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Configure WAL archiving and take a base backup. Then: (a) verify it with `pg_verifybackup`; (b) take one *without* `-Xs` and prove it cannot start; (c) delete some rows, restore to a point just before, and confirm the rows return; (d) show `pg_get_wal_replay_pause_state()` and resume.

### Exercise 2 — medium (apply it)
Break archiving three ways and show the symptom of each: (a) an `archive_command` that fails — show `.ready` files accumulating and `failed_count` rising; (b) one that exits 0 without archiving — show perfect statistics and an empty archive; (c) a gap in the middle of the archive — show recovery failing to cross it.

Then write the check that detects (b) and (c), and measure restore time at three download-concurrency settings.

### Exercise 3 — hard (production simulation)
A migration corrupts the `phone` column of 4.1 million patient records at 09:14 and replicates to all three nodes in 4 ms. Regulatory RPO is 15 minutes; RTO is 4 hours. The newest backup is 7h14m old, compression is gzip, and no restore has ever been timed.

(a) Explain why failing over to a replica is useless here, in one sentence.
(b) Compute the WAL replay time given 22 GB/hour and single-threaded replay at ~100 MB/s.
(c) Estimate the decompression time for 6 TB of gzip and explain why nobody knew this number.
(d) Two restore strategies are available. Describe both, and argue for the one that keeps production up. What does the other one destroy?
(e) 41 historical archive failures are visible. Write the command that determines whether any of them matter, and explain why it should have been running for years.
(f) Write the repair procedure, including how you distinguish migration damage from legitimate edits in the same window.
(g) Give the three RTO levers in order of impact, with the measured improvement each produced.
(h) Write the nightly restore test. What must it verify beyond "the server started"?
(i) Give the five alerts plus the dead man's switch, and explain what the dead man's switch catches that the others cannot.
(j) The root cause was two terminals and the wrong one. Give four technical controls that prevent it, and explain why this matters more than any backup improvement.

---

## Mental model checkpoint

1. Why is a replica not a backup? Give the specific failure class it cannot address.
2. Why can a base backup be taken from a running database? Which two mechanisms make it safe?
3. Why is a base backup without its spanning WAL completely unusable rather than merely stale?
4. Name the two ways `archive_command` fails, and say which is more dangerous.
5. Name the four PITR target types. Which is most precise and why?
6. Why should `recovery_target_action` be `'pause'`?
7. Give the three components of restore time and the lever for each.
8. Why is backup frequency an RTO decision?
9. Name three things `pg_dump` is good for and three reasons it is not your primary backup.
10. What does `data_checksums` buy, and what does it cost?
11. Why must retention exceed detection time?
12. What is a dead man's switch and what does it catch that failure alerts cannot?
13. For logical corruption of one column, why is a side-cluster restore better than a full PITR?

---

## Quick reference card

```ini
archive_mode = on
archive_command = 'wal-g wal-push %p'
archive_timeout = '60s'          # ★ bounds RPO when idle
data_checksums = on              # ★ set at initdb
full_page_writes = on            # ★ never off — backups depend on it
```
```bash
pg_basebackup -D /b -Ft -z -Xs -P -c fast   # ★ -Xs is mandatory
pg_verifybackup /b                          # ★ after every backup
pg_dumpall --globals-only > globals.sql     # ★ roles — daily
export WALG_COMPRESSION_METHOD=lz4          # ★ never gzip
export WALG_DOWNLOAD_CONCURRENCY=32         # ★ the biggest RTO lever
```
**PITR**
```ini
restore_command = 'wal-g wal-fetch %f %p'
recovery_target_time = '2026-08-24 14:22:00+05:30'   # or _xid / _lsn / _name
recovery_target_inclusive = false
recovery_target_action = 'pause'    # ★ inspect, then pg_wal_replay_resume()
```
```sql
SELECT pg_create_restore_point('pre_migration_v42');   -- ★ before EVERY migration
```

**Monitor — five alerts**
```sql
SELECT archived_count, failed_count, last_archived_time FROM pg_stat_archiver;
SELECT count(*) FROM pg_ls_dir('pg_wal/archive_status') f WHERE f LIKE '%.ready';
SELECT sum(checksum_failures) FROM pg_stat_database;
-- + backup age + ★ restore-test freshness + ★ A DEAD MAN'S SWITCH
```
```bash
wal-g wal-verify integrity     # ★ every 4 h — counters can lie
```

**Restore time** = download (★ concurrency) + decompress (★ lz4) + ★ **WAL replay (single-threaded, ~100 MB/s)**.
**★ Backup frequency is an RTO decision.**

**★ HA protects machines. Backups protect against humans, bugs and bits.**
**★ For logical corruption: restore to a side cluster, repair in place, keep production up.**

---

## When would I use this at work?

1. **Day one on any system you inherit.** Three questions: *when did a restore last succeed?*, *how long did it take?*, and *what compression and concurrency are configured?* In my experience the answers are "never", "unknown", and "gzip, default" — and that combination is a multi-hour RTO nobody has budgeted for.

2. **Before every migration.** `pg_create_restore_point()` costs one query and turns "restore to a guessed timestamp" into "restore to an exact named point". It should be in the migration runner, unconditionally.

3. **When someone says "we're covered, we have replicas".** The bad-`DELETE` column of the protection table settles it. Replication and backup are orthogonal, and the failure they each cover is the one the other cannot.

4. **Designing the response to logical corruption.** The instinct is a full PITR; the right move is usually a side-cluster restore and an in-place repair, because it keeps production up and preserves every legitimate transaction since the incident. Knowing that in advance is worth hours during one.

---

## Connected topics

**Understand before this:** 41 (WAL, `full_page_writes`, `archive_command`), 42 (recovery and replay throughput), 47 (why a long `pg_dump` pins `xmin`), 63 (HA — what this is *not*).

**This unlocks:**
- **65** — connection pooling
- **67** — performance investigation: recognising a restore or replay bottleneck
- **69** — security at the data layer: backup encryption and key custody
- **60** — sharding: why N shards means N backup timelines with no consistent cross-shard restore point
