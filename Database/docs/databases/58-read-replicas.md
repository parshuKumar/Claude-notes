# 58 — Read Replicas
## Phase: Denormalisation & Scale

---

## ELI5 — The Simple Analogy

One accountant keeps the company's books. Everyone who wants to know a number has to queue at her desk, and she is also the only person allowed to *write* in the books. The queue is getting long.

So you install photocopiers. Every time she writes a line, the line is copied to three other desks. Now three more people can answer questions, and she only handles writing.

Three things follow immediately, and they are the whole topic:

1. ★ **The copies are always slightly behind.** Usually by milliseconds. Occasionally — when she writes a very long entry, or a copier jams — by minutes.

2. ★ **This breaks in one specific, maddening way.** You tell her "change my address," she writes it down, you immediately walk to a copy desk and ask "what's my address?" — and it says the old one. You *just did that.* This is not a rare edge case; it's the single most common bug in every replicated system, and it has a name: **read-your-own-writes**.

3. ★ **A copy desk that gets busy doesn't slow the accountant down — but a copier that jams does.** If a copy desk is stuck on a very long question, the copier can't apply the next line to it. Now that desk falls further and further behind, and eventually it's serving data from twenty minutes ago and nobody notices.

Replicas buy read capacity. **They charge in staleness, and the bill is paid at exactly the moment the user cares most.**

---

## Where this fits in the big picture

```
   56 matviews · 57 caching — copies with a staleness window
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 58 READ REPLICAS ← YOU ARE HERE              │
        │ ★ a full copy of the database, milliseconds  │
        │   behind — and sometimes minutes             │
        └────────────────────┬─────────────────────────┘
                             ▼
              59 partitioning · 60 sharding (WRITE scaling)
              62 replication internals · 63 HA & failover
              68 CAP & consistency models
```

**★ The essential framing: a read replica scales READS. It does nothing for writes.** Every replica applies *every* write the primary does. If your problem is write throughput, replicas make it worse, not better — that's Topics 59 and 60.

---

## What is this?

A second PostgreSQL server continuously applying the primary's WAL stream (Topic 41), open for read-only queries.

```
 ┌──────────┐   WAL stream    ┌──────────┐
 │ PRIMARY  │ ──────────────► │ REPLICA  │
 │ reads +  │                 │ ★ reads  │
 │ writes   │ ──────────────► │   only   │
 └──────────┘                 └──────────┘
                              ┌──────────┐
                              │ REPLICA  │
                              └──────────┘

 ★ WHAT IT GIVES YOU
   ① read capacity — N replicas ⇒ ~N× read throughput
   ② isolation — analytics can't starve the transactional workload
   ③ ★ a standby for failover (Topic 63)
   ④ a place to take backups without touching the primary

 ★ WHAT IT DOES NOT GIVE YOU
   ✗ write capacity — ★ every replica does every write
   ✗ storage savings — each holds a FULL copy
   ✗ ★ consistency — reads are stale by construction
   ✗ protection from a bad migration or a bad DELETE
     ⇒ ★ IT REPLICATES YOUR MISTAKES IN MILLISECONDS.
       A replica is not a backup. (Topic 64.)
```

**Physical vs logical** — the distinction that decides what you can do:

| | Physical (streaming) | Logical |
|---|---|---|
| Replicates | ★ byte-identical blocks | row changes, decoded |
| Granularity | ★ the whole cluster | selected tables |
| Schema | ★ must be identical | may differ |
| Version | ★ must match | ★ can cross major versions |
| Writable | no | ★ yes (the subscriber is a real database) |
| Indexes | identical | ★ can differ per subscriber |
| Overhead | very low | higher (decode + apply) |
| Use for | ★ HA, read scaling | ★ CDC, upgrades, selective copies |

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE THE BUG IS ALWAYS THE SAME BUG, AND IT IS ALWAYS
   DISCOVERED IN PRODUCTION.

   POST /profile      → primary  → 200 OK
   GET  /profile      → replica  → ★ the OLD data
   ⇒ the user sees their change vanish. They do it again.
   ⇒ ★ "the save button doesn't work" — the most-reported bug in
     every system that adds replicas without a routing policy.

 ★ AND THE SECOND-ORDER FAILURE IS WORSE:
   a replica that falls 20 minutes behind serves 20-minute-old data
   ★ SILENTLY. There is no error. Users see stale prices, missing
   orders, and inventory that was sold an hour ago.
 ⇒ ★ A REPLICA WITHOUT LAG-AWARE ROUTING IS A CORRECTNESS BUG
   WAITING FOR A SLOW QUERY.
```

---

## The physical reality

### How a replica actually stays current

```
 ★ THE PIPELINE, AND WHERE EACH DELAY COMES FROM:

 PRIMARY                                    REPLICA
 ① a transaction commits
 ② ★ WAL is written and FLUSHED locally
 ③ walsender reads the WAL
 ④ ──── network ────────────────────────►  ⑤ walreceiver receives
                                            ⑥ ★ writes it to local WAL
                                            ⑦ ★ flushes (fsync)
                                            ⑧ ★ the STARTUP process
                                               REPLAYS it into pages

 ★ FOUR SEPARATE LSN POSITIONS, AND THE DIFFERENCE BETWEEN THEM
   IS WHERE YOUR LAG LIVES:
     sent_lsn    what the primary has sent
     write_lsn   what the replica has written to its WAL
     flush_lsn   what the replica has fsynced      ← ★ durability
     replay_lsn  what the replica has APPLIED      ← ★ VISIBILITY

   SELECT client_addr, state,
          pg_wal_lsn_diff(sent_lsn,   write_lsn)  AS write_lag_bytes,
          pg_wal_lsn_diff(write_lsn,  flush_lsn)  AS flush_lag_bytes,
          ★ pg_wal_lsn_diff(flush_lsn, replay_lsn) AS replay_lag_bytes,
          write_lag, flush_lag, replay_lag
     FROM pg_stat_replication;

 ⇒ ★ replay_lsn IS THE ONE THAT MATTERS FOR YOUR QUERIES.
   Data can be safely on the replica's disk (flush_lsn) and still
   invisible to readers (replay_lsn) — this is exactly what happens
   during a replay conflict.
