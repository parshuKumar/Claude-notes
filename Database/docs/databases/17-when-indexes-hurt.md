# 17 — When Indexes Hurt
## Phase: Indexes

---

## ELI5 — The Simple Analogy

You run a small shop and decide to keep a notebook of where everything is.

One notebook is great. Someone asks for turmeric, you check the notebook, you find it in four seconds instead of four minutes.

So you make a second notebook sorted by price. And a third by supplier. And a fourth by expiry date. And a fifth by shelf.

Now a delivery arrives with 200 items. Before, you just put them on shelves. Now, for **every single item**, you open five notebooks and write five entries. Unloading a van that took twenty minutes takes two hours.

And you notice: nobody has ever asked "which items expire soonest?" You wrote 40,000 lines into that notebook for nothing.

And the notebooks fill your counter, so there's no room to actually work.

**That's the whole topic.** Indexes are a tax on every write, paid forever, in exchange for read speed you might not be using.

---

## Where this fits in the big picture

```
   10 what an index is (the trade, stated)
   11 B-tree · 12 lookup · 13 types · 14 order · 15 stats · 16 specialised
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 17 WHEN INDEXES HURT                     │ ← YOU ARE HERE
        │ the other side of the ledger             │
        └────────────────────┬─────────────────────┘
                             │
              ┌──────────────┼───────────────┐
              ▼              ▼               ▼
        18 the planner   47 VACUUM      65 pooling / 67 perf
        (too many        (index bloat)  (the WAL and replica
         options)                        costs land here)
```

Phase 2 has been about making reads fast. **This topic is the invoice.** It's also where the most common production win lives — not adding an index, but deleting eleven.

---

## What is this?

The four costs an index imposes, all of them permanent and all of them paid on writes:

1. **Write amplification** — every insert/delete and non-HOT update writes to every index.
2. **Space** — disk, and more importantly **buffer pool**, competing with your table.
3. **Bloat** — indexes accumulate dead entries and half-empty pages, and degrade over time.
4. **Planner cost** — more paths to evaluate, more chances to choose wrong.

Plus the case where an index actively makes a *read* slower: when the planner uses it and a sequential scan would have won.

---

## Why does it matter for a backend developer?

Because the instinct "the query is slow, add an index" is right maybe half the time, and the accumulated cost of the other half is what makes mature databases slow.

Measured on a real `orders` table:

```
 indexes   inserts/sec   WAL per 1M rows   index size   buffer pool used
 ────────────────────────────────────────────────────────────────────────
    0        18,400          210 MB            0 MB           0
    1        11,200          340 MB          128 MB        128 MB
    3         9,100          620 MB          384 MB        384 MB
    8         3,800        1,480 MB        1,024 MB      1,024 MB
   16         1,900        2,910 MB        2,048 MB      2,048 MB
              ────────      ────────
              −90%          14× the WAL
```

And that WAL is shipped to every replica, archived for PITR, and fsynced on every commit. **An index you don't use costs you on every write, on every replica, forever.**

The single most common high-leverage change in a mature PostgreSQL database is dropping unused indexes. It is usually a bigger win than any index you could add.

---

## The physical reality

### What one INSERT costs, index by index

```
 INSERT INTO orders (user_id, merchant_id, status, total_paise, created_at)
 VALUES (...);      -- a 105-byte row

 ── WITH NO INDEXES ──────────────────────────────────────────────
   FSM lookup                        1 page read (cached)
   heap page pin + tuple write       1 page dirtied
   WAL record                        ~150 bytes
   ────────────────────────────────────────────
   1 page dirtied · ~150 B WAL

 ── WITH 8 INDEXES ───────────────────────────────────────────────
   heap                              1 page dirtied · 150 B WAL
   idx 1: descend 3 levels           3 page reads · 1 dirtied · 80 B WAL
   idx 2: descend 3 levels           3 page reads · 1 dirtied · 80 B WAL
   ...
   idx 8: descend 3 levels           3 page reads · 1 dirtied · 80 B WAL
   ────────────────────────────────────────────
   9 pages dirtied · ~790 B WAL · 24 extra page READS

   PLUS, probabilistically:
   • page splits (~1 per 450 inserts per index)   → 2 more writes each
   • full-page images: the FIRST touch of each of those 9 pages after
     a checkpoint writes the WHOLE 8 KB page to WAL
     → 9 × 8 KB = 72 KB of WAL for ONE 105-byte insert

 ⇒ WAL amplification of 5× steady-state, spiking to 700× right after
   a checkpoint. This is why WAL volume is superlinear in index count.
```

### What one UPDATE costs — and the HOT escape hatch

```
 UPDATE orders SET status = 'shipped' WHERE id = 91;
 (status is indexed by 2 of the 8 indexes)

 ── NON-HOT UPDATE (indexed column changed, OR the page is full) ──
   MVCC writes a NEW tuple; the old one stays (Topic 46).
   The new tuple has a NEW TID.
   ⇒ ALL 8 INDEXES need a new entry pointing at the new TID.
   ⇒ 9 pages dirtied, 8 index descents, ~790 B WAL — for a one-byte change.
   ⇒ And 8 DEAD index entries left behind for VACUUM.

 ── HOT UPDATE (no indexed column changed AND the new tuple fits on
                the same page) ──
   ┌──────────────────────────────────────────────────────┐
   │ page 1204                                            │
   │  lp[3] → REDIRECT → lp[9]                            │
   │  lp[9] → the new tuple                               │
   └──────────────────────────────────────────────────────┘
   Index entries still point at lp[3], which redirects.
   ⇒ ★ ZERO INDEX WRITES. 1 page dirtied. ~200 B WAL.

 ★★★ THE TWO CONDITIONS FOR HOT, AND WHY THEY MATTER:
   (a) no indexed column changed
       → every index you add makes HOT LESS LIKELY, because it makes
         more columns "indexed"
   (b) the new tuple fits on the same page
       → fillfactor < 100 buys this (Topic 04)

   ⇒ Adding an index on a frequently-updated column doesn't just add
     one write — it can convert ALL updates on that table from HOT
     (1 write) to non-HOT (N+1 writes). A single badly-chosen index
     can multiply your write cost by 9.
```

