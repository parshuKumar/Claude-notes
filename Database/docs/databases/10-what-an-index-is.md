# 10 — What an Index Is
## Phase: Indexes

---

## ELI5 — The Simple Analogy

A 900-page textbook on Indian history.

You want everything about the Chola dynasty. Option one: start at page 1 and read all 900 pages, noting every mention. That's a **full table scan** — guaranteed correct, and it takes all afternoon.

Option two: flip to the **index at the back**. It's alphabetical, so you find "Chola" in about four seconds by binary-searching the page edges. It says: *pages 141, 288, 412, 655*. You turn to four pages. Total time: thirty seconds.

Now notice three things people forget.

**The index is a separate object.** It's extra paper. The book is thicker and heavier because of it.

**Someone had to build it, and rebuild it.** If a second edition inserts a chapter at page 200, every page number after that shifts, and the whole index must be corrected. Writing is now more expensive.

**It doesn't always help.** Look up "India" in that index and it says *pages 1–900*. You'd have been faster just reading the book, because you'd skip the round trip through the index. **An index that matches almost everything is worse than no index at all** — and this is the single most misunderstood thing about indexes.

---

## Where this fits in the big picture

```
   PHASE 1 — you now know:
   04 pages · 05 tuples · 07 buffer pool · 09 the life of a query
                          │
                          ▼
              ┌───────────────────────┐
              │ 10 WHAT AN INDEX IS   │  ← YOU ARE HERE
              │ the trade, in pages   │
              └───────────┬───────────┘
                          │
     ┌────────────────────┼─────────────────────┐
     ▼                    ▼                     ▼
 11 B-tree           13 index types        15 selectivity
 (the structure)     (the variants)        (when it helps)
     │                    │                     │
     └────────────────────┴─────────────────────┘
                          ▼
              17 when indexes hurt · 18 the planner
```

Phase 1 taught you that **cost is measured in pages read**. Phase 2 is entirely about one question: *how do we read fewer pages?*

---

## What is this?

An **index** is a separate data structure, stored in its own file, that maps column values to the physical locations of the rows containing them. The database maintains it automatically on every insert, update, and delete of the underlying table.

It exists to convert *"examine every row"* into *"navigate directly to the few rows that match."*

It is not part of the table. It stores no data you couldn't get from the table. Drop every index in your database and every query still returns exactly the same rows — just slower. **An index is purely a performance object**, with one exception: a `UNIQUE` index also enforces a constraint (Topic 24).

---

## Why does it matter for a backend developer?

Because this is where the majority of your database performance lives, and because the trade is genuinely two-sided.

**The upside is enormous.** A query on 10 million rows:

```
 no index:  read 130,000 pages  →  1,020 ms
 index:     read 8 pages        →  0.4 ms
                                   ─────────
                                   2,550× faster
```

**The downside is real and permanent.** Every index you add:

- Makes every `INSERT` write one more B-tree entry (plus WAL for it).
- Makes every `UPDATE` of an indexed column write two more (delete + insert).
- Occupies disk, and — worse — occupies **buffer pool**, competing with your table for RAM.
- Gives the planner one more option to consider, increasing planning time.
- Can be chosen *wrongly* by the planner, making a query slower than no index at all.

Real numbers from a production `orders` table:

```
 0 indexes:  18,400 inserts/sec
 3 indexes:   9,100 inserts/sec       (−51%)
 8 indexes:   3,800 inserts/sec       (−79%)
```

So the skill is not "add indexes." It is: **add exactly the indexes your access patterns need, and no more.** This topic gives you the framework; Topics 13–18 give you the detail.

---

## The physical reality

An index is a file, exactly like a table is a file.

```
 base/16388/
 ├── 16390          orders — heap file (the rows)
 ├── 16390_fsm
 ├── 16390_vm
 ├── 16393          orders_pkey — B-tree index on (id)
 └── 16401          idx_orders_user_id — B-tree index on (user_id)

 Same 8 KB pages. Same buffer pool. Same WAL. Same VACUUM.
 An index is not a magic sidecar — it is another relation.
```

**What is actually stored in an index leaf page:**

```
 idx_orders_user_id  — a B-tree on orders(user_id)
 ┌──────────────────────────────────────────────────────────────┐
 │ PageHeader (24 B)                                            │
 │ ItemIds ...                                                  │
 │                                                              │
 │   free space                                                 │
 │                                                              │
 │ INDEX TUPLE: [ IndexTupleData hdr (8 B) │ user_id (8 B) ]    │
 │              ↑ contains the TID: (block, offset)             │
 │                                                              │
 │   key=4471 → TID (1204, 3)                                   │
 │   key=4471 → TID (2891,17)                                   │
 │   key=4472 → TID (4102, 5)                                   │
 │   ...                                                        │
 │                                                              │
 │ SPECIAL SPACE (16 B): left sibling, right sibling, flags     │
 └──────────────────────────────────────────────────────────────┘

 ⚠ Note what an index entry IS: the KEY plus a POINTER.
   ~16 bytes for a bigint key. Nothing else. No other columns.
   That is why an index is small relative to its table — and why
   an index-only scan needs the columns to be IN the index (Topic 12).
```

