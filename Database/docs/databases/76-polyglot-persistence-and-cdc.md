# 76 — Polyglot Persistence and Change Data Capture
## Phase: Beyond Relational

---

## ELI5 — The Simple Analogy

A company with one official ledger and several specialist departments.

The ledger is the truth. But the search team wants a card index sorted by keyword, the analytics team wants monthly totals, and the notifications team wants to know the moment anything changes.

★ **The wrong way: ask every clerk to write into all four places.** One clerk forgets, or gets interrupted halfway, and the four records disagree forever — with nothing to tell you which is right.

★ **The right way: one person sits beside the ledger and reads every line as it is written**, then forwards each change to whoever needs it. The clerks only ever write in the ledger. Nobody can forget, because forwarding is not their job.

★ **That reader is Change Data Capture**, and the insight that makes it work is this: **the database already writes down every change, in order, for its own crash recovery.** CDC just reads that log.

And the part that decides whether the whole thing works: ★ **the reader can fall behind, and the ledger must keep the pages until they've been read.** If the reader dies and nobody notices, the ledger room fills with paper. **That is a replication slot, and it is the single most dangerous object in this design.**

---

## Where this fits in the big picture

```
   52 the outbox · 62 logical decoding · 70–75 the specialised stores
   ★ every one of those ended with: "★ and then you own the sync"
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 76 POLYGLOT PERSISTENCE & CDC ← YOU ARE HERE │
        │ ★ how to run more than one store honestly    │
        └────────────────────┬─────────────────────────┘
                             ▼
              ★ PHASE 8 COMPLETE → Phase 9 capstones
```

★ **This is the topic that makes Phase 8 usable.** Topics 70–75 each said "adopt this only if measured, and then you own the sync problem." **This is the sync problem, solved properly** — and it closes the loop on Topic 52's dual-write problem, Topic 62's logical decoding, and Topic 51's outbox.

---

## What is this?

**Polyglot persistence**: using more than one datastore, each for what it is genuinely best at.

**Change Data Capture**: propagating every change from the system of record to the others, **by reading the database's own write-ahead log**.

```
 ★ THE FOUR WAYS TO KEEP TWO STORES IN AGREEMENT, WORST TO BEST:

 ✗ ① DUAL WRITE FROM THE APPLICATION
    db.save(x); search.index(x);
    ⇒ ★ a crash between them leaves them divergent ★ FOREVER,
      ★ silently. This is Topic 52's dual-write problem.
    ⇒ ★ AND: it is the one every team tries first.

 ✗ ② PERIODIC FULL REBUILD
    ⇒ ★ works, ★ but O(corpus) every cycle and stale in between
    ⇒ ★ acceptable ONLY for small, slowly-changing data

 ★ ③ THE TRANSACTIONAL OUTBOX (Topic 52)
    ⇒ ★ the event is written in the SAME transaction as the data
    ⇒ ★ atomic, at-least-once, ordered per aggregate
    ✗ ★ requires an application change, and an outbox table that
      is a high-churn queue (bloat — Topic 47)

 ★★ ④ CDC FROM THE WAL
    ⇒ ★ NOTHING TO FORGET. The application writes normally.
    ⇒ ★ ordered, complete, transactionally consistent
    ⇒ ★ no application change at all
    ✗ ★ a replication slot to operate (Topic 62), and ★ schema
      changes need care
```

★ **The one framing that matters:** *there is exactly **one system of record**. Everything else is a **derived view**, and a derived view must be **rebuildable from scratch** — because eventually you will have to.*

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE THE SYNC PIPELINE IS THE ACTUAL COST OF POLYGLOT
   PERSISTENCE, AND IT IS ALWAYS UNDERESTIMATED.

 ★ THE FOUR FAILURES, AND ALL FOUR ARE SILENT:
 ① ★ DIVERGENCE — the copies disagree and nothing checks
    ⇒ ★ Topic 75's example: 41 tickets/month of
      "delivered in search, in-transit in the app"
 ② ★ THE STUCK PIPELINE — the consumer dies, lag grows, ★ no error
    ⇒ ★ and the slot retains WAL until the disk fills (Topics
      41, 47, 62 — ★ the same object, a fourth time)
 ③ ★ SCHEMA DRIFT — a migration breaks the consumer, permanently
 ④ ★ NO REBUILD PATH — the derived store is corrupted and nobody
    knows how to regenerate it
    ⇒ ★ THE WORST ONE, because it is discovered during an incident

 ⇒ ★ AND THE ORGANISATIONAL VERSION:
   ★ 40% of engineering time on data plumbing (Topic 75's
   measurement). ★ That is what an unmanaged sync problem costs.
```

---

## The physical reality

### How CDC actually reads the WAL

```
 ★ THE WAL CONTAINS PAGE-LEVEL CHANGES, NOT ROWS (Topic 62).
   ⇒ ★ logical decoding reconstructs row-level events by joining
     the WAL record to the SYSTEM CATALOGS AS THEY WERE AT THAT
     MOMENT.

 ★ THE PIPELINE:
 ① a transaction commits ⇒ WAL records are durable
 ② ★ the walsender reads them for a LOGICAL replication slot
 ③ ★ the decoder resolves relation OIDs against the catalog
    ⇒ ★ this is why `catalog_xmin` is pinned, and why an abandoned
      logical slot bloats pg_attribute (Topic 62)
 ④ ★ changes are BUFFERED UNTIL COMMIT, then emitted in
    ★ COMMIT ORDER
    ⇒ ★ CONSEQUENCE: a 40M-row transaction is buffered entirely
      ⇒ ★ spills to disk in pg_replslot/ ⇒ ★ a latency cliff
      ⇒ ★ FIX: `streaming = on` (PG14+) and batch your bulk work
 ⑤ an output plugin serialises them (pgoutput, wal2json, Debezium)
 ⑥ ★ THE CONSUMER CONFIRMS AN LSN ⇒ ★ only then can the slot
    advance and the WAL be recycled

 ⇒ ★ STEP ⑥ IS THE WHOLE OPERATIONAL RISK.
   ★ A consumer that stops confirming holds WAL forever.
```

### The ordering guarantee, and its precise limits

```
 ★ WHAT CDC GUARANTEES:
   ✓ ★ changes appear in COMMIT ORDER, globally
   ✓ ★ a transaction's changes arrive together, ★ never split
   ✓ ★ at-least-once delivery, with an LSN you can checkpoint

 ★ WHAT IT DOES NOT GUARANTEE:
   ✗ ★ EXACTLY-ONCE. A consumer crash between "applied" and
     "confirmed" replays the batch.
     ⇒ ★ THE CONSUMER MUST BE IDEMPOTENT (Topic 52). ★ Always.
   ✗ ★ ORDERING ACROSS PARALLEL CONSUMERS. If you fan out by key
     for throughput, ★ you have given up global order.
     ⇒ ★ partition by the aggregate id, so per-entity order holds.
   ✗ ★ that the DERIVED STORE applies them in order — ★ that is
     the consumer's job.

 ★ AND THE ONE THAT SURPRISES PEOPLE:
   ★ CDC CAPTURES THE ROW, NOT THE INTENT.
   `UPDATE orders SET status='shipped'` arrives as a new row image.
   ⇒ ★ you cannot tell WHY it changed, or that it was a "shipment"
     rather than a correction.
   ⇒ ★ IF THE CONSUMER NEEDS INTENT, ★ THE OUTBOX IS BETTER —
     it carries a domain event, not a row diff.
   ⇒ ★ THIS IS THE REAL CDC-vs-OUTBOX DECISION, and it is not
     about mechanics.
```

### `REPLICA IDENTITY` — what the consumer can actually see

```
 ★ FROM TOPIC 62, BUT IT MATTERS DIFFERENTLY HERE:

   DEFAULT   ⇒ ★ the UPDATE event contains the NEW row and ★ ONLY
               THE PRIMARY KEY of the old row.
             ⇒ ★ you CANNOT compute a diff. You cannot tell what
               changed.
   ★ FULL    ⇒ ★ the event contains the ENTIRE OLD ROW
             ⇒ ★ diffs, audit trails and "what changed" become
               possible
             ✗ ★ substantially more WAL
             ✗ ★ and on a logical REPLICA it means a seq scan per
               change — ★ but for a CDC consumer reading a stream,
               that cost does not apply

 ⇒ ★ THE DECISION:
   ★ if the consumer only needs the current state ⇒ DEFAULT
   ★ if it needs to know WHAT CHANGED (audit, selective
     reindexing, "notify only if price changed") ⇒ ★ FULL
   ⇒ ★ MEASURED: FULL on a 40-column table increased WAL by
     ★ 2.8×. ★ Enable it per table, never globally.
