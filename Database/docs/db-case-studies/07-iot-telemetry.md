# 07 — IoT Telemetry
## When retention is the schema, and `DELETE` is the enemy

> **Read the brief. Close the file. Design it yourself. Then come back.**

---

## WHY THIS CASE STUDY EXISTS

Case studies 05 and 06 were fan-out problems: one write becoming many. This one is simpler and, at scale, more brutal — **one write staying one write, 400,000 times a second, forever.**

Nothing here is conceptually hard. There are no races, no invariants that can be violated, no double-spends. The entire difficulty is arithmetic: 400k rows/sec × 86,400 seconds × 90 days is **3.1 trillion rows**, and every naive decision multiplies a number that is already enormous.

The lesson that generalises: **at high ingest volume, the schema's job is to make deletion free.** If retention costs anything per row, you lose — not eventually, but on a schedule you can compute in advance.

---

## 1. THE BRIEF

An industrial IoT platform monitoring factory equipment.

> **200,000 devices, each emitting 2 readings/sec across ~8 metrics.**
> **400,000 rows/sec sustained. 5 TB/month raw.**
> **Retention: 90 days at full resolution, 2 years downsampled, 7 years for a regulated subset.**

Business rules:

1. **Never lose a reading** that was acknowledged — regulated equipment, audited.
2. **Late and out-of-order arrival is normal.** Edge gateways buffer during connectivity loss and flush up to 6 hours later.
3. **Duplicates are normal** — a gateway that doesn't get an ack resends.
4. **Dashboards** show "last 24 h for device X" and "fleet average per hour."
5. **Alerting** needs "any device whose temperature exceeded threshold in the last 5 minutes."
6. **Ad-hoc analysis** over months of history for the data-science team.
7. **Retention must be enforceable** and provable to an auditor.

Infrastructure: PostgreSQL 16 + TimescaleDB, Kafka, 60 Node.js ingest workers, S3.

---

## 2. ACCESS PATTERN TABLE

| # | Operation | Peak rate | Rows read | Rows written | p99 budget | Staleness |
|---|---|---|---|---|---|---|
| A | Ingest — batched from Kafka | 400,000 rows/s | 0 | 5,000/batch | 2 s | 0 |
| B | Device detail — last 24 h, one device, 8 metrics | 2,000/s | ~1,400 | 0 | 300 ms | 10 s |
| C | Fleet dashboard — hourly averages, 24 h, all devices | 400/s | **34,560,000** | 0 | 500 ms | 60 s |
| D | Alerting — threshold breach in the last 5 min | every 10 s | ~24,000,000 | ~50 | 5 s | 0 |
| E | Ad-hoc analysis — arbitrary range, arbitrary devices | 20/hour | up to 10^11 | 0 | minutes | 1 h |
| F | Retention — drop expired data | daily | — | — | n/a | n/a |
| G | Downsample — roll 90-day data into hourly | hourly | ~1.4B | ~1.7M | n/a | n/a |

**What jumps out:**

- **A is the volume**, and it's append-only. Structurally the easiest thing in the brief — *if* you don't sabotage it with indexes.
- **C reads 34.5 million rows per request, 400 times a second.** That is 13.8 billion rows/second. It cannot be served from raw data under any circumstances. It must be pre-aggregated.
- **F is the one people forget**, and it's the one that kills the system. 400k rows/sec means a 90-day `DELETE` touches 3.1 trillion rows.
- **E has no bounded shape** — which means it belongs somewhere other than the OLTP database.

---

## 3. THE INVARIANTS

```
I1.  DURABILITY: a reading acknowledged to the gateway is never lost.

I2.  IDEMPOTENCE: the same reading ingested N times is stored once.
     (device_id, metric, recorded_at) is the natural key.

I3.  RETENTION IS PROVABLE: no reading older than its class's retention
     window exists anywhere, and this is demonstrable to an auditor.

I4.  DOWNSAMPLE FIDELITY: an hourly rollup covers exactly the readings
     in that hour — no gaps, no double-counting, even with late arrivals.

I5.  MONOTONIC ROLLUPS: a rollup, once sealed, never changes. Late data
     arriving after sealing is accounted for separately, not silently
     merged. (Case study 04's `sealed_at`.)
```

**I3 is the one that dictates the physical design.** "Provable retention" plus "3.1 trillion rows" means deletion must be a metadata operation, not a row operation. That single requirement forces partitioning before you write any DDL.

---

## 4. ATTEMPT 1 — THE NAIVE SCHEMA

```sql
CREATE TABLE readings (
  id          bigserial   PRIMARY KEY,
  device_id   bigint      NOT NULL REFERENCES devices(id),
  metric      text        NOT NULL,
  value       double precision NOT NULL,
  recorded_at timestamptz NOT NULL,
  ingested_at timestamptz NOT NULL DEFAULT now()
);
CREATE INDEX idx_readings_device_time ON readings (device_id, recorded_at DESC);
CREATE INDEX idx_readings_time ON readings (recorded_at);
CREATE INDEX idx_readings_metric ON readings (metric);
```