**Size comparison, measured:**

```sql
SELECT relname, pg_size_pretty(pg_relation_size(oid)) AS size,
       CASE relkind WHEN 'r' THEN 'table' WHEN 'i' THEN 'index' END AS kind
FROM pg_class WHERE relname LIKE 'orders%' ORDER BY relkind;
```
```
        relname        |  size   | kind
-----------------------+---------+-------
 orders                | 320 MB  | table    ← 3M rows × ~105 bytes
 orders_pkey           |  64 MB  | index    ← 3M × ~16 bytes + overhead
 idx_orders_user_id    |  64 MB  | index
 idx_orders_created_at |  64 MB  | index
                        ───────
        total indexes:  192 MB   = 60% of the table's size
```

Three indexes on a four-column table cost you 60% more disk *and* 60% more buffer pool. That's the price. It is often worth it. It is never free.

---

## How it works — step by step

### The core arithmetic: why a full scan is expensive

```
 TABLE: orders, 3,000,000 rows, ~105 bytes each
 rows per 8 KB page ≈ 8160 / 109 ≈ 74
 pages = 3,000,000 / 74 ≈ 40,540 pages = 316 MB

 QUERY: SELECT * FROM orders WHERE user_id = 4471;   (17 rows match)

 ── SEQUENTIAL SCAN ──────────────────────────────────────────────
  for page in 0 .. 40539:
      read 8 KB                                  ← 40,540 page reads
      for each of ~74 tuples:
          check MVCC visibility
          evaluate (user_id = 4471)              ← 3,000,000 evaluations
          if match: emit

  I/O:  316 MB.  CPU: 3M predicate evaluations + 3M visibility checks.
  Time: ~1,020 ms cold, ~180 ms fully cached.
  ⚠ AND: it pulls 316 MB through the buffer pool.

 ── INDEX SCAN ───────────────────────────────────────────────────
  index on user_id: 3,000,000 entries × 16 B ≈ 48 MB ≈ 6,150 pages
  B-tree fanout ≈ 8160/16 ≈ 500 keys per internal page
      level 0 (root):      1 page,   500 children
      level 1:           500 pages, 250,000 children
      level 2 (leaves): 6,150 pages
      ⇒ HEIGHT = 3

  1. read root page                        [1 read, ~always cached]
  2. binary search → child pointer
  3. read internal page                    [1 read, usually cached]
  4. binary search → leaf pointer
  5. read leaf page                        [1 read]
  6. scan the leaf for key 4471 → 17 TIDs
  7. for each TID: read that heap page     [up to 17 reads]

  I/O:  ~20 page reads = 160 KB.
  Time: ~0.4 ms.

  ⇒ 40,540 reads → 20 reads.  2,027× less I/O.
```

**Notice step 7.** The index gave you *pointers*, not data. You still have to visit the heap. That "heap fetch" is where index scans get expensive when many rows match — and it's the subject of Topic 12.

### What an INSERT costs, with and without indexes

```
 INSERT INTO orders (user_id, total_paise, created_at) VALUES (...);

 WITH NO INDEXES:
   1. FSM lookup                          → which page has room
   2. pin page, write 105-byte tuple      → 1 page dirtied
   3. WAL record                          → ~150 bytes
   TOTAL: 1 page write, ~150 B WAL

 WITH 3 INDEXES:
   1–3. as above
   4. pkey:      descend 3 levels, insert leaf entry   → 1 page dirtied, ~80 B WAL
   5. user_id:   descend 3 levels, insert leaf entry   → 1 page dirtied, ~80 B WAL
   6. created_at:descend 3 levels, insert leaf entry   → 1 page dirtied, ~80 B WAL
   TOTAL: 4 pages dirtied, ~390 B WAL, 9 extra page READS for the descents

 ⇒ 4× the pages dirtied. 2.6× the WAL. Plus the descents.
   And if any leaf is full → a PAGE SPLIT: 2 more pages written, plus a
   parent update, possibly cascading to the root. (Topic 11.)
```

### What an UPDATE costs — the part people don't expect

```
 UPDATE orders SET total_paise = 250000 WHERE id = 91;
 (total_paise is NOT indexed. Three indexes exist on other columns.)

 CASE A — HOT UPDATE (new tuple fits on the same page, no indexed
          column changed):
   • new tuple written on the same page
   • old line pointer REDIRECTS to it
   • ★ ZERO index writes. All three indexes still point at the same
     line pointer, which now redirects. This is why HOT is so valuable.

 CASE B — NON-HOT (page full, or an indexed column changed):
   • new tuple written, possibly on a different page → NEW TID
   • ★ ALL THREE indexes get a NEW entry pointing at the new TID
   • the old entries remain until VACUUM
   ⇒ one logical update → 4 physical writes + 4 WAL records

 THE LESSON: fillfactor < 100 on update-heavy tables buys HOT updates,
 which is worth far more than the wasted page space. (Topic 04.)
```

---

## Concept breakdown