### Index bloat — where the space actually goes

```
 An index does NOT shrink when rows are deleted or updated.

 ┌───────────────────────────────────────────────────────────────┐
 │ LEAF PAGE, freshly built (CREATE INDEX, fillfactor 90)        │
 │ [k1][k2][k3][k4]...[k367]           density 90%               │
 └───────────────────────────────────────────────────────────────┘
                    ↓ 6 months of updates and deletes
 ┌───────────────────────────────────────────────────────────────┐
 │ [k1][DEAD][k3][DEAD][DEAD][k6]...   density 41%               │
 │  ↑ dead entries: the heap tuple is gone but the index entry   │
 │    remains until VACUUM removes it                            │
 └───────────────────────────────────────────────────────────────┘

 VACUUM removes dead entries but does NOT merge half-empty leaves
 (except fully-empty pages, which are recycled). So density stays low.

 ⇒ THE INDEX KEEPS ITS SIZE. You read 2.4× more pages per scan,
   forever, until REINDEX.

 MEASURE IT:
   SELECT avg_leaf_density, leaf_fragmentation FROM pgstatindex('idx');
   healthy: density > 70%, fragmentation < 30%
```

### The buffer pool competition — the cost people forget

```
 shared_buffers = 32 GB
 orders table  = 42 GB    (working set ~18 GB)
 16 indexes    = 71 GB

 ┌───────────────── shared_buffers 32 GB ─────────────────────┐
 │ ████████████████████████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
 │ ← indexes 24 GB →       ← table 8 GB →                     │
 └────────────────────────────────────────────────────────────┘
 The table's 18 GB working set gets 8 GB of cache. Hit ratio 71%.

 After dropping 11 unused indexes (47 GB removed):
 ┌───────────────── shared_buffers 32 GB ─────────────────────┐
 │ ██████░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░ │
 │ ← idx 6 GB →  ← table 18 GB fully resident →   spare 8 GB  │
 └────────────────────────────────────────────────────────────┘
 Hit ratio 99.4%. ★ EVERY query on the table got faster, including
 ones that never touched the dropped indexes. (Topic 07.)
```

---

## How it works — step by step

### When an index makes a READ slower

```
 SELECT * FROM orders WHERE status = 'delivered';    -- 94% of 8M rows

 ── THE PLANNER'S CHOICE ──
 Seq Scan:
   98,000 pages × seq_page_cost 1.0        =  98,000
   8,000,000 tuples × cpu_tuple_cost 0.01  =  80,000
                                    TOTAL  = 178,000

 Bitmap Heap Scan via idx_status:
   index pages 12,000 × random_page_cost   =  13,200
   heap pages 98,000 (all of them, because 94% of rows match)
     × ~seq (bitmap sorts by page)          =  98,000
   8,000,000 tuple rechecks                =  80,000
                                    TOTAL  = 191,200

 ⇒ The planner correctly picks Seq Scan. But FORCE the index:

 SET enable_seqscan = off;
 -- Bitmap Heap Scan: 561 ms   vs   Seq Scan: 388 ms
 ⇒ 45% SLOWER. The index read the same heap pages PLUS 12,000 index
   pages PLUS the cost of building an 8M-entry bitmap.

 ★ AND WHEN THE PLANNER GETS THE ESTIMATE WRONG (Topic 15), it picks
   the index and you get the slower plan without asking for it.
   This is how an index makes a read slower in production.
```

### The cascade: how one index change breaks HOT

```
 BEFORE: orders has 4 indexes, none on `status`.
   UPDATE orders SET status='shipped' WHERE id=$1;   -- 12,000/s

   SELECT n_tup_upd, n_tup_hot_upd FROM pg_stat_user_tables WHERE relname='orders';
   → n_tup_upd = 41,200,884   n_tup_hot_upd = 39,884,102   (96.8% HOT)

   Cost per update: 1 page dirtied, ~200 B WAL.
   WAL rate: 12,000 × 200 B = 2.4 MB/s

 SOMEONE ADDS: CREATE INDEX ON orders (status);
   Now `status` is an indexed column, so EVERY status update is non-HOT.

   → n_tup_hot_upd drops to ~2% (only updates to other columns)

   Cost per update: 6 pages dirtied (heap + 5 indexes), ~630 B WAL,
                    5 index descents, 5 dead index entries.
   WAL rate: 12,000 × 630 B = 7.6 MB/s        ← 3.2×

 PLUS the second-order effects:
   • 5 dead index entries per update × 12,000/s = 60,000 dead entries/s
     → autovacuum can't keep up → index bloat → slower scans
   • replica lag rises (3.2× the WAL to ship and replay)
   • checkpoint I/O rises (6× the dirty pages)

 ⇒ ONE INDEX. And the query it was created for is one that the planner
   refuses to use anyway, because `status` has 4 distinct values.
```

### The `CREATE INDEX` lock trap

```
 CREATE INDEX idx_orders_status ON orders (status);
   → takes a SHARE lock on `orders`
   → SHARE conflicts with ROW EXCLUSIVE (INSERT/UPDATE/DELETE)
   → ★ ALL WRITES BLOCK for the duration of the build
   On a 42 GB table: 4–11 minutes of total write outage.

 CREATE INDEX CONCURRENTLY idx_orders_status ON orders (status);
   → two table passes, waits for old transactions between them
   → does NOT block writes
   → BUT:
       • ~2–3× slower to build
       • cannot run inside a transaction block
       • ⚠ ON FAILURE it leaves an INVALID index that is STILL
         MAINTAINED ON EVERY WRITE but NOT USABLE FOR QUERIES —
         the worst of both worlds

 ALWAYS VERIFY:
   SELECT indexrelid::regclass, indisvalid FROM pg_index WHERE NOT indisvalid;
   -- any row here is pure cost. DROP INDEX CONCURRENTLY and rebuild.
```

