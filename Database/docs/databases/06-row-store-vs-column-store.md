# 06 — Row Store vs Column Store
## Phase: Storage Internals

---

## ELI5 — The Simple Analogy

A school keeps student records two ways.

**The office** keeps one folder per student — name, marks, address, parent's phone, all together. When a parent walks in asking about their child, the clerk pulls **one folder** and has everything. Perfect. But when the principal asks *"what is the average maths mark across all 4,000 students?"*, the clerk must open all 4,000 folders and read one line from each. Four thousand folders touched to read four thousand numbers.

**The exam department** keeps it differently — one long sheet per subject. A "Maths marks" sheet with 4,000 numbers on it. A "Name" sheet. A "Address" sheet. Now the principal's question is one sheet, read top to bottom, done in seconds. But when a parent walks in, the clerk must visit **every sheet** and find row 1,847 on each one.

Neither is better. They are optimised for opposite questions. The office is a **row store**. The exam department is a **column store**.

---

## Where this fits in the big picture

```
   04 Pages           05 Tuple layout
   (8KB folders)      (one row's bytes)
          \              /
           \            /
            ▼          ▼
        06 ROW vs COLUMN  ← YOU ARE HERE
        (the same bytes, arranged the other way)
             │
             ├──→ 07 buffer pool (why layout decides cache efficiency)
             ├──→ 19 join algorithms (columnar changes the cost model)
             ├──→ 55 denormalisation (rollup tables = poor man's columnar)
             └──→ 20 analytics warehouse (case study — OLTP/OLAP split)
```

Topics 04 and 05 showed you a row store in complete detail. **This topic shows you the alternative, so you know what you gave up.**

---

## What is this?

A **row store** (row-oriented storage) keeps all the columns of one row physically adjacent. A **column store** keeps all the values of one column physically adjacent, in separate files, aligned by position.

PostgreSQL, MySQL/InnoDB, Oracle, SQL Server (by default), MongoDB — row stores.
ClickHouse, DuckDB, BigQuery, Redshift, Snowflake, Parquet files, Apache Druid — column stores.

The choice determines which queries read 2 MB and which read 200 GB, for the same data.

---

## Why does it matter for a backend developer?

Because at some point your company will want a dashboard, and someone will point it at your production PostgreSQL, and everything will catch fire. Knowing *why* — and what the fix actually is — is the difference between "add a read replica and pray" and a real answer.

Concretely:

- `SELECT sum(total_paise) FROM orders` on 60M rows in PostgreSQL reads **every byte of every row** — 42 GB — to sum one 8-byte column. In ClickHouse it reads a ~180 MB compressed column file. That's a 230× difference from layout alone.
- That 42 GB scan **evicts your entire hot working set from `shared_buffers`**. Your checkout API's p99 goes from 80 ms to 900 ms while the dashboard loads, and nobody connects the two.
- Column stores compress 5–20×; row stores typically 2–3×. Because adjacent values in a column are the same type and often nearly identical, run-length and delta encoding become devastatingly effective.
- And the reverse trap: someone puts your **OLTP workload** on ClickHouse because "it's faster," and now a single-row update rewrites entire column blocks.

---

## The physical reality

The same 4 rows, both layouts, byte by byte.

```
LOGICAL TABLE  orders
┌────┬─────────┬──────────────┬────────┬──────────────┐
│ id │ user_id │ total_paise  │ status │ created_at   │
├────┼─────────┼──────────────┼────────┼──────────────┤
│ 91 │       7 │       249900 │ paid   │ 2026-08-01   │
│ 92 │       7 │       129900 │ paid   │ 2026-08-01   │
│ 93 │      12 │        45000 │ pending│ 2026-08-02   │
│ 94 │      12 │       899000 │ paid   │ 2026-08-02   │
└────┴─────────┴──────────────┴────────┴──────────────┘
```

**ROW STORE — one heap file, tuples laid end to end (Topics 04, 05):**

