# 62 — Replication in Depth
## Phase: Reliability & Operations

---

## ELI5 — The Simple Analogy

Two ways to keep a second copy of a ledger in another building.

**Physical replication is photocopying the pages.** Every time a line is written, the exact page — ink, smudges, page number, everything — is copied. The second building's ledger is **byte-for-byte identical**. You cannot copy just the sales section; you copy the whole book. You cannot use a different binding; the copy must match exactly. And you cannot write anything in the copy, because the next photocopy would overwrite it.

**Logical replication is dictating the changes over the phone.** *"Add a row to sales: date, amount, customer."* The second building writes it in **their own book, in their own format**. They can keep only the sales section. They can index it differently. They can even keep their own additional columns. But dictation is slower than photocopying, and **you must agree in advance on how to identify a row you want changed or deleted** — otherwise "update the amount for that customer" is ambiguous.

★ **And the piece that governs both, and causes most incidents: the sender keeps a bookmark of how far the receiver has got.** If the receiver goes away and nobody removes the bookmark, the sender **keeps every page since that bookmark, forever**, until the disk fills.

---

## Where this fits in the big picture

```
   41 WAL — what is shipped · 42 recovery — how it's applied
   58 read replicas — using them, and the staleness that follows
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 62 REPLICATION IN DEPTH ← YOU ARE HERE       │
        │ ★ slots, logical decoding, topologies,       │
        │   and the failure modes                      │
        └────────────────────┬─────────────────────────┘
                             ▼
              63 HA & failover · 64 backup & PITR
              76 CDC · 68 CAP
```

Topic 58 was **how to use a replica**. This topic is **how replication actually works, what can go wrong with it, and how to choose between the two kinds** — which is the difference between operating one and being operated by one.

---

## What is this?

Two fundamentally different mechanisms that people call by one name.

```
 ★ PHYSICAL (STREAMING) REPLICATION
   ships the WAL byte stream and replays it into identical pages.
   ⇒ the replica is a BLOCK-LEVEL CLONE.

 ★ LOGICAL REPLICATION
   DECODES the WAL into row-level change events, ships those, and
   the subscriber APPLIES them as ordinary INSERT/UPDATE/DELETE.
   ⇒ the subscriber is an INDEPENDENT DATABASE that happens to
     receive changes.
```

| | Physical | Logical |
|---|---|---|
| Unit | ★ 8 KB pages | ★ rows |
| Scope | ★ the entire cluster | selected tables |
| Schema | ★ must be identical | may differ |
| Version | ★ must match | ★ can cross major versions |
| Subscriber writable | ★ no | ★ yes |
| Indexes | identical | ★ can differ |
| Sequences | ★ replicated | ★ **not replicated** |
| DDL | ★ replicated | ★ **not replicated** |
| Overhead | ★ very low | ★ decode + apply, higher |
| Lag driver | ★ WAL volume | ★ WAL volume + apply cost |
| Use for | ★ HA, read scaling | ★ CDC, upgrades, selective copies |

★ **The two lines to internalise:** logical replication does **not** replicate DDL or sequences. Both are silent, and both cause incidents.

---

## Why does it matter for a backend developer?

```
 ★ THREE REASONS.

 ① ★ REPLICATION SLOTS ARE THE #1 CAUSE OF "THE DISK IS FULL".
    an inactive slot retains WAL FOREVER and pins xmin FOREVER.
    ⇒ Topic 41's pg_wal disk-full PANIC
    ⇒ Topic 47's 107-day vacuum failure
    ⇒ ★ both were slots. It is the same object every time.

 ② ★ LOGICAL REPLICATION IS HOW YOU UPGRADE POSTGRESQL WITH
    NEAR-ZERO DOWNTIME, and how CDC works. If you don't understand
    it, major-version upgrades become multi-hour outages.

 ③ ★ THE FAILURE MODES ARE SILENT.
    • a subscriber that stops applying: ★ no error, just growing lag
    • a missing REPLICA IDENTITY: ★ UPDATEs fail on the subscriber
      with a message nobody recognises
    • a schema change on the publisher: ★ the subscription breaks
      and stays broken
    ⇒ ★ NONE OF THESE PAGE YOU UNLESS YOU BUILT THE ALERT.
```

---

## The physical reality

### Replication slots — what they actually guarantee, and cost

```
 ★ A SLOT IS A PERSISTENT BOOKMARK ON THE PRIMARY.

 WITHOUT A SLOT:
   the primary keeps WAL according to wal_keep_size and
   max_wal_size. A replica that falls behind past that boundary
   gets:
     ERROR: requested WAL segment … has already been removed
   ⇒ ★ the replica is BROKEN and must be rebuilt from a new
     base backup.

 WITH A SLOT:
   the primary ★ REFUSES TO REMOVE WAL the slot hasn't confirmed.
   ⇒ ✓ the replica can always catch up
   ⇒ ★ ✗ AND IF IT NEVER DOES, THE DISK FILLS.

 ★ WHAT A SLOT HOLDS:
   restart_lsn    — the oldest WAL the slot still needs
                    ⇒ ★ WAL below this cannot be removed
   confirmed_flush_lsn — (logical) what the subscriber has
                    confirmed
   xmin / catalog_xmin — the oldest transaction the slot needs
                    ⇒ ★ VACUUM cannot remove tuples newer than
                      this, CLUSTER-WIDE (Topic 47)

   SELECT slot_name, slot_type, active, active_pid,
          restart_lsn, confirmed_flush_lsn, xmin, catalog_xmin,
          pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(),
                                         restart_lsn)) AS retained,
          wal_status, safe_wal_size
     FROM pg_replication_slots;

 ★ wal_status IS THE FIELD TO WATCH (PG 13+):
   reserved   — within max_wal_size, fine
   extended   — beyond max_wal_size but within max_slot_wal_keep_size
   ★ unreserved — WAL is at risk of removal
   ★ lost      — WAL has been removed; ★ THE SLOT IS DEAD and the
                 replica must be rebuilt

 ★ THE GUARD, AND IT SHOULD BE SET ON EVERY CLUSTER:
   max_slot_wal_keep_size = '128GB'
   ⇒ a runaway slot is INVALIDATED instead of filling the disk.
   ⇒ ★ THE TRADE: you lose that replica rather than the primary.
     ★ THAT IS ALWAYS THE RIGHT TRADE.
```

### Logical decoding — how a row change is reconstructed from WAL

```
 ★ THE WAL DOES NOT CONTAIN ROWS. It contains PAGE-LEVEL changes:
   "on page 42, at offset 7, write these bytes."

 ⇒ TO PRODUCE "UPDATE orders SET status='paid' WHERE id=8842",
   the decoder must:
   ① read the WAL record and identify the relation OID
   ② ★ look up the table's schema IN THE SYSTEM CATALOGS —
      as they were AT THE TIME OF THE CHANGE
   ③ decode the tuple bytes into typed column values
   ④ ★ determine WHICH ROW was changed — this needs the
      REPLICA IDENTITY (below)
   ⑤ buffer the change until the transaction COMMITS
      ⇒ ★ logical replication is TRANSACTIONALLY ORDERED: nothing
        is sent until commit (unless streaming is enabled, PG14+)
   ⑥ emit it via an output plugin (pgoutput, wal2json, …)

 ★ STEP ② IS WHY catalog_xmin EXISTS AND WHY IT MATTERS:
   the decoder needs OLD catalog rows to interpret OLD WAL.
   ⇒ ★ a logical slot pins catalog_xmin, which blocks VACUUM on
     the SYSTEM CATALOGS. An abandoned logical slot therefore
     bloats pg_attribute, pg_class and friends — a failure mode
     that looks nothing like "a slot is stuck".

 ★ STEP ⑤ HAS A CONSEQUENCE PEOPLE HIT:
   a single 40-million-row transaction is buffered ENTIRELY before
   anything is sent.
   ⇒ ★ logical_decoding_work_mem (64 MB) is exceeded ⇒ spilled to
     disk in pg_replslot/ ⇒ ★ enormous latency spike and disk use.
   ⇒ ★ FIX: PG 14+ can stream in-progress transactions
     (streaming = on), and/or ★ batch your bulk operations.
```

