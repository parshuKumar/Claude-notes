# Topic 48 — When Indexes Hurt
### SQL Mastery Curriculum — Phase 7: Indexes and Query Optimisation

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine you run a warehouse. To find products fast, you keep an index card catalogue: one card per product, sorted by product name, pointing to the shelf location. Finding "hammer" is instant — flip to H, read the shelf number, walk there.

Now the catalogue grows. You add a second catalogue sorted by colour. A third by weight. A fourth by supplier. A fifth by price. Now every time a new product arrives, you don't write one card — you write **five** cards, one for each catalogue, and file each in the right place. Every time a product's price changes, you pull the price card, rewrite it, and re-file it. Every time a product leaves, you hunt down and discard five cards.

The catalogues that made **reading** fast have made **writing** slow. A shipment that used to take an hour now takes five, because most of the work is catalogue maintenance, not shelving.

Then something worse happens. You have a catalogue sorted by "department", but your warehouse only has three departments: Hardware, Garden, Electronics. Two-thirds of everything is Hardware. When someone asks "show me all Hardware products", you *could* use the department catalogue — but it points to 60,000 of your 90,000 products scattered across every aisle. Walking card-by-card, fetching each shelf individually, is far slower than just walking the aisles front to back and grabbing everything Hardware as you pass. The catalogue is **real, correct, and useless** for that question. A smart clerk ignores it and walks the aisles.

That is the whole topic. Indexes are catalogues. They cost something on every write. They rot and bloat over time. And sometimes the correct decision — the one PostgreSQL's planner makes for you — is to ignore the index entirely and read the whole table. Understanding *when* an index stops helping, and *why the planner is right to skip it*, is what separates someone who "adds indexes until it's fast" from someone who engineers a schema that stays fast under write load.

---

## 2. Connection to SQL Internals

An index in PostgreSQL is a **separate on-disk data structure** (its own set of heap-like pages in the buffer pool) that duplicates a subset of a table's data in a searchable order. For a B-tree — the default — the structure is a balanced tree of pages, leaf pages holding `(indexed_value, ctid)` pairs where `ctid` is the physical `(page, offset)` address of the heap tuple. Understanding when indexes hurt requires naming several internal machineries:

1. **Write amplification through the heap + every index.** When you `INSERT` a row, PostgreSQL writes the heap tuple *and* inserts a leaf entry into every index on that table. Each of those is a page modification that must be journalled to the **WAL (Write-Ahead Log)** before commit. Five indexes means one heap write plus five index writes plus WAL records for all six — the physical write cost of one logical row is multiplied by the index count.

2. **MVCC and the update problem.** PostgreSQL's MVCC never updates a tuple in place (except for HOT — see below). An `UPDATE` writes a **new tuple version** at a new `ctid`, and every index must get a new entry pointing to that new `ctid`. The old index entries remain, pointing at the now-dead old tuple, until `VACUUM` removes them. This is why updates on a heavily-indexed table are expensive and why indexes **bloat**.

3. **HOT (Heap-Only Tuple) updates.** PostgreSQL's escape hatch: if an `UPDATE` does *not* change any indexed column, and there is free space on the same heap page, the new version is chained on the same page and **no index entry is written at all**. Every index you add reduces the chance of a HOT update, because it makes more columns "indexed". This is a direct, measurable way indexes hurt write throughput.

4. **Index bloat and the visibility map.** Dead index entries accumulate. B-tree pages that were split during heavy inserts don't merge back when rows are deleted. The index grows larger than the data it describes warrants, meaning more pages to traverse, worse buffer-pool cache hit ratio, and slower everything. `REINDEX` rebuilds the structure compactly.

5. **The cost model and selectivity.** The planner estimates the cost of an **Index Scan** vs a **Seq Scan** using `pg_statistic` (populated by `ANALYZE`): the number of rows a predicate will match (selectivity), the table's size in pages (`relpages`), the `random_page_cost` vs `seq_page_cost` GUCs, and the estimated **correlation** between index order and physical heap order. When an index would match a large fraction of the table, the planner computes that the random I/O of fetching scattered heap pages exceeds the sequential I/O of just reading everything — and it chooses Seq Scan. **This is correct behaviour, not a bug.**

6. **Stale statistics.** The cost model is only as good as `pg_statistic`. If a bulk load added 10M rows and `ANALYZE` hasn't run, the planner may believe the table has 1,000 rows and make a catastrophically wrong choice. "The optimizer ignored my index" is, more often than not, "my statistics are stale."

This topic lives at the intersection of the storage engine (heap, WAL, MVCC, VACUUM) and the planner (cost model, `pg_statistic`, GUCs). Every other topic in Phase 7 assumed indexes help. This one is where you learn the boundaries.

---

## 3. Logical Execution Order Context

Indexes are not part of the logical query clause pipeline (`FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → window → ORDER BY → LIMIT`). They are a **physical access-path decision** the planner makes *underneath* that logical order. But they interact with it at specific points:

```
FROM / JOIN     ← index chosen (or not) for each table's access path and for join probes
WHERE           ← index chosen (or not) to satisfy predicates; low selectivity → Seq Scan
GROUP BY        ← an index on the grouping key may allow a GroupAggregate instead of HashAggregate
ORDER BY        ← an index in the right order may eliminate an explicit Sort
LIMIT           ← LIMIT drastically changes index economics (see below)
```

Two ordering facts matter specifically for "when indexes hurt":

- **`WHERE` selectivity is evaluated by the planner *before* execution, using statistics.** The decision to use or skip an index happens at plan time, from estimates — not from the actual data. This is why stale stats break everything: the planner commits to Seq Scan or Index Scan before it sees a single real row.

- **`LIMIT` is logically last, but the planner pulls its effect forward.** A query with `ORDER BY indexed_col LIMIT 10` can become extremely cheap via an index (walk the index in order, stop after 10). The same query without `LIMIT` may prefer a Seq Scan + Sort. So the presence of `LIMIT` can flip the index decision entirely — the index *hurts* for the full scan and *helps* for the limited one. The planner models this; you must too.

Because the index decision is made from estimates at plan time, everything in this topic — write amplification aside — traces back to whether the planner's *estimate* of how the logical pipeline will behave matches reality.

---

## 4. What Is "When Indexes Hurt"?

"When indexes hurt" is the set of situations where an index is a net negative — it costs more than it saves — or where using an index for a specific query is slower than not using it. There are two distinct failure modes: **write-side cost** (the index slows down `INSERT`/`UPDATE`/`DELETE` and consumes storage, whether or not any query uses it) and **read-side misuse** (a query uses, or a naive developer forces, an index scan that is slower than the sequential scan the planner would otherwise choose).

There is no single syntax for this topic; it is a diagnostic discipline. The relevant surfaces are the DDL that creates cost, and the tools that reveal it:

```sql
CREATE INDEX idx_orders_status ON orders (status);
--     │            │              │       │
--     │            │              │       └── indexed column(s) — every write to this
--     │            │              │           column now maintains this structure
--     │            │              └────────── the table paying the write-amplification cost
--     │            └───────────────────────── index name (its own on-disk relation in pg_class)
--     └────────────────────────────────────── builds a B-tree: instant on empty table,
--                                              locks + full scan on a large one (use CONCURRENTLY)

-- Inspecting the cost side:
SELECT indexrelid::regclass          AS index_name,
       pg_size_pretty(pg_relation_size(indexrelid)) AS index_size,
       idx_scan                       AS times_used   -- from pg_stat_user_indexes
FROM   pg_stat_user_indexes
WHERE  relname = 'orders'
ORDER  BY idx_scan ASC;              -- idx_scan = 0 → an index nobody reads but everybody pays for

-- Forcing the planner's hand to compare (session-local, for diagnosis ONLY):
SET enable_indexscan = off;          -- make planner show its Seq Scan alternative
SET enable_bitmapscan = off;         -- also disable bitmap heap scans
EXPLAIN (ANALYZE, BUFFERS) SELECT ... ;
RESET enable_indexscan;              -- ALWAYS reset — never leave this off in prod

-- Fixing bloat:
REINDEX INDEX CONCURRENTLY idx_orders_status;   -- rebuild compactly, no long lock (PG 12+)

-- Fixing stale stats (the #1 cause of "planner ignored my index"):
ANALYZE orders;                      -- refresh pg_statistic so estimates match reality
```

The key mental shift: an index is not a property of a query, it is a **standing liability on a table** that happens to be an asset for *some* queries. You evaluate it on the whole workload, not on the one `SELECT` you were optimising when you created it.

---

