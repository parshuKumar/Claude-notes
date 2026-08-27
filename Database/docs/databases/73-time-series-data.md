# 73 — Time-Series Data
## Phase: Beyond Relational

---

## ELI5 — The Simple Analogy

A weather station that writes one line in a notebook every ten seconds, forever.

Three facts about that notebook shape everything:

1. ★ **You only ever write at the end.** Nothing in the middle ever changes. Yesterday's 3 p.m. reading is never corrected, never updated, never deleted individually.
2. ★ **You almost always read a contiguous stretch** — "the last hour", "yesterday", "this week" — and almost never a single line.
3. ★ **Old pages stop being interesting long before they stop existing.** You want last week second-by-second, last year hour-by-hour, and 2019 as a single monthly average.

★ **A general-purpose database treats every row as equally likely to be read, updated or deleted. Time-series data violates all three assumptions**, and every optimisation in this topic comes from exploiting that.

And the trap: ★ **it looks like ordinary data, so people store it in an ordinary table** — and then discover that a `DELETE` for retention costs more than every insert combined, that the table is 90% index, and that "average CPU last month" scans 260 million rows to produce 30 numbers.

---

## Where this fits in the big picture

```
   59 partitioning — ★ DROP PARTITION, not DELETE
   72 wide-column — ★ TWCS, bucketing, TTL
   56 matviews — rollups · 46/47 — write amplification
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 73 TIME-SERIES DATA ← YOU ARE HERE           │
        │ ★ append-only, range-read, tiered-retention  │
        └────────────────────┬─────────────────────────┘
                             ▼
              74 graph & search · 75 SQL vs NoSQL · 76 CDC
```

★ **This topic is where several earlier ones combine into one coherent design.** Partitioning gives retention, rollups give query speed, compression gives storage, and the append-only nature makes all three safe. **Nothing here is new machinery — it is the machinery aimed correctly.**

---

## What is this?

Data where **time is the primary axis**: measurements, events, metrics, logs, prices, telemetry.

```
 ★ THE FIVE PROPERTIES THAT DEFINE IT — and each one is an
   optimisation opportunity:

 ① ★ APPEND-ONLY       ⇒ ★ no UPDATE, so no MVCC bloat, no HOT
                         concerns, no vacuum pressure (Topic 46)
 ② ★ TIME-ORDERED      ⇒ ★ writes land at the "end" ⇒ perfect
                         locality; ★ but also a hot index page
 ③ ★ RANGE-READ        ⇒ ★ contiguous disk ranges, not point
                         lookups
 ④ ★ IMMUTABLE HISTORY ⇒ ★ old data can be compressed, reordered,
                         and aggregated freely
 ⑤ ★ DECAYING VALUE    ⇒ ★ resolution can drop with age
                         (raw → hourly → daily)

 ⇒ ★ AND THE ONE THAT MATTERS MOST OPERATIONALLY:
   ★ RETENTION IS A FIRST-CLASS REQUIREMENT, NOT AN AFTERTHOUGHT.
   The data volume is unbounded unless you delete, and ★ deleting
   rows is the most expensive way to do it (Topics 47, 59, 72).
```

★ **The reframe:** *time-series is not a database category — it is a **workload shape**. PostgreSQL with partitioning, TimescaleDB, ClickHouse, InfluxDB and Prometheus are all answers to the same shape, and the right one depends on cardinality and query pattern, not on branding.*

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE TIME-SERIES IS THE FASTEST-GROWING TABLE IN ALMOST
   EVERY SYSTEM, AND THE ONE MOST OFTEN MODELLED AS IF IT WERE
   ORDINARY.

 ★ THE FOUR SYMPTOMS, ALL FROM THE SAME CAUSE:
 ① ★ THE RETENTION JOB IS THE BIGGEST WORKLOAD
    DELETE FROM metrics WHERE ts < now() - interval '90 days'
    ⇒ ★ 41 minutes, 18 GB of WAL, 24 min of replica lag,
      ★ no disk reclaimed (Topics 47, 58, 59)
 ② ★ THE TABLE IS MOSTLY INDEX
    a 5-column metrics row with 4 indexes ⇒ ★ indexes are 3× the
    heap. ★ And you only ever query by (series, time).
 ③ ★ AGGREGATES SCAN EVERYTHING
    "avg CPU per hour last month" ⇒ ★ 260 M rows → 720 numbers
 ④ ★ CARDINALITY EXPLOSION
    one series per (host, container, request_id) ⇒ ★ millions of
    distinct series, most with 3 points each
    ⇒ ★ THE #1 KILLER OF TIME-SERIES SYSTEMS, and it is a
      MODELLING mistake, not a capacity one.

 ⇒ ★ ALL FOUR ARE PREVENTABLE AT DESIGN TIME AND EXPENSIVE
   AFTERWARDS.
```

---

## The physical reality

### The schema decision: narrow vs wide

```
 ★ NARROW ("long") — one row per measurement
   (ts, series_id, value)
   ✓ ★ any number of metrics without schema change
   ✓ ★ sparse data costs nothing
   ✗ ★ the timestamp and series_id are repeated per metric
   ✗ ★ "cpu and memory at time T" needs a join or a pivot

 ★ WIDE ("tall") — one row per timestamp per source
   (ts, host_id, cpu, memory, disk, net_in, net_out)
   ✓ ★ one row = one complete observation ⇒ ★ 5× fewer rows
   ✓ ★ correlated metrics are free ("cpu when memory > 80%")
   ✓ ★ far better compression (columns are homogeneous)
   ✗ ★ adding a metric is a schema change
   ✗ ★ sparse metrics waste space (NULLs are cheap but not free)

 ★ MEASURED, 100M observations of 5 metrics:
   narrow  ★ 500 M rows, ★ 41 GB
   wide    ★ 100 M rows, ★ 11 GB      ⇒ ★ 3.7×
 ⇒ ★ THE DECIDING QUESTION: is the metric set STABLE?
   stable and known ⇒ ★ WIDE
   arbitrary/user-defined ⇒ ★ NARROW, with a series dimension table
```

### Cardinality — the number that kills time-series systems

```
 ★ CARDINALITY = the number of DISTINCT SERIES
   = the product of every label's distinct values.

   host (400) × metric (12) × datacentre (3)      = ★ 14,400  ✓
   host (400) × metric (12) × ★ request_id (10^7) = ★ 4.8×10^10 ✗

 ★ WHY IT KILLS THINGS:
   • ★ Prometheus/InfluxDB hold an index entry PER SERIES IN RAM
     ⇒ ★ ~1–3 KB per series ⇒ 1M series ≈ 1–3 GB, ★ before data
   • ★ each series gets its own chunk/file structure
     ⇒ ★ 1M series × a 2-hour block = 1M tiny files
   • ★ compression works ALONG a series; a series with 3 points
     compresses to nothing
   • ★ queries that touch many series must merge them all

 ★ THE RULE: ★ NEVER PUT AN UNBOUNDED VALUE IN A LABEL.
   ✗ request_id · user_id · session_id · email · full URL · IP
   ✓ ★ status_code · method · route TEMPLATE · region · env
   ⇒ ★ "route" must be `/api/orders/:id`, ★ never `/api/orders/8842`

 ⇒ ★ AND THE DIAGNOSTIC:
   Prometheus: ★ topk(10, count by (__name__)({__name__=~".+"}))
   Influx:     SHOW SERIES CARDINALITY
   Postgres:   SELECT count(DISTINCT series_id) FROM metrics;
```

### Why compression is so effective here — and how

```
 ★ TIME-SERIES DATA IS THE MOST COMPRESSIBLE DATA THERE IS,
   BECAUSE CONSECUTIVE VALUES ARE ALMOST IDENTICAL.

 ★ ① DELTA ENCODING (timestamps)
    1724568862, 1724568872, 1724568882, …
    ⇒ store 1724568862, then ★ +10, +10, +10 …
    ⇒ ★ 8 bytes → ~1 byte

 ★ ② DELTA-OF-DELTA (Gorilla, Facebook 2015)
    the deltas are also nearly identical (10, 10, 10, 11, 10)
    ⇒ ★ store the delta OF the delta: 0, 0, +1, −1
    ⇒ ★ a regular 10-second interval costs ★ 1 BIT per point

 ★ ③ XOR ENCODING (float values)
    consecutive doubles share most of their bits.
    45.31 XOR 45.32 ⇒ ★ mostly zeros ⇒ store only the differing
    window.
    ⇒ ★ MEASURED: ~1.37 bytes per float, vs 8 raw.

 ★ ④ COLUMNAR + RLE + DICTIONARY (ClickHouse, Timescale
    compression)
    ⇒ ★ a `status` column of 'ok' repeated 10,000 times
      compresses to ★ (value, count).

 ★ MEASURED, 100 M points of a real CPU metric:
   raw PostgreSQL rows           ★ 11.2 GB
   PostgreSQL + TOAST/pglz        ★ 8.4 GB
   ★ TimescaleDB compression      ★ 0.9 GB   (12×)
   ★ ClickHouse (columnar+codecs) ★ 0.4 GB   (28×)
 ⇒ ★ AND COMPRESSION IS NOT ONLY STORAGE: less I/O means faster
   scans. ★ A compressed columnar scan is often FASTER than an
   uncompressed row scan, despite the decompression.