```
INDEX
│  └── A separate relation mapping KEY → TID, maintained by the engine.
│      Purely a performance object (except UNIQUE, which also constrains).
│
├── KEY          the indexed value(s)
├── TID          (block, offset) — the physical row address (Topic 04)
└── ACCESS METHOD  the algorithm: btree, hash, gin, gist, brin, spgist

THE THREE COSTS OF AN INDEX  (name all three; people only name the first)
│
├── 1. WRITE COST     every INSERT/DELETE and non-HOT UPDATE writes to it
├── 2. SPACE COST     disk, AND buffer pool — it competes with your table
└── 3. PLANNER COST   one more path to evaluate; and a chance to choose wrong

THE THREE THINGS AN INDEX CAN DO  (this framing predicts most plans)
│
├── 1. FILTER    find rows matching a predicate      WHERE user_id = 4471
├── 2. ORDER     return rows already sorted          ORDER BY created_at DESC
└── 3. COVER     supply the values themselves,       SELECT user_id FROM ...
                 avoiding the heap entirely           (index-only scan)
   ⇒ ONE index doing all three for one query is the ideal. Design for it.

SCAN TYPES YOU WILL SEE IN EXPLAIN
│
├── Seq Scan          read every page, filter every tuple
├── Index Scan        descend the tree, then fetch heap tuples one by one
│                     (random I/O; good for few rows)
├── Bitmap Index Scan → Bitmap Heap Scan
│                     collect ALL matching TIDs first, sort them by page,
│                     then read each heap page ONCE in physical order.
│                     ⇒ turns random I/O into semi-sequential. The planner
│                       chooses this for medium selectivity. (Topic 12.)
├── Index Only Scan   answer entirely from the index; heap never touched
│                     (requires the visibility map to say "all visible")
└── Parallel variants of all of the above

SELECTIVITY  — the number that decides everything
│
│   selectivity = rows matched / total rows
│
├── 0.0001 (17 of 3M)   → index scan wins by 1000×
├── 0.05   (150k of 3M) → bitmap scan; roughly break-even zone
└── 0.40   (1.2M of 3M) → SEQ SCAN WINS. The index is a liability.
   ⇒ The break-even is usually around 5–10% of the table, because a
     sequential read of N pages is far cheaper than N random reads.
     This is why "the planner ignored my index" is often the planner
     being right. (Topics 15, 17, 18.)
```

---

## Diagrams

**Diagram 1 — big picture: the index sits beside the table**

```
                    ┌──────────────── QUERY ────────────────┐
                    │ WHERE user_id = 4471                  │
                    └───────────────┬───────────────────────┘
                                    │
              ┌─────────────────────┴──────────────────────┐
              │                                            │
       NO INDEX                                     WITH INDEX
              │                                            │
              ▼                                            ▼
   ┌────────────────────┐                    ┌──────────────────────┐
   │ HEAP FILE 16390    │                    │ INDEX FILE 16401     │
   │ ┌──┬──┬──┬──┬──┐   │                    │      [root]          │
   │ │p0│p1│p2│..│pN│   │  read ALL          │      ╱    ╲          │
   │ └──┴──┴──┴──┴──┘   │  40,540 pages      │  [int]    [int]      │
   │                    │                    │   ╱ ╲      ╱ ╲       │
   └────────────────────┘                    │[leaf]...[leaf]       │
                                             └──────────┬───────────┘
                                                        │ TIDs
                                                        ▼
                                             ┌──────────────────────┐
                                             │ HEAP FILE 16390      │
                                             │ read only pages      │
                                             │ 1204, 2891, 4102...  │
                                             │ 17 pages             │
                                             └──────────────────────┘
```

**Diagram 2 — data flow: the read path vs the write path**

```
  READ PATH  (index helps)              WRITE PATH  (index costs)
  ──────────────────────────            ─────────────────────────────
   query                                 INSERT
     │                                     │
     ▼                                     ├──▶ heap: find page, write tuple
  descend index  ── 3 page reads           │
     │                                     ├──▶ index 1: descend, insert  ┐
     ▼                                     ├──▶ index 2: descend, insert  ├ each
  leaf: get TIDs                           └──▶ index 3: descend, insert  ┘ a page
     │                                                                      read
     ▼                                     ⚠ any leaf full → SPLIT          + write
  heap fetch  ── 1 read per row              → 2 more page writes           + WAL
     │                                       → parent update
     ▼                                       → possibly up to the root
  visibility check
     │
     ▼
   rows

  ⇒ Reads get 1000× faster. Writes get 2–5× slower. THAT IS THE TRADE.
```

**Diagram 3 — before/after: selectivity flips the winner**

```
 SELECTIVITY 0.0006  (17 rows of 3M)
 ┌──────────────────────────────────────────────────────────────┐
 │ Seq Scan   ████████████████████████████████████ 40,540 pages │
 │ Index Scan █                                        20 pages │
 └──────────────────────────────────────────────────────────────┘
                                        INDEX WINS by 2,000×

 SELECTIVITY 0.05  (150,000 rows of 3M)
 ┌──────────────────────────────────────────────────────────────┐
 │ Seq Scan     ████████████████████████████████ 40,540 seq     │
 │ Index Scan   ████████████████████████████████ ~38,000 RANDOM │
 │ Bitmap Scan  ██████████████████ ~19,000 semi-sequential      │
 └──────────────────────────────────────────────────────────────┘
                         BITMAP WINS — random I/O is ~4× the cost
                         per page, so 38k random ≈ 152k sequential

 SELECTIVITY 0.40  (1.2M rows of 3M)
 ┌──────────────────────────────────────────────────────────────┐
 │ Seq Scan   ████████████████████████████████ 40,540 sequential│
 │ Index Scan ████████████████████████████████████████████████  │
 │            40,540 heap pages (each visited MANY times)       │
 │            + 6,150 index pages + random I/O penalty          │
 └──────────────────────────────────────────────────────────────┘
                    SEQ SCAN WINS. The index makes it SLOWER.
```