### `REPLICA IDENTITY` — the setting that silently breaks `UPDATE`

```
 ★ TO REPLICATE "UPDATE … WHERE …", THE SUBSCRIBER MUST BE ABLE
   TO FIND THE ROW. The WAL's UPDATE record contains the NEW tuple;
   the OLD key must be logged separately.

 ★ FOUR MODES:
   DEFAULT   — log the PRIMARY KEY columns of the old row
               ⇒ ★ the default, and correct for most tables
   USING INDEX <name> — log the columns of a chosen UNIQUE,
               NOT NULL, non-partial index
   FULL      — ★ log the ENTIRE old row
               ⇒ works without any key, ★ but the subscriber does
                 a SEQUENTIAL SCAN per change if it has no index
               ⇒ ★ and WAL volume rises substantially
   NOTHING   — log nothing
               ⇒ ★ UPDATE and DELETE CANNOT be replicated

 ★ THE FAILURE, WHICH IS UNMISTAKABLE ONCE YOU'VE SEEN IT:
   a table with NO primary key, published, and then updated:
     ERROR:  cannot update table "events" because it does not have
             a replica identity and publishes updates
     HINT:  To enable updating the table, set REPLICA IDENTITY
            using ALTER TABLE.
   ⇒ ★ THIS FIRES ON THE PUBLISHER, ON THE UPDATE ITSELF.
     Adding a table to a publication can make previously-working
     writes start failing.

   ALTER TABLE events REPLICA IDENTITY FULL;
   -- or, better:
   ALTER TABLE events ADD PRIMARY KEY (id);

 ★ AND THE SLOW ONE:
   REPLICA IDENTITY FULL + no index on the subscriber
   ⇒ ★ every UPDATE is a seq scan on the subscriber.
   ⇒ MEASURED: 41,000 updates/sec on the publisher, ★ 12/sec
     applied on the subscriber. Lag grows without bound and
     nothing errors.
```

### What logical replication does *not* replicate

```
 ★ FOUR THINGS, ALL SILENT, ALL CAUSE INCIDENTS.

 ① ★ DDL
    ALTER TABLE orders ADD COLUMN discount_minor bigint;
    ⇒ the publisher has it; ★ the subscriber does not.
    ⇒ the next INSERT replicating that column fails:
      ERROR: logical replication target relation "public.orders"
             is missing replicated column "discount_minor"
    ⇒ ★ THE SUBSCRIPTION STOPS AND STAYS STOPPED. Lag grows.
    ⇒ ★ THE PROCEDURE: apply DDL to the SUBSCRIBER FIRST for
      additive changes, and to the PUBLISHER first for drops.
      (The expand/contract pattern — Topic 28 — applies here too.)

 ② ★ SEQUENCES
    nextval() is not replicated. After a failover to a logical
    subscriber, ★ every sequence restarts from its initial value
    ⇒ duplicate key violations everywhere.
    ⇒ ★ THE PROCEDURE: before cutover, advance every sequence on
      the subscriber:
      SELECT setval(seqname, (SELECT max(id) FROM tbl) + 1000);

 ③ ★ TRUNCATE — replicated only since PG 11, and only if the
    publication includes it (it does by default).

 ④ ★ LARGE OBJECTS (pg_largeobject) — never replicated.
```

### Cascading and topologies

```
 ★ CASCADING PHYSICAL REPLICATION
   primary → replica A → replica B
   ⇒ ★ replica A serves WAL to B, reducing load on the primary
   ⇒ ★ B's lag = A's lag + B's own lag. It compounds.
   ⇒ ★ AND: if A is promoted, B follows A's new timeline
     automatically only if configured with a slot on A.

 ★ BI-DIRECTIONAL / MULTI-MASTER
   ⇒ ★ POSTGRESQL DOES NOT SUPPORT THIS NATIVELY IN A SAFE FORM.
   Logical replication can be set up bidirectionally, but you then
   own:
     • ★ CONFLICT RESOLUTION (two nodes update the same row)
     • ★ LOOP PREVENTION (change ping-pong)
     • ★ sequence collision
   ⇒ PG 16+ adds `origin = none` to avoid loops, which makes it
     possible but ★ NOT SAFE BY DEFAULT.
   ⇒ ★ IF YOU THINK YOU NEED MULTI-MASTER, YOU ALMOST CERTAINLY
     NEED EITHER SHARDING (Topic 60) OR A SINGLE WRITER WITH
     REGIONAL READ REPLICAS.

 ★ SYNCHRONOUS TOPOLOGIES (Topic 58)
   synchronous_standby_names = 'ANY 1 (r1, r2, r3)'
   ⇒ ★ never a single name — losing it blocks ALL commits
   ⇒ FIRST 1 (r1, r2) = a priority list; ANY 1 = a quorum
```

---

## How it works — step by step

### Setting up logical replication correctly

```sql
-- ★ ON THE PUBLISHER
ALTER SYSTEM SET wal_level = 'logical';        -- ★ requires a restart
ALTER SYSTEM SET max_replication_slots = 20;
ALTER SYSTEM SET max_wal_senders = 20;
ALTER SYSTEM SET max_slot_wal_keep_size = '128GB';   -- ★ the guard

-- ★ every published table needs a replica identity
SELECT c.relname,
       CASE c.relreplident WHEN 'd' THEN 'default (PK)'
                           WHEN 'n' THEN '★ NOTHING'
                           WHEN 'f' THEN '★ FULL'
                           WHEN 'i' THEN 'index' END AS replica_identity,
       (SELECT count(*) FROM pg_index i
         WHERE i.indrelid = c.oid AND i.indisprimary) AS has_pk
  FROM pg_class c JOIN pg_namespace n ON n.oid = c.relnamespace
 WHERE c.relkind = 'r' AND n.nspname = 'public';
-- ★ any table with 'default' and has_pk = 0 will FAIL on UPDATE.

CREATE PUBLICATION orders_pub FOR TABLE orders, order_items;
-- or: FOR ALL TABLES;  ⇒ ★ dangerous — new tables join silently
```

```sql
-- ★ ON THE SUBSCRIBER — the schema must exist FIRST
--   logical replication does NOT create tables.
pg_dump -h publisher -s -t orders -t order_items shop | psql -h subscriber shop

CREATE SUBSCRIPTION orders_sub
  CONNECTION 'host=publisher dbname=shop user=repl password=…'
  PUBLICATION orders_pub
  WITH (copy_data = true,          -- ★ initial snapshot
        streaming = on,            -- ★ PG14+: stream big transactions
        binary = on);              -- ★ PG14+: faster, but types must match
```

```sql
-- ★ WATCH THE INITIAL SYNC — it is a separate phase per table
SELECT srrelid::regclass AS table, srsubstate,
       CASE srsubstate WHEN 'i' THEN 'initialising'
                       WHEN 'd' THEN '★ copying data'
                       WHEN 'f' THEN 'finished copy'
                       WHEN 's' THEN 'synchronised'
                       WHEN 'r' THEN '★ ready (streaming)' END AS state
  FROM pg_subscription_rel;
```

### Monitoring — the queries that matter