```

### The rebuild — the capability that must exist before you need it

```
 ★ EVERY DERIVED STORE MUST BE REBUILDABLE FROM THE SYSTEM OF
   RECORD, ★ AND THE PROCEDURE MUST BE TESTED.

 ★ WHY YOU WILL NEED IT:
   • a consumer bug wrote wrong data for six hours
   • a mapping/schema change requires reindexing
   • ★ the slot was dropped to save the primary's disk
   • ★ the derived store was restored from an older backup than
     the source
   • ★ a full-store corruption

 ★ THE PROCEDURE, AND THE ORDER MATTERS:
 ① ★ take a CONSISTENT SNAPSHOT of the source at a KNOWN LSN
    ⇒ ★ `pg_export_snapshot()` inside a repeatable-read
      transaction, ★ or the slot's creation LSN
 ② bulk-load the derived store from that snapshot
 ③ ★ start streaming CDC FROM THAT EXACT LSN
    ⇒ ★ this is the critical step. ★ Any gap between ① and ③ is
      lost data; any overlap is replayed (★ harmless, because the
      consumer is idempotent).
 ④ ★ verify: counts and checksums, both directions
 ⑤ swap reads over

 ⇒ ★ AND THE MEASUREMENT THAT MAKES IT REAL:
   ★ HOW LONG DOES A FULL REBUILD TAKE? ★ If you cannot answer
   in minutes, you do not have a rebuild procedure — you have a
   hope.
```

### Where the lag actually comes from

```
 ★ FOUR SEPARATE LAGS, AND PEOPLE MEASURE ONLY ONE:

 ① ★ SLOT LAG — WAL generated but not yet decoded
    pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)
    ⇒ ★ grows when the consumer is slow or stopped
 ② ★ DECODE LAG — decoded but not yet sent
    ⇒ ★ grows on large transactions (buffering)
 ③ ★ TRANSPORT LAG — sent but not yet consumed (Kafka lag)
 ④ ★ APPLY LAG — consumed but not yet visible in the derived store
    ⇒ ★ Elasticsearch refresh interval (★ default 1 s), ★ a
      consumer batch window, a commit interval

 ⇒ ★ THE USER-VISIBLE NUMBER IS THE SUM, and ★ ④ is often the
   largest and the least monitored.
   ⇒ ★ MEASURED: slot 12 ms + decode 4 ms + Kafka 40 ms +
     ★ ES refresh 1,000 ms = ★ 1,056 ms. ★ 95% of it is step ④.
   ⇒ ★ AND THE FIX FOR ④ IS USUALLY A SETTING, not a pipeline.
```

### Choosing between CDC and the outbox

```
 ★ THEY ARE NOT INTERCHANGEABLE. THE DECIDING QUESTION IS
   ★ "DOES THE CONSUMER NEED INTENT, OR STATE?"

 ★ USE CDC WHEN:
   ✓ ★ the consumer wants the CURRENT STATE of a row
     (a search index, a cache, a read model, a warehouse)
   ✓ ★ you cannot change the application (a legacy system)
   ✓ ★ you want completeness — ★ nothing can be forgotten
   ✓ ★ many consumers, added over time, from one stream

 ★ USE THE OUTBOX WHEN:
   ✓ ★ the consumer needs a DOMAIN EVENT with intent
     ("OrderCancelled" ≠ "status column changed to cancelled")
   ✓ ★ the event carries data from MULTIPLE tables
   ✓ ★ the event schema must be stable while the tables evolve
   ✓ ★ you want the event to be part of the domain, ★ reviewed and
     versioned like an API

 ★ AND THE HYBRID THAT IS USUALLY RIGHT:
   ★ CDC on the OUTBOX TABLE.
   ⇒ ★ the application writes a domain event transactionally
     (the outbox's guarantee), and ★ CDC delivers it reliably
     (CDC's guarantee) — ★ with no polling relay to operate.
   ⇒ ★ THIS IS DEBEZIUM'S "OUTBOX EVENT ROUTER" PATTERN, and it
     is the best of both.
```

---

## How it works — step by step

### The CDC pipeline, configured

```ini
# ★ postgresql.conf — the source
wal_level = ★ logical
max_replication_slots = 10
max_wal_senders = 10
★ max_slot_wal_keep_size = '128GB'      # ★ THE GUARD (Topic 62)
★ logical_decoding_work_mem = '256MB'   # ★ fewer spills
```
```sql
-- ★ per-table replica identity, chosen deliberately
ALTER TABLE orders REPLICA IDENTITY DEFAULT;   -- ★ state only
ALTER TABLE prices REPLICA IDENTITY ★ FULL;    -- ★ needs "what changed"

CREATE PUBLICATION cdc_pub FOR TABLE orders, order_items, prices;
-- ★ NOT `FOR ALL TABLES` — ★ new tables would join silently (62)
```
```json
// ★ Debezium connector — the settings that matter
{
  "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
  "plugin.name": "pgoutput",
  "slot.name": "cdc_orders",
  "publication.name": "cdc_pub",

  "★ snapshot.mode": "initial",
  //   ★ takes a consistent snapshot, THEN streams from that LSN.
  //   ★ This is the rebuild procedure, built in.

  "★ heartbeat.interval.ms": "10000",
  //   ★ CRITICAL: on a low-traffic table the slot never advances,
  //     because no changes for THAT table appear in the WAL —
  //     ★ but the WAL still grows from other tables.
  //   ★ Heartbeats force the slot to advance. ★ Without this, a
  //     quiet table's slot can retain terabytes.

  "★ topic.prefix": "logistics",
  "★ tombstones.on.delete": "true",   // ★ for Kafka log compaction
  "★ transforms": "outbox",
  "transforms.outbox.type":
    "io.debezium.transforms.outbox.EventRouter"
}
```
```
 ★ THE HEARTBEAT SETTING IS THE ONE PEOPLE MISS, AND IT CAUSES
   THE CLASSIC INCIDENT: ★ a slot on a rarely-changing table
   retaining WAL indefinitely while everyone watches the busy
   table's slot, which is fine.
```

### The idempotent consumer

```js
// ★ EVERY CDC CONSUMER MUST BE IDEMPOTENT. ★ Non-negotiable.
async function handleBatch(events) {
  // ★ ① dedupe within the batch — keep the LAST event per key
  const latest = new Map();
  for (const e of events) latest.set(e.key, e);   // ★ commit-ordered

  const ops = [];
  for (const [key, e] of latest) {
    if (e.op === 'd') {
      ops.push({ delete: { _index: 'orders', _id: key } });
    } else {
      ops.push({ index: { _index: 'orders', _id: key,
                          // ★ ② LSN AS A VERSION ⇒ out-of-order
                          //   or replayed events CANNOT overwrite
                          //   newer data
                          ★ version: lsnToNumber(e.lsn),
                          ★ version_type: 'external' } });
      ops.push(toDocument(e.after));
    }
  }
  await es.bulk({ body: ops });

  // ★ ③ confirm ONLY after the apply succeeded
  await consumer.commitOffsets();
}
```
```
 ★ THE THREE MECHANISMS, AND EACH SOLVES A DIFFERENT PROBLEM:
 ① ★ BATCH DEDUPE ⇒ 1,000 updates to one row become 1 write
 ② ★ EXTERNAL VERSIONING BY LSN ⇒ ★ a replay after a crash cannot
    overwrite newer data. ★ This is what makes at-least-once safe.
 ③ ★ COMMIT AFTER APPLY ⇒ at-least-once, ★ never at-most-once
```

### The rebuild, done properly

```js
// ★ THE FULL REBUILD — ★ tested monthly, timed, and documented
async function rebuild(indexName) {
  const client = await pool.connect();
  try {
    // ★ ① a consistent snapshot at a KNOWN LSN
    await client.query('BEGIN ISOLATION LEVEL REPEATABLE READ');
    const { rows: [{ lsn }] } =
      await client.query('SELECT pg_current_wal_lsn() AS lsn');
    const { rows: [{ snap }] } =
      await client.query('SELECT ★ pg_export_snapshot() AS snap');

    // ★ ② bulk-load from THAT snapshot, in parallel workers
    await Promise.all(range(8).map(w =>
      loadShard(w, snap, indexName + '_new')));   // ★ SET TRANSACTION SNAPSHOT

    await client.query('COMMIT');

    // ★ ③ stream CDC FROM THAT EXACT LSN — ★ the critical step
    await startConsumerFrom(lsn, indexName + '_new');
    await waitForLag({ maxMs: 5000 });

    // ★ ④ verify BOTH directions before swapping
    const [srcCount, dstCount] = await Promise.all([
      pool.query('SELECT count(*) FROM orders'),
      es.count({ index: indexName + '_new' })]);
    if (Math.abs(srcCount - dstCount) > 0)
      throw new Error(`★ count mismatch: ${srcCount} vs ${dstCount}`);
    await verifyChecksums(indexName + '_new');

    // ★ ⑤ atomic swap — an alias, so reads never see a gap
    await es.indices.updateAliases({ body: { actions: [
      { remove: { index: indexName + '_old', alias: indexName } },
      { add:    { index: indexName + '_new', alias: indexName } }]}});
  } finally { client.release(); }
}
// ★ MEASURED: 8.2M documents, 8 workers ⇒ ★ 41 minutes.
//   ★ That number is the deliverable. Without it you have a hope.
```

### The reconciler — because at-least-once is not correctness

```sql
-- ★ CHEAP, CONTINUOUS: compare counts per bucket
SELECT date_trunc('day', created_at) AS day, count(*)
  FROM orders WHERE created_at >= now() - interval '7 days'
 GROUP BY 1;
-- ★ vs the same aggregation in the derived store, compared hourly
```
```js
// ★ EXPENSIVE, PERIODIC: compare checksums per bucket
async function reconcile(day) {
  const { rows: [{ h }] } = await pool.query(`
    SELECT ★ md5(string_agg(id::text || status || updated_at::text,
                            ',' ORDER BY id)) AS h
      FROM orders WHERE created_at::date = $1`, [day]);
  const esHash = await esChecksum(day);
  if (h !== esHash) {
    metrics.increment('cdc.reconcile_mismatch', { day });
    ★ await repairDay(day);          // ★ re-emit that day from source
  }
}
// ★ nightly for the last 7 days; ★ weekly for the last 90.
// ★ AND: ★ a mismatch must trigger a REPAIR, not just an alert.
```

### The three alerts that actually matter

```sql
-- ★ ① SLOT LAG AND STATUS — ★ the one that protects the PRIMARY
SELECT slot_name, active, ★ wal_status,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn))
         AS retained,
       age(catalog_xmin) AS catalog_xmin_age
  FROM pg_replication_slots WHERE slot_type = 'logical';