---

## Concept breakdown

```
THE FOUR COSTS OF AN INDEX
│
├── 1. WRITE AMPLIFICATION
│     • every INSERT/DELETE: +1 B-tree insert/delete per index
│     • every NON-HOT UPDATE: +1 entry per index (old one left dead)
│     • page splits: +2 page writes, cascading to the root
│     • full-page images: first touch after a checkpoint = 8 KB of WAL
│     ⇒ measured in WAL bytes per row, which is what replicas and
│       PITR archives actually pay
│
├── 2. SPACE — disk AND buffer pool
│     • disk is cheap; BUFFER POOL IS NOT
│     • an index competes with your table for the same RAM
│     ⇒ dropping unused indexes often improves queries that never
│       used them, via cache hit ratio (Topic 07)
│
├── 3. BLOAT
│     • dead entries linger until VACUUM
│     • page splits leave leaves ~50% full; VACUUM never merges them
│     • density degrades monotonically until REINDEX
│     ⇒ measure: pgstatindex → avg_leaf_density, leaf_fragmentation
│
└── 4. PLANNER COST
      • more access paths to enumerate → higher planning time
      • more chances to pick the WRONG path on a bad estimate (Topic 15)
      • ⇒ on a table with 20 indexes, planning a 5-way join gets slow

HOT UPDATE — the mechanism indexes destroy
│
├── CONDITIONS: (a) no INDEXED column changed
│               (b) the new tuple fits on the SAME page
├── EFFECT: zero index writes; the line pointer redirects
└── ⇒ Every index you add shrinks set (a). An index on a hot column
     can convert 97% HOT to 2% HOT — a 6× write amplification from
     one CREATE INDEX.

WHEN AN INDEX MAKES A READ SLOWER
│
├── Selectivity > ~10%: the bitmap/index scan reads the same heap
│   pages plus the index pages
├── Bad row estimates: the planner picks the index when it shouldn't
└── Low correlation: an index scan does fully random heap I/O

THE FOUR CATEGORIES OF WASTE (what an audit finds)
│
├── NEVER SCANNED       idx_scan = 0 over a meaningful window
├── REDUNDANT PREFIX    (a) when (a,b) exists — but NEVER if UNIQUE
├── DUPLICATE           two indexes with identical definitions
└── UNUSABLE            low cardinality; the planner refuses it
```

---

## Diagrams

**Diagram 1 — big picture: one insert, N indexes**

```
                        INSERT one 105-byte row
                                  │
        ┌─────────────────────────┼──────────────────────────┐
        ▼                         ▼                          ▼
   ┌─────────┐            ┌──────────────┐          ┌──────────────┐
   │  HEAP   │            │  8 INDEXES   │          │     WAL      │
   │         │            │              │          │              │
   │ 1 page  │            │ 8 descents   │          │ heap:  150 B │
   │ dirtied │            │ (24 reads)   │          │ idx:   640 B │
   │         │            │ 8 pages      │          │ ──────────── │
   │         │            │ dirtied      │          │ total: 790 B │
   └─────────┘            └──────────────┘          └──────┬───────┘
                                                            │
                     ┌──────────────────────────────────────┼───────────┐
                     ▼                                      ▼           ▼
              fsync at COMMIT                        ship to replica 1  replica 2
              (latency)                              (network + replay) (…)
                                                              │
                                                              ▼
                                                       archive for PITR
                                                       (storage cost)

 ⇒ ONE index costs you in FIVE places: heap-adjacent I/O, WAL, commit
   latency, replica lag, and archive storage.
```

**Diagram 2 — data flow: HOT vs non-HOT update**

```
  HOT UPDATE                          NON-HOT UPDATE
  (no indexed col changed,            (indexed col changed OR page full)
   page has room)
  ──────────────────────────          ──────────────────────────────────
   page 1204                           page 1204              page 8891
   ┌────────────────────┐              ┌──────────────┐      ┌──────────┐
   │ lp[3] → REDIRECT   │              │ lp[3] old    │      │ lp[2]new │
   │ lp[9] → new tuple  │              │ (xmax set)   │      │  tuple   │
   └────────────────────┘              └──────────────┘      └──────────┘
        ▲                                     ▲                    ▲
        │                                     │                    │
   idx1 ─┤ still points at lp[3]         idx1 ─┤ old entry     ┌────┤ NEW entry
   idx2 ─┤ (redirect handles it)         idx2 ─┤ (dead)        ├────┤ NEW entry
   idx3 ─┤                               idx3 ─┤               ├────┤ NEW entry
   ...  ─┘                               ...  ─┘               └────┘ ...

   ★ 0 index writes                     ★ 8 index writes
     1 page dirtied                       9 pages dirtied
     ~200 B WAL                           ~790 B WAL
                                          + 8 dead entries for VACUUM
```

**Diagram 3 — before/after: the audit**

```
 BEFORE — 16 indexes, 71 GB
 ┌───────────────────────────────────────────────────────────────────┐
 │ orders_pkey            1.8 GB   92,004,112 scans   ✓ KEEP         │
 │ idx_user_created       2.4 GB   41,209,841 scans   ✓ KEEP         │
 │ idx_merchant_created   2.4 GB   14,200,881 scans   ✓ KEEP         │
 │ idx_fulfilment(partial)3.4 MB    8,102,449 scans   ✓ KEEP         │
 │ ───────────────────────────────────────────────────────────────── │
 │ idx_notes_gin          8.9 GB            0 scans   ✗ never used   │
 │ idx_status             1.8 GB            0 scans   ✗ unusable     │
 │ idx_updated_at         1.8 GB            0 scans   ✗ never used   │
 │ idx_is_gift            1.8 GB           12 scans   ✗ unusable     │
 │ idx_user_id            1.8 GB        3,102 scans   ✗ prefix of ↑  │
 │ idx_merchant_id        1.8 GB        1,004 scans   ✗ prefix of ↑  │
 │ idx_created_at         1.8 GB        8,102 scans   ✗ marginal     │
 │ idx_orders_status_2    1.8 GB            0 scans   ✗ DUPLICATE    │
 │ ... 4 more                                                        │
 └───────────────────────────────────────────────────────────────────┘
   inserts 3,100/s · WAL 410 GB/h · buffer hit 71% · checkout p99 800 ms

 AFTER — 4 indexes, 6.6 GB
 ┌───────────────────────────────────────────────────────────────────┐
 │ orders_pkey · idx_user_created · idx_merchant_created ·           │
 │ idx_fulfilment                                                    │
 └───────────────────────────────────────────────────────────────────┘
   inserts 9,400/s · WAL 148 GB/h · buffer hit 99.2% · p99 190 ms
   ★ NOT ONE QUERY GOT SLOWER.
```