```

### Why replay is the bottleneck — and it is single-threaded

```
 ★ THE PRIMARY WRITES WAL WITH N PARALLEL BACKENDS.
   ★ THE REPLICA REPLAYS IT WITH ONE STARTUP PROCESS.

 ⇒ MEASURED: replay throughput ≈ ★ 100 MB/s, single-threaded
   (Topic 42 — the same number that governs crash recovery)

 ⇒ CONSEQUENCE ①: a primary generating 200 MB/s of WAL will
   ★ ACCUMULATE LAG FOREVER. No tuning fixes it; the replica
   simply cannot keep up.
 ⇒ CONSEQUENCE ②: ★ THE WAL-VOLUME LEVERS FROM TOPIC 41 AND 46
   ARE ALSO YOUR LAG LEVERS:
     • wal_compression = lz4          ⇒ ~59% less WAL to replay
     • HOT updates (don't index hot columns) ⇒ 7× less WAL
     • fewer indexes on write-heavy tables
     • ★ DROP PARTITION instead of DELETE (Topic 59)
 ⇒ CONSEQUENCE ③: ★ A BULK OPERATION ON THE PRIMARY IS A LAG
   EVENT. `UPDATE orders SET …` over 40M rows generates gigabytes
   of WAL that must be replayed serially.
```

### Replay conflicts — the mechanism nobody expects

```
 ★ THE REPLICA HAS A JOB THAT CONFLICTS WITH ITSELF:
   it must apply WAL (including VACUUM's removal of dead tuples)
   AND serve queries whose snapshots may still need those tuples.

 THE CONFLICT:
   ① the primary VACUUMs and removes a dead tuple
   ② that removal arrives in the WAL
   ③ a long query on the replica has a snapshot that ★ still needs
     that tuple
   ④ ★ THE REPLICA MUST CHOOSE:
        wait to apply  ⇒ ★ LAG GROWS
        apply anyway   ⇒ ★ CANCEL THE QUERY

 ⇒ CONTROLLED BY:
     max_standby_streaming_delay = 30s   (default)
       ⇒ ★ "wait up to 30 s for queries, then cancel them"
     ⇒ the query dies with:
       ERROR: canceling statement due to conflict with recovery
       DETAIL: User query might have needed to see row versions
               that must be removed.        ★ SQLSTATE 40001
       ⇒ ★ RETRYABLE — and most ORMs treat it as fatal.

 ★ THE OTHER LEVER, AND ITS PRICE:
     hot_standby_feedback = on
       ⇒ the replica tells the primary its oldest xmin
       ⇒ the primary ★ DOESN'T VACUUM those tuples
       ⇒ ★ no more query cancellations
       ⇒ ★ BUT: a long query on the REPLICA now BLOCKS VACUUM ON
         THE PRIMARY ⇒ bloat on the primary (Topic 47's three
         xmin holders — a replica with feedback is a fourth).

 ⇒ ★ THE TRADE, STATED PLAINLY:
     feedback OFF ⇒ ★ your analytics queries get cancelled
     feedback ON  ⇒ ★ your primary bloats
   ⇒ ★ THE ACTUAL ANSWER: hot_standby_feedback = on ONLY on a
     dedicated analytics replica that no user-facing traffic
     touches, with statement_timeout to bound the damage.
     Leave it OFF on replicas serving user traffic.
```

### The three consistency modes you can actually choose

```
 ★ ① ASYNCHRONOUS (the default)
    the primary commits WITHOUT waiting for any replica.
    ✓ ★ zero write latency cost
    ✗ ★ COMMITTED TRANSACTIONS CAN BE LOST ON FAILOVER
      (whatever hadn't reached the replica)
    ⇒ RPO > 0. For most systems this is correct.

 ★ ② SYNCHRONOUS — synchronous_commit = on + synchronous_standby_names
    the primary waits for the replica before acknowledging COMMIT.
    ✓ ★ RPO = 0 for the synchronous replica
    ✗ ★ every commit pays a network round trip
      MEASURED: 0.4 ms → ★ 2.8 ms same-AZ, ★ 14 ms cross-AZ
    ✗ ★ AND THE DANGEROUS PART: if the sync replica is DOWN, the
      primary ★ BLOCKS ALL COMMITS. Availability of the primary is
      now coupled to the replica.
      ⇒ ★ MITIGATION: ANY 1 (r1, r2, r3) — wait for any one of
        three, so losing one doesn't stop writes.

 ★ ③ SYNCHRONOUS BUT NOT APPLIED — the level that matters
    synchronous_commit values:
      off          ★ don't even flush locally (data loss on crash)
      local        flush locally only
      remote_write the replica has WRITTEN it (not fsynced)
      on           the replica has ★ FLUSHED it (the default sync level)
      ★ remote_apply the replica has ★ APPLIED it — VISIBLE to readers
    ⇒ ★ ONLY remote_apply GIVES YOU READ-YOUR-WRITES ON THE REPLICA.
      `on` guarantees durability, NOT visibility.
    ⇒ ★ AND remote_apply IS EXPENSIVE: the commit waits for replay,
      which is single-threaded.

 ⇒ ★ THE PRACTICAL CHOICE FOR MOST SYSTEMS:
   async replication + ★ APPLICATION-LEVEL ROUTING (below).
   Paying remote_apply on every write to fix a read-your-writes
   problem on 2% of requests is the wrong trade.
```

---

## How it works — step by step

### The routing policy — the thing that actually prevents the bug

```
 ★ FOUR STRATEGIES, IN ORDER OF INCREASING CORRECTNESS AND COST.

 ① ★ ROUTE BY ENDPOINT (the crude default)
    writes and "critical" reads → primary
    everything else            → replica
    ✗ ★ a developer adds a read to a write path and it breaks
    ⇒ acceptable only with a very disciplined team

 ② ★ STICKY-AFTER-WRITE (the pragmatic answer)
    after a write, pin THAT USER'S reads to the primary for N seconds
    ⇒ N = p99 replication lag × safety factor (typically 2–5 s)
    ✓ simple, cheap, ★ fixes the actual user-visible bug
    ✗ a guess; wrong during a lag spike
    ⇒ ★ THIS IS WHAT MOST PRODUCTION SYSTEMS SHOULD DO.

 ③ ★★ LSN-BASED ROUTING (the correct answer)
    on write:  lsn = SELECT pg_current_wal_lsn()
               store it in the session/cookie/context
    on read:   if replica's pg_last_wal_replay_lsn() >= lsn
                 ⇒ ★ the replica is provably current enough. Use it.
               else ⇒ use the primary (or wait briefly)
    ✓ ★ EXACT. No guessing. Reads are correct by construction.
    ✓ degrades gracefully — under low lag, almost everything
      goes to a replica
    ✗ requires carrying an LSN through the request context
    ⇒ ★ THIS IS WHAT AWS AURORA, VITESS AND PLANETSCALE DO
      INTERNALLY. It is not exotic.

 ④ remote_apply synchronous commit
    ✓ no application changes
    ✗ ★ every write pays replay latency
    ⇒ only when writes are rare and reads must always be current
```

### LSN routing, implemented

```js
// ★ the write path records where the database got to
async function withWrite(ctx, fn) {
  const client = await primaryPool.connect();
  try {
    await client.query('BEGIN');
    const result = await fn(client);
    await client.query('COMMIT');
    // ★ AFTER commit — this LSN is guaranteed to include our write
    const { rows: [{ lsn }] } =
      await client.query('SELECT pg_current_wal_lsn() AS lsn');
    ctx.setMinLsn(lsn);            // → session store / signed cookie
    return result;
  } finally { client.release(); }
}

// ★ the read path picks a replica that has caught up
async function readPool(ctx) {
  const minLsn = ctx.getMinLsn();
  if (!minLsn) return replicaPool;              // no recent write

  // replay positions are refreshed by a background poller every 200 ms,
  // ★ not queried per request
  for (const r of replicas) {
    if (r.healthy && lsnGte(r.replayLsn, minLsn)) {
      metrics.increment('route.replica');
      return r.pool;                             // ★ provably current
    }
  }
  metrics.increment('route.primary_fallback');
  return primaryPool;                            // ★ fall back safely
}

// the background poller
setInterval(async () => {
  for (const r of replicas) {
    try {
      const { rows: [x] } = await r.pool.query(`
        SELECT pg_last_wal_replay_lsn() AS lsn,
               extract(epoch from now() - pg_last_xact_replay_timestamp())
                 AS lag_s,
               pg_is_in_recovery() AS in_recovery`);
      r.replayLsn = x.lsn;
      r.lagSeconds = x.lag_s;
      // ★ a replica that is no longer in recovery has been PROMOTED —
      //   remove it from the read pool immediately (Topic 63)
      r.healthy = x.in_recovery && x.lag_s < MAX_LAG_S;
    } catch { r.healthy = false; }
  }
}, 200);
```

### Measuring lag correctly

```sql
-- ★ ON THE PRIMARY — bytes and time, per replica
SELECT application_name, client_addr, state, sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS replay_lag_bytes,
       write_lag, flush_lag, replay_lag
  FROM pg_stat_replication;

-- ★ ON THE REPLICA — the number your application should use
SELECT
  pg_is_in_recovery()                      AS is_replica,
  pg_last_wal_replay_lsn()                 AS replay_lsn,
  ★ extract(epoch from now() - pg_last_xact_replay_timestamp()) AS lag_seconds;
```

```
 ★ THE TRAP IN pg_last_xact_replay_timestamp():
   on an IDLE primary, no transactions arrive, so the timestamp
   stops advancing and lag_seconds grows ★ even though the replica
   is perfectly current.
 ⇒ ★ ALWAYS COMBINE WITH THE LSN CHECK:
     lag_is_real = (pg_last_wal_replay_lsn() <> primary_current_lsn)
   or emit a heartbeat write on the primary every second and
   measure against that.
```

---

## Concept breakdown

```
WHAT A REPLICA IS
└── a server applying the primary's WAL, open for read-only queries
     ★ scales READS. Does nothing for writes. ★ Is NOT a backup —
       it replicates your mistakes in milliseconds.

PHYSICAL vs LOGICAL
├── physical  byte-identical, whole cluster, same version, ★ HA + reads
└── logical   decoded rows, selected tables, ★ cross-version,
               writable subscriber, different indexes ⇒ CDC, upgrades

★ THE FOUR LSNs — and only one governs your queries
   sent → write → flush → ★ REPLAY
   flush = durable.  ★ replay = VISIBLE.

★ REPLAY IS SINGLE-THREADED — ~100 MB/s
   ⇒ a primary generating more WAL than that accumulates lag forever
   ⇒ ★ your WAL-volume levers ARE your lag levers:
     wal_compression=lz4 · HOT updates · fewer indexes ·
     ★ DROP PARTITION not DELETE
   ⇒ ★ a bulk UPDATE on the primary IS a lag event

★ REPLAY CONFLICTS — the mechanism nobody expects
   VACUUM's tuple removal vs a long query's snapshot on the replica
   ⇒ max_standby_streaming_delay (30 s) then ★ CANCEL the query
     ERROR: canceling statement due to conflict with recovery (40001)
   ⇒ hot_standby_feedback = on stops cancellations
     ★ BUT makes the replica a FOURTH xmin holder ⇒ primary bloats
   ⇒ ★ ANSWER: feedback ON only on a dedicated analytics replica,
     OFF on user-facing ones, + statement_timeout everywhere

★ synchronous_commit LEVELS
   off · local · remote_write · on (=flushed) · ★ remote_apply
   ⇒ ★ ONLY remote_apply gives read-your-writes on a replica.
     `on` gives DURABILITY, not VISIBILITY.
   ⇒ sync replication couples primary availability to the replica
     ⇒ ★ use ANY 1 (r1,r2,r3)

★ ROUTING — the four strategies
├── ① by endpoint        crude; breaks when someone adds a read
├── ② ★ sticky-after-write  pin the user to the primary for N s
│                            ⇒ what most systems should do
├── ③ ★★ LSN-BASED       exact; store the write's LSN, only use a
│                          replica whose replay_lsn >= it
└── ④ remote_apply       no app change, ★ every write pays

★ THE TRAP
   pg_last_xact_replay_timestamp() grows on an IDLE primary
   ⇒ ★ combine with an LSN comparison or a heartbeat write
```

---

## Diagrams

**Diagram 1 — big picture: the four LSNs and where lag lives**

```
  PRIMARY                                          REPLICA
 ┌──────────────────┐                          ┌──────────────────────┐
 │ COMMIT           │                          │                      │
 │  ↓               │                          │                      │
 │ WAL flushed      │                          │                      │
 │  ↓               │                          │                      │
 │ sent_lsn ────────┼──── network ────────────►│ write_lsn            │
 │                  │      ~0.2 ms             │  ↓ fsync             │
 │                  │                          │ flush_lsn ★ DURABLE  │
 │                  │                          │  ↓                   │
 │                  │                          │ ★ STARTUP process    │
 │                  │                          │   replays serially   │
 │                  │                          │   ★ ~100 MB/s        │
 │                  │                          │  ↓                   │
 │                  │                          │ ★ replay_lsn         │
 │                  │                          │   ★ VISIBLE TO READS │
 └──────────────────┘                          └──────────────────────┘

 ★ synchronous_commit = on        waits for ── flush_lsn  (durable)
 ★ synchronous_commit = remote_apply waits for ── replay_lsn (visible)
   ⇒ ★ THE GAP BETWEEN THEM IS WHERE READ-YOUR-WRITES BUGS LIVE.

 ★ AND THE BOTTLENECK IS ALWAYS THE LAST ARROW:
   N parallel backends write the WAL; ONE process replays it.
```

**Diagram 2 — data flow: the read-your-own-writes bug, and three fixes**

```
 ✗ NAIVE ROUTING
   t=0ms    POST /profile  ──► PRIMARY   name = 'Meera Sharma'  ✓ 200
   t=12ms   GET  /profile  ──► REPLICA   ★ replay_lsn is 18 ms behind
                               ⇒ returns 'Meera Singh'
   ⇒ ★ "the save button doesn't work". The user saves again.

 ✓ FIX ② — STICKY-AFTER-WRITE
   t=0ms    POST /profile  ──► PRIMARY   ★ set session.pinnedUntil
                                            = now + 3s
   t=12ms   GET  /profile  ──► ★ PRIMARY (pinned)      ✓ correct
   t=3.1s   GET  /profile  ──► REPLICA                 ✓ caught up
   ⇒ ★ simple, cheap, fixes the real bug
   ⇒ ✗ a GUESS. During a lag spike, 3 s is not enough.

 ✓✓ FIX ③ — LSN ROUTING
   t=0ms    POST /profile  ──► PRIMARY
                               ★ lsn = pg_current_wal_lsn() = 4A/8C1220
                               ★ stored in the session
   t=12ms   GET  /profile   replica.replay_lsn = 4A/8C1180
                            ★ 4A/8C1180 < 4A/8C1220 ⇒ NOT caught up
                            ──► PRIMARY                ✓ correct
   t=40ms   GET  /profile   replica.replay_lsn = 4A/8C1400
                            ★ 4A/8C1400 >= 4A/8C1220 ⇒ caught up
                            ──► REPLICA                ✓ correct
   ⇒ ★ EXACT. Not a guess. And under normal lag, almost every read
     still reaches a replica.

 ✓ FIX ④ — remote_apply
   t=0ms    POST /profile  ──► PRIMARY, ★ commit waits for replay
            ⇒ commit latency 0.4 ms → ★ 4.2 ms
   t=12ms   GET  /profile  ──► REPLICA  ✓ correct
   ⇒ ★ every write pays, to fix reads on ~2% of requests.
```

**Diagram 3 — before/after: the replay-conflict trade**

```
 ✗ hot_standby_feedback = off  (the default)
 ┌───────────────────────────────────────────────────────────────┐
 │  PRIMARY                        ANALYTICS REPLICA              │
 │  VACUUM removes dead tuples ──► WAL arrives                    │
 │                                 ★ a 4-minute report needs      │
 │                                   those tuples                 │
 │                                 ⇒ wait max_standby_streaming_  │
 │                                   delay (30 s)                 │
 │                                 ⇒ ★ CANCEL THE QUERY           │
 │                                                                │
 │  ERROR: canceling statement due to conflict with recovery      │
 │  ⇒ ★ 41% of nightly reports failed. Retried, failed again.    │
 └───────────────────────────────────────────────────────────────┘

 ✗ hot_standby_feedback = on, on a replica serving user traffic
 ┌───────────────────────────────────────────────────────────────┐
 │  REPLICA's oldest xmin ──────► PRIMARY holds those tuples      │
 │                                                                │
 │  ★ a 40-minute BI query on the replica now blocks VACUUM       │
 │    ON THE PRIMARY for 40 minutes                               │
 │  ⇒ n_dead_tup climbs, autovacuum reclaims nothing              │
 │  ⇒ ★ the primary bloats — the exact failure of Topic 47        │
 └───────────────────────────────────────────────────────────────┘

 ✓ THE ACTUAL ANSWER — separate the workloads
 ┌───────────────────────────────────────────────────────────────┐
 │  replica-app-1, replica-app-2   ★ user-facing                 │
 │    hot_standby_feedback = off                                  │
 │    max_standby_streaming_delay = 10s                           │
 │    statement_timeout = 5s        ★ queries here are SHORT      │
 │    ⇒ ★ conflicts essentially never happen                     │
 │                                                                │
 │  replica-analytics               ★ dedicated, no user traffic  │
 │    hot_standby_feedback = on     ★ reports never cancelled     │
 │    max_standby_streaming_delay = 300s                          │
 │    statement_timeout = 900s      ★ bounds the primary's bloat  │
 │    ⇒ ★ the primary's exposure is capped at 15 minutes         │
 │                                                                │
 │  ★ + an alert on the primary: oldest xmin age > 20 min        │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**Set up a replica.**
```bash
# on the primary
psql -c "CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'x';"
psql -c "ALTER SYSTEM SET wal_level = 'replica';"
psql -c "ALTER SYSTEM SET max_wal_senders = 10;"
psql -c "ALTER SYSTEM SET max_replication_slots = 10;"
psql -c "SELECT pg_create_physical_replication_slot('replica1');"
# restart

# on the replica host
pg_basebackup -h primary -U replicator -D "$PGDATA" -Fp -Xs -P -R -S replica1
# ★ -R writes primary_conninfo and standby.signal
cat "$PGDATA/postgresql.auto.conf"
```
```
 primary_conninfo = 'host=primary user=replicator ...'
 primary_slot_name = 'replica1'
```
```bash
pg_ctl start -D "$PGDATA"
psql -c "SELECT pg_is_in_recovery();"
```
```
 pg_is_in_recovery
-------------------
 ★ t
```

**Confirm it's read-only.**
```sql
INSERT INTO orders (customer_id, total_minor) VALUES (1, 100);
```
```
ERROR:  cannot execute INSERT in a read-only transaction
```

**Watch the four LSNs.**
```sql
-- on the primary
SELECT application_name, client_addr, state, sync_state,
       sent_lsn, write_lsn, flush_lsn, replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS lag_bytes,
       write_lag, flush_lag, replay_lag
  FROM pg_stat_replication;
```
```
 application_name | state     | sync_state |  replay_lsn  | lag_bytes | replay_lag
------------------+-----------+------------+--------------+-----------+-------------
 walreceiver      | streaming | async      | 4A/8C001220  |      1184 | 00:00:00.004
```

**Prove the staleness window.**
```sql
-- primary
INSERT INTO orders (customer_id, total_minor) VALUES (42, 250000) RETURNING id;
```
```
   id
--------
 8842119
```
```bash
# replica, immediately
psql -h replica -c "SELECT count(*) FROM orders WHERE id = 8842119"
```
```
 count
-------
   ★ 0        — not replayed yet
```
```bash
sleep 0.05
psql -h replica -c "SELECT count(*) FROM orders WHERE id = 8842119"
```
```
 count
-------
     1
```

**Measure lag from the replica.**
```sql
SELECT pg_is_in_recovery() AS is_replica,
       pg_last_wal_replay_lsn() AS replay_lsn,
       pg_last_xact_replay_timestamp() AS last_xact,
       extract(epoch from now() - pg_last_xact_replay_timestamp())::numeric(10,3)
         AS lag_seconds;
```
```
 is_replica |  replay_lsn  |         last_xact         | lag_seconds
------------+--------------+---------------------------+-------------
 t          | 4A/8C001220  | 2026-08-18 09:14:22.44+05 |       0.004
```

**Prove the idle-primary trap.**
```bash
# stop all writes on the primary, wait 60 s
sleep 60
psql -h replica -c "SELECT extract(epoch from now() - pg_last_xact_replay_timestamp()) AS lag_s"
```
```
    lag_s
-----------
 ★ 60.114        — the replica is PERFECTLY CURRENT. This is not lag.
```
```bash
# ★ the correct check combines LSN equality
PRIMARY_LSN=$(psql -h primary -tAc "SELECT pg_current_wal_lsn()")
REPLICA_LSN=$(psql -h replica -tAc "SELECT pg_last_wal_replay_lsn()")
[ "$PRIMARY_LSN" = "$REPLICA_LSN" ] && echo "★ caught up" || echo "behind"
```
```
 ★ caught up
```

**Reproduce a replay conflict.**
```bash
# replica
psql -h replica -c "SET max_standby_streaming_delay = '2s'"
psql -h replica -c "BEGIN; SELECT count(*) FROM orders; SELECT pg_sleep(60);" &

# primary — generate dead tuples and vacuum them
psql -h primary -c "UPDATE orders SET total_minor = total_minor WHERE id < 500000"
psql -h primary -c "VACUUM orders"
```
```
 -- on the replica, the long query dies:
 ERROR:  canceling statement due to conflict with recovery
 DETAIL:  User query might have needed to see row versions that must be removed.
   ★ SQLSTATE 40001 — retryable, and most ORMs treat it as fatal.
```
```sql
-- count them
SELECT datname, confl_snapshot, confl_bufferpin, confl_deadlock,
       confl_lock, confl_tablespace
  FROM pg_stat_database_conflicts WHERE datname = current_database();
```
```
 datname | confl_snapshot | confl_bufferpin | confl_deadlock | confl_lock
---------+----------------+-----------------+----------------+------------
 shop    |          ★ 412 |               0 |              0 |          2
```

**And `hot_standby_feedback` preventing it — at a cost.**
```bash
psql -h replica -c "ALTER SYSTEM SET hot_standby_feedback = on; SELECT pg_reload_conf();"
# repeat the experiment — the query survives.

# ★ but now, on the PRIMARY:
psql -h primary -c "
  SELECT slot_name, xmin, catalog_xmin,
         age(xmin) AS xmin_age FROM pg_replication_slots;"
```
```
 slot_name |   xmin   | xmin_age
-----------+----------+----------
 replica1  |  8842119 |  ★ 412884
   ★ the replica is now pinning the primary's xmin.
     VACUUM on the primary cannot reclaim past it. (Topic 47.)
```

**Set up synchronous replication and measure the cost.**
```sql
-- primary
ALTER SYSTEM SET synchronous_standby_names = 'ANY 1 (replica1, replica2)';
SELECT pg_reload_conf();
```
```bash
\timing on
# asynchronous
psql -h primary -c "SET synchronous_commit = local;
                    INSERT INTO orders (customer_id,total_minor) VALUES (1,1);"
# Time: ★ 0.412 ms

# synchronous (flushed on the replica)
psql -h primary -c "SET synchronous_commit = on;
                    INSERT INTO orders (customer_id,total_minor) VALUES (1,1);"
# Time: ★ 2.884 ms      (7×)

# ★ synchronous AND applied
psql -h primary -c "SET synchronous_commit = remote_apply;
                    INSERT INTO orders (customer_id,total_minor) VALUES (1,1);"
# Time: ★ 4.204 ms      (10×)
```

**Prove `remote_apply` gives read-your-writes.**
```bash
psql -h primary -c "SET synchronous_commit = remote_apply;
  INSERT INTO orders (customer_id,total_minor) VALUES (99,777) RETURNING id" 
# id = 8842200
psql -h replica -c "SELECT count(*) FROM orders WHERE id = 8842200"
```
```
 count
-------
   ★ 1        — visible immediately, guaranteed.
```

---

## Example 2 — production scenario

**The situation.** A ticketing platform. The primary is at 78% CPU on reads; someone adds two replicas and routes all `GET` traffic to them.

```
 WEEK 1 — it works
   primary CPU     78% → 22%
   read throughput 8,400 → 24,000 rps
   p99             180 ms → 62 ms

 WEEK 2 — the bug reports start
   "I bought a ticket and my orders page is empty."        ★ 214/day
   "I changed my email and it reverted."                    ★ 88/day
   "The seat I selected shows as available to someone else."★ 41/day

 WEEK 3 — the outage
   14:22 a nightly data-fix UPDATE runs on the primary
   ⇒ ★ replicas fall 24 minutes behind
   ⇒ ★ every user sees 24-minute-old data, SILENTLY
   ⇒ 1,100 double-bookings before anyone noticed
```

**Step 1 — quantify the lag distribution, not the average.**

```sql
-- sampled every 5 s for 7 days into a monitoring table
SELECT
  percentile_cont(0.50) WITHIN GROUP (ORDER BY lag_ms) AS p50,
  percentile_cont(0.95) WITHIN GROUP (ORDER BY lag_ms) AS p95,
  percentile_cont(0.99) WITHIN GROUP (ORDER BY lag_ms) AS p99,
  percentile_cont(0.999) WITHIN GROUP (ORDER BY lag_ms) AS p999,
  max(lag_ms) AS max
FROM replica_lag_samples WHERE sampled_at > now() - interval '7 days';
```
```
 p50 |  p95  |  p99  |   p999   |    max
-----+-------+-------+----------+-----------
   4 |    18 | ★ 340 | ★ 41,200 | ★ 1,442,000
```
```
 ★ p50 IS 4 ms — WHICH IS WHY IT LOOKED FINE IN TESTING.
   p999 is 41 seconds. The max is 24 minutes.
 ⇒ ★ REPLICATION LAG IS NOT NORMALLY DISTRIBUTED. It is
   near-zero almost always and enormous occasionally, and the
   occasional case is exactly when a bulk operation runs.
```

**Step 2 — find what causes the spikes.**

```sql
SELECT date_trunc('minute', sampled_at) AS minute, max(lag_ms) AS lag
  FROM replica_lag_samples
 WHERE sampled_at > now() - interval '7 days'
 GROUP BY 1 HAVING max(lag_ms) > 10000 ORDER BY 2 DESC LIMIT 5;
```
```
        minute        |   lag
----------------------+----------
 2026-08-16 02:00:00  | ★ 1442000
 2026-08-14 02:00:00  |  ★ 884000
 2026-08-12 03:14:00  |  ★ 412000
```
```
 ★ 02:00 EVERY NIGHT. The retention job:
   DELETE FROM event_logs WHERE created_at < now() - interval '90 days';
 ⇒ 41M rows deleted ⇒ ★ 18 GB of WAL ⇒ replayed serially at
   ~100 MB/s ⇒ ★ 180 seconds minimum, and much longer with
   index cleanup.
 ⇒ ★ THE FIX IS NOT A REPLICA SETTING. It is Topic 59:
   DROP PARTITION instead of DELETE. 18 GB of WAL → ~2 KB.
```

**Step 3 — fix the WAL volume first.**

```sql
-- ① partition event_logs by month; retention becomes DROP
DROP TABLE event_logs_2026_05;      -- ★ instant, ~0 WAL

-- ② wal_compression (Topic 41)
ALTER SYSTEM SET wal_compression = 'lz4';

-- ③ find the non-HOT updates driving baseline WAL (Topic 46)
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct
  FROM pg_stat_user_tables WHERE n_tup_upd > 1000000 ORDER BY hot_pct;
```
```
   relname   | n_tup_upd | n_tup_hot_upd | hot_pct
-------------+-----------+---------------+---------
 seat_holds  | 188402118 |       2104882 |   ★ 1.1
```
```sql
-- an index on seat_holds(expires_at), the column every update touches
DROP INDEX CONCURRENTLY idx_seat_holds_expires;
ALTER TABLE seat_holds SET (fillfactor = 70);
-- + partition seat_holds by day for the expiry sweep
```
```
 ★ MEASURED AFTER ①②③:
   baseline WAL   14 GB/hr → ★ 3.2 GB/hr
   nightly spike  18 GB in 4 min → ★ ~0
   p999 lag       41,200 ms → ★ 380 ms
   max lag        1,442,000 ms → ★ 4,100 ms
 ⇒ ★ THE BIGGEST LAG IMPROVEMENT CAME FROM THE PRIMARY, NOT THE
   REPLICAS. Lag is a WAL-volume problem before it is a
   replication problem.
```

**Step 4 — LSN-based routing for the read-your-writes bug.**

```js
// ★ the LSN travels in a signed cookie, so it survives across
//   app instances and load-balancer hops
const LSN_COOKIE = 'x-db-lsn';

app.use((req, res, next) => {
  req.dbCtx = {
    minLsn: verifyCookie(req.cookies[LSN_COOKIE]),
    setMinLsn(lsn) {
      this.minLsn = lsn;
      res.cookie(LSN_COOKIE, signCookie(lsn),
                 { maxAge: 30_000, httpOnly: true, sameSite: 'lax' });
    },
  };
  next();
});

async function query(req, sql, params, { write = false } = {}) {
  if (write) {
    const client = await primaryPool.connect();
    try {
      await client.query('BEGIN');
      const r = await client.query(sql, params);
      await client.query('COMMIT');
      const { rows: [{ lsn }] } =
        await client.query('SELECT pg_current_wal_lsn() AS lsn');
      req.dbCtx.setMinLsn(lsn);       // ★ record where we got to
      return r;
    } finally { client.release(); }
  }

  const pool = pickReadPool(req.dbCtx.minLsn);
  return pool.query(sql, params);
}

function pickReadPool(minLsn) {
  if (!minLsn) {                                  // no recent write
    const r = healthyReplicas();
    return r.length ? r[Math.floor(Math.random()*r.length)].pool : primaryPool;
  }
  // ★ only a replica that has PROVABLY replayed our write
  const ready = healthyReplicas().filter(r => lsnGte(r.replayLsn, minLsn));
  if (ready.length) {
    metrics.increment('route.replica_lsn_ok');
    return ready[Math.floor(Math.random()*ready.length)].pool;
  }
  metrics.increment('route.primary_fallback');    // ★ watch this number
  return primaryPool;
}
```

```js
// ★ the poller — replay positions are cached, not queried per request
const replicas = [
  { name: 'r1', pool: r1Pool, replayLsn: null, lagS: null, healthy: false },
  { name: 'r2', pool: r2Pool, replayLsn: null, lagS: null, healthy: false },
];

setInterval(async () => {
  await Promise.all(replicas.map(async (r) => {
    try {
      const { rows: [x] } = await r.pool.query(`
        SELECT pg_is_in_recovery() AS in_recovery,
               pg_last_wal_replay_lsn() AS lsn,
               coalesce(extract(epoch from now()
                 - pg_last_xact_replay_timestamp()), 0) AS lag_s`);
      r.replayLsn = x.lsn;
      r.lagS = Number(x.lag_s);
      // ★ THREE health conditions, all necessary:
      r.healthy =
        x.in_recovery === true &&       // ★ not promoted (Topic 63)
        r.lagS < 10;                    // bounded lag
      metrics.gauge('replica.lag_seconds', r.lagS, { replica: r.name });
    } catch (e) {
      r.healthy = false;
      metrics.increment('replica.poll_failed', { replica: r.name });
    }
  }));
}, 200);
```

**Step 5 — the third bug: seat availability.**

```
 ★ "The seat I selected shows as available to someone else."
   ⇒ THIS ONE IS NOT A ROUTING PROBLEM.
   ⇒ ★ SEAT AVAILABILITY IS A CORRECTNESS INVARIANT, NOT A READ.
     It must never come from a replica at ANY lag, because the
     decision it drives (holding a seat) is a WRITE.
 ⇒ ★ THE RULE THAT FALLS OUT:
   ANY READ WHOSE RESULT IS USED TO MAKE A WRITE DECISION
   MUST COME FROM THE PRIMARY — or better, must not be a
   separate read at all (Topic 49's atomic statement).
```

```js
// ✗ before: read availability from a replica, then write
const seat = await replicaPool.query(
  'SELECT status FROM seats WHERE id=$1', [seatId]);
if (seat.rows[0].status === 'available')
  await primaryPool.query("UPDATE seats SET status='held' WHERE id=$1", [seatId]);

// ✓ after: one atomic statement on the primary (Topics 43, 49)
const { rowCount } = await primaryPool.query(
  `UPDATE seats SET status='held', held_until = now() + interval '10 minutes'
    WHERE id = $1 AND status = 'available'`, [seatId]);
if (rowCount === 0) throw new AppError('SEAT_UNAVAILABLE');
// ★ no replica involved, no race, no lag sensitivity.
```

**Step 6 — workload separation and settings.**

```sql
-- replica-app-1, replica-app-2  ★ user-facing, short queries only
ALTER SYSTEM SET hot_standby_feedback = off;
ALTER SYSTEM SET max_standby_streaming_delay = '10s';
ALTER SYSTEM SET statement_timeout = '5s';

-- replica-analytics  ★ dedicated, no user traffic
ALTER SYSTEM SET hot_standby_feedback = on;
ALTER SYSTEM SET max_standby_streaming_delay = '300s';
ALTER SYSTEM SET statement_timeout = '900s';   -- ★ bounds primary bloat
```
```sql
-- ★ and the alert on the primary that catches feedback going wrong
SELECT max(age(xmin)) AS oldest_replica_xmin_age FROM pg_replication_slots;
-- ★ alert > 20 minutes' worth
```

**Step 7 — the alerts.**

```sql
-- ① lag, per replica, ★ combining time AND lsn (avoids the idle trap)
SELECT application_name,
       extract(epoch from replay_lag) AS replay_lag_s,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes
  FROM pg_stat_replication;
-- ★ alert: replay_lag_s > 5 AND lag_bytes > 0

-- ② replay conflicts
SELECT sum(confl_snapshot + confl_lock + confl_bufferpin
           + confl_deadlock + confl_tablespace) AS conflicts
  FROM pg_stat_database_conflicts;
-- ★ alert on any increase on a user-facing replica

-- ③ ★ primary_fallback rate — from the application
--    a rising rate means lag is eating your read-scaling benefit
--    alert: route.primary_fallback / total_reads > 0.10

-- ④ ★ a replica that stopped being a replica (was promoted)
SELECT pg_is_in_recovery();     -- ★ false on a read pool member = remove it
```

**Step 8 — results.**

| | Before | After |
|---|---|---|
| p999 lag | 41,200 ms | **380 ms** (**108×**) |
| Max lag | 24 min | **4.1 s** |
| Baseline WAL | 14 GB/hr | **3.2 GB/hr** |
| "Save doesn't work" reports | 302/day | **0** |
| Double-bookings | 1,100 in one incident | **0** — structurally impossible |
| Reads served by replicas | 100% (incorrectly) | **96.4%** (LSN-verified) |
| Primary fallback rate | n/a | 3.6% |
| Read throughput | 24,000 rps | 24,000 rps (unchanged) |

```
 ★ FIVE LESSONS:
 ① ★ THE BIGGEST LAG FIX WAS ON THE PRIMARY. Replay is
   single-threaded at ~100 MB/s, so lag is a WAL-VOLUME problem
   first. DROP PARTITION instead of DELETE removed 18 GB of WAL
   per night.
 ② ★ p50 LAG OF 4 ms IS WHY IT PASSED TESTING. Lag is not
   normally distributed — near-zero almost always, enormous
   exactly when a bulk operation runs. ★ ALWAYS LOOK AT p999.
 ③ ★ LSN ROUTING COST 3.6% OF READS and eliminated 302 bug
   reports/day. Sticky-after-write would have needed a 41-second
   window to cover p999 — which is not a usable product.
 ④ ★ ONE OF THE THREE BUGS WAS NOT A ROUTING PROBLEM AT ALL.
   A read used to make a write decision must come from the
   primary — or, better, must not be a separate read.
 ⑤ ★ THE HEALTH CHECK MUST INCLUDE pg_is_in_recovery(). A
   promoted replica silently serving stale-then-diverging data
   is worse than one that is simply behind. (Topic 63.)
```

---

## Common mistakes

**1. Routing all reads to replicas with no policy.**
- *Symptom:* "the save button doesn't work" — the most-reported bug in every replicated system.
- *Fix:* sticky-after-write at minimum; LSN-based routing properly.

**2. Sizing the stickiness window from average lag.**
- *Symptom:* it works 99.9% of the time and fails exactly when a bulk job runs.
- *Fix:* size from p999, or use LSN routing, which needs no window at all.

**3. Reading from a replica to make a write decision.**
- *Symptom:* double-bookings, oversold inventory, duplicate resources.
- *Fix:* the decision and the write must be one atomic statement on the primary (Topics 43, 49).

**4. Treating a replica as a backup.**
- *Symptom:* a `DELETE` without a `WHERE` replicates in 3 ms.
- *Fix:* replicas are for availability and read scaling. Backups and PITR are Topic 64.

**5. Ignoring that replay is single-threaded.**
- *Symptom:* lag grows unboundedly under write load and no replica setting helps.
- *Fix:* reduce WAL on the primary — `wal_compression`, HOT updates, fewer indexes, `DROP PARTITION`.

**6. `hot_standby_feedback = on` everywhere.**
- *Symptom:* the primary bloats; autovacuum reclaims nothing.
- *Fix:* on only for a dedicated analytics replica, with `statement_timeout` bounding exposure, plus an alert on slot `xmin` age.

**7. `hot_standby_feedback = off` on an analytics replica.**
- *Symptom:* long reports die with `canceling statement due to conflict with recovery`.
- *Fix:* the opposite of #6 — separate the workloads and configure each for its job.

**8. Treating `40001` from a replay conflict as fatal.**
- *Symptom:* dashboards fail on a retryable error.
- *Fix:* the same retry loop as Topics 44 and 48.

**9. Using `pg_last_xact_replay_timestamp()` alone.**
- *Symptom:* false lag alarms on an idle primary.
- *Fix:* combine with an LSN comparison, or write a heartbeat row every second.

**10. Not checking `pg_is_in_recovery()` in the health check.**
- *Symptom:* a promoted replica stays in the read pool and diverges.
- *Fix:* health = `in_recovery AND lag < threshold`.

**11. Synchronous replication with a single standby name.**
- *Symptom:* the standby dies and **all writes on the primary block**.
- *Fix:* `ANY 1 (r1, r2, r3)`.

**12. Expecting `synchronous_commit = on` to give read-your-writes.**
- *Symptom:* durable but still invisible.
- *Fix:* `on` means *flushed*; only `remote_apply` means *applied*.

---

## Hands-on proof

**PROVE IT #1–#9 — Example 1** (creating a replica, proving it's read-only, the four LSNs in `pg_stat_replication`, the staleness window, the idle-primary trap and the LSN-based correction, a replay conflict cancelling a query, `pg_stat_database_conflicts`, `hot_standby_feedback` fixing it while pinning the primary's `xmin`, and the three `synchronous_commit` latencies at 0.4 / 2.9 / 4.2 ms).

**PROVE IT #10 — replay is the bottleneck.**
```bash
# generate WAL faster than 100 MB/s on the primary
pgbench -c 50 -j 8 -T 120 -f heavy_write.sql primary &
watch -n1 'psql -h primary -tAc "
  SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn)/1024/1024 AS lag_mb
    FROM pg_stat_replication"'
```
```
 lag_mb
--------
 ★ 412
 ★ 884
 ★ 1,402       — monotonically growing; the replica cannot catch up
```
```bash
# ★ now with wal_compression
psql -h primary -c "ALTER SYSTEM SET wal_compression='lz4'; SELECT pg_reload_conf();"
# repeat
```
```
 lag_mb
--------
 88
 41
 ★ 4          — it keeps up
```

**PROVE IT #11 — LSN comparison is exact.**
```bash
LSN=$(psql -h primary -tAc "
  INSERT INTO orders (customer_id,total_minor) VALUES (1,1);
  SELECT pg_current_wal_lsn()")
while true; do
  R=$(psql -h replica -tAc "SELECT pg_last_wal_replay_lsn()")
  psql -h replica -tAc "SELECT '$R'::pg_lsn >= '$LSN'::pg_lsn" | grep -q t && break
done
echo "★ replica has provably replayed our write"
```

**PROVE IT #12 — the promoted-replica hazard.**
```bash
pg_ctl promote -D "$REPLICA_PGDATA"
psql -h replica -c "SELECT pg_is_in_recovery()"
#  ★ f
psql -h replica -c "INSERT INTO orders (customer_id,total_minor) VALUES (1,1)"
#  ★ INSERT 0 1   — it now ACCEPTS WRITES and has diverged.
# ⇒ ★ this is why the health check must include pg_is_in_recovery().
```

---

## The design decision framework

```
★★★ REPLICAS SCALE READS. THEY CHARGE IN STALENESS,
    AND THE BILL ARRIVES WHEN THE USER CARES MOST. ★★★

 ① ARE READS ACTUALLY THE PROBLEM?
    ★ if the primary is write-bound, replicas make it WORSE —
      every replica applies every write.
    ⇒ write-bound ⇒ partitioning (59) or sharding (60)
    ⇒ read-bound  ⇒ ★ and first: an index (Topic 54's gate 2),
      a matview (56), or the CDN (57). Those are cheaper.

 ② MEASURE LAG AT p999, NEVER AT THE AVERAGE
    ★ lag is near-zero almost always and enormous occasionally.
      p50 4 ms · p99 340 ms · ★ p999 41 s · max 24 min
    ⇒ ★ design for p999, because that is exactly when a bulk job
      is running and your data is most out of date.

 ③ ★ FIX WAL VOLUME BEFORE FIXING REPLICAS
    replay is ★ SINGLE-THREADED at ~100 MB/s.
    ✓ wal_compression = lz4                    ~59% less
    ✓ HOT updates — don't index hot columns    up to 7× less
    ✓ fewer indexes on write-heavy tables
    ✓ ★ DROP PARTITION, never DELETE           orders of magnitude
    ⇒ ★ THE LARGEST LAG IMPROVEMENT USUALLY COMES FROM THE
      PRIMARY, NOT FROM REPLICA TUNING.

 ④ CHOOSE A ROUTING POLICY — AND WRITE IT DOWN
    ★ sticky-after-write   pragmatic; window = p999 × 2
    ★★ LSN-based           exact; costs a few % primary fallback
    remote_apply           no app change; every write pays
    ⇒ ★ AND THE RULE THAT OVERRIDES ALL OF THEM:
      ANY READ USED TO MAKE A WRITE DECISION GOES TO THE PRIMARY —
      or is not a separate read at all (an atomic statement).

 ⑤ SEPARATE WORKLOADS, CONFIGURE EACH FOR ITS JOB
    USER-FACING replicas
      hot_standby_feedback = off · max_standby_streaming_delay = 10s
      · statement_timeout = 5s
    ANALYTICS replica (★ dedicated, no user traffic)
      hot_standby_feedback = on · max_standby_streaming_delay = 300s
      · ★ statement_timeout = 900s (bounds the primary's bloat)
    ⇒ ★ never one replica doing both — the settings are opposites.

 ⑥ SYNCHRONOUS REPLICATION — ONLY IF RPO = 0 IS A REQUIREMENT
    ★ synchronous_standby_names = 'ANY 1 (r1, r2, r3)'
      — never a single name, or losing it blocks ALL writes
    ★ synchronous_commit: on = DURABLE · remote_apply = VISIBLE
    cost: 0.4 → 2.9 ms (on) → 4.2 ms (remote_apply)

 ⑦ THE FOUR ALERTS
    ✓ ★ replay lag > 5 s AND lag_bytes > 0 (avoids the idle trap)
    ✓ ★ any replay conflict on a user-facing replica
    ✓ ★ primary-fallback rate > 10% (lag is eating your benefit)
    ✓ ★ pg_is_in_recovery() = false on a read-pool member (promoted)

 ⑧ A REPLICA IS NOT A BACKUP
    ★ it replicates DROP TABLE in 3 ms. You need PITR (Topic 64).
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Set up a streaming replica. Then: (a) prove it's read-only; (b) show all four LSNs in `pg_stat_replication` and explain each; (c) insert on the primary and measure how long until it's visible on the replica; (d) stop writes for 60 seconds and show the idle-primary lag trap, then write the correct check.

### Exercise 2 — medium (apply it)
Reproduce a replay conflict: set `max_standby_streaming_delay = '2s'` on the replica, start a long query, then generate dead tuples and `VACUUM` on the primary. Show the cancellation and `pg_stat_database_conflicts`.

Then enable `hot_standby_feedback`, prove the query survives, and prove the primary's `xmin` is now pinned. Write the settings you'd use for a user-facing replica versus an analytics replica, with justification.

### Exercise 3 — hard (production simulation)
A ticketing platform added two replicas and routed all `GET`s to them. Three bug classes appeared (302/day of "my change reverted", plus 1,100 double-bookings during one incident), and lag reached 24 minutes at 02:00 nightly.

(a) The lag p50 is 4 ms and the p999 is 41 seconds. Explain why this passed testing and what it says about how to size any stickiness window.
(b) Find the 02:00 cause. Explain mechanically why deleting 41M rows produces minutes of replica lag, referencing replay's throughput and threading.
(c) Give three primary-side changes that reduce lag and quantify each. Which gives the largest improvement?
(d) Implement LSN-based routing: the write path, the read path, the background poller, and how the LSN survives across app instances.
(e) Explain why the poller caches replay positions instead of querying per request.
(f) One of the three bugs is not a routing problem. Identify it, explain why, and state the general rule it implies.
(g) Give the settings for user-facing versus analytics replicas and explain why they are opposites.
(h) Write the four alerts. Explain why the lag alert must combine time and LSN.
(i) The health check must include `pg_is_in_recovery()`. Demonstrate the failure it prevents.
(j) After the fix, 3.6% of reads fall back to the primary. Is that good or bad? What would you do if it were 40%?

---

## Mental model checkpoint

1. What does a read replica scale, and what does it not?
2. Name the four LSNs. Which governs durability and which governs visibility?
3. Why is replay single-threaded a fundamental constraint, and what levers reduce its impact?
4. Explain a replay conflict and the two settings that control it. Why can't you just enable `hot_standby_feedback` everywhere?
5. What does `synchronous_commit = on` guarantee? What does it *not*? Which level gives read-your-writes?
6. Name the four routing strategies in order of correctness and cost.
7. Explain LSN-based routing in three sentences.
8. Why is `pg_last_xact_replay_timestamp()` alone insufficient for lag alerting?
9. Give the rule about reads that drive write decisions, and why routing can't fix it.
10. Why must a replica health check include `pg_is_in_recovery()`?
11. Why is a replica not a backup?

---

## Quick reference card

```sql
-- ★ on the primary
SELECT application_name, state, sync_state,
       pg_wal_lsn_diff(pg_current_wal_lsn(), replay_lsn) AS lag_bytes,
       write_lag, flush_lag, replay_lag
  FROM pg_stat_replication;

-- ★ on the replica — the numbers your app needs
SELECT pg_is_in_recovery(),                     -- ★ false ⇒ PROMOTED, evict
       pg_last_wal_replay_lsn(),                -- ★ visibility
       extract(epoch from now() - pg_last_xact_replay_timestamp()) AS lag_s;
-- ★ combine time AND lsn — an idle primary inflates lag_s

SELECT * FROM pg_stat_database_conflicts;       -- replay conflicts
```

**The four LSNs:** `sent → write → flush` (**durable**) `→ replay` (**★ visible**).

**`synchronous_commit`:** `off` · `local` · `remote_write` · `on` = **flushed** · ★ `remote_apply` = **applied/visible**.
`synchronous_standby_names = 'ANY 1 (r1,r2,r3)'` — ★ never a single name.

**Routing**

| Strategy | Correctness | Cost |
|---|---|---|
| by endpoint | ★ brittle | free |
| ★ sticky-after-write | window = p999 × 2 | some primary load |
| ★★ **LSN-based** | ★ **exact** | ~3% primary fallback |
| `remote_apply` | exact | ★ every write pays |

**★ Override rule:** any read used to make a **write decision** goes to the primary — or isn't a separate read at all.

**Settings**

| | user-facing | ★ analytics (dedicated) |
|---|---|---|
| `hot_standby_feedback` | `off` | ★ `on` |
| `max_standby_streaming_delay` | `10s` | `300s` |
| `statement_timeout` | `5s` | ★ `900s` (bounds primary bloat) |

**Lag is a WAL-volume problem first:** `wal_compression=lz4` · HOT updates · fewer indexes · ★ `DROP PARTITION` not `DELETE`. **Replay is single-threaded at ~100 MB/s.**

**Four alerts:** lag > 5 s **and** lag_bytes > 0 · any replay conflict on a user-facing replica · primary-fallback > 10% · ★ `pg_is_in_recovery() = false` in the read pool.

**★ A replica is not a backup — it replicates `DROP TABLE` in 3 ms.**

---

## When would I use this at work?

1. **When the primary is read-bound after you've already indexed properly.** Replicas are the right answer *after* Topic 54's gate 2 and Topic 57's CDN layer, not before. They're the most expensive read-scaling option and the only one that introduces a consistency cliff.

2. **The moment anyone proposes "route all GETs to the replica."** That sentence produces the same bug in every company. Ask for the routing policy first — and if the team can carry an LSN through request context, do that rather than guessing a stickiness window.

3. **Investigating replication lag.** The instinct is to tune the replica. The lever is almost always on the primary: replay is single-threaded, so lag is a WAL-volume problem. `DROP PARTITION` instead of `DELETE` removed 18 GB of nightly WAL in the example and took p999 lag from 41 seconds to 380 ms.

4. **Separating analytics from transactional traffic.** A dedicated analytics replica with `hot_standby_feedback = on` and a bounded `statement_timeout` is one of the cleanest wins available — but putting user traffic on that same replica gives you the worst of both settings.

---

## Connected topics

**Understand before this:** 41 (WAL — what is being shipped), 42 (crash recovery — replay is the same machinery), 46/47 (MVCC and VACUUM — why replay conflicts exist and what `hot_standby_feedback` costs), 57 (caching — the cheaper read-scaling layer to try first).

**This unlocks:**
- **59** — partitioning: the `DROP PARTITION` retention that fixes lag spikes
- **60** — sharding: what to do when *writes* are the bottleneck
- **62** — replication internals: slots, logical decoding, cascading
- **63** — HA and failover: promotion, split-brain, and why `pg_is_in_recovery()` matters
- **64** — backup and PITR: what a replica is *not*
- **68** — CAP and consistency models: where async replication sits