-- ★ ALERT: active=false · wal_status <> 'reserved' · retained > 10 GB
-- ★ AND: catalog_xmin_age growing ⇒ ★ system catalogs bloating (62)
```
```
-- ★ ② END-TO-END LAG — ★ measured with a HEARTBEAT ROW, not
--    inferred from slot lag
```
```js
// ★ write a heartbeat to the source every second; ★ measure when
//   it becomes visible in the derived store.
setInterval(async () => {
  const t = Date.now();
  await pool.query(
    'INSERT INTO cdc_heartbeat (id, at) VALUES (1, now()) ' +
    'ON CONFLICT (id) DO UPDATE SET at = now()');
  const seen = await pollDerivedStore(t, { timeoutMs: 30000 });
  ★ metrics.histogram('cdc.end_to_end_ms', seen - t);
}, 1000);
// ⇒ ★ THIS IS THE ONLY NUMBER THAT REFLECTS WHAT USERS EXPERIENCE.
//   ★ Slot lag can be zero while the derived store is 60 s behind.
```
```
-- ★ ③ RECONCILIATION MISMATCHES
-- ★ ALERT: any mismatch. ★ And it must trigger a repair.
```

---

## Concept breakdown

```
★ ONE SYSTEM OF RECORD. EVERYTHING ELSE IS A DERIVED VIEW.
   ⇒ ★ and a derived view must be REBUILDABLE — ★ because you
     will have to rebuild it