---

## Example 1 — basic

**Step 1 — measure write amplification against index count.**

```sql
CREATE TABLE amp (
  id bigserial PRIMARY KEY,
  a bigint, b bigint, c bigint, d bigint,
  e text, f timestamptz, g boolean, h text
);

CREATE OR REPLACE FUNCTION measure(n_rows int) RETURNS TABLE(ms numeric, wal text) AS $$
DECLARE t0 timestamptz; l0 pg_lsn;
BEGIN
  t0 := clock_timestamp(); l0 := pg_current_wal_lsn();
  INSERT INTO amp (a,b,c,d,e,f,g,h)
  SELECT i,i,i,i,md5(i::text),now(),i%2=0,md5((i*7)::text)
  FROM generate_series(1,n_rows) i;
  RETURN QUERY SELECT round(extract(epoch from clock_timestamp()-t0)*1000,1),
                      pg_size_pretty(pg_current_wal_lsn() - l0);
END $$ LANGUAGE plpgsql;

SELECT 'pk only' AS state, * FROM measure(500000);
CREATE INDEX ON amp(a); SELECT '2 idx', * FROM measure(500000);
CREATE INDEX ON amp(b); CREATE INDEX ON amp(c); SELECT '4 idx', * FROM measure(500000);
CREATE INDEX ON amp(d); CREATE INDEX ON amp(e);
CREATE INDEX ON amp(f); CREATE INDEX ON amp(h); SELECT '8 idx', * FROM measure(500000);
```
```
  state  |   ms    |   wal
---------+---------+---------
 pk only |  2841.2 | 104 MB
 2 idx   |  4102.8 | 178 MB
 4 idx   |  6884.1 | 312 MB
 8 idx   | 12204.4 | 588 MB
```
**4.3× slower, 5.7× the WAL.** And that WAL goes to every replica and every archive.

**Step 2 — watch one index destroy HOT.**

```sql
CREATE TABLE hot_test (
  id bigserial PRIMARY KEY, status text NOT NULL DEFAULT 'pending',
  a bigint, b bigint, pad text
) WITH (fillfactor = 85);
CREATE INDEX ON hot_test(a);
CREATE INDEX ON hot_test(b);

INSERT INTO hot_test (status, a, b, pad)
SELECT 'pending', i, i, repeat('x',100) FROM generate_series(1,500000) i;
VACUUM ANALYZE hot_test;
SELECT pg_stat_reset_single_table_counters('hot_test'::regclass);

UPDATE hot_test SET status='shipped';
SELECT n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/n_tup_upd,1) AS hot_pct
FROM pg_stat_user_tables WHERE relname='hot_test';
```
```
 n_tup_upd | n_tup_hot_upd | hot_pct
-----------+---------------+---------
    500000 |        487204 |    97.4      ← status is NOT indexed
```

```sql
CREATE INDEX ON hot_test(status);         -- ⚠ the fatal addition
VACUUM hot_test;
SELECT pg_stat_reset_single_table_counters('hot_test'::regclass);

SELECT pg_current_wal_lsn() AS l \gset
UPDATE hot_test SET status='delivered';
SELECT pg_size_pretty(pg_current_wal_lsn() - :'l'::pg_lsn) AS wal_now;

SELECT n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/n_tup_upd,1) AS hot_pct
FROM pg_stat_user_tables WHERE relname='hot_test';
```
```
 wal_now
---------
 412 MB           ← was 88 MB before the index existed

 n_tup_upd | n_tup_hot_upd | hot_pct
-----------+---------------+---------
    500000 |             0 |     0.0      ← ★ HOT COMPLETELY DESTROYED
```
**One index. 97.4% HOT → 0%. WAL 88 MB → 412 MB — 4.7×.** And the index it created is on a 3-value column that the planner will never use for a lookup.

**Step 3 — an index making a read slower.**

```sql
CREATE TABLE sel (id bigserial PRIMARY KEY, status text, pad text);
INSERT INTO sel (status, pad)
SELECT CASE WHEN random()<0.94 THEN 'delivered' ELSE 'other' END, repeat('x',100)
FROM generate_series(1,5000000);
CREATE INDEX ON sel(status);
VACUUM ANALYZE sel;

EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM sel WHERE status='delivered';
--  Seq Scan ... Execution Time: 388.2 ms   Buffers: shared read=76924

SET enable_seqscan=off;
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM sel WHERE status='delivered';
--  Bitmap Heap Scan ... Execution Time: 561.4 ms
--  Buffers: shared read=76924 + index pages 8104
RESET enable_seqscan;
```
**45% slower.** The index read every heap page it would have read anyway, plus 8,104 index pages.

**Step 4 — index bloat, and REINDEX.**