```
base/16388/16390, page 0
┌───────────────────────────────────────────────────────────────────────┐
│ hdr │ lp1 lp2 lp3 lp4 │        free         │ T4 │ T3 │ T2 │ T1       │
└───────────────────────────────────────────────────────────────────────┘
                                                            │
   T1 = [24B hdr][91][7][249900]['paid'][2026-08-01]  ← 64 bytes, contiguous
   T2 = [24B hdr][92][7][129900]['paid'][2026-08-01]
   T3 = [24B hdr][93][12][45000]['pending'][2026-08-02]
   T4 = [24B hdr][94][12][899000]['paid'][2026-08-02]

   "give me order 91"    → 1 page read. Everything is right there. ✓
   "sum all total_paise" → read all 64-byte tuples to extract 4×8 bytes.
                           87.5% of the I/O is wasted.                 ✗
```

**COLUMN STORE — one file per column, aligned by row position:**

```
orders/id.col           [91][92][93][94]                    32 B  → delta-encoded: 91,+1,+1,+1 → 12 B
orders/user_id.col      [7][7][12][12]                      32 B  → RLE: (7×2)(12×2)       → 8 B
orders/total_paise.col  [249900][129900][45000][899000]     32 B  → delta+bitpack          → 18 B
orders/status.col       ['paid']['paid']['pending']['paid'] ~28 B → dictionary: {0:paid,1:pending}
                                                                     values [0][0][1][0]   → 4 B + dict
orders/created_at.col   [d1][d1][d2][d2]                    32 B  → RLE                    → 8 B

   "sum all total_paise" → read ONE file, 18 bytes compressed. ✓✓✓
   "give me order 91"    → seek into 5 separate files at position 0,
                           reassemble the row.                      ✗
```

### Why compression works so much better on columns

This is the part people underestimate. It is not a minor bonus — it is usually the *dominant* effect.

```
A ROW is heterogeneous:   [bigint][bigint][text][timestamp][bool][jsonb]
  → adjacent bytes are unrelated. General-purpose compressors (lz4, zstd)
    find few repeated patterns. Typical ratio: 2–3×.

A COLUMN is homogeneous:  [2026-08-01][2026-08-01][2026-08-01][2026-08-02]...
  → adjacent values are the same type, often nearly identical, often SORTED.
    Specialised encodings apply:

    RLE (run-length)      'paid','paid','paid','paid'  →  ('paid', 4)
    DELTA                 1000,1001,1002,1003          →  1000,+1,+1,+1
    DICTIONARY            'Mumbai','Delhi','Mumbai'    →  dict{0:Mumbai,1:Delhi}, [0,1,0]
    BIT-PACKING           values 0–3 need 2 bits, not 32
    FOR (frame of ref)    store min, then offsets

    Typical ratio: 5–20×. On low-cardinality columns: 100×+.
```

And compression compounds with the layout win: you read *fewer* columns, and each one is *smaller*.

### Where the data actually lives

```
COLUMNAR ON-DISK ORGANISATION (ClickHouse MergeTree / Parquet — same idea)

  Table
   └── PART / ROW GROUP  (e.g. 8,192 – 1,000,000 rows)
        ├── column: id            [compressed block][compressed block]...
        ├── column: user_id       [compressed block]...
        ├── column: total_paise   [compressed block]...
        └── METADATA per block:  min, max, count, null_count
             ▲
             └── THE SECOND BIG WIN: ZONE MAPS / MIN-MAX PRUNING
                 WHERE created_at >= '2026-08-01'
                 → skip every block whose max(created_at) < 2026-08-01
                 → often skips 99% of blocks WITHOUT AN INDEX

  PostgreSQL's BRIN index is exactly this idea bolted onto a row store. (Topic 16)
```

---

## How it works — step by step

**Query: `SELECT status, sum(total_paise) FROM orders WHERE created_at >= '2026-08-01' GROUP BY status;`** over 60M rows.