★ FOUR SYNC STRATEGIES
   ✗ dual write        ⇒ ★ divergence forever, silently (52)
   ✗ periodic rebuild  ⇒ O(corpus), stale between
   ★ the outbox        ⇒ atomic, ★ carries INTENT, needs an app
                         change and a high-churn table
   ★★ CDC from the WAL ⇒ ★ nothing to forget, ordered, complete,
                         ★ no app change
   ★ HYBRID: ★ CDC ON THE OUTBOX TABLE — ★ intent + reliability,
     ★ no polling relay (Debezium's EventRouter)

★ CDC MECHANICS (62)
   the WAL has pages, not rows ⇒ ★ decode against the catalog
   ⇒ ★ catalog_xmin pinned ⇒ ★ an abandoned slot bloats
     pg_attribute
   ★ buffered until COMMIT ⇒ ★ big transactions spill ⇒
     `streaming = on`
   ★ the consumer confirms an LSN ⇒ ★ only then can WAL recycle

★ GUARANTEES — precisely
   ✓ ★ commit order · ✓ transactions arrive whole ·
     ✓ at-least-once with an LSN
   ✗ ★ exactly-once ⇒ ★ THE CONSUMER MUST BE IDEMPOTENT
   ✗ ★ order across PARALLEL consumers ⇒ partition by aggregate id
   ✗ ★ INTENT — ★ CDC carries the ROW, not why it changed
     ⇒ ★ this is the real CDC-vs-outbox decision

★ REPLICA IDENTITY DECIDES WHAT THE CONSUMER SEES
   DEFAULT ⇒ new row + PK only ⇒ ★ no diffs possible
   ★ FULL  ⇒ the whole old row ⇒ ★ diffs, audit, selective work
             ★ +2.8× WAL, measured ⇒ ★ per table, never globally

★ FOUR LAGS, AND PEOPLE MEASURE ONE
   slot · decode · transport · ★ APPLY (★ often the largest)
   ⇒ measured: 12 + 4 + 40 + ★ 1,000 ms (ES refresh) = 1,056 ms
   ⇒ ★ MEASURE END-TO-END WITH A HEARTBEAT ROW

★ THE REBUILD IS THE DELIVERABLE
   ★ snapshot at a known LSN → bulk load → ★ stream FROM THAT LSN
   → verify both directions → ★ atomic alias swap
   ⇒ ★ and you must know how long it takes. In minutes.

★ THE THREE ALERTS
   ★ slot: inactive / wal_status / retained / catalog_xmin age
   ★ end-to-end lag (heartbeat)
   ★ reconciliation mismatch ⇒ ★ which must trigger a REPAIR
```

---

## Diagrams

**Diagram 1 — big picture: the four strategies, and where each breaks**

```
 ✗ ① DUAL WRITE
   app ──► ★ PostgreSQL  ✓ committed
       ──► ★ Elasticsearch  ⚡ CRASH
   ⇒ ★ PERMANENTLY DIVERGENT. ★ Nothing detects it.
   ⇒ ★ And reversing the order only changes which one is wrong.

 ✗ ② PERIODIC REBUILD
   cron ──► SELECT * FROM orders ──► reindex everything
   ⇒ ★ correct at the moment it finishes
   ⇒ ★ O(corpus) every cycle · ★ stale for the whole interval

 ★ ③ THE OUTBOX (Topic 52)
   app ──► BEGIN; INSERT orders; ★ INSERT outbox; COMMIT;
                                        │ ★ ONE fsync, atomic
                                        ▼
                        relay ──► broker ──► consumer
   ⇒ ★ carries a DOMAIN EVENT (intent)
   ⇒ ✗ ★ an app change, ★ a polling relay, ★ a high-churn table

 ★★ ④ CDC FROM THE WAL
   app ──► BEGIN; INSERT orders; COMMIT;
                       │
                       ▼  ★ the WAL is written ANYWAY, for recovery
              ┌──────────────────┐
              │ ★ logical decode │  ★ commit-ordered, complete
              │  (a slot)        │  ★ nothing to forget
              └────────┬─────────┘
                       ▼
                 broker ──► consumer ──► derived store
   ⇒ ★ NO APPLICATION CHANGE AT ALL
   ⇒ ✗ ★ a slot to operate · ★ carries ROWS, not intent

 ★★★ ⑤ THE HYBRID — ★ CDC ON THE OUTBOX TABLE
   app ──► BEGIN; INSERT orders; ★ INSERT outbox(domain event); COMMIT;
                                        │
                       ★ CDC reads the outbox table from the WAL
                                        ▼
                                 broker ──► consumer
   ⇒ ★ INTENT (from the outbox) ★ + RELIABILITY (from CDC)
   ⇒ ★ AND NO POLLING RELAY TO OPERATE
```

**Diagram 2 — data flow: the four lags, to scale**

```
  ★ A USER UPDATES AN ORDER. WHEN CAN THEY SEE IT IN SEARCH?

  t=0      COMMIT on PostgreSQL
           │
           │ ★ ① SLOT LAG — WAL written, not yet decoded
           │    ▓ 12 ms
           ▼
  t=12     logical decoding produces the row event
           │
           │ ★ ② DECODE LAG — buffering until commit
           │    ▓ 4 ms   (★ but a 40M-row txn ⇒ SECONDS, spilled)
           ▼
  t=16     emitted to Kafka
           │
           │ ★ ③ TRANSPORT LAG — broker + consumer poll
           │    ▓▓▓ 40 ms
           ▼
  t=56     the consumer applies the bulk request
           │
           │ ★ ④ APPLY LAG — ★ Elasticsearch refresh_interval
           │    ▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓▓ ★ 1,000 ms
           ▼
  t=1056   ★ VISIBLE TO SEARCH

 ★ 95% OF THE USER-VISIBLE LAG IS STEP ④ — ★ a SETTING, not a
   pipeline problem.
 ⇒ ★ AND IT IS THE ONE ALMOST NOBODY MONITORS, because slot lag
   (12 ms) looks perfect on the dashboard.
 ⇒ ★ THE FIX: `refresh_interval: "200ms"` (at a throughput cost),
   ★ or `?refresh=wait_for` on the write path for the one
   operation that needs read-your-writes.
```

**Diagram 3 — before/after: dual write vs CDC, on the same failure**

```
 ★ THE SCENARIO: 4,000 orders/minute. ★ The Elasticsearch cluster
   is unreachable for 6 minutes.

 ✗ DUAL WRITE
 ┌───────────────────────────────────────────────────────────────┐
 │ t=0     ES becomes unreachable                                 │
 │ t=0–6m  ★ every write: PostgreSQL ✓, Elasticsearch ✗          │
 │         ⇒ ★ the error is caught and logged... or not          │
 │         ⇒ ★ 24,000 orders committed, ★ NOT INDEXED            │
 │ t=6m    ES recovers                                            │
 │ t=6m+   ★ new writes index fine. ★ THE 24,000 ARE LOST        │
 │         FOREVER — ★ there is no record of which ones failed.   │
 │                                                                │
 │ ★ DISCOVERY: ★ a support ticket, weeks later.                 │
 │ ★ REPAIR: ★ a full reindex — ★ if the procedure exists.       │
 └───────────────────────────────────────────────────────────────┘

 ✓ CDC
 ┌───────────────────────────────────────────────────────────────┐
 │ t=0     ES becomes unreachable                                 │
 │ t=0–6m  ★ the consumer retries and does NOT commit offsets    │
 │         ⇒ ★ Kafka lag grows · ★ the slot holds WAL            │
 │         ⇒ ★ ALERTS FIRE at 60 s (★ end-to-end heartbeat)      │
 │ t=6m    ES recovers                                            │
 │ t=6m+   ★ the consumer resumes FROM ITS LAST COMMITTED OFFSET │
 │         ⇒ ★ all 24,000 are applied, ★ in commit order         │
 │         ⇒ ★ LSN versioning makes replay harmless              │
 │ t=7m    ★ lag back to 1,056 ms. ★ Nothing lost.               │
 │                                                                │
 │ ★ THE ONLY RISK: ★ if the outage lasted long enough for the   │
 │   slot to hit max_slot_wal_keep_size, the slot is invalidated  │
 │   ⇒ ★ and THAT is why the rebuild procedure must exist and    │
 │     be timed.                                                  │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**Set up logical decoding and watch it.**
```sql
ALTER SYSTEM SET wal_level = 'logical';
ALTER SYSTEM SET max_slot_wal_keep_size = '32GB';
-- restart

CREATE TABLE orders (
  id bigserial PRIMARY KEY, customer_id bigint NOT NULL,
  status text NOT NULL, total_minor bigint NOT NULL,
  updated_at timestamptz NOT NULL DEFAULT now());

SELECT ★ pg_create_logical_replication_slot('cdc_demo', 'test_decoding');
```
```sql
INSERT INTO orders (customer_id, status, total_minor) VALUES (42,'pending',25000);
UPDATE orders SET status='paid' WHERE id=1;
DELETE FROM orders WHERE id=1;

SELECT lsn, xid, data FROM ★ pg_logical_slot_peek_changes('cdc_demo', NULL, NULL);
```
```
    lsn     |  xid  |                        data
------------+-------+----------------------------------------------------
 0/1A2B3C40 | 88412 | BEGIN 88412
 0/1A2B3C78 | 88412 | table public.orders: ★ INSERT: id[bigint]:1
            |       |   customer_id[bigint]:42 status[text]:'pending' …
 0/1A2B3CB0 | 88412 | COMMIT 88412
 0/1A2B3CE8 | 88413 | BEGIN 88413
 0/1A2B3D20 | 88413 | table public.orders: ★ UPDATE: id[bigint]:1 …
 0/1A2B3D58 | 88413 | COMMIT 88413
 0/1A2B3D90 | 88414 | BEGIN 88414
 0/1A2B3DC8 | 88414 | table public.orders: ★ DELETE: ★ id[bigint]:1
 0/1A2B3E00 | 88414 | COMMIT 88414
```
```
 ★ NOTE THREE THINGS:
 ① ★ BEGIN/COMMIT framing — ★ transactions arrive whole
 ② ★ the DELETE shows ★ ONLY THE PRIMARY KEY (REPLICA IDENTITY
    DEFAULT)
 ③ ★ the UPDATE shows the NEW row only — ★ no old values
```

**Prove `REPLICA IDENTITY FULL` changes what you can see.**
```sql
ALTER TABLE orders REPLICA IDENTITY ★ FULL;
INSERT INTO orders (customer_id, status, total_minor) VALUES (42,'pending',25000);
UPDATE orders SET status='paid', total_minor=30000 WHERE id=2;
SELECT data FROM pg_logical_slot_peek_changes('cdc_demo', NULL, NULL)
 WHERE data LIKE '%UPDATE%';
```
```
 table public.orders: UPDATE: ★ old-key: id[bigint]:2
   customer_id[bigint]:42 ★ status[text]:'pending'
   ★ total_minor[bigint]:25000 …
   ★ new-tuple: id[bigint]:2 status[text]:'paid'
   total_minor[bigint]:30000 …
 ⇒ ★ NOW A DIFF IS POSSIBLE. status changed, total_minor changed.
```
```sql
-- ★ measure the WAL cost
SELECT pg_current_wal_lsn() AS a \gset
UPDATE orders SET status = status;   -- 100k rows, REPLICA IDENTITY FULL
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), :'a'));
```
```
 ★ 214 MB        vs ★ 76 MB with DEFAULT.  ★ 2.8×.
 ⇒ ★ enable FULL per table, only where diffs are needed.
```

**Prove `peek` vs `get` — and why it matters.**
```sql
SELECT count(*) FROM pg_logical_slot_★peek_changes('cdc_demo', NULL, NULL);
SELECT count(*) FROM pg_logical_slot_peek_changes('cdc_demo', NULL, NULL);
```
```
 ★ 12
 ★ 12        — ★ peek does NOT advance the slot
```
```sql
SELECT count(*) FROM pg_logical_slot_★get_changes('cdc_demo', NULL, NULL);
SELECT count(*) FROM pg_logical_slot_get_changes('cdc_demo', NULL, NULL);
```
```
 ★ 12
 ★ 0         — ★ get CONSUMES and advances the slot
 ⇒ ★ a consumer that uses `get` and then crashes ★ LOSES THAT
   BATCH. ★ Real consumers confirm an LSN only after applying.
```

**Prove an inactive slot retains WAL — the classic incident.**
```sql
-- ★ nothing consumes cdc_demo from now on
SELECT slot_name, active,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
  FROM pg_replication_slots;
```
```
 slot_name | active | retained
-----------+--------+----------
 cdc_demo  | ★ f    | 1,184 kB
```
```bash
pgbench -c 20 -j 4 -T 120 shop >/dev/null
```
```sql
SELECT slot_name, active, wal_status,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained,
       age(catalog_xmin) AS catalog_xmin_age
  FROM pg_replication_slots;
```
```
 slot_name | active | wal_status |  retained  | catalog_xmin_age
-----------+--------+------------+------------+------------------
 cdc_demo  | f      | reserved   | ★ 4,182 MB |       ★ 412,088
 ⇒ ★ 4 GB retained for a consumer that does not exist,
   ★ AND catalog_xmin is pinned ⇒ ★ pg_attribute is bloating (62).
```
```sql
-- ★ and the guard doing its job
ALTER SYSTEM SET max_slot_wal_keep_size = '100MB';
SELECT pg_reload_conf();
-- generate more WAL
SELECT slot_name, wal_status, safe_wal_size FROM pg_replication_slots;
```
```
 slot_name | wal_status | safe_wal_size
-----------+------------+---------------
 cdc_demo  | ★ lost     |         ★ -1
 ⇒ ★ the slot is invalidated. ★ The consumer must now do a FULL
   REBUILD — ★ which is exactly why that procedure must exist.
 ⇒ ★ AND THE PRIMARY SURVIVED. That is the right trade (62).
```

**Prove the heartbeat problem on a quiet table.**
```sql
-- ★ a publication for a table that rarely changes
CREATE PUBLICATION quiet_pub FOR TABLE rarely_changed;
SELECT pg_create_logical_replication_slot('quiet_slot','pgoutput');
```
```bash
# ★ generate heavy traffic on OTHER tables
pgbench -c 20 -j 4 -T 300 shop >/dev/null
```
```sql
SELECT slot_name,
       pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)) AS retained
  FROM pg_replication_slots WHERE slot_name='quiet_slot';
```
```
 slot_name  |  retained
------------+------------
 quiet_slot | ★ 8,204 MB
 ⇒ ★ THE TABLE HAD ZERO CHANGES. The slot still retained 8 GB,
   because it must hold ALL WAL until it confirms an LSN — ★ and
   it has nothing to confirm.
 ⇒ ★ THE FIX: ★ heartbeats. Debezium's heartbeat.interval.ms
   writes a marker so the slot can advance.
```

**Prove the consumer must be idempotent.**
```js
// ★ simulate a crash between apply and offset commit
const events = await consumer.poll();
await applyToElasticsearch(events);
// ★ CRASH HERE — offsets not committed
// ⇒ on restart, the SAME events are delivered again.

// ✗ without versioning: the replay overwrites newer data
await es.index({ index:'orders', id: e.key, body: doc });

// ✓ with LSN versioning: ★ the replay is rejected
await es.index({ index:'orders', id: e.key, body: doc,
                 ★ version: lsnToNumber(e.lsn),
                 ★ version_type: 'external' });
```
```
 ★ version_conflict_engine_exception — ★ which is SUCCESS here.
   ★ The newer document was preserved.
```

**Measure the four lags separately.**
```sql
-- ★ ① slot lag
SELECT pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS slot_lag_bytes
  FROM pg_replication_slots WHERE slot_name='cdc_demo';
```
```bash
# ★ ③ transport lag
kafka-consumer-groups --describe --group es-indexer | awk '{print $5, $6}'
```
```js
// ★ ④ apply lag — ★ the big one
await es.indices.getSettings({ index: 'orders' });
// ⇒ ★ "refresh_interval": "1s"
```
```js
// ★ END-TO-END, with a heartbeat — ★ the only number that matters
const t0 = Date.now();
await pool.query(`INSERT INTO cdc_heartbeat (id, at) VALUES (1, now())
                  ON CONFLICT (id) DO UPDATE SET at = now()`);
while (Date.now() - t0 < 30000) {
  const r = await es.get({ index:'heartbeat', id:'1' }).catch(() => null);
  if (r && new Date(r._source.at).getTime() >= t0) break;
  await sleep(20);
}
console.log('end-to-end', Date.now() - t0, 'ms');
```
```
 ★ end-to-end 1,056 ms
 ⇒ ★ slot lag was 12 ms. ★ The dashboard said everything was fine.
```

---

## Example 2 — production scenario

**The situation.** A logistics platform, post-consolidation (Topic 75), has decided one genuine specialised store is justified: search has grown to 31 million shipments and product now requires four uncapped facets.

```
 ★ THE DECISION, ALREADY MADE AND SIGNED:
   ★ trigger condition ① met: corpus > 25M documents
   ★ trigger condition ③ met: >2 uncapped facets is now a
     product requirement
 ⇒ ★ Elasticsearch is justified. ★ The question is now
   ENTIRELY about the sync pipeline.
```

**Step 1 — the sync design, decided before any code.**

```
 ★ FOUR QUESTIONS ANSWERED FIRST:

 ★ ① DOES THE CONSUMER NEED INTENT OR STATE?
    ⇒ ★ state. A search index wants the current shipment.
    ⇒ ★ CDC, not the outbox.

 ★ ② WHAT IS THE ACCEPTABLE END-TO-END LAG?
    ⇒ ★ asked of product: "a shipment appears in search within…"
    ⇒ ★ ANSWER: 5 seconds normally, 60 seconds at p999.
    ⇒ ★ WRITTEN DOWN. ★ This sets refresh_interval and the alert.

 ★ ③ WHAT MUST BE REBUILDABLE, AND HOW FAST?
    ⇒ ★ the entire index, ★ within 2 hours.
    ⇒ ★ this constrains the bulk-load parallelism.

 ★ ④ WHICH FIELDS MUST BE DENORMALISED?
    ⇒ ★ 88% of searches filter by tenant, carrier, status and
      date (Topic 74's measurement).
    ⇒ ★ all four must be IN the document ⇒ ★ CDC must join them.
    ⇒ ★ AND: a change to a CARRIER's name must reindex every
      shipment referencing it. ★ THIS IS THE HARD PART.
```

**Step 2 — the denormalisation problem, which is the real work.**

```
 ★ THE NAIVE CDC PIPELINE: shipments → Elasticsearch.
 ★ THE PROBLEM: the document needs `carrier_name`, which lives in
   another table.
 ⇒ ★ THREE OPTIONS:

 ⒜ ★ LOOK IT UP IN THE CONSUMER
    ⇒ ★ an N+1 against PostgreSQL, per event (Topic 66)
    ⇒ ★ MEASURED: 4,000 events/sec × 1 lookup = ★ 4,000 extra
      queries/sec. ★ Rejected.

 ⒝ ★ CACHE THE DIMENSION IN THE CONSUMER
    ⇒ ★ 412 carriers, ★ slowly changing ⇒ ★ an in-process Map,
      refreshed on its own CDC stream
    ⇒ ★ AND when a carrier changes ⇒ ★ re-emit every affected
      shipment
    ⇒ ★ CHOSEN.

 ⒞ ★ DENORMALISE INTO POSTGRESQL FIRST
    ⇒ a `carrier_name` column on `shipments`, maintained by a
      trigger (Topic 53)
    ⇒ ★ then CDC is trivial
    ✗ ★ but a carrier rename now rewrites millions of rows
      (Topic 54's fan-out) ⇒ ★ rejected on measurement:
      ★ avg 74,000 shipments per carrier.
```

```js
// ★ ⒝ IMPLEMENTED — a dimension cache with its own CDC stream
const carriers = new Map();

// ★ carrier changes come from the same CDC pipeline, a different topic
async function onCarrierChange(event) {
  carriers.set(event.after.id, event.after);
  if (event.op === 'u' && ★ event.before.name !== event.after.name) {
    // ★ THE EXPENSIVE CASE: re-emit every shipment for this carrier
    ★ await enqueueReindex({ carrier_id: event.after.id });
    metrics.increment('cdc.dimension_change_reindex');
  }
}

// ★ the reindex job — bounded, throttled, resumable
async function reindexByCarrier(carrierId) {
  let lastId = 0;
  for (;;) {
    const { rows } = await pool.query(
      `SELECT * FROM shipments WHERE carrier_id = $1 AND id > $2
        ORDER BY id LIMIT 5000`, [carrierId, lastId]);
    if (!rows.length) break;
    await es.bulk({ body: toBulk(rows) });
    lastId = rows[rows.length - 1].id;
    ★ await sleep(100);        // ★ don't starve live CDC
  }
}
// ★ MEASURED: 74,000 shipments per carrier ⇒ ★ 41 seconds.
//   ★ Carrier renames happen ~3 times a year. ★ Acceptable.
```

**Step 3 — the pipeline, configured with the numbers from step 1.**

```json
{
  "name": "shipments-cdc",
  "config": {
    "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
    "plugin.name": "pgoutput",
    "slot.name": "cdc_shipments",
    "publication.name": "cdc_pub",
    "topic.prefix": "logistics",

    "★ snapshot.mode": "initial",
    "★ snapshot.fetch.size": "10000",
    "★ heartbeat.interval.ms": "10000",
    "★ max.batch.size": "8192",
    "★ max.queue.size": "32768",
    "★ poll.interval.ms": "100",

    // ★ big transactions: stream them rather than buffering
    "★ streaming.enabled": "true",

    // ★ schema changes: fail loudly rather than silently skip
    "★ schema.history.internal.kafka.topic": "logistics.schema",
    "★ tombstones.on.delete": "true"
  }
}
```
```sql
-- ★ REPLICA IDENTITY, decided per table
ALTER TABLE shipments REPLICA IDENTITY ★ DEFAULT;
--   ⇒ ★ the consumer wants current state; ★ no diffs needed
ALTER TABLE carriers  REPLICA IDENTITY ★ FULL;
--   ⇒ ★ we MUST know if the NAME changed, to trigger the reindex
--   ⇒ ★ 412 rows ⇒ ★ the WAL cost is irrelevant
```
```
 ★ THIS IS THE DECISION IN MINIATURE: ★ FULL where a diff is
   needed and the table is small; ★ DEFAULT where only state
   matters and the table is huge.
```

**Step 4 — the consumer.**

```js
async function handleShipmentBatch(events) {
  // ★ ① dedupe within the batch, keeping the last per key
  const latest = new Map();
  for (const e of events) latest.set(e.after?.id ?? e.before.id, e);

  const body = [];
  for (const [id, e] of latest) {
    if (e.op === 'd') {
      body.push({ delete: { _index: 'shipments', _id: String(id) } });
      continue;
    }
    const s = e.after;
    const carrier = ★ carriers.get(s.carrier_id) ?? await loadCarrier(s.carrier_id);
    body.push({ index: { _index: 'shipments', _id: String(id),
                         ★ version: lsnToLong(e.source.lsn),
                         ★ version_type: 'external' } });
    body.push({
      tracking_number: s.tracking_number,
      status: s.status,
      // ★ the four denormalised filter fields (88% of searches)
      ★ tenant_id: s.tenant_id,
      ★ carrier_id: s.carrier_id,
      ★ carrier_name: carrier.name,
      ★ created_at: s.created_at,
      notes: s.carrier_attrs?.notes ?? null,
    });
  }

  const res = await es.bulk({ body, ★ refresh: false });
  // ★ ② version conflicts are EXPECTED and are SUCCESS
  const realErrors = res.items.filter(i =>
    i.index?.error && i.index.error.type !== ★ 'version_conflict_engine_exception');
  if (realErrors.length) ★ throw new Error('bulk failed');  // ★ don't commit

  // ★ ③ commit offsets only after a successful apply
  await consumer.commitOffsets();
  metrics.histogram('cdc.batch_size', latest.size);
}
```

**Step 5 — the four lags, tuned to the agreed 5-second target.**

```json
// ★ Elasticsearch index settings
{
  "settings": {
    "★ refresh_interval": "1s",         // ★ was 30s (the ES default
                                        //   for bulk-optimised indices)
    "number_of_replicas": 1,
    "★ translog.durability": "async",   // ★ throughput; ES is derived
    "★ translog.sync_interval": "5s"    //   ★ data, so this is safe
  }
}
```
```
 ★ MEASURED, END-TO-END, WITH THE HEARTBEAT:
   slot     ★ 14 ms
   decode   ★ 6 ms
   Kafka    ★ 38 ms
   ★ apply  ★ 1,010 ms        ← ★ the refresh interval
   ★ TOTAL  ★ 1,068 ms        ✓ ★ well inside the 5 s target

 ★ AND FOR THE ONE OPERATION THAT NEEDS READ-YOUR-WRITES —
   "create a shipment, then immediately see it in the list":
   ⇒ ★ the LIST READS FROM POSTGRESQL, not Elasticsearch.
   ⇒ ★ ONLY SEARCH READS FROM ELASTICSEARCH.
   ⇒ ★ THIS IS THE DESIGN DECISION THAT AVOIDS THE ENTIRE
     READ-YOUR-WRITES PROBLEM (Topic 68), and it cost nothing.
```

**Step 6 — the rebuild, built first and tested monthly.**

```js
async function fullRebuild() {
  const t0 = Date.now();
  const newIndex = `shipments_${Date.now()}`;
  await es.indices.create({ index: newIndex, body: MAPPING });

  // ★ ① consistent snapshot at a known LSN
  const client = await pool.connect();
  await client.query('BEGIN ISOLATION LEVEL REPEATABLE READ');
  const { rows: [{ lsn }] } = await client.query(
    'SELECT pg_current_wal_lsn() AS lsn');
  const { rows: [{ snap }] } = await client.query(
    'SELECT ★ pg_export_snapshot() AS snap');

  // ★ ② parallel bulk load from THAT snapshot
  await Promise.all(range(★ 8).map(async w => {
    const c = await pool.connect();
    await c.query('BEGIN ISOLATION LEVEL REPEATABLE READ');
    await c.query('★ SET TRANSACTION SNAPSHOT $1', [snap]);
    // ★ keyset pagination, not OFFSET (Topic 14)
    let last = 0;
    for (;;) {
      const { rows } = await c.query(
        `SELECT * FROM shipments WHERE id > $1 AND ★ id %% 8 = $2
          ORDER BY id LIMIT 10000`, [last, w]);
      if (!rows.length) break;
      await es.bulk({ index: newIndex, body: toBulk(rows), refresh: false });
      last = rows[rows.length - 1].id;
    }
    await c.query('COMMIT'); c.release();
  }));
  await client.query('COMMIT'); client.release();

  // ★ ③ stream CDC FROM THAT EXACT LSN — ★ the step that must
  //   not be skipped
  await startCatchupConsumer({ fromLsn: lsn, index: newIndex });
  await waitForLag({ maxMs: 5000, timeoutMs: 30 * 60_000 });

  // ★ ④ verify both directions
  const [src, dst] = await Promise.all([
    pool.query('SELECT count(*)::bigint AS n FROM shipments'),
    es.count({ index: newIndex })]);
  if (BigInt(src.rows[0].n) !== BigInt(dst.count))
    throw new Error(`★ count mismatch ${src.rows[0].n} vs ${dst.count}`);
  await ★ verifyChecksumsBySample(newIndex, 10000);

  // ★ ⑤ atomic alias swap — ★ readers never see a gap
  await es.indices.updateAliases({ body: { actions: [
    { remove: { index: '*', alias: 'shipments' } },
    { add: { index: newIndex, alias: 'shipments' } }]}});

  metrics.gauge('cdc.rebuild_seconds', (Date.now() - t0) / 1000);
}
```
```
 ★ MEASURED, 31 MILLION DOCUMENTS, 8 WORKERS: ★ 1 h 41 m.
   ★ Target was 2 hours. ✓
 ★ RUN MONTHLY IN STAGING, ★ QUARTERLY IN PRODUCTION.
 ⇒ ★ AND THE NUMBER IS ON THE DASHBOARD, so nobody has to guess
   during an incident.
```

**Step 7 — the reconciler and the three alerts.**

```js
// ★ nightly: checksum by day for the last 7 days
async function reconcileDay(day) {
  const { rows: [{ h, n }] } = await pool.query(`
    SELECT md5(string_agg(id::text || status || tracking_number,
                          ',' ORDER BY id)) AS h, count(*) AS n
      FROM shipments WHERE created_at::date = $1`, [day]);
  const es = await esChecksumForDay(day);
  if (h !== es.h || Number(n) !== es.n) {
    metrics.increment('cdc.reconcile_mismatch', { day });
    log.error({ day, pg: { h, n }, es }, '★ CDC reconciliation mismatch');
    ★ await reindexDay(day);       // ★ REPAIR, not just alert
  }
}
```
```sql
-- ★ ALERT 1 — the slot (protects the PRIMARY)
SELECT slot_name, active, wal_status,
       pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) AS retained,
       age(catalog_xmin) AS cat_age
  FROM pg_replication_slots WHERE slot_type='logical';
-- ★ page: active=false · wal_status<>'reserved' · retained>10GB
--         · cat_age > 100M
```
```
 ★ ALERT 2 — end-to-end lag via the heartbeat
   ★ warn > 5 s · ★ page > 60 s   (the agreed numbers)

 ★ ALERT 3 — any reconciliation mismatch
   ★ page, ★ and the repair runs automatically
```

**Step 8 — results, six months in.**

| | Dual-write (a rejected design) | CDC |
|---|---|---|
| Divergence incidents | ★ *(the previous system: 41/month)* | **0** |
| End-to-end lag p99 | n/a | **1,068 ms** |
| Application changes | ★ every write path | **0** |
| Data lost in a 6-min ES outage | ★ 24,000 documents | **0** |
| Rebuild time | ★ unknown | ★ **1 h 41 m, measured monthly** |
| Reconciliation mismatches | n/a | 3 in 6 months, **all auto-repaired** |
| Slot incidents | n/a | ★ 1 (a consumer OOM; ★ alert fired at 60 s) |
| Denormalised fields | 4 | 4, ★ **with a reindex path for each** |

```
 ★ SEVEN LESSONS:
 ① ★ THE FOUR QUESTIONS WERE ANSWERED BEFORE ANY CODE:
   intent-or-state, acceptable lag, rebuild time, and which fields
   must be denormalised. ★ Each one determined a configuration
   value.
 ② ★ THE DENORMALISATION WAS THE HARD PART, NOT THE PIPELINE.
   `carrier_name` in the document means a carrier rename must
   reindex 74,000 shipments — ★ and that path had to be built.
 ③ ★ 95% OF THE LAG WAS `refresh_interval`, a SETTING. ★ Slot lag
   was 14 ms and the dashboard looked perfect.
 ④ ★ READ-YOUR-WRITES WAS AVOIDED BY DESIGN: ★ lists read from
   PostgreSQL; ★ only search reads from Elasticsearch. ★ That one
   decision removed an entire class of problem for free.
 ⑤ ★ `REPLICA IDENTITY` WAS SET PER TABLE — ★ FULL on the 412-row
   dimension where diffs matter, DEFAULT on the 31M-row fact
   table where they don't.
 ⑥ ★ THE REBUILD WAS BUILT FIRST AND IS TIMED MONTHLY. When the
   slot was lost to a consumer OOM, ★ the recovery was a known
   1h41m procedure, not an improvisation.
 ⑦ ★ THE RECONCILER REPAIRS, IT DOES NOT ONLY ALERT. ★ Three
   mismatches in six months, ★ all fixed automatically before
   anyone looked.
```

---

## Common mistakes

**1. Dual-writing from the application.**
- *Symptom:* the stores diverge permanently and silently; discovered by a support ticket weeks later.
- *Fix:* CDC or an outbox. Never both writes in application code.

**2. A consumer that is not idempotent.**
- *Symptom:* a crash between apply and offset-commit replays a batch and overwrites newer data.
- *Fix:* external versioning by LSN, plus dedupe within the batch.

**3. Committing offsets before applying.**
- *Symptom:* at-most-once delivery — a crash loses the batch entirely.
- *Fix:* apply, verify, *then* commit.

**4. No `heartbeat.interval.ms`.**
- *Symptom:* a slot on a quiet table retains gigabytes of WAL because it has nothing to confirm.
- *Fix:* heartbeats, so the slot can advance regardless of table activity.

**5. No `max_slot_wal_keep_size`.**
- *Symptom:* a dead consumer fills `pg_wal` and PANICs the primary (Topics 41, 47, 62).
- *Fix:* always set it. Losing a derived store beats losing the primary.

**6. Measuring only slot lag.**
- *Symptom:* the dashboard shows 14 ms while users see 60-second-old search results.
- *Fix:* an end-to-end heartbeat. Apply lag is usually the largest and least monitored.

**7. No rebuild procedure.**
- *Symptom:* the derived store is corrupted or the slot is lost, and nobody knows how to regenerate it.
- *Fix:* build it first, test it monthly, and put the measured duration on the dashboard.

**8. Rebuilding without a known LSN.**
- *Symptom:* a gap between the snapshot and the stream start — silently missing rows.
- *Fix:` `pg_export_snapshot()` inside a repeatable-read transaction, then stream from that exact LSN.

**9. Forgetting that denormalised dimension fields need a reindex path.**
- *Symptom:* a carrier is renamed and 74,000 documents still show the old name.
- *Fix:* CDC the dimension table too, with `REPLICA IDENTITY FULL`, and trigger a bounded reindex on change.

**10. `REPLICA IDENTITY FULL` everywhere.**
- *Symptom:* 2.8× the WAL for tables where no consumer needs diffs.
- *Fix:* per table. `FULL` only where a diff is genuinely required.

**11. Expecting CDC to carry intent.**
- *Symptom:* the consumer cannot distinguish a cancellation from a correction.
- *Fix:* the outbox for domain events — ideally CDC *on* the outbox table.

**12. No reconciler, or a reconciler that only alerts.**
- *Symptom:* drift accumulates; a mismatch alert fires and nobody knows what to do.
- *Fix:* checksum by bucket, and make a mismatch trigger an automatic repair.

**13. Not deciding where read-your-writes reads come from.**
- *Symptom:* "I created it and it's not there" for every derived-store read path.
- *Fix:* route the operations that need immediacy to the system of record. Often free.

**14. Parallelising consumers without partitioning by aggregate.**
- *Symptom:* out-of-order application within one entity.
- *Fix:* partition by the aggregate id, so per-entity order is preserved.

---

## Hands-on proof

**PROVE IT #1–#8 — Example 1** (logical decoding output with `BEGIN`/`COMMIT` framing, `REPLICA IDENTITY FULL` revealing old values at 2.8× the WAL, `peek` vs `get` and why a naive consumer loses a batch, an inactive slot retaining 4 GB and pinning `catalog_xmin`, `max_slot_wal_keep_size` invalidating it, a quiet table's slot retaining 8 GB without heartbeats, external versioning rejecting a replay, and the four lags summing to 1,056 ms when slot lag was 12).

**PROVE IT #9 — `pg_export_snapshot` makes a parallel load consistent.**
```sql
-- session 1
BEGIN ISOLATION LEVEL REPEATABLE READ;
SELECT pg_export_snapshot();       -- ★ 00000003-0000001A-1
SELECT count(*) FROM shipments;    -- 31,204,882
```
```sql
-- session 2, concurrently
INSERT INTO shipments (...) VALUES (...);   -- ★ 1,000 new rows
```
```sql
-- session 3
BEGIN ISOLATION LEVEL REPEATABLE READ;
★ SET TRANSACTION SNAPSHOT '00000003-0000001A-1';
SELECT count(*) FROM shipments;
```
```
 ★ 31,204,882        — ★ the SAME as session 1, not 31,205,882.
 ⇒ ★ all parallel workers see one consistent instant.
```

**PROVE IT #10 — measure the rebuild, don't estimate it.**
```bash
time node scripts/rebuild.js --index shipments --workers 8
```
```
 ★ real 101m14s        — ★ put this number on the dashboard.
```
```bash
# ★ and prove parallelism matters
for w in 1 4 8 16; do
  echo -n "workers=$w  "
  /usr/bin/time -f '%e s' node scripts/rebuild.js --workers $w --dry-run 2>&1 | tail -1
done
```
```
 workers=1   ★ 41,204 s
 workers=4   ★ 11,882 s
 workers=8   ★ 6,074 s
 workers=16  ★ 5,912 s      ★ diminishing — the source is now the limit
```

**PROVE IT #11 — a version conflict is success, not failure.**
```js
const res = await es.bulk({ body: replayedBatch });
const conflicts = res.items.filter(i =>
  i.index?.error?.type === 'version_conflict_engine_exception').length;
console.log({ total: res.items.length, conflicts });
```
```
 ★ { total: 8192, conflicts: 8192 }
 ⇒ ★ a full replay applied ZERO changes. ★ Exactly correct.
```

**PROVE IT #12 — the reconciler finds real drift.**
```js
// ★ deliberately corrupt one document
await es.update({ index:'shipments', id:'8842119',
                  body:{ doc:{ status:'delivered' }}});
await reconcileDay('2026-08-26');
```
```
 ★ ERROR: CDC reconciliation mismatch
   { day: '2026-08-26', pg: { h: 'a3f9…', n: 41204 },
     es: { h: '★ 7b2c…', n: 41204 } }
 ★ reindexDay('2026-08-26') — repaired in 1.2 s
 ⇒ ★ note the counts MATCHED. ★ Only the checksum caught it.
```

---

## The design decision framework

```
★★★ ONE SYSTEM OF RECORD. EVERYTHING ELSE IS A DERIVED VIEW,
    AND A DERIVED VIEW MUST BE REBUILDABLE. ★★★

 ① ★ ANSWER FOUR QUESTIONS BEFORE WRITING ANY CODE
    ★ ⒜ DOES THE CONSUMER NEED INTENT OR STATE?
       state ⇒ ★ CDC · intent ⇒ ★ the outbox
       ⇒ ★ or the hybrid: ★ CDC ON THE OUTBOX TABLE
    ★ ⒝ WHAT END-TO-END LAG IS ACCEPTABLE?
       ⇒ ★ ask PRODUCT, in seconds. ★ It sets refresh intervals,
         batch sizes and the alert threshold.
    ★ ⒞ HOW FAST MUST A FULL REBUILD BE?
       ⇒ ★ it constrains parallelism, and ★ it is the number you
         will need during an incident.
    ★ ⒟ WHICH FIELDS MUST BE DENORMALISED?
       ⇒ ★ count how many queries FILTER. ★ Each denormalised
         dimension needs its own reindex-on-change path.

 ② ★ NEVER DUAL-WRITE
    ⇒ ★ the divergence is permanent and silent (52).

 ③ ★ THE CONSUMER MUST BE IDEMPOTENT — THREE MECHANISMS
    ✓ ★ dedupe within the batch (last write per key)
    ✓ ★ external versioning by LSN ⇒ ★ replays cannot overwrite
       newer data
    ✓ ★ commit offsets ONLY after a successful apply

 ④ ★ OPERATE THE SLOT (Topics 41, 47, 62)
    ✓ ★ max_slot_wal_keep_size — ★ ALWAYS
    ✓ ★ heartbeat.interval.ms — ★ or a quiet table's slot retains
       everything
    ✓ ★ alert on: inactive · wal_status ≠ reserved · retained size
       · ★ catalog_xmin age
    ⇒ ★ this is the same object that caused incidents in Topics
      41, 47 and 62. ★ Treat it accordingly.

 ⑤ ★ MEASURE END-TO-END, NOT SLOT LAG
    ★ four lags: slot · decode · transport · ★ APPLY
    ⇒ ★ apply is usually the largest and least monitored
    ⇒ ★ use a HEARTBEAT ROW. It is the only number users feel.

 ⑥ ★ SET `REPLICA IDENTITY` PER TABLE
    DEFAULT for large fact tables where only state matters
    ★ FULL for small dimension tables where a diff triggers work
    ⇒ ★ FULL costs 2.8× the WAL, measured.

 ⑦ ★ BUILD THE REBUILD FIRST, AND TIME IT
    ★ snapshot at a known LSN (`pg_export_snapshot`)
    → parallel bulk load → ★ stream FROM THAT LSN
    → ★ verify counts AND checksums → ★ atomic alias swap
    ⇒ ★ test monthly. ★ Put the duration on the dashboard.

 ⑧ ★ RECONCILE AND REPAIR
    ★ counts continuously, ★ checksums by bucket nightly
    ⇒ ★ a mismatch must TRIGGER A REPAIR, not just an alert.
    ⇒ ★ counts can match while checksums differ.

 ⑨ ★ DECIDE WHERE READ-YOUR-WRITES READS COME FROM
    ⇒ ★ route immediacy-sensitive reads to the SYSTEM OF RECORD.
    ⇒ ★ in the worked example this removed an entire class of
      problem for free.

 ⑩ ★ AND WRITE THE CONSISTENCY CONTRACT (Topic 68)
    per operation: which store · which model · which bound ·
    what happens when the pipeline is down. ★ Signed off.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Create a logical replication slot and use `pg_logical_slot_peek_changes` to observe `INSERT`, `UPDATE` and `DELETE`. Then: (a) show what `REPLICA IDENTITY DEFAULT` reveals about an update versus `FULL`; (b) measure the WAL difference; (c) demonstrate `peek` versus `get` and explain why a real consumer must confirm an LSN separately.

### Exercise 2 — medium (apply it)
Build a CDC consumer that indexes into a second store. Make it idempotent using external LSN versioning. Then: (a) kill it mid-batch and prove no data is lost; (b) replay a batch and prove version conflicts occur and are correct; (c) stop it entirely, generate load, and show the slot retaining WAL; (d) set `max_slot_wal_keep_size` low and show the slot being invalidated.

Finally, measure all four lags separately and with a heartbeat.

### Exercise 3 — hard (production simulation)
A platform has justified Elasticsearch (31M documents, four uncapped facets required). Design the sync pipeline.

(a) Answer the four questions before any code, and say what configuration value each one determines.
(b) 88% of searches filter by four fields that live in other tables. Give three options for denormalising them and choose one with a measurement.
(c) A carrier rename must update 74,000 documents. Design the reindex path, including throttling, and say why the alternative (a trigger-maintained column in PostgreSQL) was rejected.
(d) Set `REPLICA IDENTITY` per table and justify each choice.
(e) Write the idempotent consumer with all three mechanisms. Explain why a version conflict is a success.
(f) Measure the four lags. 95% is one number — which, and what is the fix?
(g) One operation needs read-your-writes. Give the design decision that avoids it entirely.
(h) Write the full rebuild, including `pg_export_snapshot` and the atomic swap. Explain why streaming must start from the snapshot's exact LSN.
(i) Write the reconciler. Explain why counts alone are insufficient.
(j) Write the three alerts with thresholds, and say which protects the primary database rather than the derived store.

---

## Mental model checkpoint

1. Name the four sync strategies and the specific failure of each of the first two.
2. What does "CDC carries the row, not the intent" mean, and when does it force the outbox?
3. What is the hybrid pattern, and what does it take from each parent?
4. Why must a CDC consumer be idempotent? Name the three mechanisms.
5. Why must offsets be committed *after* applying?
6. What does `heartbeat.interval.ms` prevent, and why does a quiet table need it most?
7. Name the four lags. Which is usually largest and least monitored?
8. How do you measure end-to-end lag correctly?
9. What does `REPLICA IDENTITY FULL` enable, and what does it cost?
10. Give the five steps of a correct rebuild. Why must streaming start at the snapshot's LSN?
11. Why must a reconciler compare checksums and not just counts?
12. Why does routing immediacy-sensitive reads to the system of record remove a whole class of problem?

---

## Quick reference card

**★ The four strategies:** dual write ✗ · periodic rebuild ✗ · outbox (**intent**) · ★ **CDC** (**state, nothing forgotten**) · ★★ **CDC on the outbox table** (both).

```ini
wal_level = logical
★ max_slot_wal_keep_size = '128GB'     # ★ always
★ logical_decoding_work_mem = '256MB'
```
```json
"★ heartbeat.interval.ms": "10000",   // ★ quiet tables retain WAL without it
"★ snapshot.mode": "initial",          // ★ the rebuild, built in
"★ streaming.enabled": "true"          // ★ big transactions don't spill
```

**★ Idempotent consumer — three mechanisms**
```js
① ★ dedupe within the batch (last per key)
② ★ version: lsnToLong(e.lsn), version_type: 'external'
③ ★ commitOffsets() ONLY after a successful apply
// ★ a version_conflict_engine_exception is SUCCESS
```

**★ Four lags:** slot · decode · transport · ★ **apply**. Measured: 14 + 6 + 38 + ★ **1,010** ms. ★ **Use a heartbeat row.**

**★ `REPLICA IDENTITY`:** `DEFAULT` for large fact tables (state only) · ★ `FULL` for small dimensions where a diff triggers work (★ **+2.8% WAL... 2.8×**).

**★ Rebuild** — snapshot at a known LSN (`pg_export_snapshot`) → parallel load → ★ stream **from that LSN** → verify counts **and** checksums → ★ atomic alias swap. ★ **Time it. Test it monthly.**

**★ Three alerts:** slot (inactive / `wal_status` / retained / `catalog_xmin` age) — ★ **protects the primary** · end-to-end lag · reconciliation mismatch → ★ **which repairs**.

**★ Route read-your-writes reads to the system of record.**

---

## When would I use this at work?

1. **The moment a second datastore is justified.** The query language is the easy part; this is the engineering. Answering the four questions first — intent-or-state, acceptable lag, rebuild time, which fields to denormalise — turns a vague project into four concrete configuration values.

2. **Whenever anyone proposes dual-writing.** It is the first thing every team tries and it diverges permanently and silently. The demonstration — a six-minute outage losing 24,000 documents with no record of which — usually ends the discussion.

3. **Day one on any system with a replication slot.** Check `active`, `wal_status`, retained size and `catalog_xmin` age. This is the same object behind incidents in Topics 41, 47 and 62, and an abandoned one is a latent primary-database outage with no timer on it.

4. **Before you need the rebuild.** Build it first, time it, test it monthly, and put the number on a dashboard. When a slot is lost or a consumer writes bad data for six hours, the difference between a known 1h41m procedure and an improvisation is the difference between an inconvenience and an incident.

---

## Connected topics

**Understand before this:** 52 (the outbox and the dual-write problem), 62 (logical decoding, slots, `REPLICA IDENTITY`), 41/47 (why an abandoned slot is dangerous), 68 (the consistency contract), 70–75 (each store's sync obligation).

**This unlocks:**
- **77–79** — the Phase 9 capstones, where the pipeline is part of the deliverable
- **Case study 10** — product catalogue and search · **Case study 20** — the analytics warehouse

---

> ### ★ PHASE 8 COMPLETE — Beyond Relational (70–76)
>
> **The arc:** **70** documents — and `jsonb` already is one · **71** the key *is* the query language · **72** one table per query, and the partition key is permanent · **73** cardinality decides everything · **74** the two shapes where the relational cost model breaks · **75** the five genuine limits and the seven gates · **76** how to run more than one store honestly.
>
> **The thread running through all seven:** every specialised store is **dramatically better at exactly one thing and worse at everything else** — and PostgreSQL covers most of that one thing up to a scale most systems never reach. The recurring measurement is the same each time: **96% of "search" was an exact lookup, the "graph" had 412 nodes, 4 of 350 attributes were 96% of filters, and 8.8 million series existed because of one label.** Measuring the actual workload killed most of the proposals — and when one survived, **the sync pipeline, not the query language, was the work.**
>
> **Next: Phase 9 — Capstones (77–79).** Design a complete schema from requirements, scale it under pressure, and review it the way a principal engineer would.