---

## Example 1 — basic

Build it, measure it, and watch the break-even happen.

```sql
CREATE TABLE orders (
  id          bigserial PRIMARY KEY,
  user_id     bigint      NOT NULL,
  status      text        NOT NULL,
  total_paise bigint      NOT NULL,
  created_at  timestamptz NOT NULL DEFAULT now()
);

INSERT INTO orders (user_id, status, total_paise, created_at)
SELECT (random()*200000)::bigint,
       (ARRAY['pending','paid','shipped','cancelled'])[1+(random()*3)::int],
       (random()*500000)::bigint,
       now() - (random()*365)::int * interval '1 day'
FROM generate_series(1, 3000000);

VACUUM ANALYZE orders;
```

**Step 1 — no index. Highly selective query.**

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = 4471;
```
```
Seq Scan on orders  (cost=0.00..66190.00 rows=15 width=45)
                    (actual time=2.104..184.221 rows=17 loops=1)
  Filter: (user_id = 4471)
  Rows Removed by Filter: 2999983
  Buffers: shared hit=1024 read=27666
Execution Time: 184.9 ms
```

**Step 2 — add the index.**

```sql
CREATE INDEX idx_orders_user_id ON orders (user_id);
VACUUM ANALYZE orders;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = 4471;
```
```
Index Scan using idx_orders_user_id on orders
    (cost=0.43..62.31 rows=15 width=45)
    (actual time=0.041..0.089 rows=17 loops=1)
  Index Cond: (user_id = 4471)
  Buffers: shared hit=20
Execution Time: 0.118 ms
```

**28,690 pages → 20 pages. 185 ms → 0.12 ms. 1,566× faster.**

**Step 3 — now watch the index become useless.**

```sql
CREATE INDEX idx_orders_status ON orders (status);
VACUUM ANALYZE orders;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE status = 'paid';
```
```
Seq Scan on orders  (cost=0.00..66190.00 rows=749421 width=45)
                    (actual time=0.019..312.442 rows=750128 loops=1)
  Filter: (status = 'paid')
  Rows Removed by Filter: 2249872
  Buffers: shared hit=28690
Execution Time: 388.1 ms
```

**The planner built the index and then refused to use it.** That is not a bug. `status = 'paid'` matches 25% of the table. Force it and see why:

```sql
SET enable_seqscan = off;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE status = 'paid';
```
```
Bitmap Heap Scan on orders  (actual time=71.2..498.3 rows=750128 loops=1)
  Recheck Cond: (status = 'paid')
  Heap Blocks: exact=28690
  Buffers: shared hit=30742
  ->  Bitmap Index Scan on idx_orders_status  (actual time=64.1..64.1 rows=750128)
Execution Time: 561.9 ms
SET enable_seqscan = on;
```

**Using the index is 45% SLOWER** — it read the same 28,690 heap pages *plus* 2,052 index pages, and did extra work building the bitmap. `idx_orders_status` is pure cost: it slows every write, occupies 20 MB of RAM, and helps nothing.

```sql
DROP INDEX idx_orders_status;   -- correct decision
```

**Step 4 — measure the write cost.**

```sql
CREATE TABLE t0 (id bigserial PRIMARY KEY, a bigint, b bigint, c bigint, d timestamptz);
CREATE TABLE t3 (LIKE t0 INCLUDING ALL);
CREATE INDEX ON t3(a); CREATE INDEX ON t3(b); CREATE INDEX ON t3(c);

\timing on
INSERT INTO t0 (a,b,c,d) SELECT i,i,i,now() FROM generate_series(1,1000000) i;
-- Time: 2841.204 ms
INSERT INTO t3 (a,b,c,d) SELECT i,i,i,now() FROM generate_series(1,1000000) i;
-- Time: 9204.882 ms          ← 3.2× slower
```

And the WAL:

```sql
SELECT pg_current_wal_lsn() AS b \gset
INSERT INTO t3 (a,b,c,d) SELECT i,i,i,now() FROM generate_series(1,200000) i;
SELECT pg_size_pretty(pg_current_wal_lsn() - :'b'::pg_lsn);
-- 84 MB  vs  22 MB for t0
```

---

## Example 2 — production scenario

**The situation.** You inherit a 6-year-old `orders` table. 84 million rows, 42 GB of table, **71 GB of indexes** (19 of them). Writes are slow — checkout p99 is 800 ms and the DB is disk-bound. Reads are fine.

**Step 1 — which indexes are actually used?**

```sql
SELECT s.relname AS table, s.indexrelname AS index,
       s.idx_scan AS scans,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS size,
       i.indisunique AS is_unique,
       pg_get_indexdef(s.indexrelid) AS def