```
════ ROW STORE (PostgreSQL) ════

 1. PLAN: no index on created_at that helps (60% of rows match) → Seq Scan.
 2. For each of 12,471 × 8KB pages... actually for 60M rows at 320 B/row:
       relpages ≈ 2,600,000 pages = 21 GB
 3. For each page: read 8192 bytes into a buffer  [EVICTING hot pages]
 4. For each tuple: check visibility (xmin/xmax vs snapshot)
 5. Deform the tuple: walk from t_hoff, applying alignment, to reach
    created_at (attnum 5) and total_paise (attnum 3)
 6. Evaluate the filter. Discard ~40% of rows.
 7. Feed survivors into a HashAggregate keyed on status.
 8. Emit 4 groups.

 I/O: 21 GB read. Bytes actually needed: 60M × (8 + 8 + 1) = 1.0 GB.
 EFFICIENCY: 4.8%.  Runtime: ~90 s. Buffer pool: destroyed.

════ COLUMN STORE (ClickHouse) ════

 1. PLAN: read 3 columns — created_at, total_paise, status. Ignore the other 28.
 2. ZONE MAP PRUNING: for each block, is max(created_at) < '2026-08-01'?
       → 40% of blocks skipped entirely, never decompressed.
 3. Read created_at column for surviving blocks:  RLE-compressed → 140 MB
 4. VECTORISED FILTER: process 65,536 values at a time in a SIMD loop,
    producing a bitmap of matching positions. No per-row function calls.
 5. Read total_paise for matching positions only:  ~380 MB
 6. Read status (dictionary-encoded): aggregate on the DICTIONARY CODES
    (small ints), not the strings — 4 distinct values, 4 counters.
 7. Emit 4 groups.

 I/O: ~520 MB read.  Runtime: ~0.4 s.
 EFFICIENCY: near 100%. And nothing else in the system was disturbed.

════ THE REVERSE QUERY: SELECT * FROM orders WHERE id = 91 ════

 ROW STORE:    index lookup → 1 heap page → done.            ~0.1 ms
 COLUMN STORE: locate row position 4,182,991 → seek into
               31 separate column files → decompress 31 blocks
               (each block holds ~65k values; you need 1 from each)
               → reassemble.                                  ~40 ms

 400× SLOWER for a point lookup. This is why column stores are not databases
 for your API. They are databases for your dashboard.
```

---

## Concept breakdown

```
ROW-ORIENTED  (N-ary Storage Model, "NSM")
│
├── All columns of one row are contiguous
├── Optimised for: fetch/insert/update WHOLE ROWS by identity
├── Write path: one row = one small write in one place
└── Cost: reading one column requires touching every row

COLUMN-ORIENTED  (Decomposition Storage Model, "DSM")
│
├── All values of one column are contiguous; rows reconstructed by POSITION
├── Optimised for: scan few columns over many rows
├── Write path: one row = a write into EVERY column file → batched, never single-row
└── Cost: reconstructing a row requires N seeks

THE FOUR MECHANISMS COLUMNAR WINS BY  (know all four — people only name the first)
│
├── 1. PROJECTION       read only the columns you asked for
├── 2. COMPRESSION      homogeneous data compresses 5–20× vs 2–3×
├── 3. ZONE MAPS        per-block min/max lets you skip blocks with no index
└── 4. VECTORISATION    process 65k values per loop iteration, SIMD, no
                        per-tuple function-call overhead
   ⇒ Mechanisms 2–4 are only POSSIBLE because of the layout. That's why you
     can't bolt "columnar performance" onto a row store with an index.

HYBRID / PAX  (used by Parquet, ORC, and SQL Server columnstore)
│
└── Partition rows into ROW GROUPS (e.g. 1M rows), then store COLUMN-WISE
    within each group. Gets columnar scan benefits while keeping a whole
    row within one group — so row reconstruction is one group, not one file.

OLTP  vs  OLAP  (the workload names)
│
├── OLTP: many small point reads/writes, latency-bound, high concurrency
│         → ROW STORE
└── OLAP: few huge scans + aggregates, throughput-bound, low concurrency
          → COLUMN STORE
   HTAP: the (mostly aspirational) attempt to serve both from one engine.
```

---

## Diagrams

**Diagram 1 — big picture: the same bytes, two arrangements**

```
              ROW STORE                          COLUMN STORE
        ┌──────────────────────┐          ┌────┐┌────┐┌────┐┌────┐
 row 1  │ id │uid │total│status│    id →  │ 91 ││  7 ││249k││paid│
        ├──────────────────────┤          │ 92 ││  7 ││129k││paid│
 row 2  │ id │uid │total│status│          │ 93 ││ 12 ││ 45k││pend│
        ├──────────────────────┤          │ 94 ││ 12 ││899k││paid│
 row 3  │ id │uid │total│status│          └────┘└────┘└────┘└────┘
        ├──────────────────────┤            id   uid  total status
 row 4  │ id │uid │total│status│
        └──────────────────────┘          ← separate files →

   read one ROW  ──▶  ONE seek           read one ROW  ──▶  4 seeks
   read one COL  ──▶  4 seeks            read one COL  ──▶  ONE seek
```