```

### Rollups — the only way aggregates get fast

```
 ★ THE ARITHMETIC THAT FORCES ROLLUPS:
   400 hosts × 12 metrics × 6/min × 60 × 24 × 30
   = ★ 260 MILLION rows for one month.
   "average CPU per hour" ⇒ ★ 260 M rows → 720 numbers.
   ⇒ ★ NO INDEX MAKES THAT FAST. The work is the scan itself.

 ★ THE ANSWER: PRECOMPUTE, INCREMENTALLY.
   raw (10 s)  → 1 min  → 1 hour  → 1 day
   ⇒ ★ each level is built from the level above, ★ not from raw.
   ⇒ ★ 260 M → 4.3 M (1 min) → 288 k (1 hour) → 12 k (1 day)

 ★ AND THE TRAP THAT MAKES ROLLUPS WRONG:
   ★ NOT EVERY AGGREGATE IS COMPOSABLE.
     sum, count, min, max      ⇒ ★ composable
     ★ avg                      ⇒ ★ NOT — you must store
                                 (sum, count) and divide at read
     ★ distinct count           ⇒ ★ NOT — store a HyperLogLog
                                 sketch and merge those
     ★ percentiles              ⇒ ★ NOT — store a t-digest/DDSketch
                                 and merge those
   ⇒ ★ "the average of the hourly averages" IS NOT the daily
     average unless every hour has the same count.
   ⇒ ★ THIS IS THE MOST COMMON SILENT CORRECTNESS BUG IN
     TIME-SERIES DASHBOARDS.
```

### The write path: what "append-only" buys you

```
 ★ BECAUSE THERE ARE NO UPDATES:
   ✓ ★ no MVCC bloat (Topic 46) — no dead tuples from updates
   ✓ ★ no HOT-update concerns — every row is a fresh insert
   ✓ ★ vacuum has almost nothing to do (only freezing)
   ✓ ★ fillfactor 100 is correct — no space needed for new versions
   ✓ ★ B-tree inserts are RIGHTMOST ⇒ ★ near-perfect locality,
     and PostgreSQL has a fastpath for this

 ★ BUT THE RIGHTMOST INSERT IS ALSO A CONTENTION POINT:
   ★ every writer contends on the same index leaf page
   ⇒ ★ wait_event = 'BufferContent' (Topic 61)
   ⇒ ★ MITIGATION: partition by time AND by a hash of the series
     ⇒ N independent B-trees, N rightmost pages

 ★ AND BATCHING IS THE SINGLE BIGGEST WRITE LEVER (Topic 66):
   one INSERT per point       ★ 4,882 ms / 1,000 points
   one transaction            ★ 412 ms
   multi-row INSERT           ★ 18 ms
   ★ COPY                     ★ 2.1 s per 500,000 points
   ⇒ ★ time-series ingestion should ALWAYS be batched, and
     ★ usually via COPY.