FROM pg_stat_user_indexes s
JOIN pg_index i ON i.indexrelid = s.indexrelid
WHERE s.relname = 'orders'
ORDER BY s.idx_scan ASC, pg_relation_size(s.indexrelid) DESC;
```
```
 table  |            index               |  scans   | size    | is_unique
--------+--------------------------------+----------+---------+-----------
 orders | idx_orders_notes_gin           |        0 | 8912 MB | f     ⚠
 orders | idx_orders_status              |        0 | 1804 MB | f     ⚠
 orders | idx_orders_updated_at          |        0 | 1804 MB | f     ⚠
 orders | idx_orders_is_gift             |       12 | 1804 MB | f     ⚠
 orders | idx_orders_user_id             |     3102 | 1804 MB | f     ⚠
 orders | idx_orders_created_at          |    88240 | 1804 MB | f
 orders | idx_orders_user_id_created_at  | 41209841 | 2712 MB | f
 orders | orders_pkey                    | 92004112 | 1804 MB | t
 ...
```

⚠ **Important caveat before you delete anything:** `idx_scan` is cumulative since the last `pg_stat_reset()`. Check when that was, and confirm on **all replicas** — a replica may be the only thing using an index, and its counters are separate.

```sql
SELECT stats_reset FROM pg_stat_database WHERE datname = current_database();
-- 2024-11-02  → 9 months of data. Trustworthy.
```

**Step 2 — the four categories of waste.**

| Category | Indexes | Size | Why it's waste |
|---|---|---|---|
| **Never scanned** | `notes_gin`, `status`, `updated_at` | 12.5 GB | Built for a feature that shipped differently, or for a one-off query |
| **Low selectivity** | `is_gift` (boolean, 99.7% false) | 1.8 GB | 12 scans in 9 months; the planner correctly refuses it |
| **Redundant prefix** | `user_id` | 1.8 GB | `(user_id, created_at)` already serves every `user_id = ?` query — a composite index serves any **leftmost prefix** of its columns (Topic 14) |
| **Duplicate** | two indexes with identical definitions | 1.8 GB | Created by two migrations with different names |

**Step 3 — find redundant prefixes systematically.**

```sql
-- Any index whose column list is a strict leftmost prefix of another
SELECT a.indexrelid::regclass AS redundant,
       b.indexrelid::regclass AS covered_by,
       pg_size_pretty(pg_relation_size(a.indexrelid)) AS reclaimable
FROM pg_index a
JOIN pg_index b ON a.indrelid = b.indrelid
                AND a.indexrelid <> b.indexrelid
                AND a.indkey::int2[] <@ b.indkey::int2[]   -- rough prefix test
                AND array_length(string_to_array(a.indkey::text,' '),1)
                  < array_length(string_to_array(b.indkey::text,' '),1)
WHERE a.indrelid = 'orders'::regclass
  AND NOT a.indisunique                                    -- ★ never drop a unique
  AND NOT a.indisprimary;
```

⚠ Two things this check must respect:
- **Never drop a `UNIQUE` index** on prefix grounds. `UNIQUE(a)` and `INDEX(a,b)` are *not* interchangeable — the first enforces a constraint the second does not.
- Check the **first column matches**, not just set containment. `(b, a)` does not cover `(a)`.

**Step 4 — drop safely.**

```sql
-- 1. Make it invisible to the planner WITHOUT dropping it (PG 15+ trick):
--    there is no true "invisible index" in PostgreSQL, so instead:
BEGIN;
DROP INDEX CONCURRENTLY idx_orders_status;   -- ⚠ CONCURRENTLY can't run in a txn
ROLLBACK;

-- The real procedure:
--   a) Confirm zero scans across primary AND all replicas.
--   b) Record the exact definition so you can rebuild it:
SELECT pg_get_indexdef('idx_orders_status'::regclass);
--   c) Drop without blocking:
DROP INDEX CONCURRENTLY idx_orders_status;
--   d) Watch pg_stat_statements for regressions for 48 hours.
--   e) Rebuild with CREATE INDEX CONCURRENTLY if anything degrades.
```

**Step 5 — the result.**

```
                     BEFORE          AFTER
 indexes              19              8
 index size          71 GB          24 GB       −66%
 checkout p99       800 ms         190 ms       −76%
 inserts/sec         3,100          9,400       +203%
 WAL per hour        410 GB         148 GB      −64%
 buffer pool freed     —            47 GB worth of pressure removed