**Diagram 2 — data flow: what each layout reads for `SUM(total_paise)`**

```
 ROW STORE                            COLUMN STORE
 ─────────────────────────────        ────────────────────────────────
 page 0  ████████████████ read        total_paise.col
 page 1  ████████████████ read           block 0  ██ read (compressed)
 page 2  ████████████████ read           block 1  ██ read
   ...   (2.6M pages)                    block 2  ██ read
 page N  ████████████████ read           ...      (few hundred blocks)

 ████ = bytes read                    ██ = bytes read
 ░░░░ = bytes needed                  ██ = bytes needed  (they're the same)

 21 GB read / 1 GB needed             520 MB read / 480 MB needed
        4.8% efficient                       ~92% efficient
```

**Diagram 3 — before/after: what the analytics query does to your OLTP system**

```
 BEFORE the dashboard query runs
 ┌─────────────── shared_buffers (16 GB) ───────────────┐
 │ users idx  │ orders idx │ products │ sessions │ hot  │
 │   3 GB     │   5 GB     │  2 GB    │  1 GB    │ 5 GB │
 └───────────────────────────────────────────────────────┘
   buffer hit ratio: 99.4%      API p99: 80 ms

 DURING the 21 GB sequential scan
 ┌─────────────── shared_buffers (16 GB) ───────────────┐
 │ orders heap pages from 2026-01 … 2026-08 (useless    │
 │ to the API, evicted everything the API needed)       │
 └───────────────────────────────────────────────────────┘
   buffer hit ratio: 71%        API p99: 900 ms   ← THE INCIDENT

 ⚠ PostgreSQL mitigates this: sequential scans of tables larger than
   shared_buffers/4 use a small RING BUFFER (256 KB) instead of the whole
   pool. It helps a lot — but index scans, sorts, and hash joins from the
   same query still evict, and the OS page cache is still trashed.
```

---

## Example 1 — basic

Prove the projection and compression differences inside PostgreSQL itself.

```sql
CREATE TABLE orders_wide (
  id bigserial PRIMARY KEY,
  user_id bigint, total_paise bigint, status text,
  created_at timestamptz,
  shipping_json jsonb, notes text, gateway_ref text
);
INSERT INTO orders_wide (user_id, total_paise, status, created_at, shipping_json, notes, gateway_ref)
SELECT (random()*100000)::bigint, (random()*500000)::bigint,
       (ARRAY['paid','pending','shipped','cancelled'])[1+(random()*3)::int],
       now() - (random()*365)::int * interval '1 day',
       jsonb_build_object('city','Mumbai','pin', (400000+(random()*99)::int)),
       repeat('note ', 20), md5(random()::text)
FROM generate_series(1, 2000000);

VACUUM ANALYZE orders_wide;

EXPLAIN (ANALYZE, BUFFERS) SELECT sum(total_paise) FROM orders_wide;
```
```
Finalize Aggregate  (actual time=2841.2..2841.2 rows=1 loops=1)
  Buffers: shared read=76924                    ← 76,924 pages = 601 MB
Execution Time: 2843.9 ms
```

601 MB read to sum 2,000,000 × 8 bytes = **16 MB of actual data. 2.6% efficient.**

Now simulate a column store — a table containing *only* that column:

```sql
CREATE TABLE orders_total_only AS SELECT total_paise FROM orders_wide;
VACUUM ANALYZE orders_total_only;
EXPLAIN (ANALYZE, BUFFERS) SELECT sum(total_paise) FROM orders_total_only;
```
```
Aggregate  (actual time=178.4..178.4 rows=1 loops=1)
  Buffers: shared read=8850                     ← 8,850 pages = 69 MB
Execution Time: 179.1 ms
```

**16× faster, 8.7× less I/O — from projection alone**, with no compression, no vectorisation, no zone maps. Those three would take it another 10–20× further.

And prove the compression argument:

```sql
-- column-homogeneous data vs row-heterogeneous data, same total bytes
SELECT pg_size_pretty(pg_relation_size('orders_wide'))       AS row_layout,
       pg_size_pretty(pg_relation_size('orders_total_only')) AS one_column;
```
```
 row_layout | one_column
------------+------------
 601 MB     | 69 MB
```