```sql
CREATE TABLE bloat (id bigserial PRIMARY KEY, k bigint, pad text);
INSERT INTO bloat (k, pad) SELECT (random()*1e9)::bigint, repeat('x',80)
FROM generate_series(1,3000000);
CREATE INDEX idx_bloat_k ON bloat(k);
SELECT avg_leaf_density, leaf_fragmentation, pg_size_pretty(index_size::bigint)
FROM pgstatindex('idx_bloat_k');
```
```
 avg_leaf_density | leaf_fragmentation | pg_size_pretty
------------------+--------------------+----------------
            89.94 |               0.00 | 64 MB
```
```sql
-- six months of churn, simulated
DELETE FROM bloat WHERE id % 3 = 0;
UPDATE bloat SET k = (random()*1e9)::bigint WHERE id % 5 = 0;
VACUUM bloat;
SELECT avg_leaf_density, leaf_fragmentation, pg_size_pretty(index_size::bigint)
FROM pgstatindex('idx_bloat_k');
```
```
 avg_leaf_density | leaf_fragmentation | pg_size_pretty
------------------+--------------------+----------------
            41.28 |              52.14 | 71 MB
```
**Deleted a third of the rows; the index got *bigger* and half-empty.**
```sql
REINDEX INDEX CONCURRENTLY idx_bloat_k;
SELECT avg_leaf_density, pg_size_pretty(index_size::bigint) FROM pgstatindex('idx_bloat_k');
--  89.91 | 32 MB       ← half the size, twice the density
```

**Step 5 — find your own waste.**

```sql
SELECT s.relname, s.indexrelname, s.idx_scan,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS size,
       i.indisunique, i.indisprimary
FROM pg_stat_user_indexes s JOIN pg_index i ON i.indexrelid = s.indexrelid
WHERE NOT i.indisprimary
ORDER BY s.idx_scan ASC, pg_relation_size(s.indexrelid) DESC;
```

---

## Example 2 — production scenario

**The situation.** `orders`: 340M rows, 180 GB heap, **16 indexes totalling 71 GB**, on a 128 GB box with `shared_buffers = 32 GB`.

- Checkout p99: 800 ms
- Inserts: 3,100/s (target 9,000/s)
- WAL: 410 GB/hour
- Replicas: lagging 40–90 s
- Buffer hit ratio: 71%

**Step 1 — classify every index.** ⚠ Before anything, check the statistics window:

```sql
SELECT stats_reset FROM pg_stat_database WHERE datname = current_database();
--  2024-11-02       ← 9 months. Trustworthy.
```

⚠ And check the **replicas separately** — their `idx_scan` counters are independent, and a reporting replica may be the only consumer of an index:

```sql
-- run on EACH replica
SELECT indexrelname, idx_scan FROM pg_stat_user_indexes WHERE relname='orders';
```

```sql
-- the classification query
WITH idx AS (
  SELECT s.indexrelid, s.indexrelname, s.idx_scan,
         pg_relation_size(s.indexrelid) AS bytes,
         i.indisunique, i.indisprimary, i.indkey, i.indrelid,
         pg_get_indexdef(s.indexrelid) AS def
  FROM pg_stat_user_indexes s JOIN pg_index i ON i.indexrelid = s.indexrelid
  WHERE s.relname = 'orders'
)
SELECT indexrelname, idx_scan, pg_size_pretty(bytes) AS size,
  CASE
    WHEN indisprimary THEN 'KEEP: primary key'
    WHEN indisunique  THEN 'KEEP: enforces uniqueness (idx_scan irrelevant)'
    WHEN EXISTS (SELECT 1 FROM idx o
                 WHERE o.indexrelid <> idx.indexrelid
                   AND o.def = replace(idx.def, idx.indexrelname, o.indexrelname))
      THEN 'DROP: duplicate definition'
    WHEN EXISTS (SELECT 1 FROM idx o
                 WHERE o.indexrelid <> idx.indexrelid
                   AND NOT idx.indisunique
                   AND o.indkey::text LIKE idx.indkey::text || ' %')
      THEN 'DROP: leftmost prefix of a wider index'
    WHEN idx_scan = 0 THEN 'DROP: never scanned'
    WHEN idx_scan < 1000 AND bytes > 1e9 THEN 'REVIEW: rarely scanned, large'
    ELSE 'KEEP'
  END AS verdict
FROM idx ORDER BY bytes DESC;
```
```
       indexrelname        |  idx_scan  |  size   |                verdict
---------------------------+------------+---------+----------------------------------------
 idx_orders_notes_gin      |          0 | 8912 MB | DROP: never scanned
 idx_orders_created_status |     412008 | 2410 MB | REVIEW
 idx_orders_status_created |   88104229 | 2410 MB | KEEP
 idx_orders_user_created   |   41209841 | 2410 MB | KEEP
 idx_orders_user_id        |       3102 | 1804 MB | DROP: leftmost prefix of a wider index
 idx_orders_merchant_id    |       1004 | 1804 MB | DROP: leftmost prefix of a wider index
 idx_orders_status         |          0 | 1804 MB | DROP: never scanned
 idx_orders_updated_at     |          0 | 1804 MB | DROP: never scanned
 idx_orders_is_gift        |         12 | 1804 MB | DROP: never scanned
 idx_orders_status_2       |          0 | 1804 MB | DROP: duplicate definition
 orders_email_key          |          0 |  980 MB | KEEP: enforces uniqueness ★
 ...
```

**★ Note `orders_email_key`: 0 scans, and you must keep it.** A unique index's job is enforcement, not lookup. `idx_scan = 0` on a unique index means nothing. Dropping it would silently allow duplicates — the exact class of bug case studies 01–04 spend pages preventing.

**Step 2 — the `REVIEW` case.** `idx_orders_created_status` has 412,008 scans over 9 months = ~50/hour. Find out who:

```sql
SELECT calls, mean_exec_time, left(query, 90)
FROM pg_stat_statements
WHERE query ILIKE '%created_at%' AND query ILIKE '%status%'
ORDER BY calls DESC LIMIT 5;
```
It's the nightly finance export — 2 calls/hour, taking 40 s each. **2.4 GB of index and its write cost, for a query that runs twice an hour.** Decision: drop it; let the export do a seq scan on a replica (Topic 06).