```js
// the ingest worker
for (const r of batch) {
  await db.query(
    `INSERT INTO readings (device_id, metric, value, recorded_at)
     VALUES ($1,$2,$3,$4)`, [r.deviceId, r.metric, r.value, r.recordedAt]);
}
```

This is a perfectly ordinary schema. It works flawlessly at 400 rows/sec. Every single element of it fails at 400,000.

---

## 5. WHERE IT BREAKS

### 5.1 The row is 3× larger than the data

```
 TUPLE LAYOUT (Topic 05), in declaration order:
   24  header
    8  id          bigint
    8  device_id   bigint
   ~14 metric      text  ('temperature' = 1 hdr + 11)
    2  pad
    8  value       float8
    8  recorded_at timestamptz
    8  ingested_at timestamptz
   ───
   80 bytes + 4 (line pointer) = 84 bytes per reading

 THE ACTUAL INFORMATION: device (8) + metric (1) + value (4) + time (8)
                       = 21 bytes.
 ⇒ 4× overhead. At 400k/s:
      84 B × 400,000 × 86,400 = 2.9 TB/day of heap
   plus indexes (below), plus WAL.
```

`metric` as `text`, repeated 34.5 billion times a day for 8 distinct values, is 14 bytes where 1 would do. `ingested_at` is 8 bytes nobody queries. `id` is 8 bytes that serves no purpose — `(device_id, metric, recorded_at)` is already unique.

### 5.2 The indexes cost more than the data

```sql
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid)), idx_scan
FROM pg_stat_user_indexes WHERE relname='readings';
```
```
        indexrelname          |  size   |  idx_scan
------------------------------+---------+-----------
 readings_pkey                | 1.9 TB  |         0     ← ⚠ never used
 idx_readings_device_time     | 2.8 TB  |  41200884
 idx_readings_time            | 1.9 TB  |      8102
 idx_readings_metric          | 1.4 TB  |         0     ← ⚠ 8 distinct values
```

**8 TB of indexes on 2.9 TB/day of data**, two of which are never used and one of which (Topic 17) the planner will never choose because `metric` has 8 distinct values.

And the write cost:

```
 4 indexes × ~80 B of WAL each + heap ≈ 470 B of WAL per reading
 × 400,000/s = 188 MB/s of WAL
 = 16 TB/day, shipped to every replica and archived for PITR.
```

### 5.3 Ingest cannot reach 400k/s

```bash
pgbench -f /tmp/single_insert.sql -c 60 -j 8 -T 60
```
```
tps = 18,204          ← target 400,000. Short by 22×.
```

Three separate causes:

```
 ① ONE ROW PER STATEMENT
    Each INSERT is a network round trip + a parse + a plan + a commit
    fsync. At ~3 ms per round trip, one connection does 330/s.

 ② FOUR B-TREE INSERTS PER ROW
    idx_readings_device_time is keyed on (device_id, recorded_at).
    device_id is effectively RANDOM across 200,000 devices, so every
    insert lands in a random leaf of a 2.8 TB index. Once that index
    exceeds RAM — which it does within a day — every insert is a random
    disk read + a dirty page. (Topic 11's cliff.)

 ③ THE FOREIGN KEY
    REFERENCES devices(id) runs an indexed lookup and takes FOR KEY
    SHARE on the parent row, per insert. At 400k/s across 200k devices
    that is real contention on hot devices (Topic 23).
```

### 5.4 The fleet dashboard is arithmetically impossible

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT date_trunc('hour', recorded_at) AS h, metric, avg(value)
FROM readings
WHERE recorded_at >= now() - interval '24 hours'
GROUP BY 1,2;
```
```
Finalize GroupAggregate  (actual time=184102.4..184110.2 rows=192 loops=1)
  ->  Gather Merge (workers=4)
        ->  Partial GroupAggregate
              ->  Sort  (actual rows=8640000 loops=4)
                    Sort Method: external merge  Disk: 1204880kB
                    ->  Parallel Seq Scan on readings (actual rows=8640000 loops=4)
  Buffers: shared hit=41204 read=42104882