## 5. Why Mastering "When Indexes Hurt" Matters in Production

1. **Write throughput collapses silently.** A table with 12 indexes can have 5–10× the write cost of the same table with 2 indexes. Nobody notices at 100 writes/sec. At 10,000 writes/sec — a payment ingest, an audit_logs firehose, an IoT feed — the WAL volume and index-maintenance CPU become the bottleneck, and the fix (dropping indexes) is scary because "an index might be needed." Engineers who can't reason about write amplification either over-index and throttle writes, or under-index and throttle reads. Both are outages waiting to happen.

2. **Bloat turns a fast index into a slow one over months.** An index that was 200MB and snappy at launch is 1.2GB of mostly-dead entries a year later on a high-churn table. Cache hit ratio drops, latency creeps up, and the team blames "the database getting slow" instead of running `REINDEX`. Knowing to monitor `pg_stat_user_indexes` and index bloat is the difference between a 5-minute fix and a quarter-long "we need to shard" panic.

3. **"The optimizer is ignoring my index" wastes senior time.** This is one of the most common escalations. 90% of the time it is stale statistics, a low-selectivity predicate where Seq Scan is genuinely correct, a type mismatch preventing index use, or a function wrapping the column. An engineer who understands the cost model resolves it in minutes with `ANALYZE` and `EXPLAIN`. One who doesn't files a bug against PostgreSQL, or worse, forces the index with `enable_seqscan = off` in production and makes it slower.

4. **Storage and backup cost.** Indexes are data. On a large system, indexes can exceed the size of the tables themselves. Every unused index inflates disk usage, backup time, restore time, and replication lag. `idx_scan = 0` over a representative window is a signal to drop.

5. **Correct capacity planning.** When you understand that each index multiplies write I/O and WAL, you size instances, tune `checkpoint` behaviour, and design partitioning correctly. Without it, you provision for read load and get surprised by write load.

The theme of Phase 7 so far (Topics 40–47) has been "indexes make reads fast." This topic is the counterweight that makes you *trustworthy* with indexes: you know their price, and you know when the planner is right to leave them on the shelf.

---

## 6. Deep Technical Content

### 6.1 Write Amplification — The Precise Mechanics

Every index on a table is maintained synchronously, inside the same transaction as the write, before commit. Consider a table with a primary key and three secondary indexes:

```sql
CREATE TABLE orders (
  id           bigint PRIMARY KEY,           -- index 1 (PK, unique B-tree)
  customer_id  bigint,
  status       text,
  total_amount numeric,
  created_at   timestamptz
);
CREATE INDEX idx_orders_customer ON orders (customer_id);   -- index 2
CREATE INDEX idx_orders_status   ON orders (status);        -- index 3
CREATE INDEX idx_orders_created  ON orders (created_at);    -- index 4
```

**On INSERT of one row:**
- 1 heap tuple written to a heap page.
- 4 index entries written (PK, customer, status, created) — each may cause a leaf-page read, modify, and possibly a page split.
- WAL records for the heap insert **and** all 4 index inserts. On a busy system this WAL is the real cost — it must be flushed (`fsync`) at commit and shipped to replicas.

The rule of thumb: **write cost scales roughly linearly with the number of indexes.** A table with N indexes costs approximately (1 + N) page-modification units per insert versus 1 for an unindexed table. This is why a bulk-load best practice is *drop the indexes, load, recreate the indexes* — building an index once over sorted data is far cheaper than maintaining it incrementally over random inserts.

**On UPDATE — the MVCC multiplier:**

PostgreSQL does not update in place. An `UPDATE` creates a new tuple version:

```sql
UPDATE orders SET status = 'shipped' WHERE id = 12345;
```

Naively, this must:
- Write a new heap tuple (new `ctid`).
- Insert a new entry into **every index** pointing to the new `ctid` — even indexes on columns that didn't change — because the physical row address changed.
- Mark the old tuple dead (later cleaned by VACUUM), leaving stale entries in every index.

That is potentially 4 new index entries plus 4 future dead entries to vacuum, for changing one column.

**HOT (Heap-Only Tuple) — the crucial optimization:**

PostgreSQL avoids that cost when two conditions hold:
1. **No indexed column changed.** `status` is indexed, so the update above is *not* HOT-eligible. But `UPDATE orders SET total_amount = 99 WHERE id = 12345` — where `total_amount` is *not* indexed — is HOT-eligible.
2. **There is free space on the same heap page** for the new version.

When both hold, the new version is placed on the same page and linked via a HOT chain; **no index entries are written**. The index still points at the old `ctid`, and the HOT chain lets PostgreSQL follow it to the live version.

The production consequence is sharp: **every index you add shrinks the set of HOT-eligible updates.** If you index `total_amount`, the update above stops being HOT and starts writing 4 index entries. This is a direct, often-overlooked way indexes hurt: they don't just cost on writes to their own column — they can poison HOT eligibility for the whole row. A frequently-updated column should almost never be indexed unless a read query truly requires it.

You can observe HOT activity:

```sql
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 1) AS hot_pct
FROM   pg_stat_user_tables
WHERE  relname = 'orders';
-- A low hot_pct on a high-update table = index-induced write amplification
```

### 6.2 Index Bloat — Why Indexes Grow and Rot

Bloat is dead space inside an index (or table) that is no longer needed but has not been reclaimed. For indexes it accumulates from:

- **Dead tuples from UPDATE/DELETE.** Each leaves an index entry pointing at a dead heap tuple. VACUUM removes these, but only lazily, and only marks the space reusable — it does not shrink the index file or merge sparse pages.
- **Page splits that never merge.** B-tree pages split on insert when full. When rows are later deleted, PostgreSQL does *not* merge half-empty sibling pages back together. A table that saw heavy inserts then heavy deletes ends up with many half-empty index pages — the index occupies twice the space its live entries need.
- **Non-sequential insert patterns.** Random-order inserts (e.g., indexing a UUIDv4 primary key) cause splits all over the tree, versus append-only patterns (a `bigserial` or time-ordered key) that fill pages neatly at the right edge. This is a strong argument for UUIDv7 / time-ordered keys over random UUIDs when the column is indexed.

**Detecting bloat.** There is no single built-in number; the community `pgstattuple` extension gives an authoritative answer:

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstatindex('idx_orders_customer');
-- Look at: leaf_fragmentation (%) and avg_leaf_density (%)
-- avg_leaf_density well below ~90% and high leaf_fragmentation → bloated
```

A quick approximation without the extension compares actual size to the theoretical size implied by row count and average key width, but `pgstattuple` is the honest measure.

**Fixing bloat.**

```sql
-- OLD, DANGEROUS: takes an ACCESS EXCLUSIVE lock, blocks all reads+writes
REINDEX INDEX idx_orders_customer;

-- MODERN (PG 12+): rebuilds concurrently, only a brief lock at start/end
REINDEX INDEX CONCURRENTLY idx_orders_customer;