```sql
-- ★ ① ON THE PUBLISHER — slots and their retention
SELECT slot_name, slot_type, active, wal_status,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
         AS retained_wal,
       pg_size_pretty(safe_wal_size) AS headroom,
       age(xmin) AS xmin_age, age(catalog_xmin) AS catalog_xmin_age
  FROM pg_replication_slots ORDER BY 5 DESC;
-- ★ ALERT: active = false · wal_status <> 'reserved' ·
--          retained > 10 GB · xmin_age > 100M

-- ★ ② ON THE PUBLISHER — the sender side
SELECT application_name, state, sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), sent_lsn)   AS pending_bytes,
       pg_wal_lsn_diff(sent_lsn, replay_lsn)             AS in_flight_bytes,
       write_lag, flush_lag, replay_lag
  FROM pg_stat_replication;

-- ★ ③ ON THE SUBSCRIBER — is it actually applying?
SELECT subname, pid, received_lsn, latest_end_lsn, latest_end_time,
       extract(epoch from now() - latest_end_time) AS seconds_since_data
  FROM pg_stat_subscription;
-- ★ pid IS NULL ⇒ the worker is NOT RUNNING. This is the silent one.

-- ★ ④ ON THE SUBSCRIBER — errors (PG 15+)
SELECT subname, relid::regclass, command, xid, last_error_message,
       stats_reset FROM pg_stat_subscription_stats;
-- ★ apply_error_count > 0 ⇒ the subscription is stuck.
```

### Diagnosing a stuck subscription

```
 ★ THE SEQUENCE, IN ORDER:

 ① IS THE WORKER RUNNING?
    SELECT subname, pid FROM pg_stat_subscription;
    ⇒ ★ pid IS NULL means it crashed and is restarting in a loop.

 ② WHAT IS THE ERROR?
    ⇒ PG 15+: pg_stat_subscription_stats.last_error_message
    ⇒ earlier: ★ THE SUBSCRIBER'S SERVER LOG. There is nowhere else.
    grep 'logical replication' /var/log/postgresql/*.log | tail -20

 ③ THE THREE COMMON ERRORS:
    ★ "is missing replicated column"       ⇒ DDL drift (fix the schema)
    ★ "duplicate key value violates …"     ⇒ the subscriber has a
       row the publisher is re-sending, usually after a manual write
       to the subscriber
    ★ "cannot update table … replica identity" ⇒ on the PUBLISHER

 ④ ★ SKIPPING A BAD TRANSACTION (PG 15+) — the escape hatch
    ALTER SUBSCRIPTION orders_sub SKIP (lsn = '0/1A2B3C4D');
    ⇒ ★ THIS DISCARDS THAT TRANSACTION ENTIRELY. The subscriber
      is now permanently inconsistent for those rows.
    ⇒ ★ ALWAYS follow with a targeted resync:
      ALTER SUBSCRIPTION orders_sub REFRESH PUBLICATION
        WITH (copy_data = true);
      -- or re-copy one table:
      ALTER SUBSCRIPTION s SET (slot_name = NONE); -- then recreate
```

---

## Concept breakdown

```
TWO MECHANISMS
├── ★ PHYSICAL   ships WAL bytes, replays into identical pages
│    whole cluster · same version · read-only · ★ low overhead
│    ⇒ HA, read scaling, standbys
└── ★ LOGICAL    decodes WAL into row events, applies as SQL
     selected tables · ★ cross-version · ★ writable subscriber
     ⇒ CDC, near-zero-downtime upgrades, selective copies

★ WHAT LOGICAL DOES NOT REPLICATE — all silent
   ① ★ DDL       ⇒ the subscription STOPS and stays stopped
   ② ★ SEQUENCES ⇒ duplicate keys after a cutover
   ③ TRUNCATE (only PG11+)      ④ large objects

★ REPLICATION SLOTS — the #1 operational hazard
   restart_lsn      ⇒ WAL below it CANNOT be removed
   xmin/catalog_xmin ⇒ ★ VACUUM blocked CLUSTER-WIDE
   ★ wal_status: reserved → extended → unreserved → ★ lost
   ★ max_slot_wal_keep_size = the guard. Lose the replica, not
     the primary.
   ⇒ ★ an inactive slot caused BOTH Topic 41's disk-full PANIC
     AND Topic 47's 107-day vacuum failure.

★ LOGICAL DECODING MECHANICS
   WAL has PAGES, not rows ⇒ the decoder needs the CATALOGS as they
   were AT THE TIME ⇒ ★ catalog_xmin pins system-catalog vacuum
   ⇒ ★ an abandoned logical slot bloats pg_attribute/pg_class
   ★ changes are buffered until COMMIT ⇒ a huge transaction spills
     to disk (logical_decoding_work_mem) ⇒ ★ use streaming = on

★ REPLICA IDENTITY — how the subscriber finds the row
   DEFAULT (PK) · USING INDEX · ★ FULL (whole row) · ★ NOTHING
   ⇒ ★ no PK + published ⇒ UPDATE FAILS ON THE PUBLISHER
   ⇒ ★ FULL + no subscriber index ⇒ a SEQ SCAN PER CHANGE
     (41,000/s published, ★ 12/s applied)

TOPOLOGIES
├── cascading  primary → A → B ⇒ ★ lag compounds
├── ★ sync     'ANY 1 (r1,r2,r3)' — ★ never a single name
└── ★ multi-master — NOT safe by default. You own conflict
     resolution, loop prevention and sequence collisions.
     ⇒ ★ you probably want sharding (60) or one writer + regional
       read replicas.

★ THE FOUR ALERTS
   inactive slot · wal_status ≠ reserved · ★ subscription pid IS NULL
   · apply_error_count > 0
```

---

## Diagrams

**Diagram 1 — big picture: physical vs logical, end to end**

```
 ★ PHYSICAL — the WAL byte stream, replayed into identical pages
 ┌─────────────────────────┐              ┌─────────────────────────┐
 │ PRIMARY                 │              │ REPLICA                 │
 │  backend writes tuple   │              │                         │
 │   ↓                     │              │                         │
 │  WAL record: "page 42,  │  ─────────►  │  startup process:       │
 │   offset 7, bytes …"    │   walsender  │   apply to page 42      │
 │   ↓                     │              │   ↓                     │
 │  ★ page 42 dirtied      │              │  ★ page 42 — IDENTICAL  │
 └─────────────────────────┘              └─────────────────────────┘
   ★ same version · same schema · whole cluster · read-only
   ★ overhead: near zero — it is a memcpy and a page write

 ★ LOGICAL — decode to rows, apply as SQL
 ┌─────────────────────────┐              ┌─────────────────────────┐
 │ PRIMARY (publisher)     │              │ SUBSCRIBER              │
 │  WAL record: "page 42…" │              │                         │
 │   ↓                     │              │                         │
 │  ★ LOGICAL DECODER      │              │                         │
 │   ① relation OID → ★ look up the       │                         │
 │      CATALOG AS IT WAS  │              │                         │
 │   ② decode tuple bytes  │              │                         │
 │   ③ ★ old key from      │              │                         │
 │      REPLICA IDENTITY   │              │                         │
 │   ④ ★ buffer until      │              │                         │
 │      COMMIT             │              │                         │
 │   ↓ pgoutput            │  ─────────►  │  apply worker:          │
 │  {UPDATE orders         │              │   ★ UPDATE orders       │
 │   SET status='paid'     │              │     SET status='paid'   │
 │   WHERE id=8842}        │              │    WHERE id=8842        │
 └─────────────────────────┘              └─────────────────────────┘
   ★ cross-version · selected tables · ★ writable · own indexes
   ★ overhead: decode + a real SQL apply — much higher
   ★ ✗ no DDL · ✗ no sequences
```

**Diagram 2 — data flow: how a slot fills the disk**

```
  DAY 0 — healthy
   primary WAL  ▓▓▓▓▓▓▓▓▓░░░░░  restart_lsn ──┐
                          ▲                    │ 240 MB retained
                          └── slot bookmark ───┘
   ⇒ WAL below the bookmark is recycled normally.

  DAY 1 — the replica is decommissioned; ★ the slot is not dropped
   primary WAL  ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓░░░░
                          ▲
                          └── ★ the bookmark NEVER MOVES
   ⇒ nothing below it can be removed.
   ⇒ ★ AND xmin is pinned ⇒ VACUUM reclaims nothing, cluster-wide.

  DAY 14
   pg_wal  ████████████████████████████████████  ★ 812 GB
   xmin    ★ frozen 14 days ago
   ⇒ ★ n_dead_tup in the hundreds of millions
   ⇒ ★ age(datfrozenxid) marching toward wraparound (Topic 47)

  DAY 21
   ★ PANIC: could not write to file "pg_wal/…": No space left on device
   ★ THE PRIMARY SHUTS DOWN.

 ✓ WITH THE GUARD
   max_slot_wal_keep_size = '128GB'
   ⇒ at 128 GB the slot's wal_status becomes ★ 'lost'
   ⇒ the replica is dead and must be rebuilt
   ⇒ ★ THE PRIMARY SURVIVES. Always the right trade.
```