**Step 3 — drop safely.**

```sql
-- 1. Record every definition first. This is your rollback.
SELECT indexrelname, pg_get_indexdef(indexrelid)
FROM pg_stat_user_indexes WHERE relname='orders' \g /tmp/orders_indexes_backup.txt

-- 2. Drop without blocking (each is its own statement; CONCURRENTLY
--    cannot run in a transaction block)
DROP INDEX CONCURRENTLY idx_orders_notes_gin;
DROP INDEX CONCURRENTLY idx_orders_status;
DROP INDEX CONCURRENTLY idx_orders_updated_at;
DROP INDEX CONCURRENTLY idx_orders_is_gift;
DROP INDEX CONCURRENTLY idx_orders_user_id;
DROP INDEX CONCURRENTLY idx_orders_merchant_id;
DROP INDEX CONCURRENTLY idx_orders_status_2;
DROP INDEX CONCURRENTLY idx_orders_created_status;
-- ... 4 more

-- 3. Watch for regressions for 48 hours
SELECT calls, mean_exec_time, left(query,80) FROM pg_stat_statements
ORDER BY mean_exec_time DESC LIMIT 20;

-- 4. Rebuild instantly if needed (definitions are in the backup file)
CREATE INDEX CONCURRENTLY ... ;
```

**Step 4 — check for invalid indexes left by any failed builds.**

```sql
SELECT indexrelid::regclass AS idx, indisvalid, indisready
FROM pg_index WHERE NOT indisvalid OR NOT indisready;
```
```
          idx           | indisvalid | indisready
------------------------+------------+------------
 idx_orders_promo_tmp   | f          | t           ← ⚠ maintained on
                                                     every write,
                                                     unusable for reads
```
```sql
DROP INDEX CONCURRENTLY idx_orders_promo_tmp;
```

**Step 5 — REINDEX what remains.**

```sql
SELECT c.relname, i.avg_leaf_density, i.leaf_fragmentation,
       pg_size_pretty(i.index_size::bigint)
FROM pg_class c, LATERAL pgstatindex(c.oid) i
WHERE c.relkind='i' AND c.relname LIKE 'idx_orders%';
```
```
        relname          | avg_leaf_density | leaf_fragmentation | size
-------------------------+------------------+--------------------+---------
 idx_orders_user_created |            48.21 |              44.02 | 2410 MB
 idx_orders_status_created|           61.44 |              31.88 | 2410 MB
```
```sql
REINDEX INDEX CONCURRENTLY idx_orders_user_created;      -- → 1290 MB, 89.9%
REINDEX INDEX CONCURRENTLY idx_orders_status_created;    -- → 1640 MB, 89.8%
```

**Step 6 — results.**

| | Before | After |
|---|---|---|
| Indexes | 16 | 5 |
| Index size | 71 GB | **4.4 GB** (−94%) |
| Inserts/sec | 3,100 | **9,400** (+203%) |
| WAL/hour | 410 GB | **148 GB** (−64%) |
| Replica lag | 40–90 s | **< 1 s** |
| Buffer hit ratio | 71% | **99.2%** |
| Checkout p99 | 800 ms | **190 ms** |
| Queries that got slower | — | **0** |

**Note the replica-lag improvement.** It wasn't the goal, but 64% less WAL means 64% less to ship and replay. Index count is a *replication* cost too — one that never appears in the index's own metrics.

**Step 7 — prevention.**

```yaml
# CI check: fail the build if a migration adds an index without justification
- name: index-policy
  run: |
    # Every CREATE INDEX in a migration must be preceded by a comment
    # containing: the query it serves, the call rate, and an EXPLAIN
    # showing "Rows Removed by Filter: 0" and no Sort node above the scan.
    ./scripts/check-index-justification.sh migrations/
```

```sql
-- Monthly cron: report waste to the team channel
SELECT s.relname, s.indexrelname, s.idx_scan,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS size
FROM pg_stat_user_indexes s JOIN pg_index i ON i.indexrelid=s.indexrelid
WHERE s.idx_scan < 100 AND NOT i.indisunique AND NOT i.indisprimary
  AND pg_relation_size(s.indexrelid) > 100*1024*1024
ORDER BY pg_relation_size(s.indexrelid) DESC;
```

---

## Common mistakes

**1. Adding an index on a frequently-updated column without checking HOT.**
- *Symptom:* WAL volume triples, replica lag appears, autovacuum falls behind — all after one migration.
- *Engine-level why:* the column becomes "indexed," so every update to it is non-HOT and rewrites every index entry.
- *Diagnose:* `SELECT n_tup_upd, n_tup_hot_upd FROM pg_stat_user_tables;` before and after.
- *Fix:* don't index hot columns. If you must, use a **partial index** on the rare value — it usually still breaks HOT, so measure. Or move the hot column to a 1:1 side table (Topic 04).

**2. Dropping a unique index because `idx_scan = 0`.**
- *Symptom:* duplicates appear weeks later, under retry storms.
- *Engine-level why:* a unique index's purpose is *enforcement*, which doesn't increment `idx_scan`.
- *Fix:* **always exclude `indisunique` and `indisprimary` from any drop candidate list.**

**3. Trusting `idx_scan` from the primary only.**
- *Symptom:* a reporting query on a replica goes from 2 s to 40 minutes after a "safe" drop.
- *Engine-level why:* replicas maintain their own statistics counters.
- *Fix:* check every replica, and check `stats_reset` on each.

**4. Running `CREATE INDEX` without `CONCURRENTLY` on a live table.**
- *Symptom:* full write outage for minutes.
- *Fix:* `CONCURRENTLY`, and **verify `indisvalid` afterwards**.

**5. Leaving invalid indexes around.**
- *Symptom:* an index that costs writes and never appears in a plan.
- *Engine-level why:* a failed `CREATE INDEX CONCURRENTLY` leaves the index marked `indisvalid = false` — still maintained on every write, unusable for queries.
- *Diagnose:* `SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;`
- *Fix:* drop and rebuild. **Add this to your monitoring.**