---

## Example 2 — production scenario

**The situation.** Your e-commerce backend, PostgreSQL primary + 2 replicas. The BI team built a Metabase dashboard with 14 tiles. It refreshes every 5 minutes and points at replica #2.

At 09:00 every weekday, checkout p99 jumps from 90 ms to 1.4 s for four minutes. Nobody can reproduce it. It doesn't correlate with traffic.

**Step 1 — find it.**

```sql
SELECT calls, round(total_exec_time::numeric/1000) AS total_s,
       round(mean_exec_time::numeric) AS mean_ms, rows,
       shared_blks_read, left(query, 70) AS q
FROM pg_stat_statements ORDER BY total_exec_time DESC LIMIT 5;
```
```
 calls | total_s | mean_ms |  rows  | shared_blks_read |                q
-------+---------+---------+--------+------------------+-----------------------------
   288 |   24910 |   86493 |    288 |        892400000 | SELECT date_trunc('day', ...
   288 |   19204 |   66681 |   8640 |        710200000 | SELECT c.name, sum(oi.qty...
```

Two dashboard tiles have read **892 million + 710 million buffers**. Each run scans 18 months of `orders` joined to `order_items`.

**Step 2 — why the replica didn't protect production.** It's a *physical* replica: it replays WAL from the primary. It has its own `shared_buffers`, so the buffer eviction is contained to the replica — good. But:

- The dashboard's long-running queries hold **old snapshots**, which (with `hot_standby_feedback = on`) prevent the *primary* from vacuuming dead tuples. `orders` bloat climbed 40% in three weeks.
- With `hot_standby_feedback = off` instead, the dashboard queries get cancelled mid-flight with `ERROR: canceling statement due to conflict with recovery` — so the BI team turned it on.
- Replica #2's disk is now saturated at 09:00, so replay lag hits 40 s, so pattern-E reads routed there return stale orders.

**Step 3 — the options, honestly compared.**

| Option | Effort | Effect | When it's right |
|---|---|---|---|
| **Add indexes** | low | ~nothing. These queries touch 60% of the table; the planner correctly chooses a seq scan | Never for this shape |
| **Materialised views** refreshed nightly | low | Dashboard reads a 40 MB matview instead of 21 GB. Tiles go 86 s → 40 ms | ✅ **Do this first.** Solves it for most companies, permanently (Topic 56) |
| **BRIN index on `created_at`** | low | If `orders` is naturally time-ordered, BRIN gives you zone-map-style block pruning in PostgreSQL — a 40 KB index that skips 90% of blocks for time-ranged queries | ✅ Cheap, big win for date-filtered scans (Topic 16) |
| **Partition `orders` by month** | medium | Partition pruning: "last 30 days" reads 1 partition, not 18 months | ✅ Do this anyway for retention (Topic 59) |
| **Dedicated columnar store** (ClickHouse/DuckDB) fed by CDC | high | Tiles go to sub-second on *any* query, including ones nobody predicted. Zero impact on OLTP | ✅ When the BI team writes ad-hoc queries you can't pre-materialise (Topic 76) |
| **Migrate everything to columnar** | catastrophic | Point lookups become 40 ms; single-row updates rewrite blocks | ❌ Never |

**Step 4 — what you actually ship.** Matviews + BRIN + monthly partitioning, in that order, in one sprint. Tiles drop from 86 s to under 200 ms. Bloat stops. Replica lag returns to under 1 s.

You revisit ClickHouse when — and only when — the BI team's queries become genuinely ad-hoc and you're refreshing 40 matviews.

**The transferable lesson:** the columnar *idea* (read fewer columns, prune blocks, precompute) can be bought incrementally inside PostgreSQL. You rarely need the columnar *engine* until your analytics workload is unpredictable.

---

## Common mistakes

**1. Running analytics on the OLTP primary.**
- *Symptom:* periodic, unexplained p99 spikes that correlate with a cron schedule, not with traffic.
- *Engine-level why:* a large sequential scan consumes buffer pool, disk bandwidth, and (via long snapshots) blocks vacuum. The workloads compete for physically shared resources.
- *Diagnose:* `pg_stat_statements` ordered by `shared_blks_read`; correlate the timestamps with your latency graph.
- *Fix:* replica → matview → partition → columnar store, in that order of cost.