**Diagram 3 — before/after: the major-version upgrade**

```
 ✗ pg_upgrade — the traditional route
 ┌───────────────────────────────────────────────────────────────┐
 │ ① stop the primary                    ★ DOWNTIME STARTS       │
 │ ② pg_upgrade --link                     4–20 minutes          │
 │ ③ ★ REBUILD STATISTICS (ANALYZE)        ★ 40+ minutes on 4 TB │
 │    — the database is UP but every plan is wrong until done     │
 │ ④ rebuild every replica from scratch    ★ hours                │
 │                                                                │
 │ ★ TOTAL USER-VISIBLE IMPACT: 45–90 minutes                    │
 │ ★ ROLLBACK: restore from backup. Hours.                       │
 └───────────────────────────────────────────────────────────────┘

 ✓ LOGICAL REPLICATION — near-zero downtime
 ┌───────────────────────────────────────────────────────────────┐
 │ ① build a PG 18 server alongside the PG 16 primary            │
 │ ② copy the schema (pg_dump -s)                                │
 │ ③ CREATE SUBSCRIPTION … WITH (copy_data = true)               │
 │    ⇒ ★ initial sync runs for hours, ONLINE, no impact         │
 │ ④ wait until lag ≈ 0                                          │
 │ ⑤ ★ ANALYZE on the new server — done BEFORE cutover           │
 │ ⑥ ★ CUTOVER:                                                  │
 │      • stop writes (a brief read-only window)                  │
 │      • wait for lag = 0                                        │
 │      • ★ ADVANCE ALL SEQUENCES — logical does NOT replicate    │
 │        them, and forgetting this is the classic failure        │
 │      • repoint the application                                 │
 │    ⇒ ★ USER-VISIBLE IMPACT: 20–60 SECONDS                     │
 │ ⑦ ★ ROLLBACK: repoint back. The old primary is untouched      │
 │    and still current if you set up reverse replication.        │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**Physical replication with a slot.**
```bash
# primary
psql -c "ALTER SYSTEM SET wal_level='replica';
         ALTER SYSTEM SET max_wal_senders=10;
         ALTER SYSTEM SET max_replication_slots=10;
         ALTER SYSTEM SET max_slot_wal_keep_size='64GB';"   # ★ the guard
pg_ctl restart
psql -c "SELECT pg_create_physical_replication_slot('replica1');"

# replica
pg_basebackup -h primary -U repl -D "$PGDATA" -Fp -Xs -P -R -S replica1
pg_ctl start -D "$PGDATA"
```

**Watch the slot.**
```sql
SELECT slot_name, slot_type, active, active_pid, wal_status,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
         AS retained,
       pg_size_pretty(safe_wal_size) AS headroom
  FROM pg_replication_slots;
```
```
 slot_name | slot_type | active | wal_status | retained | headroom
-----------+-----------+--------+------------+----------+----------
 replica1  | physical  | t      | reserved   | 1,184 kB | 63 GB
```

**Prove a stopped replica accumulates WAL.**
```bash
pg_ctl stop -D "$REPLICA_PGDATA"
# generate WAL on the primary
pgbench -c 20 -j 4 -T 60 shop
psql -c "SELECT slot_name, active, wal_status,
         pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
           AS retained FROM pg_replication_slots;"
```
```
 slot_name | active | wal_status |  retained
-----------+--------+------------+------------
 replica1  | ★ f    | reserved   | ★ 4,184 MB
   ★ the primary is holding 4 GB of WAL for a server that is off.
```
```sql
-- ★ and it pins xmin too
UPDATE orders SET total_minor = total_minor WHERE id < 500000;
VACUUM (VERBOSE) orders;
```
```
DETAIL:  ★ 499999 dead row versions cannot be removed yet,
         oldest xmin: 8842119
```

**Prove the guard works.**
```sql
ALTER SYSTEM SET max_slot_wal_keep_size = '100MB';
SELECT pg_reload_conf();
-- generate more WAL
```
```sql
SELECT slot_name, active, wal_status, safe_wal_size FROM pg_replication_slots;
```
```
 slot_name | active | wal_status | safe_wal_size
-----------+--------+------------+---------------
 replica1  | f      | ★ lost     |          ★ -1
   ★ the slot is invalidated. The replica must be rebuilt.
   ★ THE PRIMARY IS FINE. That is the trade, and it is correct.
```

**Logical replication end to end.**
```sql
-- publisher
ALTER SYSTEM SET wal_level = 'logical';   -- ★ restart required
CREATE TABLE orders (
  id bigserial PRIMARY KEY,
  customer_id bigint NOT NULL,
  status text NOT NULL,
  total_minor bigint NOT NULL);
INSERT INTO orders (customer_id, status, total_minor)
  SELECT g, 'pending', g*100 FROM generate_series(1,1000) g;

CREATE PUBLICATION orders_pub FOR TABLE orders;
```
```bash
# subscriber — ★ the schema must exist first
pg_dump -h publisher -s -t orders shop | psql -h subscriber shop
```
```sql
-- subscriber
CREATE SUBSCRIPTION orders_sub
  CONNECTION 'host=publisher dbname=shop user=repl'
  PUBLICATION orders_pub
  WITH (copy_data = true, streaming = on);
```
```sql
SELECT srrelid::regclass, srsubstate FROM pg_subscription_rel;
```
```
 srrelid | srsubstate
---------+------------
 orders  | ★ r          — ready, streaming
```
```sql
SELECT count(*) FROM orders;   -- ★ 1000, copied
```

**Prove DDL is not replicated.**
```sql
-- publisher
ALTER TABLE orders ADD COLUMN discount_minor bigint DEFAULT 0;
INSERT INTO orders (customer_id, status, total_minor, discount_minor)
  VALUES (9999, 'pending', 5000, 500);
```
```sql
-- subscriber
SELECT count(*) FROM orders;
```
```
 count
-------
 ★ 1000        — the new row never arrived
```
```bash
tail -5 /var/log/postgresql/postgresql.log
```
```
ERROR:  logical replication target relation "public.orders" is missing
        replicated column "discount_minor"
   ★ THE SUBSCRIPTION IS STUCK. It will retry forever and never succeed.
```
```sql
-- ★ the fix, and the ordering rule
-- subscriber FIRST for additive changes:
ALTER TABLE orders ADD COLUMN discount_minor bigint DEFAULT 0;
-- ⇒ the worker retries and catches up automatically
SELECT count(*) FROM orders;   -- ★ 1001
```

**Prove sequences are not replicated.**
```sql
-- publisher
SELECT last_value FROM orders_id_seq;
```
```
 last_value
------------
     ★ 1001
```
```sql
-- subscriber
SELECT last_value FROM orders_id_seq;
```
```
 last_value
------------
        ★ 1        — ★ after a cutover, every INSERT would collide
```
```sql
-- ★ the cutover step people forget
SELECT setval('orders_id_seq', (SELECT max(id) FROM orders) + 1000);
```

**Prove `REPLICA IDENTITY` breaks `UPDATE`.**
```sql
-- publisher
CREATE TABLE events (id bigint, payload text);   -- ★ no primary key
INSERT INTO events VALUES (1, 'a');
ALTER PUBLICATION orders_pub ADD TABLE events;

UPDATE events SET payload = 'b' WHERE id = 1;
```
```
ERROR:  cannot update table "events" because it does not have a
        replica identity and publishes updates
HINT:  To enable updating the table, set REPLICA IDENTITY using
       ALTER TABLE.
   ★ ADDING A TABLE TO A PUBLICATION MADE A PREVIOUSLY-WORKING
     UPDATE START FAILING.