-- Rebuild every index on a table concurrently:
REINDEX TABLE CONCURRENTLY orders;
```

`REINDEX CONCURRENTLY` builds a fresh index alongside the old one, then swaps. It needs disk space for both copies simultaneously and is slower in wall-clock time, but it does not block the workload. Routine autovacuum keeps *dead-tuple* bloat in check; it does **not** compact splits, so periodic `REINDEX` is still needed on high-churn indexes. `VACUUM FULL` also rebuilds indexes but takes a full table lock — avoid on live systems.

### 6.3 The Optimizer Correctly Ignoring an Index

This is the heart of the "read-side" story. The planner ignores an available index in several legitimate situations. In every one, forcing the index would make the query slower.

**(a) Low selectivity — the predicate matches too much of the table.**

If `WHERE status = 'active'` matches 70% of a 1M-row table, using the index means: read 700K index leaf entries, then for each, fetch the heap page it points to. Because those heap pages are scattered, this is 700K (up to) *random* page fetches. A Seq Scan reads all the table's pages sequentially — far fewer total pages, sequential I/O, and no index traversal. The planner computes both costs and correctly picks Seq Scan.

The tipping point is roughly when the predicate matches more than ~5–10% of rows (it depends heavily on `random_page_cost` and correlation), but there is no fixed threshold — it is a cost comparison. A **Bitmap Index Scan** is the planner's middle ground: it collects all matching `ctid`s from the index, sorts them by physical page, then fetches heap pages in physical order (sequential-ish). Bitmap scans extend the useful range of an index to higher selectivities before Seq Scan wins outright.

**(b) Stale statistics — the #1 real-world cause.**

The planner estimates selectivity from `pg_statistic`. If those stats are stale — a bulk load happened, or a column's distribution shifted — the estimate is wrong and the plan is wrong. Classic case: you load 10M rows into a freshly created table and query it; autovacuum's `ANALYZE` hasn't run yet, the planner thinks the table is tiny, and it picks a plan that's disastrous at real scale.

```sql
-- Symptom in EXPLAIN ANALYZE: estimated rows wildly different from actual rows
-- e.g.  (cost=... rows=100 ...) (actual ... rows=850000 ...)
ANALYZE orders;   -- refreshes pg_statistic; re-plan usually fixes it
```

Always suspect stale stats when estimated and actual row counts diverge by an order of magnitude.

**(c) The cost model's assumptions — `random_page_cost` and correlation.**

The default `random_page_cost = 4.0` assumes random I/O is 4× more expensive than sequential — true for spinning disks, pessimistic for SSDs and cloud storage. On SSD-backed systems, `random_page_cost = 4.0` can make the planner *over*-prefer Seq Scans, skipping indexes that would actually be faster. Tuning it to `1.1`–`1.5` on SSDs often "unlocks" index usage. This is a case where the planner ignoring the index is arguably wrong *because it was told the wrong hardware costs* — the fix is configuration, not fighting the planner.

**Correlation** matters too: `pg_stats.correlation` measures how well the index order matches physical heap order. A perfectly correlated index (e.g., a `created_at` index on a table where rows are inserted in time order) means an index range scan fetches nearly-sequential heap pages — cheap. A poorly correlated index means scattered fetches — expensive — and the planner rightly favours Seq Scan sooner.

**(d) The index simply can't be used for the predicate.**

- **Function/expression on the column:** `WHERE lower(email) = 'x'` cannot use a plain index on `email`; it needs an expression index `CREATE INDEX ON users (lower(email))`.
- **Type mismatch / implicit cast:** `WHERE user_id = '42'` (text vs bigint) or a `varchar` column compared to an integer may prevent index use.
- **Leading-column rule:** a composite index on `(a, b)` cannot serve `WHERE b = ?` alone.
- **`LIKE '%foo'`** (leading wildcard) cannot use a standard B-tree; needs a trigram (`pg_trgm`) index.
- **`OR` across columns** often can't use a single index (a bitmap OR of two indexes may, though).

These aren't the planner "ignoring" a usable index — the index is genuinely inapplicable. `EXPLAIN` will show a Seq Scan with a Filter; recognising *why* is the skill.

### 6.4 Too Many Indexes — The Aggregate Cost

Beyond per-index write cost, a surfeit of indexes causes systemic problems:

- **Planner time.** With many indexes and many join orders, planning time itself grows. For simple queries this is negligible; for complex analytic queries it is measurable.
- **Redundant / overlapping indexes.** An index on `(a)` is fully covered by an index on `(a, b)` for any query that only needs `a` as a leading column — the single-column index is redundant and can be dropped, saving its write cost. Detecting these requires auditing index definitions.
- **Buffer pool contention.** Every index competes for shared_buffers cache. Bloated and unused indexes evict useful pages.
- **The "index everything" anti-pattern.** ORMs and well-meaning migrations add an index per foreign key, per frequently-filtered column, "just in case." Auditing `idx_scan` from `pg_stat_user_indexes` after a representative period reveals which are never used.

```sql
-- Find unused indexes (never scanned since stats were last reset)
SELECT s.relname            AS table_name,
       s.indexrelname       AS index_name,
       s.idx_scan           AS scans,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS size
FROM   pg_stat_user_indexes s
JOIN   pg_index i ON i.indexrelid = s.indexrelid
WHERE  s.idx_scan = 0
  AND  NOT i.indisunique          -- keep unique/PK indexes even if unscanned (they enforce constraints)
ORDER  BY pg_relation_size(s.indexrelid) DESC;
```

Two caveats before dropping: unique/PK indexes enforce constraints (never drop for being "unused"); and `idx_scan = 0` only means unused *since the stats were last reset* — a monthly report's index looks unused for 29 days. Check `pg_stat_reset` timing and observe over a full business cycle.

### 6.5 When a Sequential Scan Is Genuinely Faster

A Seq Scan is faster than an Index Scan when the total pages touched sequentially is cheaper than (index pages traversed + heap pages fetched randomly). Concretely, Seq Scan wins when:

- **High selectivity predicate** — matching a large fraction of rows (see 6.3a).
- **Small table** — a table that fits in a few pages is faster to scan wholesale than to traverse an index; the planner will Seq Scan tiny tables even for a unique lookup. This is correct: for a 3-page table, index overhead exceeds the scan.
- **Reading most columns of most rows anyway** — e.g., a full-table aggregate `SELECT count(*) ... GROUP BY status`. There's nothing to filter; you must read everything.
- **Poor correlation + moderate selectivity** — scattered fetches make even a 5% match more expensive than a sequential read.

The counter-intuitive production case: a query with a WHERE clause on an indexed column that you *expected* to use the index does a Seq Scan, and it is **correct and fast**. Verify with `EXPLAIN (ANALYZE, BUFFERS)`: if the Seq Scan's execution time is low and buffer counts are reasonable, the planner made the right call. Forcing an index scan (via `SET enable_seqscan = off`) and finding it *slower* is the definitive proof.

### 6.6 Partial and Covering Indexes as the Antidote

Sometimes the fix for "this index hurts" is a *better-shaped* index, not no index:

- **Partial index** — index only the rows queries actually target, slashing both size and write cost:
  ```sql
  -- Instead of indexing all 100M orders on status (70% are 'completed'),
  -- index only the rare, frequently-queried states:
  CREATE INDEX idx_orders_pending ON orders (created_at)
    WHERE status IN ('pending', 'processing');
  -- Tiny index; writes to 'completed' rows never touch it; queries for
  -- pending work are fast. Best of both sides.
  ```
- **Covering index** (`INCLUDE`) — let an Index-Only Scan (Topic 47) avoid heap fetches entirely, which changes the selectivity math because there's no random heap I/O:
  ```sql
  CREATE INDEX idx_orders_cust_covering ON orders (customer_id)
    INCLUDE (status, total_amount);
  ```

Partial indexes are the single most under-used tool for reducing write amplification while preserving read speed on skewed data.

### 6.7 A Note on `enable_*` Flags — Diagnosis Only

PostgreSQL exposes `enable_seqscan`, `enable_indexscan`, `enable_bitmapscan`, etc. These do not *hard*-disable a scan type; they add a large cost penalty so the planner avoids it if any alternative exists. Their **only** legitimate use is diagnosis: toggle one off, re-run `EXPLAIN ANALYZE`, and compare the alternative's real cost. **Never** ship `enable_seqscan = off` in production config — it globally distorts planning and will cause the planner to pick terrible index scans for queries where Seq Scan was right. If you find yourself wanting to force a plan permanently, the real fix is stats, index shape, or `random_page_cost`.

---

## 7. EXPLAIN — Index vs Seq Scan Decisions in the Plan

### 7.1 The planner correctly choosing Seq Scan for low selectivity

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total_amount
FROM   orders
WHERE  status = 'completed';   -- matches ~70% of 2,000,000 rows
```

```
Seq Scan on orders  (cost=0.00..43620.00 rows=1400000 width=16)
                    (actual time=0.02..180.4 rows=1403912 loops=1)
  Filter: (status = 'completed')
  Rows Removed by Filter: 596088
Buffers: shared hit=512 read=18108
Planning Time: 0.14 ms
Execution Time: 235.7 ms
```

**Reading it:**
- Despite an index on `status`, the planner chose `Seq Scan`. Estimated 1.4M rows match — 70% of the table.
- `Rows Removed by Filter: 596088` — the 30% that didn't match, discarded during the scan.
- Using the index would mean ~1.4M random heap fetches; the sequential read of ~18K pages is far cheaper. **The planner is right.**

Force the index to prove it:

```sql
SET enable_seqscan = off;
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total_amount FROM orders WHERE status = 'completed';
```

```
Bitmap Heap Scan on orders  (cost=15230.0..98450.0 rows=1400000 width=16)
                            (actual time=95.2..640.8 rows=1403912 loops=1)
  Recheck Cond: (status = 'completed')
  Heap Blocks: exact=18342
  ->  Bitmap Index Scan on idx_orders_status
        (cost=0.00..14880.0 rows=1400000 width=0)
        (actual time=88.1..88.1 rows=1403912 loops=1)
        Index Cond: (status = 'completed')
Buffers: shared hit=20 read=20455
Execution Time: 712.3 ms
```