Execution Time: 184112.8 ms
```

**184 seconds, 344 GB read, for one dashboard load.** At 400 requests/second the required throughput is 137 TB/second. There is no configuration of any database that serves this from raw rows. **Pattern C must be pre-aggregated. That is not an optimisation; it is the only possibility.**

### 5.5 Retention is a multi-day outage

```sql
DELETE FROM readings WHERE recorded_at < now() - interval '90 days';
```

```
 Rows to delete: 400,000/s × 86,400 × 1 day = 34.5 BILLION per day of
 retention drift.

 EACH DELETE:
   • sets t_xmax on the heap tuple (Topic 46) — the row stays
   • leaves 4 dead index entries
   • generates WAL
   • must later be vacuumed, which re-reads every page

 34.5 billion × (1 heap + 4 index) dead entries = 172 billion dead
 entries per day of retention.

 MEASURED: the DELETE ran for 31 hours and was killed. Autovacuum never
 caught up. The table grew every single day DESPITE the deletion.
```

**This is the failure that ends the design.** Not slow queries — an unbounded table that cannot be pruned.

### 5.6 Duplicates and late data have no handling

I2 requires idempotence; there is no unique constraint, so a gateway resend duplicates the reading and every average is wrong. And there is no notion of "this hour is final," so a rollup computed at 01:00 differs from the same rollup computed at 07:00 — violating I4 and I5.

---

## 6. ATTEMPT 2 — THE FIX THAT ISN'T ENOUGH

**Batch the inserts and drop the useless indexes.** The two obvious fixes.

```js
// COPY instead of INSERT — one round trip for 5,000 rows
const stream = client.query(copyFrom(
  `COPY readings (device_id, metric_id, value, recorded_at) FROM STDIN BINARY`));
for (const r of batch) stream.write(encodeBinary(r));
```

```sql
DROP INDEX idx_readings_metric;      -- 8 distinct values, never used
DROP INDEX idx_readings_time;        -- 8,102 scans in 90 days
ALTER TABLE readings DROP CONSTRAINT readings_device_id_fkey;   -- + reconciliation job
```

Measure:

```
 ingest: 18,204/s → 214,000/s          ← 11.7×
 WAL:    188 MB/s → 71 MB/s
 index storage: 8 TB → 4.7 TB
```

**A large, genuine win. And the design is still dead:**

| Still broken | Why batching doesn't help |
|---|---|
| **Retention (5.5)** | unchanged — still 34.5B `DELETE`s/day |
| **Fleet dashboard (5.4)** | unchanged — still 184 s |
| Ingest ceiling | 214k/s, target 400k/s. `idx_readings_device_time` is still a random-insert B-tree larger than RAM |
| Row width | unchanged — still 84 B for 21 B of information |
| Idempotence (I2) | unchanged — and adding a `UNIQUE` index would add back a random-insert index |
| Late data (I4/I5) | unchanged |

**The realisation:** batching fixes the *statement* overhead. It does nothing about the two structural problems — **a B-tree that must accept random inserts forever, and a table that cannot be pruned.** Both are solved by the same thing.

---

## 7. THE PRODUCTION DESIGN

```
 PROBLEM                          SOLUTION                          TOPIC
 ─────────────────────────────────────────────────────────────────────────
 Retention = 34.5B DELETEs   →  PARTITION BY TIME; retention = DROP   59
 Random-insert B-tree > RAM  →  Per-partition indexes; the ACTIVE
                                one is small enough to cache          11
 84 B/row for 21 B of data   →  Type discipline + column order        05,25
 Dashboard reads 34.5M rows  →  CONTINUOUS AGGREGATES (rollups)       55,56
 Ad-hoc over months          →  COMPRESSED chunks + columnar          06
 Duplicates / late data      →  ON CONFLICT + sealed watermarks       52,04
```

### 7.1 The schema

```sql
-- ═══════════════════════════════════════════════════════════════════
-- REFERENCE DATA — small, cached, never in the hot path
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE devices (
  id            bigint GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
  serial        text   NOT NULL UNIQUE,
  site_id       int    NOT NULL REFERENCES sites(id),
  retention_class smallint NOT NULL DEFAULT 1,   -- 1=90d 2=2y 3=7y regulated
  installed_at  timestamptz NOT NULL DEFAULT now()
);

CREATE TABLE metrics (
  id   smallint PRIMARY KEY,        -- ★ 2 bytes, not 14
  code text     NOT NULL UNIQUE,    -- 'temperature','vibration',…
  unit text     NOT NULL
);