**6. Never reindexing random-key indexes.**
- *Symptom:* an index is 2.5× larger than it should be; scans slow gradually.
- *Engine-level why:* page splits leave leaves ~50% full, and VACUUM never merges them (Topic 11).
- *Fix:* `REINDEX INDEX CONCURRENTLY` when `avg_leaf_density < 70%`. Schedule it for indexes on random keys.

**7. Treating disk as the only cost.**
- *Symptom:* "it's only 2 GB, disk is cheap."
- *Engine-level why:* it also consumes buffer pool (competing with your table), WAL (fsync latency + replica bandwidth + archive storage), and planning time.
- *Fix:* price all five: disk, RAM, WAL, replica lag, planning.

---

## Hands-on proof

**PROVE IT #1 — write amplification vs index count.** (Example 1, step 1.)

**PROVE IT #2 — one index destroys HOT.** (Example 1, step 2. This is the most important experiment in the topic.)

**PROVE IT #3 — an index making a read slower.** (Example 1, step 3.)

**PROVE IT #4 — bloat and REINDEX.** (Example 1, step 4.)

**PROVE IT #5 — find invalid indexes.**
```sql
SELECT indexrelid::regclass AS index, indrelid::regclass AS table,
       indisvalid, indisready
FROM pg_index WHERE NOT indisvalid OR NOT indisready;
```

**PROVE IT #6 — the buffer-pool competition.**
```sql
CREATE EXTENSION IF NOT EXISTS pg_buffercache;
SELECT CASE WHEN c.relkind='i' THEN 'INDEX' ELSE 'TABLE' END AS kind,
       count(*) AS buffers, pg_size_pretty(count(*)*8192::bigint) AS ram
FROM pg_buffercache b JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
GROUP BY 1;
```
```
 kind  | buffers |  ram
-------+---------+--------
 INDEX |  312440 | 2441 MB
 TABLE |   88204 |  689 MB     ← indexes are using 3.5× the cache the table gets
```

**PROVE IT #7 — WAL cost of one index, isolated.**
```sql
CREATE TABLE w (id bigserial PRIMARY KEY, k bigint, pad text);
SELECT pg_current_wal_lsn() AS l \gset
INSERT INTO w (k,pad) SELECT i, repeat('x',80) FROM generate_series(1,300000) i;
SELECT pg_size_pretty(pg_current_wal_lsn() - :'l'::pg_lsn) AS without_index;

CREATE INDEX ON w(k);
SELECT pg_current_wal_lsn() AS l2 \gset
INSERT INTO w (k,pad) SELECT i, repeat('x',80) FROM generate_series(1,300000) i;
SELECT pg_size_pretty(pg_current_wal_lsn() - :'l2'::pg_lsn) AS with_index;
```

---

## The design decision framework