**2. Believing a read replica fully isolates analytics.**
- *Symptom:* replica added, primary still bloats and still slows down.
- *Engine-level why:* `hot_standby_feedback = on` propagates the replica's oldest snapshot back to the primary, pinning dead tuples cluster-wide. Off, and long queries get cancelled by replay conflicts.
- *Diagnose:* on the primary, `SELECT max(age(backend_xmin)) FROM pg_stat_replication;`
- *Fix:* a dedicated replica with `max_standby_streaming_delay` tuned and `hot_standby_feedback = off`, accepting cancellations — or move analytics off PostgreSQL entirely.

**3. Adding indexes to fix aggregate scans.**
- *Symptom:* six new indexes, no improvement, slower writes.
- *Engine-level why:* an index scan that returns >5–10% of the table costs *more* than a sequential scan (random I/O vs sequential). The planner knows this and ignores your index. The problem is the volume of data read, which an index cannot reduce below the rows that match.
- *Fix:* a covering index (index-only scan) helps if you need few columns — that is literally you hand-building a column store. Otherwise: precompute.

**4. Putting OLTP on a column store.**
- *Symptom:* `UPDATE users SET last_seen = now() WHERE id = 7` takes 200 ms; the system falls over at 500 writes/sec.
- *Engine-level why:* one row update touches every column file, and each file is compressed in large blocks — modifying one value requires decompressing, editing, and recompressing a block. Column stores handle this with append-only parts + background merges, which means single-row updates are pathologically expensive and deletes are logical, not physical.
- *Fix:* column stores ingest in **batches** (10k+ rows). If you need single-row mutation, you need a row store.

**5. Treating "HTAP" marketing as solved.**
- *Symptom:* a vendor promises one engine for both; six months later, both workloads are mediocre.
- *Reality:* the layouts are genuinely opposed. Real HTAP systems keep *two* copies (a row store for writes, a column store for reads) and sync them — which is exactly the two-store architecture, just managed for you. Evaluate it as that, and price the sync lag.

---

## Hands-on proof

**PROVE IT #1 — the projection penalty (repeat of Example 1, condensed).**
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT sum(total_paise) FROM orders_wide;      -- read≈76900
EXPLAIN (ANALYZE, BUFFERS) SELECT sum(total_paise) FROM orders_total_only;-- read≈8850
```

**PROVE IT #2 — build a covering index and watch PostgreSQL become semi-columnar.**
```sql
CREATE INDEX idx_orders_cover ON orders_wide (created_at) INCLUDE (total_paise, status);
VACUUM ANALYZE orders_wide;
EXPLAIN (ANALYZE, BUFFERS)
SELECT status, sum(total_paise) FROM orders_wide
WHERE created_at > now() - interval '30 days' GROUP BY status;
```
```
HashAggregate ...
  ->  Index Only Scan using idx_orders_cover on orders_wide
        Heap Fetches: 0                    ← the heap was NEVER touched
        Buffers: shared hit=284 read=1902  ← 2,186 pages instead of 76,924
```
**That index is a column store containing three columns.** This is the single most useful columnar technique available inside PostgreSQL (Topic 12).

**PROVE IT #3 — BRIN as a zone map.**
```sql
CREATE INDEX idx_orders_brin ON orders_wide USING BRIN (created_at)
  WITH (pages_per_range = 32);
SELECT pg_size_pretty(pg_relation_size('idx_orders_brin'));
```
```
 pg_size_pretty
----------------
 72 kB              ← 72 KB for a 601 MB table
```
```sql
-- physically order the table by time so BRIN's min/max are tight:
CLUSTER orders_wide USING idx_orders_cover;   -- (or insert in time order naturally)
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM orders_wide
WHERE created_at BETWEEN '2026-07-01' AND '2026-07-31';
```
```
Bitmap Heap Scan on orders_wide
  Recheck Cond: ...
  ->  Bitmap Index Scan on idx_orders_brin
  Buffers: shared read=6431               ← 91% of blocks skipped