`712ms` forced-index vs `236ms` Seq Scan — **3× slower**. Proof the planner's Seq Scan was correct. `RESET enable_seqscan;` immediately.

### 7.2 The planner correctly choosing Index Scan for high selectivity

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total_amount FROM orders WHERE status = 'refunded';  -- matches ~0.3%
```

```
Bitmap Heap Scan on orders  (cost=68.0..7120.0 rows=6000 width=16)
                            (actual time=1.1..8.4 rows=6012 loops=1)
  Recheck Cond: (status = 'refunded')
  Heap Blocks: exact=5820
  ->  Bitmap Index Scan on idx_orders_status
        (cost=0.00..66.5 rows=6000 width=0)
        (actual time=0.6..0.6 rows=6012 loops=1)
        Index Cond: (status = 'refunded')
Buffers: shared hit=40 read=5810
Execution Time: 9.1 ms
```

Same column, same index — but `'refunded'` matches 0.3%, so the index wins decisively (`9ms`). The *value*, not just the column, determines whether the index helps. This is why the planner re-evaluates per query.

### 7.3 Stale statistics producing a wrong plan

```sql
-- After a bulk load of 10M rows, before ANALYZE ran:
EXPLAIN ANALYZE
SELECT * FROM audit_logs WHERE action = 'login' AND created_at > now() - interval '1 day';
```

```
Index Scan using idx_audit_action on audit_logs
      (cost=0.29..8.31 rows=1 width=120)          ← estimate: 1 row
      (actual time=0.03..4210.5 rows=2841002 loops=1)   ← reality: 2.8M rows
  Index Cond: (action = 'login')
  Filter: (created_at > (now() - '1 day'::interval))
Execution Time: 4380.2 ms
```

**The tell:** `rows=1` estimated vs `rows=2841002` actual — six orders of magnitude off. The planner thought one row matched and chose a Nested-Loop-friendly Index Scan; in reality millions matched and the plan is a disaster. Fix:

```sql
ANALYZE audit_logs;
-- Re-plan now estimates ~2.8M rows and switches to Seq Scan or a bitmap scan → seconds, not minutes.
```

### 7.4 Signals table

| EXPLAIN signal | Meaning | Action |
|----------------|---------|--------|
| Seq Scan chosen despite index, high `Rows Removed by Filter` but low % | Possibly wrong; check selectivity | Compare with `enable_seqscan=off` |
| Seq Scan chosen, predicate matches >10% of rows | Correct — index would be slower | Leave it; maybe partial index |
| `rows=` estimate vs `actual rows=` differ by >10× | Stale statistics | `ANALYZE` the table |
| `Bitmap Heap Scan` with huge `Heap Blocks` | Selectivity marginal; near the tipping point | Consider partial/covering index |
| Index Scan with `Rows Removed by Filter` high | Index used for one column, filtering the rest | Composite or covering index |
| `Filter:` on the "indexed" column itself | Function/type mismatch blocks index | Expression index or fix the cast |

---

## 8. Query Examples

### Example 1 — Basic: Proving a Seq Scan Is Correct

```sql
-- A boolean-ish column with two values: is_active true/false, ~50/50 split.
-- An index on is_active is almost useless — neither value is selective.
CREATE INDEX idx_users_active ON users (is_active);   -- likely a mistake

EXPLAIN (ANALYZE, BUFFERS)
SELECT id, email FROM users WHERE is_active = true;   -- matches ~50%
-- Planner chooses Seq Scan; the index will show idx_scan = 0 forever.
-- Lesson: don't index low-cardinality columns unless combined (partial/composite).
```

### Example 2 — Intermediate: Diagnosing "The Optimizer Ignores My Index"

```sql
-- Developer complaint: "I have an index on email but this is slow / Seq Scans."
CREATE INDEX idx_users_email ON users (email);

-- The failing query:
EXPLAIN ANALYZE
SELECT id FROM users WHERE lower(email) = 'alice@example.com';
--                          ^^^^^ function on the column defeats the plain index
-- Plan: Seq Scan on users, Filter: (lower(email) = 'alice@example.com')

-- The fix is an expression index matching the predicate shape:
CREATE INDEX idx_users_lower_email ON users (lower(email));
ANALYZE users;
-- Now: Index Scan using idx_users_lower_email. The original index wasn't
-- "ignored" — it was inapplicable. The predicate must match the index expression.
```

### Example 3 — Production Grade: Auditing Write Amplification and Bloat on a Hot Table

```sql
-- Context:
--   Table:  order_items, 80,000,000 rows, ~4,000 INSERTs/sec at peak, frequent
--           UPDATEs to `fulfilled_qty` (a running counter).
--   Indexes: PK(id), (order_id), (product_id), (created_at), (fulfilled_qty)
--   Symptom: write latency climbed from 2ms to 22ms over 6 months; replica lag rising.
--   Suspicion: too many indexes + fulfilled_qty index killing HOT updates.

-- Step 1: measure HOT update ratio — low ratio confirms index-induced amplification.
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/NULLIF(n_tup_upd,0),1) AS hot_pct
FROM   pg_stat_user_tables WHERE relname = 'order_items';
--  relname     | n_tup_upd | n_tup_hot_upd | hot_pct
-- order_items  | 410288122 |     12308643  |    3.0     ← only 3% HOT — bad

-- Step 2: which indexes are actually read? (justify their write cost)
SELECT indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM   pg_stat_user_indexes WHERE relname = 'order_items'
ORDER  BY idx_scan;
--  indexrelname                | idx_scan | size
--  order_items_fulfilled_idx   |        0 | 3200 MB   ← never read, huge write cost
--  order_items_created_idx     |     8123 | 3400 MB
--  order_items_product_idx     | 51203348 | 3600 MB
--  order_items_order_idx       | 88134002 | 3600 MB
--  order_items_pkey            | 90551210 | 1800 MB

-- Step 3: check bloat on the churned indexes.
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT avg_leaf_density, leaf_fragmentation FROM pgstatindex('order_items_order_idx');
--  avg_leaf_density | leaf_fragmentation
--        58.2       |       31.4            ← density < 90%, fragmented → bloated

-- Step 4: remediation.
--   (a) fulfilled_qty is indexed but never scanned AND it's the hot-updated column.
--       Dropping it recovers HOT eligibility AND removes 3.2GB + its write cost.
DROP INDEX CONCURRENTLY order_items_fulfilled_idx;

--   (b) rebuild the bloated but useful index without blocking writes.
REINDEX INDEX CONCURRENTLY order_items_order_idx;

-- Step 5: re-measure — expect hot_pct to jump (fewer indexed cols on the row),
--         write latency to drop, and replica lag to recover.

-- EXPLAIN of a representative write path is not meaningful (INSERT has no scan),
-- but the read query that justified order_idx should stay fast:
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, product_id, quantity FROM order_items WHERE order_id = 84213771;
```
```
Index Scan using order_items_order_idx on order_items
      (cost=0.57..44.20 rows=6 width=20) (actual time=0.04..0.06 rows=5 loops=1)
  Index Cond: (order_id = 84213771)
Buffers: shared hit=5
Execution Time: 0.08 ms
```
The read path stays sub-millisecond after remediation, while dropping the unused `fulfilled_qty` index restores HOT updates and slashes write amplification. This is the whole topic in one operation: **the index you removed was pure cost; the index you kept and rebuilt was a justified asset.**

---

## 9. Wrong → Right Patterns

### Wrong 1: Forcing the index because "index scan is always faster"

```sql
-- WRONG: developer sees a Seq Scan, assumes it's a bug, disables seq scans
SET enable_seqscan = off;   -- shipped this in a session / connection init hook
SELECT id, total FROM orders WHERE status = 'completed';  -- matches 70%
-- RESULT: forced Bitmap/Index scan does 1.4M random heap fetches.
-- Measured 712ms vs the 236ms Seq Scan the planner wanted. 3x SLOWER.
-- Worse: enable_seqscan=off now poisons EVERY query on this connection —
-- unrelated queries get terrible forced index plans.
```

**Why it's wrong at the execution level:** for a 70%-selectivity predicate, the index yields ~1.4M `ctid`s whose heap pages are scattered across the table. Random-fetching them costs more page reads (with random_page_cost penalty) than sequentially streaming all ~18K table pages once. The planner's cost model captured this correctly; the human overrode it with a global hammer.

```sql
-- RIGHT: trust the planner; if you doubt it, PROVE it with EXPLAIN, then RESET.
EXPLAIN (ANALYZE, BUFFERS) SELECT id, total FROM orders WHERE status = 'completed';
-- Confirm Seq Scan's actual time is low. If a rare-status query needs speed,
-- build a PARTIAL index for the rare values instead of forcing plans globally:
CREATE INDEX idx_orders_rare ON orders (created_at)
  WHERE status IN ('refunded', 'disputed');