```
```sql
ALTER TABLE events REPLICA IDENTITY FULL;
UPDATE events SET payload = 'b' WHERE id = 1;   -- ★ now works
```

**And prove `FULL` without a subscriber index is catastrophic.**
```sql
-- publisher: 1M rows, REPLICA IDENTITY FULL, no PK
-- subscriber: same table, ★ no index
UPDATE events SET payload = payload || 'x';    -- 1M updates
```
```sql
-- subscriber, during apply
SELECT query, state, now()-query_start AS running FROM pg_stat_activity
 WHERE backend_type = 'logical replication worker';
```
```
        query         | state  |   running
----------------------+--------+-------------
 UPDATE events SET …  | active | 00:04:12
```
```sql
EXPLAIN SELECT * FROM events WHERE id=1 AND payload='a';
```
```
 ★ Seq Scan on events  (cost=0.00..18334.00 rows=1)
   ★ EVERY replicated UPDATE is a full scan.
     41,000/s published ⇒ ★ 12/s applied.
```
```sql
-- ★ the fix
CREATE INDEX ON events (id);        -- on the SUBSCRIBER
-- ⇒ apply rate 12/s → ★ 38,000/s
```

**Monitor the subscription.**
```sql
SELECT subname, pid, received_lsn, latest_end_lsn,
       extract(epoch from now() - latest_end_time)::int AS seconds_idle
  FROM pg_stat_subscription;
```
```
   subname   |  pid  | seconds_idle
-------------+-------+--------------
 orders_sub  | 41202 |            0
-- ★ pid IS NULL would mean the worker is dead. THE key alert.
```
```sql
-- PG 15+
SELECT subname, apply_error_count, sync_error_count, last_error_message
  FROM pg_stat_subscription_stats;
```

---

## Example 2 — production scenario

**The situation.** A retail platform on PostgreSQL 14, 4.2 TB, must upgrade to 18 before 14 goes end-of-life. The business allows **five minutes** of write downtime.

```
 THE CONSTRAINTS
   database size          ★ 4.2 TB
   tables                 ★ 1,840
   sequences              ★ 412
   largest table          ★ 1.8 TB (orders, partitioned)
   write rate             ★ 18,000/sec peak
   allowed downtime       ★ 5 minutes
   pg_upgrade estimate    ★ 45–90 min (mostly post-upgrade ANALYZE)
```

**Step 1 — establish why `pg_upgrade` doesn't fit.**

```bash
# a dry run on a restored copy
pg_upgrade --check -d /old -D /new -b /usr/lib/postgresql/14/bin \
                                   -B /usr/lib/postgresql/18/bin
```
```
 Performing Consistency Checks
 ...
 *Clusters are compatible*
```
```bash
time pg_upgrade --link -d /old -D /new -b … -B …
```
```
 real  ★ 4m18s        — the upgrade itself is fast
```
```sql
-- ★ but then:
\timing on
ANALYZE;
```
```
 Time: ★ 2,884,201 ms      (48 minutes)
```
```
 ★ THE UPGRADE IS 4 MINUTES. THE STATISTICS REBUILD IS 48.
   And during those 48 minutes the database is UP and every plan
   is wrong — measured p99 during a partial ANALYZE: ★ 41,000 ms.
 ⇒ ★ pg_upgrade's "downtime" number is misleading. The
   user-visible impact is the ANALYZE, not the upgrade.
 ⇒ AND: every replica must be rebuilt from scratch afterwards.
```

**Step 2 — the logical replication plan, and its obstacles.**

```sql
-- ★ obstacle 1: which tables lack a replica identity?
SELECT n.nspname, c.relname,
       CASE c.relreplident WHEN 'd' THEN 'default' WHEN 'n' THEN 'NOTHING'
                           WHEN 'f' THEN 'FULL' WHEN 'i' THEN 'index' END AS ri,
       EXISTS (SELECT 1 FROM pg_index i
                WHERE i.indrelid=c.oid AND i.indisprimary) AS has_pk
  FROM pg_class c JOIN pg_namespace n ON n.oid=c.relnamespace
 WHERE c.relkind='r' AND n.nspname NOT IN ('pg_catalog','information_schema')
   AND NOT EXISTS (SELECT 1 FROM pg_index i
                    WHERE i.indrelid=c.oid AND i.indisprimary);
```
```
 nspname |      relname       |   ri    | has_pk
---------+--------------------+---------+--------
 public  | ★ event_log        | default | f
 public  | ★ import_staging   | default | f
 public  | ★ session_activity | default | f
   ★ THREE TABLES WOULD FAIL ON UPDATE the moment they were
     published. Two are append-only (never updated) — safe.
     session_activity IS updated ⇒ ★ must be fixed first.
```
```sql
ALTER TABLE session_activity ADD COLUMN id bigserial;
ALTER TABLE session_activity ADD PRIMARY KEY (id);
-- ★ done weeks in advance, with CONCURRENTLY where possible
```

```
 ★ OBSTACLE 2: 1.8 TB of partitioned `orders`.
   PG 13+ supports publishing a partitioned table with
   publish_via_partition_root, but ★ the initial COPY of 1.8 TB
   over the network took 14 hours in testing.
 ⇒ ★ THE MITIGATION: split the subscription.
   • one subscription for `orders` alone, started FIRST
   • a second for everything else
   ⇒ they sync in parallel and a failure in one doesn't restart
     the other.
```

```
 ★ OBSTACLE 3: 412 sequences, none replicated.
   ⇒ a script, tested on the rehearsal, that advances every one.
```

**Step 3 — build it.**

```sql
-- ★ ON THE PUBLISHER (PG 14)
ALTER SYSTEM SET wal_level = 'logical';
ALTER SYSTEM SET max_replication_slots = 20;
ALTER SYSTEM SET max_wal_senders = 20;
ALTER SYSTEM SET max_slot_wal_keep_size = '512GB';   -- ★ generous but bounded
ALTER SYSTEM SET logical_decoding_work_mem = '256MB'; -- ★ fewer spills
-- restart

CREATE PUBLICATION pub_orders FOR TABLE orders
  WITH (publish_via_partition_root = true);
CREATE PUBLICATION pub_rest FOR ALL TABLES;   -- ★ then exclude orders
-- (in practice: an explicit table list, generated from the catalog,
--  because FOR ALL TABLES silently includes future tables)
```

```bash
# ★ schema-only dump to the new server
pg_dump -h pg14 -s --no-publications --no-subscriptions shop \
  | psql -h pg18 shop

# ★ CRITICAL: drop indexes on the subscriber BEFORE the initial copy,
#   recreate after. Copying 1.8 TB while maintaining 40 indexes is
#   3–5× slower.
psql -h pg18 -c "
  SELECT format('DROP INDEX %I;', indexname)
    FROM pg_indexes WHERE schemaname='public'
     AND indexname NOT IN (SELECT conname FROM pg_constraint
                            WHERE contype IN ('p','u'))" \
  | psql -h pg18
```

```sql
-- ★ ON THE SUBSCRIBER (PG 18)
ALTER SYSTEM SET max_logical_replication_workers = 8;
ALTER SYSTEM SET max_sync_workers_per_subscription = 4;  -- ★ parallel copy
ALTER SYSTEM SET maintenance_work_mem = '8GB';
ALTER SYSTEM SET max_wal_size = '64GB';                  -- ★ absorb the copy
-- ★ and, ONLY during the initial load:
ALTER SYSTEM SET synchronous_commit = 'off';   -- ★ revert before cutover
ALTER SYSTEM SET autovacuum = off;             -- ★ revert before cutover

CREATE SUBSCRIPTION sub_orders
  CONNECTION 'host=pg14 dbname=shop user=repl'
  PUBLICATION pub_orders
  WITH (copy_data = true, streaming = on, binary = on);