```

**Not one query got slower.** Eleven indexes were doing nothing but taxing every write, occupying RAM the hot tables needed, and generating WAL that had to be shipped to two replicas.

**The prevention rule** (this is standing rule #4 in the case-studies folder):

> Every index must map to a line in a written access-pattern table.
> Adding one in a PR requires naming the query it serves.
> An index with no named query is rejected.

---

## Common mistakes

**1. "Add an index" as a reflex for any slow query.**
- *Symptom:* 19 indexes, writes crawling, and the slow query is still slow.
- *Engine-level why:* the planner will not use an index that doesn't reduce pages read. Low selectivity, a leading wildcard, a function on the column, or a type mismatch all disqualify it.
- *Diagnose:* `EXPLAIN (ANALYZE)` **after** creating it, then `pg_stat_user_indexes.idx_scan` a week later.
- *Fix:* diagnose first (Topic 09's six questions), index second.

**2. Indexing low-cardinality columns.**
- *Symptom:* an index on `status`, `is_active`, `type`, `is_deleted` with near-zero scans.
- *Engine-level why:* matching 25% of rows means reading 25% of pages randomly, which costs more than reading 100% sequentially.
- *Diagnose:* `SELECT n_distinct FROM pg_stats WHERE tablename='orders' AND attname='status';`
- *Fix:* a **partial index** (`WHERE status = 'pending'`) if you only query the rare value — that's tiny and highly effective (Topic 13). Or a composite index where the low-cardinality column is *not* first.

**3. Not knowing that a composite index serves its leftmost prefixes.**
- *Symptom:* separate indexes on `(user_id)` and `(user_id, created_at)`.
- *Engine-level why:* the B-tree is sorted by `user_id` first, so a search on `user_id` alone descends normally and scans a contiguous range. The single-column index adds nothing.
- *Diagnose:* the redundancy query in Example 2.
- *Fix:* drop the prefix index. Keep the composite. (Full treatment in Topic 14.)

**4. Wrapping the indexed column in a function.**
- *Symptom:* index exists, `Seq Scan` in the plan.
```sql
WHERE lower(email) = 'a@shop.in'          -- idx on (email) is UNUSABLE
WHERE created_at::date = '2026-08-01'     -- idx on (created_at) is UNUSABLE
WHERE user_id::text = '4471'              -- type coercion kills it too
```
- *Engine-level why:* the B-tree is sorted by `email`, not by `lower(email)`. There is no way to descend a tree ordered by X using a predicate on f(X).
- *Fix:* an **expression index** — `CREATE INDEX ON users (lower(email));` — or rewrite the predicate to a range: `WHERE created_at >= '2026-08-01' AND created_at < '2026-08-02'`. The range rewrite is usually better: it's sargable *and* needs no extra index.

**5. Creating indexes with a plain `CREATE INDEX` on a live table.**
- *Symptom:* the application freezes for 4 minutes.
- *Engine-level why:* `CREATE INDEX` takes a `SHARE` lock — it blocks all writes for the whole build.
- *Fix:* `CREATE INDEX CONCURRENTLY`. It's slower (two table passes) and cannot run inside a transaction, and it can leave an `INVALID` index if it fails — check with `SELECT indisvalid FROM pg_index` and drop/retry. But it does not block writes. (Topic 28.)

**6. Forgetting that indexes bloat too.**
- *Symptom:* an index is 3× the size it should be; scans got slower over time.
- *Engine-level why:* MVCC dead entries and page splits leaving leaves half full (Topic 08).
- *Diagnose:* `SELECT * FROM pgstatindex('idx_orders_user_id');` — check `avg_leaf_density` (want >70%) and `leaf_fragmentation`.
- *Fix:* `REINDEX INDEX CONCURRENTLY`.

**7. Judging an index by disk cost alone.**
- *Symptom:* "it's only 1.8 GB, disk is cheap."
- *Engine-level why:* it also occupies buffer pool, competing with your hot tables (Topic 07); it adds WAL that must be fsynced and shipped to every replica; and it adds a planning alternative on every query.
- *Fix:* price all four costs, not just disk.

---

## Hands-on proof

**PROVE IT #1 — an index is a file.**
```sql
CREATE INDEX idx_orders_user_id ON orders(user_id);
SELECT pg_relation_filepath('idx_orders_user_id');   -- base/16388/16401
SELECT relname, relkind, relpages FROM pg_class WHERE relname LIKE 'idx_orders%';
```

**PROVE IT #2 — look inside an index page.**
```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;
SELECT * FROM bt_metap('idx_orders_user_id');
```
```
 magic  | version | root | level | fastroot | fastlevel
--------+---------+------+-------+----------+-----------
 340322 |       4 |  412 |     2 |      412 |         2
```
`level 2` means a 3-level tree (levels 2, 1, 0). Now read a leaf:
```sql
SELECT itemoffset, ctid, itemlen, data
FROM bt_page_items('idx_orders_user_id', 3) LIMIT 5;
```
```
 itemoffset |   ctid    | itemlen |          data
------------+-----------+---------+-------------------------
          1 | (1204,3)  |      16 | 77 11 00 00 00 00 00 00
          2 | (2891,17) |      16 | 77 11 00 00 00 00 00 00
          3 | (4102,5)  |      16 | 78 11 00 00 00 00 00 00