```

### Wrong 2: Indexing a hot-updated counter column

```sql
-- WRONG: a view_count column updated on every page view, then indexed
--        "so we can sort by popularity fast"
CREATE INDEX idx_products_views ON products (view_count);
-- Every UPDATE products SET view_count = view_count + 1 now:
--   - is NO LONGER HOT-eligible (view_count is indexed)
--   - writes a new index entry every single view
--   - leaves a dead index entry for VACUUM every single view
-- On a popular catalogue this can 5x the write cost and bloat the index hourly.
```

**Why it's wrong at the execution level:** indexing `view_count` makes every increment change an indexed column, which disqualifies the HOT optimization. Instead of an in-page tuple update with zero index writes, each increment writes a heap tuple + an index entry + a future dead entry. The index that was meant to speed up an occasional "top products" query throttles the constant write path.

```sql
-- RIGHT: don't index the volatile column. Serve the ranking query differently:
--  (a) a periodically-refreshed materialized view / summary table, OR
--  (b) accept a Seq Scan + Sort for the infrequent leaderboard query, OR
--  (c) keep counts in a separate low-index table, or Redis, and sync in batches.
DROP INDEX idx_products_views;
CREATE MATERIALIZED VIEW top_products AS
  SELECT id, name, view_count FROM products ORDER BY view_count DESC LIMIT 100;
-- REFRESH MATERIALIZED VIEW CONCURRENTLY top_products;  -- on a schedule
```

### Wrong 3: Blaming the planner for stale-stats mis-estimates

```sql
-- WRONG: after a nightly 10M-row bulk import, the report query is 100x slower.
--        "PostgreSQL's optimizer is broken, it picks the wrong plan."
SELECT * FROM audit_logs WHERE action = 'login' AND created_at > now() - interval '1 day';
-- EXPLAIN shows rows=1 estimated, rows=2,800,000 actual → Nested Loop disaster.
```

**Why it's wrong at the execution level:** the planner isn't broken; it planned from `pg_statistic` that predates the import. It estimated 1 matching row and chose a plan optimal for 1 row (index-driven nested loop), which is catastrophic for millions.

```sql
-- RIGHT: refresh statistics as the LAST step of any bulk load.
ANALYZE audit_logs;
-- Better: make it part of the ETL. Also consider raising the stats target on
-- skewed columns so estimates are sharper:
ALTER TABLE audit_logs ALTER COLUMN action SET STATISTICS 1000;
ANALYZE audit_logs;
```

### Wrong 4: Keeping redundant overlapping indexes

```sql
-- WRONG: migrations accreted three overlapping indexes
CREATE INDEX idx_a  ON orders (customer_id);
CREATE INDEX idx_ab ON orders (customer_id, created_at);
CREATE INDEX idx_ac ON orders (customer_id, status);
-- idx_a is fully redundant: any query using a leading customer_id can use
-- idx_ab or idx_ac. idx_a adds write cost + storage for zero unique benefit.
```

**Why it's wrong at the execution level:** a composite index on `(customer_id, created_at)` can satisfy every predicate that a single `(customer_id)` index can — the leading column is usable alone. So `idx_a` is pure write amplification: every insert maintains it, but no query needs it that another index doesn't already serve.

```sql
-- RIGHT: drop the prefix-redundant single-column index.
DROP INDEX idx_a;
-- Verify first that no query truly needs idx_a's slightly smaller size for a
-- covering/index-only scan; usually it doesn't. Audit with pg_stat_user_indexes.
```

---

## 10. Performance Profile

### Write-side cost by index count (per INSERT, approximate)

| Indexes on table | Relative write cost | WAL volume | Notes |
|------------------|--------------------|-----------|-------|
| 0 (heap only) | 1× | 1× | Baseline; append is cheapest |
| 1 (PK only) | ~1.5× | ~1.5× | Unavoidable minimum for a keyed table |
| 4 | ~3–4× | ~3–4× | Common; watch HOT ratio |
| 8+ | ~6–10× | ~6–10× | Write throughput likely the bottleneck |

### Scaling behaviour at size

| Rows | Index concern | What dominates |
|------|--------------|----------------|
| 1M | Bloat negligible; write cost linear in index count | Read plans; get selectivity right |
| 10M | Bloat starts mattering on high-churn indexes; REINDEX quarterly | HOT ratio, ANALYZE freshness |
| 100M | Every index is GBs; write amplification is a capacity constraint; partial indexes essential | WAL volume, VACUUM/autovacuum tuning, replica lag |

### The selectivity tipping point (single-column B-tree, default GUCs)

| Fraction of table matched | Typical planner choice | Why |
|---------------------------|------------------------|-----|
| < 1% | Index Scan | Few, targeted heap fetches |
| 1–5% | Bitmap Index Scan | Sort ctids, fetch pages in physical order |
| 5–15% | Bitmap or Seq (correlation-dependent) | The gray zone; correlation & random_page_cost decide |
| > 15–20% | Seq Scan | Random fetches exceed one sequential pass |

These thresholds shift with `random_page_cost` (lower on SSD → index used at higher selectivity), `pg_stats.correlation` (high correlation → index viable at higher selectivity), and whether an **index-only scan** is possible (no heap fetch → index viable at much higher selectivity).

### Optimization techniques specific to this topic

- **Partial indexes** to cut both size and write cost on skewed columns (index only rare, queried values).
- **Covering indexes (`INCLUDE`)** to enable index-only scans, which change the selectivity math by removing heap I/O.
- **Drop-load-recreate** for bulk imports: building an index once over the final data beats maintaining it per-row.
- **Tune `random_page_cost`** to your storage (≈1.1 on SSD/cloud) so the planner's index/seq decisions match your hardware.
- **Raise `default_statistics_target`** (or per-column `SET STATISTICS`) on skewed columns so selectivity estimates are accurate — the cheapest fix for bad plans.
- **`REINDEX CONCURRENTLY`** on a schedule for high-churn indexes to reclaim split-induced bloat autovacuum can't.
- **Audit `pg_stat_user_indexes.idx_scan`** over a full business cycle and drop the genuinely unused (non-constraint) indexes.

---

## 11. Node.js Integration

### 11.1 Auditing index health from the app / a maintenance script

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Report unused, non-constraint indexes larger than a threshold — candidates to drop.
async function findUnusedIndexes(minBytes = 50 * 1024 * 1024) {
  const { rows } = await pool.query(
    `SELECT s.relname            AS table_name,
            s.indexrelname       AS index_name,
            s.idx_scan           AS scans,
            pg_relation_size(s.indexrelid) AS size_bytes,
            pg_size_pretty(pg_relation_size(s.indexrelid)) AS size
     FROM   pg_stat_user_indexes s
     JOIN   pg_index i ON i.indexrelid = s.indexrelid
     WHERE  s.idx_scan = 0
       AND  NOT i.indisunique
       AND  pg_relation_size(s.indexrelid) >= $1
     ORDER  BY size_bytes DESC`,
    [minBytes]
  );
  return rows;
}
```

### 11.2 Monitoring HOT-update ratio (write-amplification early warning)

```javascript
// Surface tables where indexing is killing HOT updates. Alert when hot_pct is low
// AND update volume is high — that combination is the write-amplification signature.
async function hotUpdateReport() {
  const { rows } = await pool.query(
    `SELECT relname,
            n_tup_upd,
            n_tup_hot_upd,
            round(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd, 0), 1) AS hot_pct
     FROM   pg_stat_user_tables
     WHERE  n_tup_upd > 100000
     ORDER  BY hot_pct ASC NULLS LAST
     LIMIT  20`
  );
  return rows.filter(r => r.hot_pct !== null && Number(r.hot_pct) < 50);
}
```

### 11.3 Diagnosing a suspect query at runtime with EXPLAIN

```javascript
// Programmatically capture the plan to detect estimate/actual skew (stale stats)
// or an unexpected Seq Scan. Never run this on hot paths — it executes the query.
async function explainAndCheck(sql, params = []) {
  const { rows } = await pool.query(
    `EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON) ${sql}`,
    params
  );
  const plan = rows[0]['QUERY PLAN'][0].Plan;
  const estimate = plan['Plan Rows'];
  const actual = plan['Actual Rows'];
  const skew = actual > 0 ? estimate / actual : 1;
  // A skew far from 1 (e.g. < 0.1 or > 10) strongly suggests stale statistics.
  if (skew < 0.1 || skew > 10) {
    console.warn(`Estimate/actual skew ${skew.toFixed(3)} on ${plan['Node Type']} — consider ANALYZE`);
  }
  return { nodeType: plan['Node Type'], estimate, actual, skew };
}
```