CREATE SUBSCRIPTION sub_rest
  CONNECTION 'host=pg14 dbname=shop user=repl'
  PUBLICATION pub_rest
  WITH (copy_data = true, streaming = on, binary = on);
```

**Step 4 — watch the initial sync.**

```sql
SELECT s.subname, r.srrelid::regclass AS tbl, r.srsubstate,
       pg_size_pretty(pg_total_relation_size(r.srrelid)) AS size
  FROM pg_subscription_rel r JOIN pg_subscription s ON s.oid = r.srsubid
 WHERE r.srsubstate <> 'r' ORDER BY pg_total_relation_size(r.srrelid) DESC;
```
```
  subname   |     tbl     | srsubstate |  size
------------+-------------+------------+---------
 sub_orders | orders_2026 | ★ d        | 412 GB
 sub_rest   | event_log   | d          |  88 GB
   ★ 'd' = copying. Watch this shrink to zero.
```
```sql
-- ★ and the publisher's slot retention during the copy
SELECT slot_name, wal_status,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
         AS retained FROM pg_replication_slots;
```
```
  slot_name  | wal_status |  retained
-------------+------------+------------
 sub_orders  | reserved   | ★ 184 GB
 sub_rest    | reserved   |  41 GB
   ★ THE COPY TAKES HOURS, AND WAL ACCUMULATES THE WHOLE TIME.
     max_slot_wal_keep_size = 512 GB was sized for exactly this.
   ★ MONITOR IT. If retention approaches the limit, the slot is
     invalidated and the whole copy restarts.
```

**Step 5 — after the copy, prepare for cutover.**

```sql
-- ★ ① recreate the indexes, in parallel
--    (a script running CREATE INDEX with 8 concurrent sessions)

-- ★ ② revert the load-time settings
ALTER SYSTEM SET synchronous_commit = 'on';
ALTER SYSTEM SET autovacuum = on;
SELECT pg_reload_conf();

-- ★ ③ ANALYZE — done BEFORE cutover, while nobody depends on it
ANALYZE;   -- ★ 48 minutes, entirely off the critical path

-- ★ ④ verify row counts, table by table
SELECT relname, n_live_tup FROM pg_stat_user_tables ORDER BY 1;
-- compared against the same query on pg14

-- ★ ⑤ verify checksums on the highest-value tables
SELECT md5(string_agg(id::text || status || total_minor::text, ',' ORDER BY id))
  FROM orders WHERE created_at >= '2026-08-01';
-- ★ must match on both sides
```

**Step 6 — the cutover.**

```bash
#!/usr/bin/env bash
set -euo pipefail
# ★ REHEARSED THREE TIMES ON A RESTORED COPY BEFORE THE REAL RUN.

# ① stop writes — the application enters read-only mode
curl -X POST https://api.internal/admin/readonly/enable
sleep 2

# ② ★ on the publisher, verify no in-flight write transactions
psql -h pg14 -c "SELECT count(*) FROM pg_stat_activity
                  WHERE state='idle in transaction' OR
                        (state='active' AND backend_type='client backend')"

# ③ ★ wait for lag to reach exactly zero
until [ "$(psql -h pg14 -tAc "
    SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), confirmed_flush_lsn)
      FROM pg_replication_slots WHERE slot_name='sub_orders'")" = "0" ]; do
  sleep 0.5
done

# ④ ★★ ADVANCE ALL 412 SEQUENCES — the step that fails cutovers
psql -h pg14 -tAc "
  SELECT format('SELECT setval(%L, %s);', s.seqname, s.last_value + 10000)
    FROM (SELECT schemaname||'.'||sequencename AS seqname,
                 last_value FROM pg_sequences) s" \
  | psql -h pg18

# ⑤ verify a sample
psql -h pg18 -c "SELECT sequencename, last_value FROM pg_sequences LIMIT 5"

# ⑥ ★ set up REVERSE replication before repointing — this is the
#    rollback path, and it must exist before you need it
psql -h pg18 -c "CREATE PUBLICATION rollback_pub FOR ALL TABLES;"
psql -h pg14 -c "CREATE SUBSCRIPTION rollback_sub
  CONNECTION 'host=pg18 dbname=shop user=repl' PUBLICATION rollback_pub
  WITH (copy_data = false, create_slot = true);"

# ⑦ repoint the application
kubectl set env deployment/api DATABASE_URL="$PG18_URL"
kubectl rollout status deployment/api --timeout=120s

# ⑧ re-enable writes
curl -X POST https://api.internal/admin/readonly/disable
```

```
 ★ MEASURED CUTOVER: 47 SECONDS.
   ① read-only          2 s
   ③ lag → 0            8 s
   ④ 412 sequences     11 s
   ⑥ reverse repl       4 s
   ⑦ rollout           19 s
   ⑧ writes on         3 s
```

**Step 7 — what went wrong in the rehearsals.**

```
 ★ REHEARSAL 1 — the slot filled the disk.
   The initial copy took 14 hours; peak write rate during that
   window produced 640 GB of WAL. max_slot_wal_keep_size was set
   to 256 GB.
   ⇒ ★ the slot was invalidated at hour 9 and the entire copy
     restarted.
   ⇒ FIX: raised to 512 GB AND ★ moved the copy to a low-traffic
     window AND ★ split into two subscriptions so a failure in one
     didn't restart the other.

 ★ REHEARSAL 2 — sequences were forgotten.
   The cutover script advanced sequences for `public` only.
   ⇒ ★ 41 sequences in other schemas were missed.
   ⇒ every INSERT into those tables failed with a duplicate key
     violation, 90 seconds after go-live.
   ⇒ FIX: generate the list from `pg_sequences` (all schemas), and
     ★ add a post-cutover verification query.

 ★ REHEARSAL 3 — a DDL change landed mid-sync.
   A routine migration added a column on PG 14 during the copy.
   ⇒ ★ the subscription stopped with "missing replicated column".
   ⇒ FIX: ★ a migration freeze for the duration of the sync, and
     an alert on `pg_stat_subscription_stats.apply_error_count`.

 ⇒ ★ ALL THREE FAILURES WERE FOUND IN REHEARSAL AND NONE
   OCCURRED IN PRODUCTION. THAT IS THE ENTIRE VALUE OF REHEARSING.
```

**Step 8 — results.**

| | `pg_upgrade` | Logical replication |
|---|---|---|
| Write downtime | 4 min | **47 seconds** |
| Degraded-plan window | ★ **48 min** | **0** (ANALYZE done in advance) |
| Total user impact | 45–90 min | **47 seconds** |
| Replica rebuild | required, hours | not required |
| Rollback | restore from backup, hours | ★ **repoint, seconds** |
| Preparation | 1 day | ★ **3 weeks + 3 rehearsals** |
| Extra hardware | none | ★ a full second cluster |

```
 ★ FIVE LESSONS:
 ① ★ pg_upgrade's headline number is misleading. The upgrade was
   4 minutes; the ANALYZE was 48, and during it every plan was
   wrong.
 ② ★ THE SLOT IS THE RISK DURING A LONG INITIAL COPY. 14 hours of
   copying accumulated 640 GB of WAL on the publisher. Size
   max_slot_wal_keep_size for the copy, not for steady state, and
   monitor retention continuously.
 ③ ★ SEQUENCES ARE THE CLASSIC CUTOVER FAILURE. Not replicated,
   silent, and the failure appears 90 seconds after go-live.
   Generate the list from the catalog, across all schemas, and
   verify afterwards.
 ④ ★ DDL DURING SYNC BREAKS EVERYTHING. Freeze migrations for the
   duration and alert on apply_error_count.
 ⑤ ★ SET UP REVERSE REPLICATION BEFORE REPOINTING. The rollback
   path must exist before you need it — and with it, rollback is
   a repoint rather than a restore.