```
**That's the whole index entry: 8 bytes of key + a TID, 16 bytes total.** `0x1177` = 4471 little-endian. Now you have physically seen what an index stores.

**PROVE IT #3 — watch the break-even move.**
```sql
CREATE INDEX idx_o_total ON orders(total_paise);
VACUUM ANALYZE orders;
-- vary the range width and watch the plan flip:
EXPLAIN SELECT count(*) FROM orders WHERE total_paise < 500;        -- Index Scan
EXPLAIN SELECT count(*) FROM orders WHERE total_paise < 25000;      -- Bitmap
EXPLAIN SELECT count(*) FROM orders WHERE total_paise < 250000;     -- Seq Scan
```
Find *your* break-even by bisection. It depends on row width, `random_page_cost`, and cache state — which is exactly why the planner computes it rather than using a fixed rule.

**PROVE IT #4 — function calls disable the index.**
```sql
CREATE INDEX idx_o_created ON orders(created_at);
EXPLAIN SELECT * FROM orders WHERE created_at::date = '2026-08-01';   -- Seq Scan
EXPLAIN SELECT * FROM orders WHERE created_at >= '2026-08-01'
                               AND created_at <  '2026-08-02';        -- Index Scan
-- and the expression-index alternative:
CREATE INDEX idx_o_created_date ON orders ((created_at::date));
EXPLAIN SELECT * FROM orders WHERE created_at::date = '2026-08-01';   -- Index Scan
```

**PROVE IT #5 — the write cost, isolated.**
```sql
\timing on
CREATE TABLE w0 (id bigserial PRIMARY KEY, a bigint, b bigint, c bigint);
INSERT INTO w0 (a,b,c) SELECT i,i,i FROM generate_series(1,1000000) i;   -- baseline
CREATE INDEX ON w0(a); CREATE INDEX ON w0(b); CREATE INDEX ON w0(c);
INSERT INTO w0 (a,b,c) SELECT i,i,i FROM generate_series(1,1000000) i;   -- with indexes
```

**PROVE IT #6 — find unused indexes on your own database.**
```sql
SELECT s.relname, s.indexrelname, s.idx_scan,
       pg_size_pretty(pg_relation_size(s.indexrelid)) AS size
FROM pg_stat_user_indexes s
JOIN pg_index i ON i.indexrelid = s.indexrelid
WHERE s.idx_scan < 50
  AND NOT i.indisunique AND NOT i.indisprimary
  AND pg_relation_size(s.indexrelid) > 10*1024*1024
ORDER BY pg_relation_size(s.indexrelid) DESC;
```
Run this on your actual production database today. Most teams find several gigabytes.

---

## The design decision framework

```
CREATE AN INDEX WHEN:
  ✓ The query runs frequently (check pg_stat_statements `calls`)
  ✓ Selectivity is high — it matches < ~5% of the table
  ✓ It serves a FILTER, an ORDER BY, or a JOIN key that you can name
  ✓ You have measured the query as slow, not assumed it
  ✓ The table's read:write ratio favours reads for this access path

DO NOT CREATE AN INDEX WHEN:
  ✗ The column has few distinct values (boolean, small enum, status)
     → unless you use a PARTIAL index on the rare value
  ✗ A composite index already covers it as a leftmost prefix
  ✗ The table is small (< ~1,000 rows) — a seq scan is one page read
  ✗ The table is write-heavy and this path is queried rarely
  ✗ You cannot name the exact query it serves

CHOOSE THE VARIANT:
  point lookup / equality        → B-tree (default)
  range / ORDER BY / prefix LIKE → B-tree
  multiple columns filtered together → composite B-tree (order matters! T14)
  only a rare subset queried     → PARTIAL index          (T13)
  predicate uses a function      → EXPRESSION index       (T13)
  need to avoid the heap fetch   → COVERING (INCLUDE)     (T12)
  JSONB / arrays / full-text     → GIN                    (T16)
  huge, naturally time-ordered   → BRIN                   (T16)
  ranges / geometry / exclusion  → GiST                   (T16, T24)

THE SIGNAL TO LOOK FOR:
  Before creating:
      SELECT count(*) FILTER (WHERE <your predicate>)::float / count(*)
      FROM <table>;
  • < 0.01  → index will be used and will win big
  • 0.01–0.10 → probably a bitmap scan; measure before and after
  • > 0.10  → the planner will likely ignore it. Don't create it.
    Instead: change the query, add a partial index, or accept the scan.

  After creating, ALWAYS verify:
      EXPLAIN (ANALYZE, BUFFERS) <the query>;
  If the plan didn't change, DROP THE INDEX. An unused index is pure cost.

  And one week later:
      SELECT idx_scan FROM pg_stat_user_indexes WHERE indexrelname = '...';
  Zero → drop it.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build the 3M-row `orders` table from Example 1. Without creating any index, use `pg_class` to compute: rows per page, total pages, and the megabytes a sequential scan must read. Then create `idx_orders_user_id` and use `bt_metap` and `pg_relation_size` to compute the index's height and size. Finally, predict the page count for `WHERE user_id = 4471` **before** running `EXPLAIN`, then check.

### Exercise 2 — medium (apply it)
Using the same table, find the exact selectivity break-even point on your machine where the planner switches from `Index Scan` → `Bitmap Heap Scan` → `Seq Scan`, by bisecting on a range predicate. Then:
(a) set `random_page_cost = 1.1` and find the break-even again,
(b) explain in terms of the cost model why it moved and in which direction,
(c) state which value is correct for an NVMe SSD and why the default is what it is.