-- ═══════════════════════════════════════════════════════════════════
-- READINGS — the hot table. Every byte and every index justified.
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE readings (
  recorded_at timestamptz NOT NULL,   -- 8 B, first: it's the partition key
  device_id   bigint      NOT NULL,   -- 8 B
  value       real        NOT NULL,   -- ★ 4 B: float4 gives 6 significant
                                      --   digits — ample for a sensor whose
                                      --   own accuracy is ±0.5%
  metric_id   smallint    NOT NULL,   -- ★ 2 B instead of 14
  quality     smallint    NOT NULL DEFAULT 0,   -- sensor status flags
  PRIMARY KEY (recorded_at, device_id, metric_id)
) PARTITION BY RANGE (recorded_at);
--   ★ NO surrogate id — the natural key IS unique (I2)
--   ★ NO ingested_at — nobody queries it
--   ★ NO foreign key to devices — 400k/s cannot afford the check;
--     a nightly reconciliation job covers it (Topic 23's discipline)

-- TUPLE: 24 hdr + 8 + 8 + 4 + 2 + 2 = 48 B, no padding, +4 line pointer
--        = 52 B, vs 84 B. 38% smaller, and every scan is 38% cheaper.

-- ★ THE ONLY INDEX. It IS the primary key, and it serves pattern B.
--   Per-partition, so the active one is ~9 GB, not 2.8 TB.
--   Older partitions' indexes are cold but read-only — that's fine.

-- ═══════════════════════════════════════════════════════════════════
-- ROLLUPS — patterns C, D, E read these. Never `readings`.
-- ═══════════════════════════════════════════════════════════════════
CREATE TABLE readings_1m (
  bucket      timestamptz NOT NULL,
  device_id   bigint      NOT NULL,
  metric_id   smallint    NOT NULL,
  n           integer     NOT NULL,
  sum_value   double precision NOT NULL,   -- ★ sum + n, NOT avg (see 7.4)
  min_value   real        NOT NULL,
  max_value   real        NOT NULL,
  sealed_at   timestamptz NULL,            -- ★ I5
  PRIMARY KEY (bucket, device_id, metric_id)
) PARTITION BY RANGE (bucket);

CREATE TABLE readings_1h (LIKE readings_1m INCLUDING ALL) PARTITION BY RANGE (bucket);
CREATE TABLE readings_1d (LIKE readings_1m INCLUDING ALL) PARTITION BY RANGE (bucket);

-- ★ FLEET-LEVEL rollup: pattern C aggregates across ALL devices, so
--   pre-aggregate that dimension away too.
CREATE TABLE fleet_1h (
  bucket    timestamptz NOT NULL,
  site_id   int         NOT NULL,
  metric_id smallint    NOT NULL,
  n         bigint      NOT NULL,
  sum_value double precision NOT NULL,
  min_value real        NOT NULL,
  max_value real        NOT NULL,
  PRIMARY KEY (bucket, site_id, metric_id)
);
--   192 rows per hour for the whole fleet. Pattern C reads 192 rows.
```

**Every decision justified:**

| Decision | Why |
|---|---|
| Partition by `recorded_at` | ★ retention becomes `DROP TABLE` — O(1), zero dead tuples (I3) |
| `recorded_at` first in the PK | it's the partition key; must be in the PK (Topic 22) |
| `real` not `double precision` | 4 B vs 8 B × 3.1 trillion rows = **12 TB saved**; 6 significant digits exceeds sensor accuracy |
| `smallint metric_id` | 2 B vs 14 B × 3.1T = **37 TB saved** |
| No `id` column | the natural key is unique; 8 B × 3.1T = 25 TB saved, plus a whole index |
| No `ingested_at` | 8 B nobody queries |
| **One index total** | it's the PK. Every additional index is ~80 B of WAL per row at 400k/s |
| No FK to `devices` | the check + `FOR KEY SHARE` is unaffordable at this rate. **Replaced by a nightly reconciliation job** — the explicit trade from Topic 23 |
| `sum` + `n`, not `avg` | ★ averages don't compose; sums do (see 7.4) |
| `sealed_at` on rollups | I5 — a sealed bucket never changes |

### 7.2 Partitioning — the design that makes everything else possible

```sql
-- Daily partitions: 34.5B rows/day is already large for one table.
-- pg_partman pre-creates them.
SELECT partman.create_parent(
  p_parent_table := 'public.readings',
  p_control      := 'recorded_at',
  p_interval     := '1 day',
  p_premake      := 14                     -- 14 days of runway
);
```

```
 WHY DAILY, NOT MONTHLY OR HOURLY:

  HOURLY   → 2,160 partitions for 90 days. Planning time balloons
             (Topic 18: the planner must consider each partition),
             and pg_class bloats.
  DAILY    → 90 partitions. ★ Each holds 34.5B rows / 52 B ≈ 1.8 TB.
             Hmm — that's still large. Which tells you something:
  ⇒ AT THIS VOLUME, ONE DIMENSION ISN'T ENOUGH.

 ★ COMPOSITE PARTITIONING: by day, then sub-partitioned by hash(device_id)
   CREATE TABLE readings_2026_08_09 PARTITION OF readings
     FOR VALUES FROM ('2026-08-09') TO ('2026-08-10')
     PARTITION BY HASH (device_id);
   -- 16 hash sub-partitions per day
   ⇒ each leaf partition: ~113 GB, index ~9 GB — ★ fits in RAM
   ⇒ 90 days × 16 = 1,440 leaf partitions. Manageable.
   ⇒ AND ingest spreads across 16 B-trees instead of hammering one,
     which is what finally breaks the write ceiling.
```

**The retention operation:**

```sql
-- Retention class 1 (90 days) — O(1), no dead tuples, no vacuum
ALTER TABLE readings DETACH PARTITION readings_2026_05_11 CONCURRENTLY;
-- export to S3 as Parquet for the audit trail (I3), then:
DROP TABLE readings_2026_05_11;
```
```
 34.5 billion rows removed in ~200 ms.
 vs the naive DELETE: 31 hours, killed, and the table grew anyway.
```

⚠ **Retention classes complicate this.** Devices in class 3 (7 years, regulated) live in the same partitions as class 1. Two options:

```
 (a) SEPARATE TABLES per retention class.
     readings_std (90d) and readings_regulated (7y), both partitioned.
     ✓ DROP works per class
     ✗ the ingest worker must route by device class
     ★ CHOSEN — routing is one lookup in a cached map, and it keeps
       retention provable per class (I3).

 (b) One table; before dropping a partition, copy out the regulated rows.
     ✗ reintroduces a row-level operation on the retention path.
       That is exactly the thing partitioning exists to avoid.
```

### 7.3 Ingest — batched, idempotent, and spread

```js
const BATCH = 5_000;

async function ingest(rows) {                 // from a Kafka consumer
  // ① route by retention class (a cached map, refreshed every 5 min)
  const std = [], reg = [];
  for (const r of rows) (deviceClass.get(r.deviceId) === 3 ? reg : std).push(r);

  await Promise.all([
    copyInto('readings_std', std),
    copyInto('readings_regulated', reg),
  ]);
}

async function copyInto(table, rows) {
  if (!rows.length) return;
  const client = await pool.connect();
  try {
    // ② COPY into a per-worker UNLOGGED staging table.
    //    UNLOGGED = no WAL for the staging write. Safe because Kafka
    //    is the durable source: on crash we replay the offset. (I1)
    await client.query(`CREATE TEMP TABLE stg (LIKE ${table}) ON COMMIT DROP`);
    const stream = client.query(copyFrom(`COPY stg FROM STDIN BINARY`));
    for (const r of rows) stream.write(encodeBinary(r));
    await finished(stream);

    // ③ MERGE into the real table. ON CONFLICT gives I2 for free —
    //    the PK is (recorded_at, device_id, metric_id).
    await client.query(`
      INSERT INTO ${table} (recorded_at, device_id, value, metric_id, quality)
      SELECT recorded_at, device_id, value, metric_id, quality FROM stg
      ON CONFLICT (recorded_at, device_id, metric_id) DO NOTHING`);
  } finally { client.release(); }
}
```

```
 WHY THE STAGING STEP:
   • COPY is ~10× faster than multi-row INSERT (no per-row parse)
   • the INSERT…SELECT is ONE statement, so the PK's B-tree inserts
     are sorted by the copy order → far better locality than 5,000
     independent inserts
   • ON CONFLICT DO NOTHING handles gateway resends (I2) with no
     extra index and no SELECT
   • the temp table is dropped on commit — no cleanup

 MEASURED: 18,204/s → 214,000/s (batching alone)
                   → 438,000/s (batching + hash sub-partitions)
 ⇒ ★ the hash sub-partitioning is what crossed 400k, because it
   spread the B-tree inserts across 16 trees instead of one.
```

### 7.4 Rollups — and the detail that matters most

```sql
-- Every 1 minute: roll the previous complete minute
INSERT INTO readings_1m (bucket, device_id, metric_id, n, sum_value, min_value, max_value)
SELECT date_trunc('minute', recorded_at), device_id, metric_id,
       count(*), sum(value::double precision), min(value), max(value)
FROM readings
WHERE recorded_at >= $1 AND recorded_at < $2
GROUP BY 1,2,3
ON CONFLICT (bucket, device_id, metric_id) DO UPDATE
  SET n         = readings_1m.n + EXCLUDED.n,
      sum_value = readings_1m.sum_value + EXCLUDED.sum_value,
      min_value = LEAST(readings_1m.min_value, EXCLUDED.min_value),
      max_value = GREATEST(readings_1m.max_value, EXCLUDED.max_value)
  WHERE readings_1m.sealed_at IS NULL;      -- ★ I5: never touch a sealed bucket
```

```
 ★★★ WHY sum + n, AND NOT avg ★★★

 AVERAGES DO NOT COMPOSE.
   avg(avg(a,b), avg(c,d,e))  ≠  avg(a,b,c,d,e)
   Rolling minute-averages into an hourly average by averaging them
   weights each minute equally — but minutes have different row counts
   when data is missing or late. The result is silently WRONG.

 SUMS AND COUNTS COMPOSE PERFECTLY:
   hourly_avg = sum(minute.sum_value) / sum(minute.n)
   ⇒ EXACT, and computable at any level from any lower level.

 THE SAME APPLIES TO:
   ✓ compose: sum, count, min, max
   ✗ do NOT compose: avg, median, percentiles, distinct counts
   ⇒ for percentiles you must store a SKETCH (t-digest / HdrHistogram),
     not a value. TimescaleDB's `percentile_agg` does exactly this.
     For distinct counts, HyperLogLog.

 ★ THIS IS THE SINGLE MOST COMMON BUG IN TIME-SERIES ROLLUPS, and it
   is invisible: the dashboard shows plausible numbers that are wrong.
```

```sql
-- Cascade upward: 1m → 1h → 1d, always from the level below
INSERT INTO readings_1h (bucket, device_id, metric_id, n, sum_value, min_value, max_value)
SELECT date_trunc('hour', bucket), device_id, metric_id,
       sum(n), sum(sum_value), min(min_value), max(max_value)
FROM readings_1m WHERE bucket >= $1 AND bucket < $2 AND sealed_at IS NOT NULL
GROUP BY 1,2,3
ON CONFLICT … ;

-- Fleet level: aggregate the device dimension away
INSERT INTO fleet_1h (bucket, site_id, metric_id, n, sum_value, min_value, max_value)
SELECT r.bucket, d.site_id, r.metric_id,
       sum(r.n), sum(r.sum_value), min(r.min_value), max(r.max_value)
FROM readings_1h r JOIN devices d ON d.id = r.device_id
WHERE r.bucket >= $1 AND r.bucket < $2
GROUP BY 1,2,3
ON CONFLICT … ;

-- Sealing: 6 h after the bucket ends (the late-arrival window)
UPDATE readings_1m SET sealed_at = now()
WHERE sealed_at IS NULL AND bucket < now() - interval '6 hours';
```

**Late data after sealing** goes to `late_readings` and is reported separately — never silently merged, because that would change a number a dashboard already showed (I5, case study 04's lesson).

### 7.5 The queries, now trivial

```sql
-- Pattern C: fleet dashboard, hourly, 24 h
SELECT bucket, metric_id, sum(sum_value)/sum(n) AS avg_value,
       min(min_value), max(max_value)
FROM fleet_1h
WHERE bucket >= now() - interval '24 hours'
GROUP BY 1,2 ORDER BY 1;
```
```
 GroupAggregate (actual time=0.4..0.8 rows=192 loops=1)
   ->  Index Scan using fleet_1h_pkey (actual rows=4608 loops=1)
   Buffers: shared hit=42
 Execution Time: 0.9 ms         ← was 184,113 ms.  204,000×.
```

```sql
-- Pattern B: one device, 24 h, raw resolution
SELECT recorded_at, metric_id, value FROM readings
WHERE device_id = $1
  AND recorded_at >= now() - interval '24 hours'
ORDER BY recorded_at DESC;
```
⚠ **Note the ordering problem:** the PK is `(recorded_at, device_id, metric_id)`, so `device_id = $1` is not a leftmost prefix (Topic 14). Partition pruning limits it to one day's partitions, but within those it's a scan.

```
 THE FIX — and it is a real trade:
   (a) add a second index (device_id, recorded_at) per partition
       → +80 B of WAL per row at 400k/s = +32 MB/s. Expensive but
         bounded, and pattern B is 2,000/s.
   (b) reorder the PK to (device_id, recorded_at, metric_id)
       → ✗ recorded_at must be the leading partition key… actually no:
         the partition key must be IN the PK, not FIRST. So this works:
         PRIMARY KEY (device_id, recorded_at, metric_id)
       → ★ CHOSEN. Pattern B becomes an index scan, ingest keeps one
         index, and partition pruning still works on recorded_at.
       ⚠ COST: inserts now go to a random leaf (device_id is random),
         which is why the hash sub-partitioning matters — it keeps each
         B-tree small enough to stay cached.
```

### 7.6 Compression — the last 10× on storage

```sql
-- TimescaleDB: compress chunks older than 7 days
ALTER TABLE readings SET (
  timescaledb.compress,
  timescaledb.compress_segmentby = 'device_id, metric_id',
  timescaledb.compress_orderby   = 'recorded_at DESC'
);
SELECT add_compression_policy('readings', INTERVAL '7 days');
```

```
 HOW IT WORKS (Topic 06): rows are rewritten COLUMNAR within a chunk.
   recorded_at → delta-encoded (consecutive, 2 s apart)  → ~1 bit each
   device_id   → run-length encoded within a segment      → ~0 bits
   metric_id   → RLE                                      → ~0 bits
   value       → delta-of-delta / XOR (Gorilla)           → ~2 B each

 MEASURED: 113 GB per leaf partition → 8.4 GB.  13.5×.
 ⇒ 90 days: 10.2 TB → 760 GB.

 ⚠ COMPRESSED CHUNKS ARE EFFECTIVELY READ-ONLY. Late data cannot be
   inserted into them without decompressing. That is exactly why the
   compression policy is 7 days and the late-arrival window is 6 hours —
   the gap is deliberate.
```

### 7.7 Ad-hoc analysis (pattern E) leaves the database

```
 20 queries/hour over arbitrary ranges, up to 10^11 rows, minutes of
 latency tolerated. This has NO bounded shape, and running it on the
 OLTP primary would evict the entire working set (Topic 07).

 ⇒ Export sealed daily partitions to S3 as Parquet, and query with
   DuckDB / Athena / Spark.
   ✓ columnar, compressed, arbitrarily parallel
   ✓ zero impact on ingest
   ✓ ★ and it doubles as the retention audit trail (I3) — the Parquet
     files are the provable record that data existed and was removed
     on schedule.
```

---

## 8. THE NUMBERS

| Metric | Attempt 1 | Attempt 2 | Production |
|---|---|---|---|
| Ingest rate | 18,204/s | 214,000/s | **438,000/s** |
| Bytes per row (heap) | 84 | 84 | **52** → **~4 compressed** |
| Indexes | 4 (8 TB) | 1 (2.8 TB) | **1, per-partition, ~9 GB active** |
| WAL | 188 MB/s | 71 MB/s | **31 MB/s** |
| 90-day storage | ~31 TB + 8 TB idx | ~31 TB | **760 GB compressed** |
| Fleet dashboard (C) | 184,113 ms | 184,113 ms | **0.9 ms** |
| Device detail (B) | 2,104 ms | 2,104 ms | **3.2 ms** |
| Alerting (D) | 41,000 ms | 41,000 ms | **12 ms** |
| Retention (F) | **31 h, killed** | same | **200 ms (`DROP`)** |
| Duplicates handled | no | no | **yes (`ON CONFLICT`)** |
| Rollup correctness | n/a | n/a | **exact (sum+n)** |

---

## 9. FAILURE MODES AND WHAT YOU MONITOR

| Failure | What happens | Design response |
|---|---|---|
| **Partition not pre-created** | Ingest fails at midnight with "no partition found" | `pg_partman` premakes 14 days. **Alert if runway < 5 days** — this is the #1 partitioning outage |
| **Gateway floods 6 h of buffered data** | 8.6M rows arrive at once | Kafka absorbs it; the ingest worker is rate-limited. Batches are bounded, so no single transaction is huge |
| **Late data after sealing** | Would change a shown number | Goes to `late_readings`, reported separately (I5). Alert if late volume > 0.5% |
| **A device sends garbage (NaN, ±∞)** | `real` accepts them; averages become NaN | `CHECK (value = value AND value BETWEEN -1e30 AND 1e30)` — the `value = value` idiom rejects NaN |
| **Clock skew on a gateway** | Readings land in the wrong partition, or the future | `CHECK (recorded_at < now() + interval '5 minutes')` at ingest; quarantine the rest |
| **Compression policy runs on a chunk with late data** | Late inserts fail | 7-day compression vs 6-hour late window — a deliberate 6.75-day gap |
| **Rollup worker dies** | Dashboards go stale, raw data intact | Rollups are **derived and rebuildable** from `readings` (standing rule #3). Alert on rollup lag |
| **Reconciliation finds unknown device_ids** | Orphan readings (no FK) | Nightly job; quarantine and alert. **This is the price of dropping the FK, and it must actually be paid** |
| **Disk fills** | Everything stops | Retention + compression are the controls. Alert at 70%, because `DROP PARTITION` needs no headroom but compression does |

**The five alerts:**

```sql
-- 1. Partition runway — the classic 3am outage
SELECT max(to_date(substring(relname from '\d{4}_\d{2}_\d{2}'),'YYYY_MM_DD')) - current_date
FROM pg_class WHERE relname ~ '^readings_\d{4}_\d{2}_\d{2}$';
-- ALERT if < 5

-- 2. Ingest lag (Kafka consumer offset vs head)
-- ALERT if > 60 s

-- 3. Rollup lag
SELECT now() - max(bucket) FROM readings_1m;
-- ALERT if > 3 min

-- 4. Late data after sealing
SELECT count(*) FROM late_readings WHERE ingested_at > now() - interval '1 hour';
-- ALERT if > 0.5% of ingest

-- 5. Orphan readings (the FK we deliberately dropped)
SELECT count(DISTINCT r.device_id) FROM readings r
LEFT JOIN devices d ON d.id = r.device_id
WHERE r.recorded_at > now() - interval '1 day' AND d.id IS NULL;
-- ALERT if > 0
```

---

## 10. WHAT BREAKS AT 10×

**2M devices, 4M rows/sec, 50 TB/month.**

| Wall | Why | Next design |
|---|---|---|
| Single-node write capacity | 4M rows/s exceeds one PostgreSQL primary | **Shard by `device_id`.** Devices are independent — no cross-device queries at the raw level, so this shards perfectly (Topic 60) |
| Rollup computation | 4M rows/s to aggregate every minute | Move rollups into the **stream** (Kafka Streams / Flink). PostgreSQL receives only 1-minute rollups; raw data goes straight to S3 |
| Partition count | 1,440 leaf partitions × 10 shards | Keep leaf count per node under ~2,000 — planning time grows with it (Topic 18) |
| S3 export | 50 TB/month of Parquet | Fine — that's what object storage is for. Partition the Parquet by date and site for pruning |
| Alerting (D) | 24M rows per evaluation | Evaluate in the stream, not the database. The DB stores alert *state*, not the evaluation |

**At 10×, the pattern inverts: PostgreSQL stops being the ingest target and becomes the serving layer for rollups.** Raw data lives in object storage. That is the natural endpoint of every telemetry system.

---

## 11. THE SEVEN QUESTIONS

1. Why is `DELETE`-based retention structurally impossible at 400k rows/sec? Compute the dead-tuple count for one day of retention drift.
2. Why does `sum` + `n` compose correctly into higher-level rollups while `avg` does not? Give a concrete example where averaging averages gives the wrong answer.
3. Why is `real` acceptable for a sensor value where `float8` would be wasteful, and why is neither acceptable for money?
4. The naive schema had 4 indexes. Why is exactly one correct here, and what specifically justifies that one?
5. Explain why hash sub-partitioning by `device_id` was what finally broke the write ceiling, when daily partitioning alone did not.
6. Why is the compression policy 7 days when the late-arrival window is 6 hours? What breaks if you set compression to 12 hours?
7. The FK to `devices` was deliberately dropped. What did that buy, what did it cost, and what do you now owe?

---

## 12. TRANSFERABLE LESSONS

**1. At high volume, retention is the schema.**
Ask "how does data leave?" *before* "how does data arrive." If deletion costs anything per row, the table becomes unmanageable on a schedule you can compute in advance. Partition by the retention dimension from day one — retrofitting it onto a 30 TB table is a quarter of work. → Reappears in **08 (audit log)**, **18 (scan events)**, and any table with a time-based lifecycle.

**2. Store `sum` and `count`, never `avg`.**
Sums, counts, mins and maxes compose into higher aggregates exactly. Averages, medians, percentiles and distinct counts do not — they need sketches (t-digest, HyperLogLog). This bug is invisible: the dashboard shows plausible wrong numbers for years. → Reappears in **13 (leaderboards)**, **14 (ad metrics)**, **20 (analytics warehouse)**.

**3. Every byte is multiplied by the row count — so audit types before you ship.**
`text` → `smallint` for `metric` saved 37 TB. `float8` → `real` saved 12 TB. Dropping a useless `id` saved 25 TB plus an entire index. **At a trillion rows, type selection is an architecture decision, not a style preference.** → Topics 05, 25.

**4. Index count is the ingest ceiling.**
Each index is ~80 B of WAL and one random B-tree insert per row. At 400k/s, four indexes is 188 MB/s of WAL and four random-write streams. **Ask of every index: which named query, at what rate?** — and be willing to answer "none, drop it." → Topic 17.

**5. Pre-aggregate anything read more often than it is written.**
The fleet dashboard reads 34.5M rows 400 times a second and writes nothing. Reading it from raw data is 137 TB/s; reading it from a rollup is 192 rows. **When reads outnumber writes on the same data, move the work to write time.** → Reappears in **05 (timelines)**, **06 (unread counts)**, **13**, **19**.

**6. Seal your buckets, and account for late data separately.**
`sealed_at` is what makes a rollup reproducible. Without it, the same query at 01:00 and 07:00 returns different numbers and nobody can say which was right. Late data goes to a separate table and is *reported*, not silently merged. → Case study 04's lesson, applied to time-series.

**7. Dropping a foreign key is a trade you must pay for.**
Removing the FK to `devices` bought real throughput at 400k/s. It cost referential integrity, and the payment is a nightly reconciliation job plus an alert. **An FK dropped without a reconciliation job is not an optimisation — it's a bug you haven't found.** → Topic 23.

---

## FILES TO READ NEXT

- **08 — Audit log** (this folder): the same partition-and-drop shape, plus tamper evidence and legal retention
- **20 — Analytics warehouse** (this folder): where pattern E properly lives
- **59 — Partitioning** (curriculum): the mechanism this whole design rests on
- **73 — Time-series data modelling** (curriculum): the general theory
- **06 — Row vs column store** (curriculum): why compression works so well here
- **25 — Data types** (curriculum): the byte-level decisions that saved 74 TB