```

---

## Common mistakes

**1. Creating a replication slot without `max_slot_wal_keep_size`.**
- *Symptom:* an inactive slot fills `pg_wal` and PANICs the primary; or pins `xmin` and stops VACUUM cluster-wide.
- *Fix:* always set the guard. Losing a replica is always better than losing the primary.

**2. Not alerting on inactive slots.**
- *Symptom:* a decommissioned replica's slot survives for months (Topics 41, 47).
- *Fix:* alert on `active = false` and on `wal_status <> 'reserved'`.

**3. Publishing a table with no primary key.**
- *Symptom:* `cannot update table … because it does not have a replica identity` — on the **publisher**, breaking previously-working writes.
- *Fix:* audit `relreplident` and primary keys *before* creating the publication.

**4. `REPLICA IDENTITY FULL` with no subscriber index.**
- *Symptom:* the subscriber applies 12 updates/sec against 41,000/sec published; lag grows without bound and nothing errors.
- *Fix:* index the identity columns on the subscriber. Better: add a real primary key.

**5. Forgetting that DDL is not replicated.**
- *Symptom:* the subscription stops with `missing replicated column` and stays stopped.
- *Fix:* additive changes on the subscriber first, drops on the publisher first; freeze migrations during initial sync; alert on `apply_error_count`.

**6. Forgetting sequences at cutover.**
- *Symptom:* duplicate key violations 90 seconds after go-live.
- *Fix:* generate the advance script from `pg_sequences` across **all** schemas, and verify afterwards.

**7. `CREATE PUBLICATION … FOR ALL TABLES` in production.**
- *Symptom:* new tables silently join the publication; a table without a PK breaks writes the day it's created.
- *Fix:* an explicit table list, generated and reviewed.

**8. Copying with indexes in place.**
- *Symptom:* the initial sync takes 3–5× longer, extending the window during which the slot accumulates WAL.
- *Fix:* drop non-constraint indexes on the subscriber before the copy; recreate in parallel after.

**9. Ignoring `logical_decoding_work_mem`.**
- *Symptom:* a huge transaction spills to `pg_replslot/`, causing a latency spike and disk usage.
- *Fix:* raise it, enable `streaming = on` (PG 14+), and batch bulk operations.

**10. Not monitoring `pg_stat_subscription.pid`.**
- *Symptom:* the apply worker crashed and is restarting in a loop; lag grows with no error anywhere visible.
- *Fix:* `pid IS NULL` is the alert.

**11. Using `ALTER SUBSCRIPTION … SKIP` without resyncing.**
- *Symptom:* the subscription resumes but the subscriber is permanently missing rows.
- *Fix:* always follow a skip with a targeted re-copy of the affected table.

**12. Attempting bidirectional replication casually.**
- *Symptom:* update loops, conflicting writes, sequence collisions.
- *Fix:* PostgreSQL does not make this safe by default. Use sharding or a single writer with regional read replicas.

---

## Hands-on proof

**PROVE IT #1–#9 — Example 1** (a physical slot retaining 4 GB for a stopped replica, the same slot pinning `xmin`, `max_slot_wal_keep_size` invalidating it with `wal_status = 'lost'`, logical replication end to end, DDL breaking the subscription and the subscriber-first fix, sequences at 1 vs 1001, `REPLICA IDENTITY` failing an `UPDATE` on the publisher, and `FULL` without an index at 12 applies/sec).

**PROVE IT #10 — logical decoding output, directly.**
```sql
SELECT pg_create_logical_replication_slot('peek', 'test_decoding');
UPDATE orders SET status = 'paid' WHERE id = 1;
SELECT lsn, xid, data FROM pg_logical_slot_peek_changes('peek', NULL, NULL);
```
```
    lsn     |  xid  |                        data
------------+-------+-----------------------------------------------------
 4A/8C001220| 88412 | BEGIN 88412
 4A/8C001258| 88412 | ★ table public.orders: UPDATE: id[bigint]:1
            |       |   customer_id[bigint]:1 status[text]:'paid' …
 4A/8C0012A0| 88412 | COMMIT 88412
   ★ this is exactly what the decoder produces. Note the BEGIN/COMMIT
     framing — changes are buffered until commit.
```
```sql
SELECT pg_drop_replication_slot('peek');   -- ★ ALWAYS clean up
```

**PROVE IT #11 — `catalog_xmin` blocks system-catalog vacuum.**
```sql
SELECT pg_create_logical_replication_slot('stuck', 'pgoutput');
-- nothing consumes it
SELECT slot_name, xmin, catalog_xmin, age(catalog_xmin) FROM pg_replication_slots;
```
```
 slot_name | xmin | catalog_xmin | age
-----------+------+--------------+------
 stuck     |      |    ★ 8842119 | 4,102
```
```sql
-- create catalog churn
DO $$ BEGIN FOR i IN 1..1000 LOOP
  EXECUTE format('CREATE TABLE t%s (id int); DROP TABLE t%s;', i, i);
END LOOP; END $$;
VACUUM (VERBOSE) pg_attribute;
```
```
DETAIL:  ★ N dead row versions cannot be removed yet, oldest xmin: 8842119
   ★ the logical slot is bloating the SYSTEM CATALOGS — a failure
     mode that looks nothing like "a slot is stuck".
```

**PROVE IT #12 — a large transaction spilling to disk.**
```sql
ALTER SYSTEM SET logical_decoding_work_mem = '1MB';
SELECT pg_reload_conf();
BEGIN;
INSERT INTO orders (customer_id, status, total_minor)
  SELECT g, 'pending', g FROM generate_series(1, 1000000) g;
COMMIT;
```
```bash
ls -la "$PGDATA/pg_replslot/orders_pub_slot/"
```
```
 -rw------- 1 postgres postgres ★ 412M xid-88412-lsn-4A-8C.spill
   ★ the entire transaction was spilled to disk before anything
     was sent. ⇒ raise logical_decoding_work_mem, enable
     streaming = on, or batch the insert.
```

**PROVE IT #13 — the four monitoring queries, as one dashboard.**
```sql
-- publisher
SELECT 'slot' AS kind, slot_name AS name, active::text AS ok,
       wal_status AS detail,
       pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS bytes
  FROM pg_replication_slots
UNION ALL
SELECT 'sender', application_name, (state='streaming')::text, state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)
  FROM pg_stat_replication;
```
```sql
-- subscriber
SELECT subname, (pid IS NOT NULL)::text AS worker_running,
       extract(epoch from now()-latest_end_time)::int AS seconds_idle
  FROM pg_stat_subscription
UNION ALL
SELECT subname, (apply_error_count = 0)::text, apply_error_count
  FROM pg_stat_subscription_stats;