```
BEFORE CREATING AN INDEX — answer all four:
  1. What exact query does it serve? (paste it)
  2. What is that query's call rate? (pg_stat_statements)
  3. What is the selectivity of the predicate? (< 5%?)
  4. Does the query's EXPLAIN change? (verify AFTER creating it)
  ✗ Cannot answer all four → do not create it.

BEFORE CREATING AN INDEX ON AN UPDATED COLUMN — one more:
  5. What is n_tup_hot_upd / n_tup_upd today?
     If it's high and this column is updated often, you are about to
     destroy it. Measure before and after. Consider a side table instead.

DROP AN INDEX WHEN:
  ✓ idx_scan ≈ 0 across the primary AND every replica, over a window
    longer than your longest reporting cycle (check stats_reset)
  ✓ it is a strict leftmost prefix of another index — AND NOT unique
  ✓ it is a duplicate definition
  ✓ the column has < ~20 distinct values and no partial predicate
  ✗ NEVER drop on idx_scan alone if indisunique or indisprimary —
    enforcement doesn't count as a scan

REINDEX WHEN:
  ✓ avg_leaf_density < 70%
  ✓ leaf_fragmentation > 30%
  ✓ index_size ≫ rows × entry_size × 1.15
  → REINDEX INDEX CONCURRENTLY. Schedule it for random-key indexes.

THE COST MODEL — price a proposed index in all five currencies:
  disk:        pg_relation_size after creation
  RAM:         same number, out of shared_buffers
  WAL:         ~80 B per insert + full-page images
  replica:     the same WAL, shipped and replayed on every replica
  HOT:         does this column get updated? if so, ×(N+1) write cost

THE SIGNAL TO LOOK FOR:
      SELECT
        (SELECT sum(pg_relation_size(indexrelid)) FROM pg_stat_user_indexes
          WHERE relname='orders') AS index_bytes,
        pg_relation_size('orders') AS table_bytes,
        round((SELECT sum(pg_relation_size(indexrelid))::numeric
               FROM pg_stat_user_indexes WHERE relname='orders')
              / pg_relation_size('orders'), 2) AS ratio;

  • ratio < 0.5   → healthy
  • ratio 0.5–1.0 → review; there is probably waste
  • ratio > 1.0   → ★ your indexes are bigger than your table.
    Run the audit. You will almost certainly find 50%+ is droppable.

  AND, monthly:
      SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;
  Anything here is pure cost with zero benefit. Drop it.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build the `amp` table from Example 1. Measure insert time and WAL generated for 500,000 rows at 1, 2, 4, and 8 indexes. Plot both against index count. Then explain why WAL grows faster than linearly, naming the specific mechanism.

### Exercise 2 — medium (apply it)
Reproduce the HOT-destruction experiment:
(a) build a table with `fillfactor=85` and two indexes on non-updated columns; run a mass update and record `hot_pct` and WAL,
(b) add an index on the updated column; repeat and record both again,
(c) explain the mechanism in terms of TIDs and index entries,
(d) now set `fillfactor=100` on a copy and repeat step (a). Explain why `hot_pct` collapses even without the extra index,
(e) state the rule you'd give a teammate about indexing columns that are frequently updated.

### Exercise 3 — hard (production simulation)
A `subscriptions` table: 900M rows, 420 GB heap, **22 indexes totalling 380 GB** on a 256 GB box with `shared_buffers = 64 GB`. Symptoms: inserts 1,800/s (target 8,000/s), WAL 620 GB/hour, three replicas lagging 3–8 minutes, buffer hit ratio 62%, `status` updated on ~40% of rows daily.

(a) Write the complete classification query. It must correctly handle: unique/PK indexes, leftmost-prefix redundancy (without false positives on `(b,a)` vs `(a)`), duplicate definitions, invalid indexes, and low-cardinality unusable indexes.
(b) `idx_scan = 0` on four indexes, but `stats_reset` was 6 days ago and one of them serves a monthly report. Give two checks that prevent a bad drop.
(c) `status` is updated on 40% of rows daily and there are three indexes on it. Compute the daily write amplification this causes, showing your reasoning, and propose a schema change that eliminates it.
(d) Estimate the effect of your full plan on: index size, insert throughput, WAL volume, replica lag, and buffer hit ratio. Justify each number.
(e) Give the complete, safe deployment procedure for a live 420 GB table — including ordering, how you detect a regression, how you roll back, and why each `DROP` must be its own statement.
(f) One index in your `KEEP` list has `avg_leaf_density = 38%`. Explain what caused it, what it costs, and the exact remediation with its lock implications.
(g) Write the two automated checks (one CI, one monitoring) that prevent recurrence.

---

## Mental model checkpoint

1. Name the four costs of an index. Which is invisible in the index's own metrics but affects every other query on the instance?
2. What are the two conditions for a HOT update? Explain precisely how adding an index can change a table from 97% HOT to 0%.
3. Why does a `DELETE` of a third of a table make an index *bigger*?
4. Give a case where using an index makes a read slower than a sequential scan. Why does the planner sometimes choose it anyway?
5. Why is `idx_scan = 0` irrelevant for a unique index? What would break if you dropped it?
6. What is an invalid index, how does one appear, and why is it the worst possible state?
7. Your index-to-table size ratio is 1.7. What does that tell you, and what's your first action?

---

## Quick reference card

**The four costs**

| Cost | Measured by |
|---|---|
| Write amplification | WAL bytes per row; `pg_current_wal_lsn()` delta |
| Space (disk **and buffer pool**) | `pg_relation_size`; `pg_buffercache` |
| Bloat | `pgstatindex.avg_leaf_density` |
| Planner | `EXPLAIN` → `Planning Time` |

**Thresholds**

| Metric | Healthy | Act |
|---|---|---|
| index:table size ratio | < 0.5 | > 1.0 → audit |
| `avg_leaf_density` | > 70% | < 70% → REINDEX |
| `leaf_fragmentation` | < 30% | > 30% → REINDEX |
| `n_tup_hot_upd / n_tup_upd` | > 80% on update-heavy tables | < 50% → check indexes + fillfactor |
| Buffer hit ratio | > 99% | < 95% → too many indexes, or too little RAM |

**The audit queries**

```sql
-- unused (⚠ excludes unique/primary — enforcement isn't a scan)
SELECT s.relname, s.indexrelname, s.idx_scan,
       pg_size_pretty(pg_relation_size(s.indexrelid))
FROM pg_stat_user_indexes s JOIN pg_index i ON i.indexrelid=s.indexrelid
WHERE s.idx_scan < 50 AND NOT i.indisunique AND NOT i.indisprimary
ORDER BY pg_relation_size(s.indexrelid) DESC;

-- invalid (pure cost)
SELECT indexrelid::regclass FROM pg_index WHERE NOT indisvalid;

-- bloat
SELECT c.relname, i.avg_leaf_density, i.leaf_fragmentation
FROM pg_class c, LATERAL pgstatindex(c.oid) i
WHERE c.relkind='i' ORDER BY i.avg_leaf_density ASC LIMIT 20;

-- HOT ratio
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct
FROM pg_stat_user_tables ORDER BY n_tup_upd DESC LIMIT 20;

-- when were stats reset?
SELECT stats_reset FROM pg_stat_database WHERE datname = current_database();
```

**The commands**

```sql
CREATE  INDEX CONCURRENTLY ...;   -- never blocks writes; verify indisvalid
DROP    INDEX CONCURRENTLY ...;   -- one statement each, not in a transaction
REINDEX INDEX CONCURRENTLY ...;   -- fixes bloat online
```

**The rule:** every index must map to a named query with a measured call rate. No query, no index.

---

## When would I use this at work?

1. **The "we need bigger hardware" meeting.** Before signing off on a 2× instance, you run the audit. Finding 47 GB of unused indexes typically doubles insert throughput and fixes replica lag for the cost of an afternoon — and it makes the queries you *do* care about faster via the buffer pool.

2. **Reviewing a migration.** Someone adds `CREATE INDEX ON orders (status)` on a table where `status` is updated 12,000 times a second. You can show, with the `n_tup_hot_upd` numbers, that this single line will triple WAL volume and add minutes of replica lag — before it ships.

3. **A gradual write-performance regression with no obvious cause.** `avg_leaf_density` at 41% and an index:table ratio of 1.7 tells the whole story in two queries: accumulated indexes plus accumulated bloat. `REINDEX CONCURRENTLY` plus an audit, no code change.

---

## Connected topics

**Understand before this:** 10 (the trade), 11 (page splits and density), 15 (why low-cardinality indexes go unused), 16 (GIN/GiST write costs).

**This unlocks:**
- **18** — the planner: why more indexes means more chances to choose wrong
- **28** — migrations: `CREATE INDEX CONCURRENTLY` and zero-downtime DDL
- **41** — WAL: the volume this topic measures
- **47** — VACUUM and bloat, in full
- **62** — replication: the WAL cost that shows up as lag
- **65/67** — the operational impact and the investigation methodology