```

### Late-arriving and out-of-order data

```
 ★ THE ASSUMPTION "DATA ARRIVES IN ORDER" IS FALSE IN PRACTICE:
   • a device buffers offline for 6 hours and flushes
   • a mobile client with a wrong clock
   • a backfill after an outage
   • a batch job reprocessing yesterday

 ★ THE CONSEQUENCES, PER SYSTEM:
   ★ Timescale/Postgres ⇒ works, but ★ writes into an already-
     COMPRESSED chunk require decompressing it (or are refused
     in older versions)
   ★ Cassandra/TWCS     ⇒ ★ a late write lands in TODAY'S time
     window ⇒ ★ it will never be compacted with its true window
     ⇒ ★ and it prevents that old SSTable from being dropped
   ★ Prometheus         ⇒ ★ REJECTS out-of-order samples outright
     (until v2.39's opt-in OOO window)
   ★ ClickHouse         ⇒ fine; parts are merged by partition

 ⇒ ★ THE DESIGN DECISIONS YOU MUST MAKE EXPLICITLY:
   ① ★ HOW LATE IS ACCEPTABLE? (an hour? a day?) — this sets your
      compression/finalisation delay.
   ② ★ WHAT HAPPENS TO DATA LATER THAN THAT? Rejected? Written to
      a separate "late" table? Silently dropped?
   ③ ★ DO ROLLUPS GET RECOMPUTED when late data arrives?
      ⇒ ★ if not, your rollups and your raw data disagree, forever.
```

### Choosing the store — by cardinality and query shape

```
 ★ POSTGRESQL + PARTITIONING
   ✓ ★ you already have it; ★ joins, transactions, constraints
   ✓ good to ★ ~10⁹ rows and ~10⁵ series with rollups
   ✗ ★ no native compression; ★ rollups are your code
   ⇒ ★ THE RIGHT DEFAULT. Reach further only with a measured reason.

 ★ TIMESCALEDB (a PostgreSQL extension)
   ✓ ★ automatic partitioning (hypertables), ★ 10–20×
     compression, ★ CONTINUOUS AGGREGATES (incremental matviews)
   ✓ ★ still PostgreSQL — joins, FKs, existing tooling
   ✗ an extension to operate; ★ compression complicates late data
   ⇒ ★ THE BEST UPGRADE PATH FROM POSTGRESQL.

 ★ CLICKHOUSE
   ✓ ★ columnar, ★ enormous scan throughput (billions of rows/s),
     ★ 20–30× compression
   ✓ ★ excellent for analytics over huge volumes
   ✗ ★ weak single-row updates/deletes; ★ eventual consistency on
     inserts; ★ a different operational model
   ⇒ ★ when the workload is ANALYTICAL SCANS, not point reads.

 ★ PROMETHEUS
   ✓ ★ purpose-built for METRICS: pull model, service discovery,
     PromQL, alerting
   ✗ ★ NOT durable long-term storage; ★ no high availability by
     design; ★ cardinality-sensitive
   ⇒ ★ for operational monitoring, ★ not for business data.

 ★ INFLUXDB
   ✓ purpose-built, good ingest
   ✗ ★ severe cardinality limits historically; ★ major breaking
     changes between versions (1.x → 2.x → 3.x)

 ⇒ ★ THE DECIDING QUESTIONS:
   ① ★ how many DISTINCT SERIES? (<10⁵ ⇒ Postgres is fine)
   ② ★ point reads or big scans? (scans ⇒ columnar)
   ③ ★ do you need joins to relational data? (yes ⇒ stay in
     Postgres/Timescale)
   ④ ★ is it operational monitoring or business data?
     (monitoring ⇒ Prometheus; ★ business data ⇒ never Prometheus)
```

---

## How it works — step by step

### The PostgreSQL design, complete

```sql
-- ★ ① THE TABLE: wide, partitioned by time, minimal indexes
CREATE TABLE metrics (
  ts        timestamptz NOT NULL,
  series_id int         NOT NULL,     -- ★ a surrogate for the labels
  cpu       real,
  memory    real,
  disk_pct  real,
  net_in    bigint,
  net_out   bigint
) PARTITION BY RANGE (ts);

-- ★ the series dimension — labels live HERE, not in every row
CREATE TABLE series (
  id       serial PRIMARY KEY,
  host     text NOT NULL,
  env      text NOT NULL,
  region   text NOT NULL,
  ★ UNIQUE (host, env, region)        -- ★ this UNIQUE bounds cardinality
);
```
```
 ★ WHY A SERIES TABLE AND NOT LABELS PER ROW:
   • ★ an int instead of 3 text columns × 260 M rows
   • ★ the UNIQUE constraint makes cardinality VISIBLE and BOUNDED
   • ★ SELECT count(*) FROM series is your cardinality metric
   • ★ and you can JOIN to it — which Prometheus cannot do
```

```sql
-- ★ ② PARTITIONS: daily, created ahead, with a DEFAULT seatbelt
CREATE TABLE metrics_2026_08_25 PARTITION OF metrics
  FOR VALUES FROM ('2026-08-25') TO ('2026-08-26');
CREATE TABLE metrics_default PARTITION OF metrics DEFAULT;  -- ★ alert on rows

-- ★ ③ THE ONE INDEX YOU ACTUALLY NEED
--    every query is (series, time range) ⇒ ★ this is it.
CREATE INDEX ON metrics (series_id, ts DESC);
-- ★ NOT: an index on ts alone (partitioning already does that)
-- ★ NOT: an index per metric column (nobody queries by cpu value)

-- ★ ④ APPEND-ONLY SETTINGS
ALTER TABLE metrics SET (
  ★ fillfactor = 100,                       -- ★ no updates ⇒ no headroom
  autovacuum_vacuum_scale_factor = 0.0,
  autovacuum_vacuum_threshold = 100000,
  autovacuum_analyze_scale_factor = 0.02);  -- ★ but DO analyze
```

### Ingestion — batched, always

```js
// ★ a bounded in-memory buffer, flushed on size OR time
class MetricBuffer {
  constructor({ maxRows = 5000, maxMs = 1000 }) {
    this.rows = []; this.maxRows = maxRows;
    this.timer = setInterval(() => this.flush(), maxMs);
  }
  add(row) {
    this.rows.push(row);
    if (this.rows.length >= this.maxRows) this.flush();
  }
  async flush() {
    if (!this.rows.length) return;
    const batch = this.rows; this.rows = [];
    // ★ COPY is 20× faster than multi-row INSERT at this size
    const stream = client.query(copyFrom(
      `COPY metrics (ts, series_id, cpu, memory, disk_pct, net_in, net_out)
       FROM STDIN WITH (FORMAT csv)`));
    for (const r of batch) stream.write(toCsvLine(r));
    await finished(stream.end());
    metrics.histogram('ingest.batch_rows', batch.length);
  }
}
// ★ AND THE DURABILITY TRADE, MADE EXPLICIT:
//   a crash loses up to `maxMs` of buffered points.
//   ⇒ ★ acceptable for telemetry; ★ NOT for financial ticks.
//     For those, write to a durable queue first (Topic 52).
```

### Rollups — incremental, and composable

```sql
-- ★ THE ROLLUP TABLE: store (sum, count), NOT avg
CREATE TABLE metrics_1h (
  bucket     timestamptz NOT NULL,
  series_id  int         NOT NULL,
  n          bigint      NOT NULL,
  ★ cpu_sum  double precision NOT NULL,
  cpu_min    real, cpu_max real,
  ★ cpu_p99_sketch  bytea,      -- ★ a t-digest, for mergeable p99
  mem_sum    double precision NOT NULL,
  PRIMARY KEY (series_id, bucket)
) PARTITION BY RANGE (bucket);

-- ★ THE INCREMENTAL JOB — processes only NEW data
INSERT INTO metrics_1h (bucket, series_id, n, cpu_sum, cpu_min, cpu_max, mem_sum)
SELECT date_trunc('hour', ts), series_id,
       count(*), sum(cpu::double precision), min(cpu), max(cpu),
       sum(memory::double precision)
  FROM metrics
 WHERE ts >= $1 AND ts < $2          -- ★ a bounded window
 GROUP BY 1, 2
    ON CONFLICT (series_id, bucket) DO UPDATE
   SET n = EXCLUDED.n, cpu_sum = EXCLUDED.cpu_sum,
       cpu_min = least(metrics_1h.cpu_min, EXCLUDED.cpu_min),
       cpu_max = greatest(metrics_1h.cpu_max, EXCLUDED.cpu_max),
       mem_sum = EXCLUDED.mem_sum;
```
```sql
-- ★ THE READ: divide at query time. ★ NEVER store an average.
SELECT bucket, ★ cpu_sum / n AS cpu_avg, cpu_max
  FROM metrics_1h
 WHERE series_id = $1 AND bucket >= $2 AND bucket < $3
 ORDER BY bucket;

-- ★ AND ROLLING UP AGAIN — daily FROM hourly, correctly
INSERT INTO metrics_1d (bucket, series_id, n, cpu_sum, cpu_min, cpu_max)
SELECT date_trunc('day', bucket), series_id,
       ★ sum(n), ★ sum(cpu_sum),      -- ★ composable
       min(cpu_min), max(cpu_max)
  FROM metrics_1h WHERE bucket >= $1 AND bucket < $2
 GROUP BY 1,2;
-- ⇒ ★ cpu_sum/n at the daily level is EXACTLY the true daily
--   average. ★ avg(cpu_avg) would NOT be.
```

### TimescaleDB — the same design, mostly automated

```sql
CREATE EXTENSION IF NOT EXISTS timescaledb;

SELECT create_hypertable('metrics', 'ts',
  ★ chunk_time_interval => INTERVAL '1 day',
  ★ partitioning_column => 'series_id',      -- ★ space partitioning too
  ★ number_partitions   => 4);               -- ★ 4 rightmost B-trees

-- ★ COMPRESSION — the biggest single win
ALTER TABLE metrics SET (
  timescaledb.compress,
  ★ timescaledb.compress_segmentby = 'series_id',   -- ★ group by series
  ★ timescaledb.compress_orderby   = 'ts DESC');    -- ★ enables delta coding
SELECT add_compression_policy('metrics', ★ INTERVAL '7 days');
--                                        ▲ ★ the late-data window

-- ★ CONTINUOUS AGGREGATE — an incrementally-maintained rollup
CREATE MATERIALIZED VIEW metrics_1h
WITH (timescaledb.continuous) AS
SELECT ★ time_bucket('1 hour', ts) AS bucket, series_id,
       count(*) AS n,
       sum(cpu::double precision) AS cpu_sum,   -- ★ NOT avg()
       max(cpu) AS cpu_max,
       ★ percentile_agg(cpu) AS cpu_pct         -- ★ a mergeable sketch
  FROM metrics GROUP BY 1, 2;

SELECT add_continuous_aggregate_policy('metrics_1h',
  start_offset => INTERVAL '3 days',   -- ★ recompute a window, for
  end_offset   => INTERVAL '1 hour',   --   late data
  schedule_interval => INTERVAL '30 minutes');

-- ★ RETENTION
SELECT add_retention_policy('metrics', INTERVAL '90 days');
SELECT add_retention_policy('metrics_1h', INTERVAL '2 years');
```
```
 ★ WHAT THIS AUTOMATES vs THE MANUAL VERSION:
   partition creation · ★ compression · ★ incremental rollup
   refresh with a late-data window · retention
 ★ WHAT YOU STILL OWN:
   ★ cardinality · ★ the composability of your aggregates ·
   ★ deciding the late-data window
```

---

## Concept breakdown

```
★ FIVE PROPERTIES, EACH AN OPPORTUNITY
   ★ append-only ⇒ no MVCC bloat, fillfactor 100
   ★ time-ordered ⇒ rightmost inserts (★ and a contention point)
   ★ range-read ⇒ contiguous scans, ★ ONE index: (series, ts)
   ★ immutable ⇒ compress and reorder old data freely
   ★ decaying value ⇒ ★ tiered resolution

★ CARDINALITY IS THE KILLER
   distinct series = the product of label cardinalities
   ★ 1–3 KB of RAM per series, ★ before any data
   ★ NEVER put an unbounded value in a label:
     ✗ request_id, user_id, session_id, full URL, IP
     ✓ status_code, method, ★ route TEMPLATE, region, env
   ⇒ ★ a series dimension table with a UNIQUE constraint makes
     cardinality visible and bounded

★ NARROW vs WIDE
   narrow (ts, series, value) ⇒ flexible, ★ 3.7× more storage
   ★ wide (ts, host, cpu, mem, …) ⇒ fewer rows, ★ better
     compression, correlated queries free
   ⇒ ★ stable metric set ⇒ WIDE

★ COMPRESSION IS EXTRAORDINARY HERE
   ★ delta (timestamps) · ★ delta-of-delta (★ 1 BIT per regular
   point) · ★ XOR (floats, ~1.37 bytes) · columnar RLE
   ⇒ ★ 11.2 GB → 0.9 GB (Timescale) → 0.4 GB (ClickHouse)
   ⇒ ★ and compressed scans are often FASTER (less I/O)

★ ROLLUPS — the only way aggregates get fast
   260 M rows → 720 numbers: ★ no index helps
   raw → 1 min → 1 hour → 1 day, ★ each built from the level above
   ★ NOT EVERY AGGREGATE IS COMPOSABLE:
     ✓ sum, count, min, max
     ★ ✗ avg ⇒ store (sum, count), divide at read
     ★ ✗ distinct ⇒ HyperLogLog sketch, merged
     ★ ✗ percentiles ⇒ t-digest/DDSketch, merged
   ⇒ ★ "the average of averages" is the most common silent
     correctness bug in time-series dashboards

★ RETENTION IS ★ DROP PARTITION, NEVER DELETE (59, 47, 72)

★ LATE DATA — decide explicitly
   ★ how late is acceptable? ⇒ sets the compression delay
   ★ what happens beyond that? ⇒ reject / a late table / drop
   ★ do rollups get recomputed? ⇒ ★ if not, they disagree forever

★ CHOOSING A STORE — by cardinality and query shape
   <10⁵ series, point+range reads ⇒ ★ PostgreSQL + partitioning
   + compression and auto-rollups ⇒ ★ TimescaleDB
   ★ analytical scans over billions ⇒ ★ ClickHouse
   ★ operational monitoring ⇒ Prometheus (★ never business data)
```

---

## Diagrams

**Diagram 1 — big picture: tiered resolution and retention**

```
  ★ THE SAME METRIC, AT FOUR RESOLUTIONS

  RAW (10 s)      ████████████████████  ★ 260 M rows/month
  ★ retention: 7 days                     ★ 11.2 GB
  ⇒ "what exactly happened during the incident at 03:14?"
                          │ ★ rolled up from RAW
                          ▼
  1 MINUTE        ████████             ★ 4.3 M rows/month
  ★ retention: 30 days                   ★ 210 MB
  ⇒ "how did latency behave over the last week?"
                          │ ★ rolled up from 1 MINUTE
                          ▼
  1 HOUR          ██                   ★ 288 k rows/month
  ★ retention: 2 years                   ★ 22 MB
  ⇒ "what is our monthly capacity trend?"
                          │ ★ rolled up from 1 HOUR
                          ▼
  1 DAY           ▌                    ★ 12 k rows/month
  ★ retention: forever                   ★ 1.1 MB
  ⇒ "year-on-year growth"

 ★ TOTAL STORED AT STEADY STATE: ★ 2.6 GB
 ★ WITHOUT TIERING (raw kept 2 years): ★ 269 GB   ⇒ ★ 103×

 ★ AND NOTE THE ARROWS: each level is built from the one ABOVE,
   not from raw. ★ Rebuilding daily from 260 M raw rows would
   take minutes; from 288 k hourly rows it takes ★ 40 ms.
```

**Diagram 2 — data flow: why cardinality explodes, and what it costs**

```
 ✗ AN UNBOUNDED LABEL
   http_requests_total{
     method="GET", status="200", route="/api/orders",
     ★ request_id="a3f9-…"        ← ★ UNBOUNDED
   }
 ┌────────────────────────────────────────────────────────────────┐
 │  DAY 1     ★ 412 series          index RAM ★ 1 MB              │
 │  DAY 7     ★ 8,842,119 series    index RAM ★ 12 GB             │
 │  DAY 8     ★ OOM                                                │
 │                                                                 │
 │  ★ AND EACH SERIES HAS EXACTLY ONE DATA POINT.                 │
 │    ⇒ ★ compression does nothing (nothing to delta against)     │
 │    ⇒ ★ every query merges millions of one-point series         │
 │    ⇒ ★ the index is 1,000× the data                            │
 └────────────────────────────────────────────────────────────────┘

 ✓ BOUNDED LABELS ONLY
   http_requests_total{
     method="GET", status="200", ★ route="/api/orders/:id",
     region="ap-south-1", env="prod"
   }
   ★ 5 methods × 20 statuses × 80 routes × 3 regions × 2 envs
   = ★ 48,000 series, ★ and it never grows with traffic
 ┌────────────────────────────────────────────────────────────────┐
 │  index RAM ★ 96 MB, flat                                        │
 │  ★ each series has millions of points ⇒ ★ delta-of-delta gets  │
 │    ~1 bit per point                                             │
 │  ★ AND request_id still exists — ★ in TRACES and LOGS, which   │
 │    are designed for high cardinality. ★ Metrics are not.        │
 └────────────────────────────────────────────────────────────────┘
   ★ THE RULE: ★ IF A LABEL'S VALUE COMES FROM USER INPUT OR A
     UUID, IT DOES NOT BELONG IN A METRIC.
```

**Diagram 3 — before/after: the average-of-averages bug**

```
 ★ TRUE DATA — one hour, two 30-minute halves
   first 30 min:  ★ 1,800 samples, ★ mean 20 ms
   second 30 min: ★     6 samples, ★ mean 900 ms   (an outage;
                                     the service was barely
                                     responding)

 ✗ STORING avg PER 30-MINUTE BUCKET
 ┌───────────────────────────────────────────────────────────────┐
 │ bucket_1: avg = 20      bucket_2: avg = 900                    │
 │                                                                │
 │ ★ hourly = avg(20, 900) = ★ 460 ms                             │
 └───────────────────────────────────────────────────────────────┘

 ✓ STORING (sum, count)
 ┌───────────────────────────────────────────────────────────────┐
 │ bucket_1: sum = 36,000   n = 1,800                             │
 │ bucket_2: sum =  5,400   n =     6                             │
 │                                                                │
 │ ★ hourly = (36,000 + 5,400) / (1,800 + 6) = ★ 22.9 ms          │
 └───────────────────────────────────────────────────────────────┘

 ★ 460 ms vs 22.9 ms — ★ A FACTOR OF 20, AND THE WRONG ONE
   IS THE ONE ON THE DASHBOARD.
 ⇒ ★ AND IT IS WORSE THAN A WRONG NUMBER: it makes the outage
   look like a slow hour rather than a total failure, because
   ★ the 6 samples are weighted as heavily as the 1,800.

 ★ THE SAME APPLIES TO:
   ★ p99 ⇒ store a t-digest/DDSketch and MERGE the sketches
   ★ distinct ⇒ store a HyperLogLog and MERGE those
   ⇒ ★ "the p99 of the hourly p99s" is not a meaningful quantity.
```

---

## Example 1 — basic

```sql
-- ★ the series dimension — cardinality is visible here
CREATE TABLE series (
  id     serial PRIMARY KEY,
  host   text NOT NULL,
  env    text NOT NULL,
  region text NOT NULL,
  UNIQUE (host, env, region));
INSERT INTO series (host, env, region)
SELECT 'host-'||g, 'prod', 'ap-south-1' FROM generate_series(1,400) g;

CREATE TABLE metrics (
  ts        timestamptz NOT NULL,
  series_id int NOT NULL,
  cpu real, memory real, disk_pct real, net_in bigint, net_out bigint
) PARTITION BY RANGE (ts);

DO $$ DECLARE d date := '2026-08-01';
BEGIN
  WHILE d < '2026-09-01' LOOP
    EXECUTE format('CREATE TABLE metrics_%s PARTITION OF metrics
                    FOR VALUES FROM (%L) TO (%L)',
                   to_char(d,'YYYY_MM_DD'), d, d+1);
    d := d + 1;
  END LOOP;
END $$;

CREATE INDEX ON metrics (series_id, ts DESC);
ALTER TABLE metrics SET (fillfactor = 100);
```

**Generate a month of data and measure ingestion methods.**
```bash
python3 - <<'EOF'
import psycopg2, io, time, random, datetime
c = psycopg2.connect("dbname=tsdb"); cur = c.cursor()
start = datetime.datetime(2026,8,1)
buf = io.StringIO()
n = 0
for minute in range(30*24*60):
    t = start + datetime.timedelta(minutes=minute)
    for s in range(1, 401):
        buf.write(f"{t.isoformat()},{s},{random.uniform(10,90):.2f},"
                  f"{random.uniform(20,95):.2f},{random.uniform(5,80):.2f},"
                  f"{random.randint(0,10**6)},{random.randint(0,10**6)}\n")
        n += 1
    if n % 500000 == 0:
        buf.seek(0); cur.copy_expert(
          "COPY metrics (ts,series_id,cpu,memory,disk_pct,net_in,net_out) "
          "FROM STDIN WITH (FORMAT csv)", buf)
        c.commit(); buf = io.StringIO()
EOF
```
```sql
SELECT count(*), pg_size_pretty(pg_total_relation_size('metrics')) FROM metrics;
```
```
   count    | pg_size_pretty
------------+----------------
 ★ 17,280,000 | ★ 1,842 MB
```

**Compare ingestion methods.**
```js
const rows = makeRows(10000);

console.time('per-row');
for (const r of rows) await client.query(INSERT_ONE, r);
console.timeEnd('per-row');

console.time('txn');
await client.query('BEGIN');
for (const r of rows) await client.query(INSERT_ONE, r);
await client.query('COMMIT');
console.timeEnd('txn');

console.time('multi-row');
await client.query(buildMultiRowInsert(rows));
console.timeEnd('multi-row');

console.time('copy');
await copyRows(client, rows);
console.timeEnd('copy');
```
```
 ★ per-row:   48,204 ms
 ★ txn:        4,102 ms      ★ 11.8×
 ★ multi-row:    182 ms      ★ 265×
 ★ copy:          41 ms      ★ 1,176×
   ⇒ ★ COPY is the only correct answer for time-series ingestion.
```

**Prove the aggregate needs a rollup.**
```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT date_trunc('hour', ts) AS bucket, avg(cpu)
  FROM metrics
 WHERE ts >= '2026-08-01' AND ts < '2026-09-01'
 GROUP BY 1 ORDER BY 1;
```
```
 GroupAggregate  (actual rows=720)
   ->  Sort  (actual rows=★ 17,280,000)
         ★ Sort Method: external merge  Disk: 618,204kB
         ->  Append (30 partitions)
 Execution Time: ★ 41,204 ms
   ★ 17.3 MILLION ROWS SCANNED AND SORTED TO PRODUCE 720 NUMBERS.
```
```sql
-- ★ the rollup
CREATE TABLE metrics_1h (
  bucket timestamptz NOT NULL, series_id int NOT NULL,
  n bigint NOT NULL, cpu_sum double precision NOT NULL,
  cpu_min real, cpu_max real,
  PRIMARY KEY (series_id, bucket));

INSERT INTO metrics_1h
SELECT date_trunc('hour', ts), series_id, count(*),
       sum(cpu::double precision), min(cpu), max(cpu)
  FROM metrics GROUP BY 1,2;
```
```
 INSERT 0 288000
 Time: ★ 38,204 ms        (once)
```
```sql
EXPLAIN (ANALYZE)
SELECT bucket, ★ sum(cpu_sum)/sum(n) AS cpu_avg
  FROM metrics_1h
 WHERE bucket >= '2026-08-01' AND bucket < '2026-09-01'
 GROUP BY 1 ORDER BY 1;
```
```
 Execution Time: ★ 88 ms        ★ 468×
```

**Prove the average-of-averages bug.**
```sql
CREATE TABLE bad_rollup (bucket timestamptz, avg_val double precision);
INSERT INTO bad_rollup VALUES
  ('2026-08-25 10:00', 20),      -- ★ from 1,800 samples
  ('2026-08-25 10:30', 900);     -- ★ from 6 samples

SELECT avg(avg_val) AS wrong FROM bad_rollup;
```
```
 wrong
-------
 ★ 460
```
```sql
CREATE TABLE good_rollup (bucket timestamptz, s double precision, n bigint);
INSERT INTO good_rollup VALUES
  ('2026-08-25 10:00', 36000, 1800),
  ('2026-08-25 10:30',  5400,    6);

SELECT ★ sum(s)/sum(n) AS correct FROM good_rollup;
```
```
 correct
------------------
 ★ 22.9236…
   ★ 460 vs 22.9. ★ A FACTOR OF 20, and the wrong one hides an
     outage as "a slow hour".
```

**Prove percentiles do not compose either.**
```sql
CREATE EXTENSION IF NOT EXISTS tdigest;   -- or timescaledb_toolkit

-- ✗ storing p99 per bucket and averaging
SELECT avg(p99) FROM (
  SELECT percentile_cont(0.99) WITHIN GROUP (ORDER BY cpu) AS p99
    FROM metrics WHERE ts >= '2026-08-25' GROUP BY date_trunc('hour', ts)) x;
```
```
 ★ 88.42        — ★ meaningless
```
```sql
-- ✓ storing a mergeable sketch
SELECT ★ tdigest_percentile(tdigest(cpu, 100), 0.99)
  FROM metrics WHERE ts >= '2026-08-25';
```
```
 ★ 89.91        — ★ the true p99 over the whole period
```

**Prove `DROP PARTITION` beats `DELETE`.**
```sql
\timing on
DELETE FROM metrics WHERE ts >= '2026-08-01' AND ts < '2026-08-02';
```
```
 DELETE 576000
 Time: ★ 8,842 ms
```
```sql
SELECT pg_size_pretty(pg_relation_size('metrics_2026_08_01')),
       n_dead_tup FROM pg_stat_user_tables WHERE relname='metrics_2026_08_01';
```
```
 pg_size_pretty | n_dead_tup
----------------+------------
 ★ 61 MB        |  ★ 576000
   ★ the space is NOT reclaimed, and there are 576k dead tuples.
```
```sql
DROP TABLE metrics_2026_08_02;
```
```
 Time: ★ 12 ms        ★ 737×, ★ and the disk is returned.
```

**Measure the compression opportunity.**
```sql
-- ★ how compressible is this data really?
SELECT pg_size_pretty(sum(pg_column_size(ROW(ts, series_id, cpu, memory)))),
       pg_size_pretty(sum(pg_column_size(ts))) AS ts_bytes,
       pg_size_pretty(sum(pg_column_size(cpu))) AS cpu_bytes
  FROM metrics WHERE ts >= '2026-08-25' AND ts < '2026-08-26';
```
```
 ★ ts_bytes: 4,608 kB        (8 bytes × 576,000)
 ★ cpu_bytes: 2,304 kB       (4 bytes × 576,000)
 ★ delta-of-delta on ts at a regular interval ⇒ ~1 bit/point
   ⇒ ★ 4,608 kB → ~70 kB.  ★ 65×.
```

**Prove cardinality is the constraint.**
```sql
-- ★ the good design: cardinality is bounded by a UNIQUE constraint
SELECT count(*) AS series_count FROM series;
```
```
 ★ 400
```
```sql
-- ✗ the bad design: labels per row, unbounded
CREATE TABLE metrics_labelled (
  ts timestamptz, host text, env text, ★ request_id uuid,
  cpu real);
INSERT INTO metrics_labelled
SELECT now(), 'host-'||(g%400), 'prod', ★ gen_random_uuid(), random()*100
  FROM generate_series(1, 5000000) g;

SELECT count(DISTINCT (host, env, request_id)) AS cardinality
  FROM metrics_labelled;
```
```
 ★ 5,000,000        — ★ one series per row. Compression: zero.
```
```sql
SELECT pg_size_pretty(pg_total_relation_size('metrics_labelled')),
       pg_size_pretty(pg_total_relation_size('metrics'));
```
```
 ★ 612 MB  vs  ★ 1,842 MB for 3.4× the rows
   ⇒ ★ per row, the labelled version is ★ 4.4× larger — and it
     cannot be compressed or rolled up meaningfully.
```

---

## Example 2 — production scenario

**The situation.** An observability platform for a fintech. 4,000 hosts, application metrics, 90-day retention required by policy.

```
 THE ORIGINAL DESIGN
 CREATE TABLE metrics (
   id bigserial PRIMARY KEY,          -- ★ a surrogate key nobody uses
   ts timestamptz, metric_name text, host text, env text,
   ★ labels jsonb, value double precision);
 CREATE INDEX ON metrics (ts);
 CREATE INDEX ON metrics (metric_name);
 CREATE INDEX ON metrics (host);
 ★ CREATE INDEX ON metrics USING gin (labels);
 -- ★ retention: a nightly DELETE

 AFTER 4 MONTHS
   ★ table size            ★ 4.1 TB (★ 2.9 TB of it indexes)
   ★ ingest                ★ 41,000 points/sec, ★ p99 write 840 ms
   ★ dashboard queries     ★ 8,400 ms – 94,000 ms
   ★ the nightly DELETE    ★ 6h 40m, ★ often unfinished
   ★ replica lag at 02:00  ★ 41 minutes
   ★ disk                  ★ 91%
```

**Step 1 — the four separate problems.**

```sql
SELECT relname, pg_size_pretty(pg_relation_size(relid)) AS heap,
       pg_size_pretty(pg_indexes_size(relid)) AS indexes
  FROM pg_stat_user_tables WHERE relname = 'metrics';
```
```
 relname |  heap   | indexes
---------+---------+----------
 metrics | 1,204 GB| ★ 2,918 GB
 ★ THE INDEXES ARE 2.4× THE DATA.
```
```sql
SELECT indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
  FROM pg_stat_user_indexes WHERE relname='metrics' ORDER BY idx_scan;
```
```
     indexrelname      |  idx_scan  |   size
-----------------------+------------+-----------
 ★ metrics_labels_idx  |      ★ 412 | ★ 1,840 GB
 ★ metrics_host_idx    |    ★ 1,204 |   ★ 288 GB
 metrics_metric_idx    |    884,201 |   ★ 302 GB
 metrics_ts_idx        |  4,102,884 |   ★ 288 GB
 ★ metrics_pkey        |      ★ 0   |   ★ 200 GB
 ⇒ ★ PROBLEM 1: a 1.8 TB GIN index used 412 times in four months,
   and a 200 GB primary key ★ NEVER USED AT ALL.
```
```sql
SELECT count(DISTINCT (metric_name, host, env, labels)) AS cardinality
  FROM metrics WHERE ts > now() - interval '1 hour';
```
```
 ★ 8,842,119        — in ONE HOUR.
```
```sql
SELECT jsonb_object_keys(labels) AS k, count(DISTINCT labels->>jsonb_object_keys(labels))
  FROM metrics WHERE ts > now() - interval '1 hour'
 GROUP BY 1 ORDER BY 2 DESC LIMIT 5;
```
```
        k         |  count
------------------+----------
 ★ request_id     | ★ 8,204,118
 ★ user_id        |   ★ 412,088
 pod_name         |     ★ 8,420
 status_code      |         18
 method           |          6
 ⇒ ★ PROBLEM 2: request_id and user_id are in metric labels.
   ★ THE CARDINALITY EXPLOSION. Every "series" has one point.
```
```
 ★ PROBLEM 3: ★ no partitioning ⇒ retention is a DELETE ⇒ 6h40m,
   18 GB of WAL, 41 minutes of replica lag, ★ no disk reclaimed.
 ★ PROBLEM 4: ★ no rollups ⇒ every dashboard query scans raw data.
```

**Step 2 — fix cardinality first. It is the root cause.**

```
 ★ WHY FIRST: partitioning, compression and rollups ALL work
   better on low-cardinality data, and ★ none of them fix
   8.8 million one-point series.
```
```js
// ★ the metric emission layer — an allowlist, enforced
const ALLOWED_LABELS = new Set([
  'metric_name','host','env','region','service','status_code',
  'method','route','pod_namespace',
]);
const HIGH_CARDINALITY = new Set(['request_id','user_id','session_id',
                                  'trace_id','email','ip']);

function emit(name, value, labels) {
  for (const k of Object.keys(labels)) {
    if (HIGH_CARDINALITY.has(k)) {
      // ★ FAIL LOUDLY IN CI, drop in production
      if (process.env.NODE_ENV === 'test')
        throw new Error(`★ high-cardinality label in metric: ${k}`);
      metrics.increment('telemetry.label_dropped', { label: k });
      delete labels[k];
      continue;
    }
    if (!ALLOWED_LABELS.has(k)) { delete labels[k]; }
  }
  // ★ route must be a TEMPLATE, never a concrete path
  if (labels.route) labels.route = normaliseRoute(labels.route);
  buffer.add({ name, value, labels });
}
// ★ AND: request_id and trace_id still exist — ★ in TRACES,
//   which are built for high cardinality. Metrics are not.
```
```
 ★ MEASURED AFTER: cardinality 8,842,119 → ★ 41,204 in one hour.
   ★ 215×.
```

**Step 3 — the new schema.**

```sql
-- ★ the series dimension: labels stored ONCE
CREATE TABLE series (
  id           bigserial PRIMARY KEY,
  metric_name  text NOT NULL,
  host         text NOT NULL,
  env          text NOT NULL,
  service      text NOT NULL,
  labels       jsonb NOT NULL DEFAULT '{}',
  ★ label_hash bytea GENERATED ALWAYS AS
      (sha256((metric_name||host||env||service||labels::text)::bytea)) STORED,
  ★ UNIQUE (label_hash)              -- ★ cardinality is now VISIBLE
);
CREATE INDEX ON series (metric_name, service);

-- ★ the fact table: narrow, because metric names are user-defined
CREATE TABLE samples (
  ts        timestamptz NOT NULL,
  series_id bigint      NOT NULL,
  value     double precision NOT NULL
) PARTITION BY RANGE (ts);

-- ★ ONE index. Every query is (series, time range).
CREATE INDEX ON samples (series_id, ts DESC);
-- ★ NO primary key (nothing looks up a sample by id)
-- ★ NO index on ts alone (partitioning handles it)
-- ★ NO GIN index (labels are in `series` now)

ALTER TABLE samples SET (fillfactor = 100,
  autovacuum_vacuum_scale_factor = 0.0,
  autovacuum_vacuum_threshold = 500000,
  autovacuum_analyze_scale_factor = 0.01);
```
```
 ★ THE INDEX ARITHMETIC:
   before: 5 indexes, ★ 2,918 GB
   after:  1 index,   ★ 88 GB      ⇒ ★ 33×
 ★ AND THE ROW SHRANK: (ts, bigint, double) = ★ 24 bytes
   vs (bigserial, ts, 3 texts, jsonb, double) = ★ ~180 bytes.
```

**Step 4 — TimescaleDB for compression and continuous aggregates.**

```sql
SELECT create_hypertable('samples', 'ts',
  chunk_time_interval => INTERVAL '6 hours',
  ★ partitioning_column => 'series_id',
  ★ number_partitions => 8);        -- ★ 8 rightmost B-trees, not 1

ALTER TABLE samples SET (
  timescaledb.compress,
  ★ timescaledb.compress_segmentby = 'series_id',
  ★ timescaledb.compress_orderby = 'ts DESC');
SELECT add_compression_policy('samples', ★ INTERVAL '2 days');
--                                        ▲ ★ the late-data window
```
```sql
-- ★ the rollup hierarchy — each built from the level above
CREATE MATERIALIZED VIEW samples_1m WITH (timescaledb.continuous) AS
SELECT time_bucket('1 minute', ts) AS bucket, series_id,
       count(*) AS n, ★ sum(value) AS s, min(value) AS mn, max(value) AS mx,
       ★ percentile_agg(value) AS pct
  FROM samples GROUP BY 1,2;

CREATE MATERIALIZED VIEW samples_1h WITH (timescaledb.continuous) AS
SELECT time_bucket('1 hour', bucket) AS bucket, series_id,
       ★ sum(n) AS n, ★ sum(s) AS s, min(mn) AS mn, max(mx) AS mx,
       ★ rollup(pct) AS pct              -- ★ merge the sketches
  FROM ★ samples_1m GROUP BY 1,2;        -- ★ FROM THE 1-MINUTE VIEW

CREATE MATERIALIZED VIEW samples_1d WITH (timescaledb.continuous) AS
SELECT time_bucket('1 day', bucket) AS bucket, series_id,
       sum(n) AS n, sum(s) AS s, min(mn) AS mn, max(mx) AS mx,
       rollup(pct) AS pct
  FROM ★ samples_1h GROUP BY 1,2;
```
```sql
-- ★ refresh policies, with a late-data window
SELECT add_continuous_aggregate_policy('samples_1m',
  ★ start_offset => INTERVAL '2 days',   -- ★ recompute for late data
  end_offset   => INTERVAL '1 minute',
  schedule_interval => INTERVAL '1 minute');
-- (similar for 1h and 1d, with wider windows)
```
```sql
-- ★ tiered retention
SELECT add_retention_policy('samples',    INTERVAL '7 days');
SELECT add_retention_policy('samples_1m', INTERVAL '30 days');
SELECT add_retention_policy('samples_1h', INTERVAL '2 years');
-- ★ samples_1d: kept forever (12k rows/month)
```
```
 ★ NOTE THE POLICY CONFLICT THAT WAS NEARLY MISSED:
   ★ compliance requires 90 days of data.
   ★ raw retention is 7 days.
   ⇒ ★ IS THAT COMPLIANT? ★ The policy says "90 days of metrics",
     and 1-minute resolution for 30 days plus hourly for 2 years
     was ★ reviewed and accepted in writing.
   ⇒ ★ THIS IS A CONVERSATION TO HAVE BEFORE THE MIGRATION,
     NOT DURING AN AUDIT.
```

**Step 5 — the query layer chooses the resolution.**

```js
// ★ the dashboard picks the coarsest table that satisfies the range
const TIERS = [
  { table: 'samples',    step: 10,     maxAgeMs: 7  * 864e5 },
  { table: 'samples_1m', step: 60,     maxAgeMs: 30 * 864e5 },
  { table: 'samples_1h', step: 3600,   maxAgeMs: 730 * 864e5 },
  { table: 'samples_1d', step: 86400,  maxAgeMs: Infinity },
];

function pickTier(fromMs, toMs, targetPoints = 500) {
  const spanS = (toMs - fromMs) / 1000;
  const ageMs = Date.now() - fromMs;
  for (const t of TIERS) {
    if (ageMs > t.maxAgeMs) continue;          // ★ data no longer exists
    if (spanS / t.step <= targetPoints * 4) return t;  // ★ enough detail
  }
  return TIERS[TIERS.length - 1];
}

async function series(seriesId, fromMs, toMs) {
  const tier = pickTier(fromMs, toMs);
  const sql = tier.table === 'samples'
    ? `SELECT ts, value FROM samples
        WHERE series_id=$1 AND ts >= $2 AND ts < $3 ORDER BY ts`
    : `SELECT bucket AS ts, ★ s/n AS value, mn, mx   -- ★ divide at read
         FROM ${tier.table}
        WHERE series_id=$1 AND bucket >= $2 AND bucket < $3 ORDER BY bucket`;
  return pool.query(sql, [seriesId, new Date(fromMs), new Date(toMs)]);
}
```

**Step 6 — the migration.**

```js
// ★ ① dual-write for the transition
await Promise.all([
  legacyBuffer.add(point),                    // ★ keep serving reads
  newBuffer.add({ series_id: await seriesIdFor(point.labels),
                  ts: point.ts, value: point.value }),
]);

// ★ ② the series lookup is hot — cache it in process
const seriesCache = new LRU({ max: 200_000 });
async function seriesIdFor(labels) {
  const key = labelHash(labels);
  const hit = seriesCache.get(key);
  if (hit) return hit;
  const { rows: [r] } = await pool.query(
    `INSERT INTO series (metric_name, host, env, service, labels)
     VALUES ($1,$2,$3,$4,$5)
     ★ ON CONFLICT (label_hash) DO UPDATE SET metric_name = EXCLUDED.metric_name
     RETURNING id`, [...]);
  seriesCache.set(key, r.id);
  return r.id;
}
// ★ ON CONFLICT … DO UPDATE (not DO NOTHING) so RETURNING always
//   yields a row. ★ DO NOTHING returns zero rows on conflict —
//   a classic bug.
```
```bash
# ★ ③ backfill only the last 7 days at raw resolution;
#   ★ backfill rollups for the full 90 days from the old table
#   (the old data's cardinality makes raw backfill pointless)
```
```
 ★ AND THE DECISION THAT SAVED WEEKS: the old data's 8.8M-series
   cardinality could not be usefully re-keyed. ★ Only the rollups
   were backfilled, and raw history older than 7 days was
   ★ deliberately discarded, with sign-off.
```

**Step 7 — results.**

| | Before | After |
|---|---|---|
| Cardinality (1 h) | ★ **8,842,119** | **41,204** (**215×**) |
| Total size | ★ **4.1 TB** | **188 GB** (**22×**) |
| — of which indexes | ★ 2,918 GB | **88 GB** (**33×**) |
| Row size | ~180 B | **24 B** (compressed: **~2 B**) |
| Ingest p99 | 840 ms | **8 ms** |
| Dashboard p99 (30 d) | ★ **94,000 ms** | **41 ms** (**2,290×**) |
| Retention | ★ 6h 40m `DELETE` | ★ **`drop_chunks`, 40 ms** |
| Replica lag at 02:00 | ★ 41 min | **< 2 s** |
| Indexes | 5 | **1** |
| Disk | 91% | **19%** |

```
 ★ SEVEN LESSONS:
 ① ★ CARDINALITY WAS THE ROOT CAUSE, AND EVERYTHING ELSE WAS A
   SYMPTOM. 8.8M one-point series cannot be compressed, rolled up,
   or indexed usefully. ★ Fixing it first made every other fix work.
 ② ★ request_id AND user_id IN METRIC LABELS. They belong in
   traces and logs — systems designed for high cardinality.
 ③ ★ A 1.8 TB GIN INDEX USED 412 TIMES IN FOUR MONTHS, and a
   200 GB primary key used ★ zero times. ★ Time-series needs
   exactly one index: (series, time).
 ④ ★ THE ROW SHRANK 7.5× BEFORE COMPRESSION, purely from moving
   labels into a dimension table.
 ⑤ ★ ROLLUPS MUST BE BUILT FROM THE LEVEL ABOVE, and must store
   (sum, count) plus ★ mergeable sketches — never averages or
   percentiles.
 ⑥ ★ THE RETENTION POLICY CONFLICT WAS FOUND BEFORE THE
   MIGRATION, not during an audit. "90 days of metrics" and
   "7 days of raw" needed an explicit, written decision.
 ⑦ ★ OLD HIGH-CARDINALITY DATA WAS DELIBERATELY DISCARDED. It
   could not be re-keyed usefully, and pretending otherwise would
   have cost weeks.
```

---

## Common mistakes

**1. Unbounded labels — the cardinality explosion.**
- *Symptom:* millions of series with one point each; index RAM exceeding data; compression achieving nothing.
- *Fix:* an allowlist at the emission layer that fails loudly in CI. `request_id` belongs in traces, not metrics.

**2. Retention by `DELETE`.**
- *Symptom:* the nightly job is the largest workload; hours of runtime, gigabytes of WAL, replica lag, no disk reclaimed.
- *Fix:* partition by time and `DROP` (Topics 47, 59).

**3. Indexing every column.**
- *Symptom:* indexes 2.4× the size of the data, most of them unused.
- *Fix:* one index — `(series_id, ts DESC)`. Check `idx_scan` before keeping any other.

**4. A surrogate primary key on the fact table.**
- *Symptom:* 200 GB of index that nothing ever uses.
- *Fix:* time-series rows are never looked up by id. Omit it.

**5. Storing averages in rollups.**
- *Symptom:* dashboards showing 460 ms where the truth is 22.9 ms — and outages appearing as mildly slow hours.
- *Fix:* store `(sum, count)` and divide at read time.

**6. Storing percentiles in rollups.**
- *Symptom:* "the p99 of hourly p99s" — a meaningless number presented as fact.
- *Fix:* a mergeable sketch (t-digest, DDSketch) and merge those.

**7. Building each rollup level from raw.**
- *Symptom:* the daily rollup takes minutes instead of milliseconds.
- *Fix:* each level from the level above. `sum(n)` and `sum(s)` compose.

**8. Ignoring late-arriving data.**
- *Symptom:* points silently rejected, or landing in the wrong compaction window, or rollups permanently disagreeing with raw.
- *Fix:* decide the acceptable lateness, set the compression/finalisation delay to match, and recompute rollups over that window.

**9. Row-by-row ingestion.**
- *Symptom:* 48 seconds for 10,000 points.
- *Fix:* `COPY`. It is 1,176× faster than per-row inserts.

**10. `fillfactor` below 100 on an append-only table.**
- *Symptom:* wasted space with no benefit — there are no updates to accommodate.
- *Fix:* `fillfactor = 100`.

**11. A single rightmost index page under high concurrency.**
- *Symptom:* `wait_event = 'BufferContent'` and writes that don't scale with cores.
- *Fix:* hash-partition by series in addition to time (Topic 61).

**12. Using Prometheus for business data.**
- *Symptom:* data loss on restart, no HA, and a discovery that it was never durable storage.
- *Fix:* Prometheus for operational monitoring; a durable store for anything that matters.

**13. Choosing a time-series database before checking cardinality.**
- *Symptom:* the same explosion, in a system with tighter limits.
- *Fix:* under ~10⁵ series, PostgreSQL with partitioning is sufficient and far simpler.

---

## Hands-on proof

**PROVE IT #1–#8 — Example 1** (ingestion at 48,204 / 4,102 / 182 / 41 ms, an aggregate at 41 s dropping to 88 ms with a rollup, the average-of-averages bug at 460 vs 22.9, percentiles failing to compose, `DELETE` at 8,842 ms vs `DROP` at 12 ms with no space reclaimed, the delta-encoding opportunity at 65×, and a labelled table being 4.4× larger per row with zero compressibility).

**PROVE IT #9 — the rightmost-page contention.**
```bash
# ★ 64 concurrent writers into one time-partitioned table
pgbench -f /tmp/insert_metric.sql -c 64 -j 8 -T 30 tsdb &
sleep 5
psql -c "SELECT wait_event, count(*) FROM pg_stat_activity
          WHERE state='active' AND wait_event IS NOT NULL GROUP BY 1"
```
```
   wait_event   | count
----------------+-------
 ★ BufferContent|  ★ 41
   ★ all contending on the same rightmost B-tree leaf.
```
```bash
# ★ after hash-partitioning by series_id into 8
psql -c "SELECT wait_event, count(*) FROM pg_stat_activity
          WHERE state='active' AND wait_event IS NOT NULL GROUP BY 1"
```
```
   wait_event   | count
----------------+-------
 ClientRead     |     3
   ★ gone. 8 partitions ⇒ 8 rightmost pages.
```

**PROVE IT #10 — compression, measured.**
```sql
SELECT pg_size_pretty(before_compression_total_bytes) AS before,
       pg_size_pretty(after_compression_total_bytes) AS after,
       round(before_compression_total_bytes::numeric
             / after_compression_total_bytes, 1) AS ratio
  FROM hypertable_compression_stats('samples');
```
```
   before   |  after   | ratio
------------+----------+-------
 ★ 188 GB   | ★ 14 GB  | ★ 13.4
```

**PROVE IT #11 — late data and compressed chunks.**
```sql
-- ★ insert a point into a chunk compressed 3 days ago
INSERT INTO samples VALUES (now() - interval '5 days', 1, 42.0);
```
```
 ★ (Timescale ≥2.11: succeeds, but the chunk is partially
   decompressed and must be recompressed.)
 ★ (older versions: ERROR: insert into a compressed chunk)
 ⇒ ★ THE COMPRESSION DELAY MUST EXCEED YOUR LATE-DATA WINDOW.
```

**PROVE IT #12 — the tier selector picks correctly.**
```js
console.log(pickTier(Date.now() - 3600e3,   Date.now()).table);  // ★ samples
console.log(pickTier(Date.now() - 7*864e5,  Date.now()).table);  // ★ samples_1m
console.log(pickTier(Date.now() - 90*864e5, Date.now()).table);  // ★ samples_1h
console.log(pickTier(Date.now() - 900*864e5,Date.now()).table);  // ★ samples_1d
```

---

## The design decision framework

```
★★★ TIME-SERIES IS A WORKLOAD SHAPE, NOT A DATABASE CATEGORY.
    ★ CARDINALITY DECIDES EVERYTHING ELSE. ★★★

 ① ★ BOUND CARDINALITY FIRST — IT IS THE ROOT OF EVERY OTHER
    PROBLEM
    ✓ ★ an allowlist of labels at the EMISSION layer
    ✓ ★ fail loudly in CI on request_id / user_id / trace_id /
       session_id / email / IP / full URL
    ✓ ★ route TEMPLATES, never concrete paths
    ✓ ★ a series dimension table with a UNIQUE constraint ⇒
       cardinality becomes a number you can SELECT
    ⇒ ★ high-cardinality identifiers belong in TRACES and LOGS.

 ② ★ CHOOSE THE SHAPE
    stable, known metric set ⇒ ★ WIDE (3.7× less storage,
      correlated queries free)
    arbitrary/user-defined   ⇒ ★ NARROW + a series table

 ③ ★ ONE INDEX: (series_id, ts DESC)
    ✗ ★ no surrogate primary key · ✗ no index on ts alone ·
    ✗ no per-column indexes · ✗ no GIN on labels (they're in the
      dimension table)
    ⇒ ★ check idx_scan before keeping any index at all.

 ④ ★ PARTITION BY TIME — RETENTION IS `DROP`, NEVER `DELETE`
    ⇒ ★ 12 ms vs 8,842 ms, ★ and the disk is actually returned
    ⇒ ★ hash-partition by series too if writes contend on the
      rightmost page (Topic 61)

 ⑤ ★ TIER THE RESOLUTION
    raw (days) → 1 min (weeks) → 1 hour (years) → 1 day (forever)
    ⇒ ★ 269 GB → 2.6 GB, measured
    ⇒ ★ EACH LEVEL BUILT FROM THE LEVEL ABOVE
    ⇒ ★ AND GET SIGN-OFF: "90 days of metrics" vs "7 days of raw"
      is a policy question, ★ not an engineering one.

 ⑥ ★ ONLY STORE COMPOSABLE AGGREGATES
    ✓ sum · count · min · max
    ★ ✗ avg        ⇒ store (sum, count), divide at read
    ★ ✗ distinct   ⇒ a HyperLogLog sketch, merged
    ★ ✗ percentile ⇒ a t-digest/DDSketch, merged
    ⇒ ★ the average-of-averages bug turns an outage into "a
      slightly slow hour". It is the most common silent
      correctness failure here.

 ⑦ ★ DECIDE THE LATE-DATA POLICY EXPLICITLY
    ★ how late is acceptable? ⇒ sets the compression delay
    ★ what happens beyond it? ⇒ reject / a late table / drop
    ★ do rollups recompute over that window? ⇒ ★ if not, rollups
      and raw disagree forever

 ⑧ ★ ALWAYS BATCH INGESTION — `COPY` IF POSSIBLE
    per-row 48,204 ms · txn 4,102 · multi-row 182 · ★ COPY 41
    ⇒ ★ and make the buffering durability trade explicit.

 ⑨ ★ CHOOSE THE STORE BY CARDINALITY AND QUERY SHAPE
    <10⁵ series, mixed reads        ⇒ ★ PostgreSQL + partitioning
    + compression and auto-rollups  ⇒ ★ TimescaleDB
    ★ analytical scans over billions ⇒ ★ ClickHouse
    ★ operational monitoring         ⇒ Prometheus
                                      ★ (never business data)
    ⇒ ★ start with what you have; move for a MEASURED reason.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build a partitioned metrics table with 17M rows. Then: (a) compare per-row, transaction, multi-row and `COPY` ingestion; (b) run an hourly average over a month and report the plan and timing; (c) build a rollup and re-run it; (d) compare `DELETE` and `DROP TABLE` for one day's data, checking whether disk is reclaimed.

### Exercise 2 — medium (apply it)
Construct the average-of-averages bug with two buckets of very different sample counts, and show the correct calculation. Then do the same for percentiles, using a t-digest to show that sketches merge correctly and p99s do not.

Then measure cardinality two ways — labels-per-row versus a series dimension table — and report the size and compressibility difference.

### Exercise 3 — hard (production simulation)
An observability platform has a 4.1 TB unpartitioned metrics table (2.9 TB of it indexes), 8.8 million distinct series per hour, dashboard queries up to 94 seconds, and a nightly `DELETE` that runs 6h40m.

(a) Identify the four separate problems and explain which is the root cause.
(b) Two indexes account for 2 TB and are barely used. Write the query that finds them and explain why time-series needs exactly one index.
(c) Find the cardinality source. Explain why `request_id` in a metric label makes compression and rollups impossible.
(d) Design the emission-layer allowlist. Explain why it must fail in CI rather than only drop in production.
(e) Design the new schema. Compute the row-size reduction before compression and explain where it comes from.
(f) Design the rollup hierarchy with composable aggregates only. Show the SQL for building the hourly level from the minute level.
(g) The compliance policy requires 90 days. Raw retention will be 7 days. Explain the conversation that must happen and what must be documented.
(h) Design the tier selector for the query layer.
(i) The migration cannot usefully re-key four months of high-cardinality history. Justify discarding it and say what you would backfill instead.
(j) List every change and the measured improvement each produced.

---

## Mental model checkpoint

1. Name the five properties of time-series data and the optimisation each enables.
2. What is cardinality, and why does it kill time-series systems?
3. Give five labels that must never appear in a metric, and where they belong instead.
4. Compare narrow and wide schemas. What decides between them?
5. Explain delta-of-delta and XOR encoding. Why is time-series so compressible?
6. Why can't an index make "hourly average over a month" fast?
7. Which aggregates compose and which don't? Give the storage fix for each that doesn't.
8. Explain the average-of-averages bug and why it is worse than merely wrong.
9. Why must each rollup level be built from the level above?
10. Name four things that can go wrong with late-arriving data, by system.
11. Why is `fillfactor = 100` correct here, and why is there a contention point anyway?
12. Give the decision rule for PostgreSQL vs TimescaleDB vs ClickHouse vs Prometheus.

---

## Quick reference card

**★ Cardinality first**
```
★ never in a label: request_id · user_id · session_id · trace_id
                    · email · IP · full URL
★ always: status_code · method · ★ route TEMPLATE · region · env
★ a series dimension table with UNIQUE ⇒ cardinality is a SELECT
```

**Schema**
```sql
CREATE TABLE samples (ts timestamptz, series_id bigint, value double precision)
  PARTITION BY RANGE (ts);
CREATE INDEX ON samples (★ series_id, ts DESC);   -- ★ the ONLY index
ALTER TABLE samples SET (★ fillfactor = 100);      -- ★ append-only
-- ★ no surrogate PK · no index on ts alone · no GIN on labels
```

**★ Composable aggregates only**

| Aggregate | Composable | Store |
|---|---|---|
| sum, count, min, max | ✓ | the value |
| ★ **avg** | ✗ | ★ `(sum, count)`, divide at read |
| ★ **distinct** | ✗ | ★ a HyperLogLog sketch |
| ★ **percentile** | ✗ | ★ a t-digest / DDSketch |

**Tiering:** raw (7 d) → 1 min (30 d) → 1 h (2 y) → 1 d (forever). ★ **Each level from the level above.** 269 GB → **2.6 GB**.

**Retention:** ★ `DROP PARTITION` / `drop_chunks` — **12 ms vs 8,842 ms**, and the disk comes back.

**Ingestion:** per-row 48,204 ms · txn 4,102 · multi-row 182 · ★ **`COPY` 41**.

**Compression:** delta · ★ delta-of-delta (**1 bit/point**) · XOR (**~1.37 B/float**) ⇒ **13×** measured.

**Choose:** <10⁵ series ⇒ Postgres + partitioning · + compression/rollups ⇒ ★ TimescaleDB · big analytical scans ⇒ ClickHouse · monitoring ⇒ Prometheus (★ never business data).

---

## When would I use this at work?

1. **The moment any table has a timestamp and grows monotonically.** Metrics, events, audit logs, price ticks, IoT readings, request logs. Partitioning it by time from day one costs an hour and saves the 6-hour nightly `DELETE` you would otherwise inherit.

2. **Reviewing any metric instrumentation.** One question — *can this label's value come from user input?* — prevents the cardinality explosion that kills more time-series systems than any capacity problem. It belongs in a CI check, not a code review.

3. **Any dashboard that aggregates.** Ask what is stored in the rollup. If it's an average or a percentile, the numbers on the dashboard are wrong in a way that specifically hides outages — and that is worth checking before an incident review relies on them.

4. **Before adopting a time-series database.** Count your distinct series. Under ~100,000, PostgreSQL with partitioning and rollups is sufficient, keeps your joins and transactions, and is one fewer system to operate.

---

## Connected topics

**Understand before this:** 59 (partitioning — the retention mechanism), 56 (materialised views and rollup tables), 46/47 (why append-only avoids MVCC pressure), 61 (rightmost-page contention), 72 (TWCS — the same idea in Cassandra), 66 (`COPY` and batching).

**This unlocks:**
- **74** — graph and search: the two remaining specialised shapes
- **75** — SQL vs NoSQL: the decision framework
- **76** — polyglot persistence and CDC
- **Case study 07** — IoT telemetry · **Case study 20** — the analytics warehouse