### 11.4 Running maintenance safely (never inside a transaction)

```javascript
// REINDEX CONCURRENTLY and CREATE/DROP INDEX CONCURRENTLY cannot run inside a
// transaction block. node-postgres runs each query in its own implicit txn unless
// you BEGIN — so a bare pool.query is fine, but never wrap these in BEGIN/COMMIT.
async function reindexConcurrently(indexName) {
  // Allowlist the identifier; you cannot parameterize DDL object names with $1.
  if (!/^[a-z_][a-z0-9_]*$/i.test(indexName)) {
    throw new Error(`Unsafe index name: ${indexName}`);
  }
  const client = await pool.connect();
  try {
    // No BEGIN — must run outside a transaction block.
    await client.query(`REINDEX INDEX CONCURRENTLY ${indexName}`);
  } finally {
    client.release();
  }
}
```

**On ORMs:** no ORM decides *whether* the planner uses an index — that is PostgreSQL's job, and no ORM can override the cost model. What ORMs control is index *creation* (in migrations) and whether a query's shape (a function on a column, a type mismatch) accidentally defeats an index. The write-amplification and bloat concerns are entirely schema-level and invisible to the ORM's query API; you manage them with migrations and maintenance scripts, not ORM calls.

---

## 12. ORM Comparison

The framing for this topic is different from a query-shape topic. "When indexes hurt" is about **which indexes exist** (write amplification, bloat, unused indexes) and **whether the planner uses one** (selectivity, stale stats). No ORM changes the planner's decision — that is PostgreSQL's cost model, full stop. So the ORM axis here is narrower and specific: (1) how the ORM's migration DSL lets you *create* the indexes that become liabilities, (2) whether it can express the good-shaped antidotes (partial, covering, expression indexes), (3) whether it silently auto-creates indexes you didn't ask for, and (4) whether it can accidentally defeat an index by wrapping a column in a function or forcing a cast. Bloat, `REINDEX`, `ANALYZE`, and `idx_scan` auditing are outside every ORM's query API — they are raw SQL / maintenance-script territory in all of them.

---

### Prisma

**Can Prisma express the indexes that matter?** — Partially. Prisma's schema supports plain and composite indexes and `@@index`/`@@unique`, but **partial indexes, covering (`INCLUDE`) indexes, and expression indexes are not expressible in the Prisma schema** — you add them with raw SQL in a migration.

```prisma
model Order {
  id          BigInt   @id @default(autoincrement())
  customerId  BigInt
  status      String
  totalAmount Decimal
  createdAt   DateTime @default(now())

  @@index([customerId])            // fine — plain B-tree
  @@index([status])                // DANGER: low-cardinality; likely idx_scan=0
}
```

```sql
-- The good-shaped antidotes require a raw migration (prisma migrate --create-only, then edit):
CREATE INDEX idx_orders_pending ON orders (created_at)
  WHERE status IN ('pending', 'processing');          -- partial: not expressible in schema
CREATE INDEX idx_orders_cust_cov ON orders (customer_id)
  INCLUDE (status, total_amount);                     -- covering: not expressible in schema
CREATE INDEX idx_users_lower_email ON users (lower(email));  -- expression: not expressible
```

**Where it breaks**: Prisma encourages an `@@index` per frequently-filtered field, and developers add `@@index([status])`-style low-cardinality indexes that the planner will never use but every write must maintain. Prisma cannot express the partial index that would fix it. There is no Prisma API for `ANALYZE`, `REINDEX`, or reading `pg_stat_user_indexes` — use `$executeRawUnsafe` / `$queryRaw`.

**Verdict**: Fine for declaring plain indexes. For every write-amplification antidote (partial/covering/expression) and all maintenance, drop to raw SQL migrations and scripts. Audit the indexes Prisma generated — do not assume each `@@index` earns its keep.

---

### Drizzle ORM

**Can Drizzle express the indexes that matter?** — Best of the group. Drizzle's schema builder supports partial indexes (`.where()`), expression indexes, and index storage parameters directly in TypeScript.

```typescript
import { pgTable, bigint, text, numeric, timestamp, index } from 'drizzle-orm/pg-core';
import { sql } from 'drizzle-orm';

export const orders = pgTable('orders', {
  id: bigint('id', { mode: 'bigint' }).primaryKey(),
  customerId: bigint('customer_id', { mode: 'bigint' }),
  status: text('status'),
  totalAmount: numeric('total_amount'),
  createdAt: timestamp('created_at').defaultNow(),
}, (t) => ({
  // Plain index on the join column — justified
  customerIdx: index('idx_orders_customer').on(t.customerId),
  // PARTIAL index — the write-amplification antidote, expressible natively:
  pendingIdx: index('idx_orders_pending')
    .on(t.createdAt)
    .where(sql`${t.status} IN ('pending', 'processing')`),
}));
```

**Where it breaks**: Covering `INCLUDE` columns and `REINDEX`/`ANALYZE` still fall outside the builder — run those via `db.execute(sql\`...\`)`. Drizzle will faithfully create whatever you declare, including a low-cardinality index that becomes pure write cost, so the discipline is still yours.

**Verdict**: The strongest schema-side story for this topic — partial and expression indexes are first-class, so the main antidote to write amplification is native. Maintenance (`REINDEX CONCURRENTLY`, `ANALYZE`, index audits) is raw SQL via `db.execute`.

---

### Sequelize

**Can Sequelize express the indexes that matter?** — Plain and composite indexes yes; partial indexes via a `where` option on the index; covering/expression indexes effectively no.

```javascript
const Order = sequelize.define('Order', {
  id: { type: DataTypes.BIGINT, primaryKey: true },
  customerId: DataTypes.BIGINT,
  status: DataTypes.STRING,
  totalAmount: DataTypes.DECIMAL,
}, {
  indexes: [
    { fields: ['customerId'] },                       // plain
    { fields: ['status'] },                            // DANGER: low-cardinality
    { fields: ['createdAt'], where: { status: ['pending', 'processing'] } }, // partial — supported
  ],
});
```

**Where it breaks**: Sequelize's `sync({ alter: true })` is notorious for creating and dropping indexes it thinks the model implies, and for adding an index on every association foreign key — a direct route to the "too many indexes" anti-pattern. `INCLUDE` and expression indexes need raw migrations (`queryInterface.sequelize.query`). No API for HOT-ratio or bloat.

**Verdict**: Usable for plain and partial indexes; never let `alter: true` manage indexes on a production table — it silently churns the exact objects this topic warns about. Maintenance and covering indexes are raw SQL.

---

### TypeORM

**Can TypeORM express the indexes that matter?** — Plain, composite, unique, and partial (`where`) indexes via the `@Index` decorator; expression indexes only through a raw string; covering `INCLUDE` not supported.

```typescript
import { Entity, PrimaryColumn, Column, Index } from 'typeorm';

@Entity('orders')
@Index('idx_orders_customer', ['customerId'])
@Index('idx_orders_pending', ['createdAt'], { where: `status IN ('pending', 'processing')` }) // partial
export class Order {
  @PrimaryColumn('bigint') id: string;
  @Column('bigint') customerId: string;
  @Column('text') status: string;
  @Column('numeric') totalAmount: string;
}
```

**Where it breaks**: `synchronize: true` is the same footgun as Sequelize's `alter` — it will create every decorated index and drop anything not decorated, with no regard for write cost; keep it off in production and use migrations. Covering indexes and all maintenance (`REINDEX`, `ANALYZE`, `pg_stat_user_indexes` audits) are raw SQL via `dataSource.query()`.

**Verdict**: Decorators cover plain and partial indexes cleanly. Disable `synchronize` in production, hand-write covering/expression indexes and all maintenance. TypeORM will not stop you from over-indexing.

---

### Knex.js

**Can Knex express the indexes that matter?** — As a query/migration builder it exposes `table.index()` for plain/composite, and everything else through `knex.raw` in a migration — which is honestly the right level for this topic.