```
BRIN only works when the column correlates with physical row order. Check it:
```sql
SELECT attname, correlation FROM pg_stats
WHERE tablename='orders_wide' AND attname='created_at';
-- correlation near 1.0 or -1.0 → BRIN is great. Near 0 → BRIN is useless.
```

**PROVE IT #4 — compression ratio, column vs row.**
```bash
# Export both ways and compare compressed sizes
psql -c "COPY (SELECT * FROM orders_wide) TO STDOUT" | zstd | wc -c        # row-wise
psql -c "COPY (SELECT total_paise FROM orders_wide) TO STDOUT" | zstd | wc -c
psql -c "COPY (SELECT status FROM orders_wide) TO STDOUT" | zstd | wc -c   # 4 distinct values
```
The `status` column — 2M values, 4 distinct — compresses to a few kilobytes. The same 2M values interleaved inside rows compress far worse.

**PROVE IT #5 — a real column store, on your laptop, in 30 seconds.**
```sql
-- DuckDB reads PostgreSQL directly, and is columnar + vectorised
-- $ duckdb
INSTALL postgres; LOAD postgres;
ATTACH 'dbname=shop user=postgres host=localhost' AS pg (TYPE postgres);
CREATE TABLE local_orders AS SELECT * FROM pg.orders_wide;   -- now columnar
.timer on
SELECT status, sum(total_paise) FROM local_orders GROUP BY status;
```
Compare with the PostgreSQL timing on identical data and hardware. Typical result: 2,840 ms → 45 ms.

---

## The design decision framework

```
USE A ROW STORE WHEN:
  ✓ You fetch, insert, or update whole rows by identity
  ✓ Concurrency is high and transactions are short
  ✓ Single-row mutation is a normal operation
  ✓ Queries are selective (touch < 1% of rows)
  ✓ You need real transactions, foreign keys, and constraints
  → This is your API's database. Always.

USE A COLUMN STORE WHEN:
  ✓ Queries scan millions of rows and touch a handful of columns
  ✓ Data is append-only or batch-loaded (not single-row updated)
  ✓ Aggregations dominate: sum, count, avg, group by, window functions
  ✓ Queries are ad-hoc — you can't predict them well enough to pre-materialise
  ✓ Storage volume makes 10× compression financially meaningful
  → This is your dashboard's database. Never your API's.

BUY COLUMNAR BEHAVIOUR INSIDE POSTGRESQL FIRST — in this order:
  1. COVERING INDEX (INCLUDE)     → index-only scan = a 3-column column store
  2. BRIN index                   → zone maps, if data is physically time-ordered
  3. PARTITIONING by time         → partition pruning
  4. MATERIALISED VIEW            → precompute the aggregate entirely
  5. SUMMARY/ROLLUP TABLE         → incremental, refreshed on write
  Only after all five are exhausted does a separate columnar engine pay for itself.

THE SIGNAL TO LOOK FOR:
      SELECT round(shared_blks_read * 8192 / 1024/1024) AS mb_read,
             rows, round(mean_exec_time) AS ms, left(query,60)
      FROM pg_stat_statements ORDER BY shared_blks_read DESC LIMIT 10;

  For each query compute:   bytes_needed / bytes_read
      bytes_needed ≈ rows_returned × sum(width of selected columns)
  • ratio > 20%  → row store is fine, this query is doing real work
  • ratio < 5% AND it runs often → you have a columnar-shaped query.
    Work down the five-step list above.
  • And the decisive one: does this query's I/O volume exceed shared_buffers?
    If yes, it is evicting your OLTP working set, and it must move — replica,
    matview, or another engine.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Take `orders_wide` from Example 1. For `SELECT count(*) FROM orders_wide WHERE status = 'paid'`, compute: bytes actually needed, bytes read (from `EXPLAIN BUFFERS`), and the efficiency ratio. Then create the narrowest index that makes this an index-only scan, and recompute all three numbers.

### Exercise 2 — medium (apply it)
You have a 40-column `events` table, 200M rows. The three dashboard queries all filter on `occurred_at` (a time range) and aggregate 2–3 columns. Without adding a new database:
(a) design the covering index(es),
(b) decide whether BRIN or B-tree is right and prove it with `pg_stats.correlation`,
(c) measure the before/after in pages read,
(d) state precisely what you'd have to change about the *write* path for BRIN to keep working over time.

### Exercise 3 — hard (production simulation)
Your `orders` table is 2.1 TB. The BI team runs ~200 distinct ad-hoc queries per day; you cannot pre-materialise them because they're genuinely unpredictable. Analytics currently runs on a replica with `hot_standby_feedback = on`, and the primary's bloat is growing 3% per week.