### Exercise 3 — hard (production simulation)
You are given a `orders` table: 60M rows, 28 GB table, 52 GB of indexes across 16 indexes. Checkout p99 is 900 ms; the primary is disk-bound at 88% utilisation; two replicas are lagging 12 s.

(a) Write the complete query that classifies every index into: unique/PK (never drop), never-scanned, redundant-prefix, duplicate-definition, low-selectivity, and actively-used. Your prefix check must not produce false positives on `(b,a)` vs `(a)`, and must exclude unique indexes.
(b) `idx_scan` is 0 for one index, but it was created 3 days ago for a monthly report. How do you avoid dropping it? Name two checks.
(c) Give the exact, safe drop procedure for a live system, including how you'd detect and revert a regression.
(d) Estimate the effect on: WAL volume, replica lag, buffer pool pressure, and insert throughput. Show your reasoning for each.
(e) One of the "unused" indexes is `UNIQUE (email)`. Explain why `idx_scan = 0` is completely irrelevant for it, and what would break if you dropped it.
(f) Write the CI check that prevents this from recurring.

---

## Mental model checkpoint

1. What does one B-tree index entry physically contain? How many bytes for a `bigint` key?
2. Name the three costs of an index. Which one do people forget, and why does it matter more than disk?
3. Name the three things an index can do for a query. What's the ideal case?
4. Why is an index on a boolean column usually useless? At roughly what selectivity does the planner abandon an index, and why is the number what it is?
5. What is a bitmap heap scan, and what problem with plain index scans does it solve?
6. `WHERE lower(email) = 'a@x.com'` won't use an index on `email`. Explain why at the B-tree level, and give two fixes.
7. You have indexes on `(user_id)` and `(user_id, created_at)`. Which is redundant and why? Name the one case where it would *not* be redundant.

---

## Quick reference card

| Concept | One line |
|---|---|
| Index | A separate relation mapping key → TID |
| TID | (block, offset) — the row's physical address |
| Selectivity | rows matched / total rows |
| Index Scan | descend + fetch heap rows one at a time (random I/O) |
| Bitmap Heap Scan | collect all TIDs, sort by page, read each page once |
| Index Only Scan | answered from the index alone; heap untouched |
| Partial index | `WHERE ...` — indexes only a subset |
| Expression index | indexes `f(column)` instead of `column` |
| Covering index | `INCLUDE (...)` — extra columns for index-only scans |

**Numbers to memorise**

| Thing | Value |
|---|---|
| B-tree index entry (bigint) | ~16 bytes |
| B-tree fanout (bigint keys) | ~500 per page |
| Tree height, 3M rows | 3 levels |
| Tree height, 1B rows | 4–5 levels |
| Index-scan break-even | ~**5–10%** of rows |
| Write slowdown per index | roughly 1.5–2× for the first, less each after |
| `random_page_cost` correct on SSD | **1.1** (default 4.0) |

**The three commands**

```sql
CREATE INDEX CONCURRENTLY idx_name ON tbl (col);   -- never blocks writes
DROP   INDEX CONCURRENTLY idx_name;
REINDEX INDEX CONCURRENTLY idx_name;               -- fixes bloat
```

**The two queries to run on production today**

```sql
-- unused indexes
SELECT s.relname, s.indexrelname, s.idx_scan,
       pg_size_pretty(pg_relation_size(s.indexrelid))
FROM pg_stat_user_indexes s JOIN pg_index i ON i.indexrelid=s.indexrelid
WHERE s.idx_scan < 50 AND NOT i.indisunique AND NOT i.indisprimary
ORDER BY pg_relation_size(s.indexrelid) DESC;

-- index bloat
SELECT indexrelname, pg_size_pretty(pg_relation_size(indexrelid))
FROM pg_stat_user_indexes ORDER BY pg_relation_size(indexrelid) DESC LIMIT 20;
```

---

## When would I use this at work?

1. **The "database is slow" sprint.** Instead of adding indexes, you run the unused-index query and find 40 GB of pure write tax. Dropping it improves writes 2–3× and frees buffer pool — usually a bigger win than any index you could have added, and it takes an afternoon.

2. **Reviewing a migration PR.** Someone adds `CREATE INDEX ON orders (is_gift)`. You can immediately say: boolean, 99.7% one value, the planner will never use it, and it costs a write on every insert forever. Ask for the query it serves; usually there isn't one, or a partial index is correct.

3. **A "we need to shard" conversation.** Writes are maxed out. Before agreeing to a six-month project, you check index count and WAL volume. Sixteen indexes generating 400 GB of WAL an hour is a schema problem, not a scale problem — and it's a two-week fix.

---

## Connected topics

**Understand before this:** 04 (pages, TIDs), 07 (buffer pool, random vs sequential I/O), 09 (the query trace this optimises).

**This unlocks:**
- **11** — the B-tree structure in full: nodes, splits, fanout, height
- **12** — the lookup traced end to end, heap fetches, index-only scans
- **13** — partial, expression, unique, covering indexes
- **14** — composite indexes and why column order is a different data structure
- **15** — selectivity, cardinality, and the statistics the planner uses
- **17** — when indexes hurt: write amplification and bloat in detail
- **18** — the planner: how it actually computes the break-even you found by bisection