```javascript
// Migration
exports.up = async (knex) => {
  await knex.schema.alterTable('orders', (t) => {
    t.index(['customer_id'], 'idx_orders_customer');   // plain
  });
  // Partial / covering / expression — raw, fully under your control:
  await knex.raw(`
    CREATE INDEX idx_orders_pending ON orders (created_at)
      WHERE status IN ('pending', 'processing')
  `);
  await knex.raw(`
    CREATE INDEX idx_orders_cust_cov ON orders (customer_id)
      INCLUDE (status, total_amount)
  `);
};

// Maintenance script — CONCURRENTLY must run outside a transaction:
await knex.raw('REINDEX INDEX CONCURRENTLY idx_orders_customer');
await knex.raw('ANALYZE orders');
```

**Where it breaks**: Nothing is auto-created, which is the point — Knex never invents an index behind your back, so the "too many indexes" problem is entirely a function of what you write. `CREATE INDEX CONCURRENTLY` / `REINDEX CONCURRENTLY` must not be wrapped in Knex's transaction-wrapped migrations (`config: { transaction: false }` on that migration, or run in a plain script).

**Verdict**: The most honest tool for this topic — no hidden indexes, raw SQL for the antidotes and all maintenance. You get exactly the indexes you write and pay for exactly those. That transparency is what you want when reasoning about write cost.

---

### ORM Summary Table

| ORM | Partial index | Covering (`INCLUDE`) | Expression index | Auto-creates indexes? | Maintenance (REINDEX/ANALYZE/audit) |
|-----|---------------|----------------------|------------------|-----------------------|-------------------------------------|
| Prisma | Raw SQL only | Raw SQL only | Raw SQL only | Only what you declare | Raw SQL (`$executeRaw`) |
| Drizzle | Native `.where()` | Raw SQL | Native | Only what you declare | Raw SQL (`db.execute`) |
| Sequelize | `where` option | Raw SQL | Raw SQL | Yes — `sync/alter` + FK indexes (danger) | Raw SQL |
| TypeORM | `@Index({where})` | Raw SQL | Raw string | Yes — `synchronize` (danger) | Raw SQL (`dataSource.query`) |
| Knex | Raw SQL | Raw SQL | Raw SQL | No | Raw SQL (`knex.raw`) |

The through-line: **no ORM helps you decide whether an index is worth its write cost, and none exposes bloat or HOT-ratio.** Drizzle and Knex give you the cleanest control over the *antidote* (partial indexes). Sequelize and TypeORM's auto-sync features actively work against you here — they are the tools most likely to create the unused, low-cardinality, write-amplifying indexes this whole topic is about. Whatever the ORM, the audit (`pg_stat_user_indexes`, `pg_stat_user_tables.n_tup_hot_upd`, `pgstattuple`) and the fixes (`REINDEX`, `ANALYZE`, `DROP INDEX CONCURRENTLY`, `random_page_cost`) live in raw SQL.

---


## 13. Practice Exercises

### Exercise 1 — Basic

Given `orders(id, customer_id, status, total_amount, created_at)` — a high-write table taking ~2,000 INSERTs/sec — someone has already created these indexes:

```sql
CREATE INDEX idx_orders_status      ON orders (status);
CREATE INDEX idx_orders_customer    ON orders (customer_id);
CREATE INDEX idx_orders_created_at  ON orders (created_at);
CREATE INDEX idx_orders_total       ON orders (total_amount);
```

`status` has exactly 4 distinct values (`pending`, `paid`, `shipped`, `cancelled`), and 92% of live rows are `paid`. Every INSERT must now maintain all four indexes.

1. Which of these four indexes is the weakest candidate — the one most likely costing write throughput while returning little on reads? Explain in one sentence why low cardinality makes it a poor B-tree index.
2. Write the query that lists each index on `orders` together with how many times it has been used for reads, so you can confirm your suspicion with real data.

```sql
-- Write your query here
```

---

### Exercise 2 — Intermediate

The `sessions(id, user_id, token, created_at, expires_at, revoked)` table holds 40M rows. It is written constantly (every login inserts, every logout updates `revoked`) and rows older than 30 days are bulk-deleted nightly. Reads only ever look for **active** sessions: `WHERE revoked = FALSE AND expires_at > NOW()`. Active sessions are about 3% of the table.

There is currently a plain index `CREATE INDEX idx_sessions_expires ON sessions (expires_at)` that is enormous and, after each nightly delete, heavily bloated.

1. Explain why the nightly bulk-delete leaves this index bloated and why the bloat is not automatically reclaimed for reuse in a way that shrinks the on-disk file.
2. Write the single `CREATE INDEX` statement that replaces this with a **partial index** covering only the rows reads actually touch, so the index is ~3% of the size and its write/bloat cost collapses accordingly.

```sql
-- Write your query here
```

---

### Exercise 3 — Hard (Production Simulation)

Production incident: `audit_logs(id, actor_id, action, entity_type, entity_id, created_at, payload jsonb)` grew to 800M rows. A dashboard query runs:

```sql
SELECT date_trunc('day', created_at) AS day, COUNT(*)
FROM audit_logs
WHERE created_at >= NOW() - INTERVAL '7 days'
GROUP BY 1;
```

There is an index `idx_audit_created_at ON audit_logs (created_at)`. Last week this query used the index and ran in 40ms. Today it does a **Seq Scan** and takes 90 seconds, even though 7 days is still a tiny fraction of the table. Nothing about the schema changed. `pg_stat_user_tables` shows `n_mod_since_analyze` is in the tens of millions and `last_analyze` is 9 days ago.

1. Give the most likely root cause for the planner abandoning the index, in terms of the statistics it relies on.
2. Write the two-statement fix: the command that refreshes what the planner knows, followed by the query that confirms the planner's row estimate for the 7-day predicate is now close to reality (use `EXPLAIN`).
3. Separately, name the config knob you would inspect if — after fixing stats — the planner *still* preferred a Seq Scan on SSD-backed storage, and state the direction you would move it.

```sql
-- Write your query here
```

---

### Exercise 4 — Interview Simulation

You are handed `products(id, name, category_id, price, sku, description, active)` in an interview. The candidate before you added, over two years, every index anyone ever asked for:

```sql
CREATE INDEX idx_products_category        ON products (category_id);
CREATE INDEX idx_products_category_price  ON products (category_id, price);
CREATE INDEX idx_products_price           ON products (price);
CREATE INDEX idx_products_active          ON products (active);
CREATE INDEX idx_products_cat_active_price ON products (category_id, active, price);
CREATE UNIQUE INDEX idx_products_sku       ON products (sku);
```

The table takes heavy bulk price updates (a supplier feed rewrites `price` on ~200K rows every hour).

1. Identify which indexes are **redundant** because another index already serves any query they could serve as a leftmost prefix. Name the redundant ones and the index that subsumes each.
2. Explain specifically why the hourly price rewrite is expensive given this index set, and which indexes make it worse.
3. Write the audit query that would let you prove, from live usage stats, that `idx_products_active` is safe to drop.

```sql
-- Write your query here
```

---

## 14. Interview Questions

### Q1 — "You added an index and writes got slower. Why?"

**Junior answer**: Indexes take up disk space, so writing more data is slower.

**Principal answer**: Every non-HOT INSERT, UPDATE, and DELETE must maintain *every* index on the table, not just the heap. An INSERT adds an entry to N index B-trees; an UPDATE that changes an indexed column (or that can't do a HOT update because the new tuple won't fit on the same page) writes a new index entry in *all* indexes and marks the old ones dead. That is write amplification: one logical write becomes 1 + N physical index writes plus WAL for each. The index also defeats HOT updates — PostgreSQL can only do a Heap-Only Tuple update (which skips index maintenance entirely) when *no* indexed column changed and there is room on the page; adding an index over a frequently-updated column silently disables that optimization. So the cost isn't disk space, it's the per-row maintenance multiplied across every write and every index, plus lost HOT updates, plus more WAL, plus more vacuum work later.

**Follow-up**: How would you measure whether a specific index is worth its write cost? — Compare its read usage (`pg_stat_user_indexes.idx_scan`) against the table's write volume, and check `pg_stat_user_tables.n_tup_hot_upd` vs `n_tup_upd`: a low HOT ratio on a hot-updated table points at an index over the updated column.

---

### Q2 — "The optimizer is ignoring my index. Force it to use the index."

**Junior answer**: Add an index hint, or set `enable_seqscan = off` so it's forced to use the index.