(a) Explain the exact mechanism linking `hot_standby_feedback` to primary bloat.
(b) Explain what changes if you set it to `off`, and what new failure the BI team sees.
(c) Design a CDC pipeline that feeds a columnar store from PostgreSQL. Name what carries the changes, what the lag budget is, and how you handle the initial 2.1 TB backfill without taking a lock.
(d) The BI team asks for "real-time" dashboards. Explain what latency is actually achievable through your pipeline and where the floor comes from.
(e) One BI query needs to join against `users` — which lives only in PostgreSQL and changes constantly. Give two designs and their staleness characteristics.

---

## Mental model checkpoint

1. Name the four mechanisms by which a column store beats a row store on an aggregate query. Which ones are *impossible* in a row store, and why?
2. Why does a column of 2M `status` values compress ~100× while the same values inside rows compress ~3×?
3. Why is a point lookup (`WHERE id = 91`) slower in a column store? Quantify roughly.
4. What is a zone map / min-max block index, and which PostgreSQL feature is the same idea?
5. A covering index with `INCLUDE` is described as "a poor man's column store." Explain exactly why that's accurate.
6. Your dashboard query runs on a read replica. Name two mechanisms by which it can *still* hurt the primary.
7. Why can't you get columnar performance in PostgreSQL just by adding the right index? (Answer must reference at least two of the four mechanisms.)

---

## Quick reference card

| | Row store | Column store |
|---|---|---|
| Layout | all columns of a row contiguous | all values of a column contiguous |
| Point lookup | ~0.1 ms | ~10–50 ms |
| Full-column aggregate | reads whole table | reads one file |
| Compression | 2–3× | 5–20× (100×+ low cardinality) |
| Single-row UPDATE | cheap | pathological |
| Bulk load | fine | ideal |
| Concurrency | high, short txns | low, long queries |
| Transactions/FKs | full | limited or absent |
| Examples | PostgreSQL, MySQL, Oracle, Mongo | ClickHouse, DuckDB, BigQuery, Parquet |

**The four columnar mechanisms:** projection · compression · zone maps · vectorisation

**PostgreSQL's columnar toolkit, cheapest first**

| Tool | Buys you | Cost |
|---|---|---|
| `INCLUDE` covering index | projection + index-only scan | write amplification, disk |
| BRIN | zone maps | requires physical correlation |
| Partitioning | block pruning | DDL complexity |
| Materialised view | full precompute | staleness, refresh cost |
| Rollup table | incremental precompute | write-path complexity |

**Numbers**

| Thing | Value |
|---|---|
| Efficiency ratio worth investigating | < 5% |
| Index scan beats seq scan below | ~5–10% of rows matched |
| Typical row-store compression | 2–3× |
| Typical column-store compression | 5–20× |
| Vectorised batch size | ~1,024–65,536 values |

---

## When would I use this at work?

1. **The 09:00 latency spike.** You correlate `pg_stat_statements` with the graph, find two dashboard queries reading 890M buffers, and can immediately explain the mechanism (buffer eviction + snapshot pinning) and propose matviews rather than "we need a bigger instance."

2. **A vendor pitch.** Someone proposes migrating your OLTP database to a columnar engine "because it benchmarks 100× faster." You can name the benchmark's shape (scan-and-aggregate), point out that your top 10 queries by call count are point lookups and single-row updates, and ask what a single-row update costs in their engine.

3. **Designing an events table.** Before it's written, you know it will be append-only, time-ranged, and aggregated. So you partition by month, add BRIN on `occurred_at`, and plan the CDC path to a columnar store from day one — instead of discovering all of it at 200M rows.

---

## Connected topics

**Understand before this:** 04 (pages), 05 (tuple layout — the row-store layout in full detail).

**This unlocks:**
- **12** — index-only scans, the covering-index mechanism used here
- **16** — BRIN indexes as zone maps
- **19** — join algorithms, whose cost model differs sharply in columnar engines
- **56** — materialised views: precompute as the cheap alternative
- **59** — partitioning and partition pruning
- **73** — time-series storage, which is columnar in practice
- **76** — CDC pipelines to feed a derived columnar store
- **Case study 20** — analytics warehouse, the full OLTP/OLAP separation