```

---

## The design decision framework

```
★★★ PHYSICAL FOR AVAILABILITY. LOGICAL FOR MOVEMENT. ★★★

 ① WHICH KIND?
    ★ PHYSICAL when you need:
      • a standby for failover (63)
      • read scaling with identical data (58)
      • the lowest possible overhead
    ★ LOGICAL when you need:
      • ★ a major-version upgrade with seconds of downtime
      • ★ CDC to a warehouse, search index or event stream (76)
      • a subset of tables, or different indexes on the copy
      • a writable target
    ⇒ ★ MANY SYSTEMS RUN BOTH: physical standbys for HA, one
      logical subscription for CDC.

 ② ★ SLOTS — THE RULES, ON EVERY CLUSTER
    ✓ ★ max_slot_wal_keep_size ALWAYS SET
       ⇒ lose a replica, never the primary
    ✓ ★ ALERT on: active=false · wal_status≠'reserved' ·
       retained > 10 GB · age(xmin) > 100M
    ✓ ★ DROP slots when decommissioning a replica —
       put it in the runbook
    ⇒ ★ THE SAME OBJECT CAUSED THE DISK-FULL PANIC (41) AND THE
      107-DAY VACUUM FAILURE (47). Treat it with respect.

 ③ ★ BEFORE PUBLISHING ANYTHING, AUDIT REPLICA IDENTITY
    every published table needs a PK, or an explicit identity.
    ⇒ ★ REPLICA IDENTITY FULL requires an INDEX ON THE SUBSCRIBER
      or apply degrades to a seq scan per row.
    ⇒ ★ adding a table to a publication can break writes that
      worked yesterday.

 ④ ★ PLAN FOR WHAT LOGICAL DOESN'T CARRY
    ✓ ★ DDL — subscriber-first for additive, publisher-first for
       drops; ★ freeze migrations during initial sync
    ✓ ★ SEQUENCES — generate the advance script from pg_sequences
       across ALL schemas; verify after cutover
    ✓ large objects — never replicated

 ⑤ INITIAL SYNC IS THE RISKY PHASE
    ✓ ★ drop non-constraint indexes on the subscriber first
    ✓ ★ split into multiple subscriptions so one failure doesn't
       restart everything
    ✓ ★ size max_slot_wal_keep_size for the COPY WINDOW, not for
       steady state
    ✓ ★ monitor slot retention continuously during the copy
    ✓ run it in a low-traffic window

 ⑥ ★ CUTOVER CHECKLIST — REHEARSE IT AT LEAST TWICE
    ① read-only mode  ② verify no in-flight writes
    ③ ★ wait for lag EXACTLY zero
    ④ ★ ADVANCE ALL SEQUENCES        ⑤ verify a sample
    ⑥ ★ SET UP REVERSE REPLICATION (the rollback path)
    ⑦ repoint  ⑧ writes on  ⑨ ★ verify counts and checksums
    ⇒ ★ ANALYZE ON THE NEW SERVER BEFORE CUTOVER, not after.

 ⑦ TOPOLOGY
    ✓ cascading reduces primary load — ★ but lag compounds
    ✓ synchronous: ★ 'ANY 1 (r1,r2,r3)', never a single name
    ✗ ★ multi-master: not safe by default. If you want it, you
      probably want sharding (60) or one writer + regional reads.

 ⑧ THE FOUR ALERTS THAT MATTER
    ✓ ★ any slot with active = false
    ✓ ★ any slot with wal_status ≠ 'reserved'
    ✓ ★ pg_stat_subscription.pid IS NULL
    ✓ ★ apply_error_count > 0
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Create a physical replication slot and a replica. Then: (a) stop the replica and show WAL accumulating; (b) show the slot pinning `xmin` via `VACUUM VERBOSE`; (c) set `max_slot_wal_keep_size` low and show `wal_status` becoming `lost`; (d) explain why that outcome is correct.

### Exercise 2 — medium (apply it)
Set up logical replication between two clusters. Then reproduce and fix each failure: (a) DDL drift breaking the subscription; (b) sequences not advancing; (c) a published table with no primary key failing `UPDATE` on the publisher; (d) `REPLICA IDENTITY FULL` without a subscriber index.

For (d), measure the apply rate before and after adding the index.

### Exercise 3 — hard (production simulation)
A 4.2 TB retail database on PostgreSQL 14 must upgrade to 18 with at most five minutes of write downtime. There are 1,840 tables, 412 sequences, and a 1.8 TB partitioned `orders` table.

(a) `pg_upgrade --link` takes 4m18s. Explain why the real user-visible impact is far larger, and quantify it.
(b) Audit for tables that would break on publication. Write the query and explain what you'd do about each result.
(c) The initial copy takes 14 hours and the publisher writes 640 GB of WAL in that window. Explain what goes wrong with `max_slot_wal_keep_size = 256GB` and give three mitigations.
(d) Why split into two subscriptions rather than one?
(e) Give the subscriber settings for the initial load phase, and say which must be reverted before cutover and why.
(f) Why drop indexes before the copy, and how do you recreate them?
(g) Write the cutover script. Explain the ordering of every step, especially why sequences come before repointing and why reverse replication comes before that.
(h) Three rehearsals each failed differently. For each failure, give the root cause and the fix.
(i) Explain why `ANALYZE` on the new server must happen before cutover, not after.
(j) Compare the two approaches on downtime, rollback, hardware and preparation. When would `pg_upgrade` still be the right choice?

---

## Mental model checkpoint

1. State the difference between physical and logical replication in terms of what is shipped.
2. Name four things logical replication does not replicate. Which two cause the most incidents?
3. What does a replication slot guarantee, and what are its two costs?
4. Explain `wal_status` and its four values. Which one means the replica is dead?
5. Why does logical decoding need `catalog_xmin`, and what does an abandoned logical slot bloat?
6. Why are logical changes buffered until commit, and what does that cause for large transactions?
7. Name the four `REPLICA IDENTITY` modes. Which breaks `UPDATE` entirely, and which is slow and why?
8. What happens when DDL is applied to the publisher but not the subscriber? What is the correct ordering rule?
9. Why is `pg_stat_subscription.pid IS NULL` the most important subscription alert?
10. Give the cutover step most often forgotten, and describe how the failure manifests.
11. Why is `max_slot_wal_keep_size` always the right setting, even though it can destroy a replica?

---

## Quick reference card

**Monitor — publisher**
```sql
SELECT slot_name, slot_type, active, wal_status,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained,
       age(xmin) AS xmin_age, age(catalog_xmin) AS cat_xmin_age
  FROM pg_replication_slots;
-- ★ alert: active=false · wal_status≠'reserved' · retained>10GB · xmin_age>100M
```
**Monitor — subscriber**
```sql
SELECT subname, pid, extract(epoch from now()-latest_end_time) AS idle_s
  FROM pg_stat_subscription;                 -- ★ pid IS NULL = worker dead
SELECT subname, apply_error_count, last_error_message
  FROM pg_stat_subscription_stats;           -- ★ >0 = stuck
SELECT srrelid::regclass, srsubstate FROM pg_subscription_rel;  -- sync state
```

| | Physical | Logical |
|---|---|---|
| Ships | ★ WAL bytes | ★ decoded rows |
| Scope | whole cluster | selected tables |
| Version | ★ must match | ★ can differ |
| DDL | ★ replicated | ★ **not** |
| Sequences | ★ replicated | ★ **not** |
| Writable target | no | ★ yes |

**★ Always set:** `max_slot_wal_keep_size` — lose a replica, never the primary.

**`REPLICA IDENTITY`:** `DEFAULT` (PK) · `USING INDEX` · ★ `FULL` (needs a **subscriber index**) · ★ `NOTHING` (`UPDATE`/`DELETE` impossible).

**Cutover:** read-only → lag = 0 → ★ **advance all sequences (all schemas)** → ★ reverse replication → repoint → writes on → verify.
**★ `ANALYZE` on the new server *before* cutover, not after.**

**Initial sync:** drop non-constraint indexes · split subscriptions · size the slot guard for the **copy window** · monitor retention · low-traffic window · ★ freeze migrations.

---

## When would I use this at work?

1. **Every major-version upgrade.** `pg_upgrade`'s headline number omits the statistics rebuild, which is where the real impact lives. Logical replication turns 45–90 minutes of degradation into under a minute — at the cost of a second cluster and three weeks of preparation.

2. **The first day on any cluster you inherit.** Check `pg_replication_slots`. An inactive slot is a latent disk-full PANIC and a latent vacuum failure with no timer on it, and it is invisible on every normal dashboard.

3. **Setting up CDC to a warehouse, search index or event stream.** Logical replication is the mechanism underneath Debezium and every CDC product — and understanding `REPLICA IDENTITY`, `catalog_xmin` and the DDL rule is what separates a CDC pipeline that runs for years from one that silently stops.

4. **When someone proposes multi-master.** PostgreSQL does not make this safe by default, and the conflict-resolution, loop-prevention and sequence problems are yours to solve. The honest answer is usually sharding or a single writer with regional read replicas.

---

## Connected topics

**Understand before this:** 41 (WAL — what is shipped), 42 (recovery — the replay machinery), 47 (VACUUM — what `xmin` pinning costs), 58 (read replicas — using them).

**This unlocks:**
- **63** — HA and failover: promotion, timelines, split-brain
- **64** — backup and PITR: base backups and WAL archiving
- **76** — CDC and polyglot persistence: logical decoding in production
- **68** — CAP and consistency models: the formal framing of what replication gives you