**Principal answer**: First assume the planner is *right* — it chose a Seq Scan because its cost model estimated that as cheaper, and it's usually correct. Three real causes to check before overriding anything: (1) **Stale statistics** — if `ANALYZE` hasn't run since a big data change, the planner's row estimate for the predicate is wrong; the fix is `ANALYZE`, not a hint. (2) **Low selectivity** — if the predicate actually matches a large fraction of the table (say >5-10%), a Seq Scan genuinely *is* cheaper than an index scan plus that many random heap fetches; the index is the wrong tool. (3) **Cost parameters that don't match the hardware** — `random_page_cost` defaults to 4.0, tuned for spinning disks; on SSD/NVMe the real ratio is closer to 1.1, and leaving it at 4.0 makes the planner systematically overvalue sequential scans. PostgreSQL deliberately has no index hints; `enable_seqscan = off` is a diagnostic to see the alternative plan's cost, never a production fix. If after fixing stats and cost params the planner still avoids the index, the honest conclusion is often that the index doesn't help this query.

**Follow-up**: When *is* a Seq Scan legitimately faster than using an available index? — When the query returns a large fraction of rows, when the table is small enough to fit in a few pages, or when the index would require so many random heap lookups that sequential I/O over the whole table wins.

---

### Q3 — "This table has 15 indexes. Is that a problem?"

**Junior answer**: No, more indexes just means more queries can be fast.

**Principal answer**: Almost certainly yes. Indexes are not free reads-only structures — each one is a tax on every write, consumes cache and disk, must be vacuumed, and can be chosen wrongly by the planner. The specific pathologies with 15 indexes: (1) **write amplification** — every INSERT touches 15 B-trees; (2) **redundancy** — an index on `(a)` is fully subsumed by an index on `(a, b)` as a leftmost prefix, so it's pure write cost with zero unique read value; (3) **unused indexes** — many are created "just in case" and never chosen, which you can prove with `idx_scan = 0` in `pg_stat_user_indexes` over a representative window; (4) **planner overhead and mis-selection** — more candidate paths, and more chances to pick a worse one. The move is to audit: find zero/low-scan indexes and redundant prefixes, and `DROP INDEX CONCURRENTLY` the dead ones. A healthy OLTP table usually needs the PK, its real foreign keys, and a small number of composite indexes matching actual query shapes — not fifteen.

**Follow-up**: Two indexes exist, `(customer_id)` and `(customer_id, created_at)`. Which do you drop and why? — Drop `(customer_id)`: any query it can serve is also served by the leftmost prefix of `(customer_id, created_at)`, so it adds only write cost. Keep the composite. (Caveat: the single-column index is slightly smaller, so on an extreme write-heavy path with no query needing the second column you might occasionally prefer the narrow one — but as a default, the composite wins.)

---

## 15. Mental Model Checkpoint

1. **A table has one narrow row per INSERT and 6 indexes. Roughly how many physical B-tree writes does a single INSERT trigger, and what else does each one generate that costs you later?** Think past the heap write — what happens in each index and in the WAL?

2. **You change one non-indexed column on a row via UPDATE, and the new tuple fits on the same page. How many index entries are written? Now you add an index on that column — what changes, and what optimization did you just lose?**

3. **Reads on `orders` only ever filter `WHERE status = 'paid' AND created_at > ...`, and 92% of rows are already `paid`. Why is a plain index on `status` nearly worthless, and how does a *partial* index on the non-paid rows flip the economics for the queries that look for `pending`/`cancelled`?**

4. **A query that used an index yesterday does a Seq Scan today after a bulk load, with no schema change. Walk the chain: what does the planner rely on, why did the bulk load break it, and what is the one command that most often fixes it?**

5. **Your storage is NVMe but the planner keeps preferring sequential scans over index scans it "should" use. Which single cost parameter is the prime suspect, what value does it default to, what does that default assume about the hardware, and which direction do you move it?**

6. **After a nightly job deletes 30M of 40M rows, the index on that table is still huge on disk and reads feel slow. What is physically taking up the space, why didn't the delete give it back, and what are the two commands that reclaim it — one blocking, one not?**

7. **You inherit a table with indexes `(category_id)`, `(category_id, price)`, and `(category_id, active, price)`. Which are redundant, by what rule, and how would you prove from live stats that dropping one is safe before you run `DROP INDEX CONCURRENTLY`?**

---

## 16. Quick Reference Card

```sql
-- ============ WHEN INDEXES HURT — DIAGNOSIS & FIXES ============

-- 1. WRITE AMPLIFICATION: every write maintains EVERY index.
--    N indexes => 1 heap write + up to N index writes + WAL for each.
--    An index over a hot-updated column also KILLS HOT updates.

-- Check HOT-update ratio (want n_tup_hot_upd close to n_tup_upd):
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0 * n_tup_hot_upd / NULLIF(n_tup_upd,0), 1) AS hot_pct
FROM pg_stat_user_tables ORDER BY n_tup_upd DESC;

-- 2. UNUSED / LOW-VALUE INDEXES: idx_scan tells the truth.
SELECT schemaname, relname, indexrelname, idx_scan,
       pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
ORDER BY idx_scan ASC, pg_relation_size(indexrelid) DESC;
-- idx_scan = 0 over a representative window  => candidate to DROP.

-- 3. REDUNDANT INDEXES: (a) is subsumed by (a, b) as leftmost prefix.
--    Keep the composite, drop the standalone prefix (usually).

-- 4. DROP without locking out reads/writes:
DROP INDEX CONCURRENTLY idx_products_active;   -- not inside a txn block

-- 5. BLOAT after mass DELETE/UPDATE: dead entries stay, file won't shrink.
--    Measure:
SELECT * FROM pgstattuple('idx_sessions_expires');   -- needs pgstattuple ext
--    Reclaim:
REINDEX INDEX CONCURRENTLY idx_sessions_expires;      -- rebuild, non-blocking
--    (plain REINDEX / VACUUM FULL are blocking — avoid on live tables)

-- 6. OPTIMIZER IGNORING AN INDEX — check in this order:
--    a) Stale stats  => planner's row estimate is wrong:
ANALYZE audit_logs;
EXPLAIN SELECT ... ;          -- compare estimated vs actual rows
--    b) Low selectivity => Seq Scan is genuinely cheaper (index won't help).
--    c) Cost params wrong for hardware (SSD/NVMe):
SET random_page_cost = 1.1;   -- default 4.0 assumes spinning disk
--    NEVER ship enable_seqscan=off as a fix; it's a diagnostic only.

-- 7. PARTIAL INDEX = the antidote to low-cardinality / hot-flag indexes.
--    Index only the rows reads actually touch:
CREATE INDEX idx_sessions_active ON sessions (expires_at)
  WHERE revoked = FALSE;      -- ~3% the size, ~3% the write cost

-- 8. COVERING INDEX (index-only scan, avoids heap fetches):
CREATE INDEX idx_orders_cust_cov ON orders (customer_id)
  INCLUDE (status, total_amount);

-- ============ RULE OF THUMB ============
-- An index earns its place only if READ benefit > WRITE cost.
-- Prove reads with idx_scan; prove write cost with table write volume
-- and the HOT ratio. Seq scan winning is often CORRECT, not a bug.
```

---

## Connected Topics

- **Topic 45 — Index Fundamentals (B-tree internals)**: The structure whose per-write maintenance cost is the entire reason indexes hurt; understand the B-tree before reasoning about what maintaining N of them costs.
- **Topic 46 — Choosing the Right Index (cardinality & selectivity)**: The flip side — low cardinality and poor selectivity are precisely what turn an index from asset into pure write tax here.
- **Topic 47 — Composite & Covering Indexes**: The leftmost-prefix rule that makes standalone indexes redundant, and the `INCLUDE` covering trick used as an antidote in this topic.
- **Topic 49 — Partial & Expression Indexes**: The primary antidote covered here — indexing only the rows reads touch to collapse write and bloat cost.
- **Topic 50 — Reading EXPLAIN / EXPLAIN ANALYZE**: How you actually confirm a Seq Scan vs Index Scan choice and compare estimated vs actual rows when the planner "ignores" an index.
- **Topic 51 — The Query Planner & Cost Model**: Why the planner picks Seq Scans, and the cost parameters (`random_page_cost`, `seq_page_cost`) behind those decisions.
- **Topic 52 — Table Statistics & ANALYZE**: The statistics whose staleness is the single most common cause of the planner abandoning a good index after a bulk load.
- **Topic 53 — VACUUM, Bloat & HOT Updates**: Where index bloat comes from, why mass DELETE/UPDATE doesn't shrink files, and how indexes over hot columns defeat Heap-Only Tuple updates.
- **Topic 54 — Online Maintenance (CONCURRENTLY)**: How to `REINDEX`/`CREATE`/`DROP INDEX CONCURRENTLY` without locking a live table when applying every fix in this topic.
