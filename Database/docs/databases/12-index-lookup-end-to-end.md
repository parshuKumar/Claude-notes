# 12 — Index Lookup End to End
## Phase: Indexes

---

## ELI5 — The Simple Analogy

Back to the textbook index. You look up "Chola" and it says *pages 141, 288, 412, 655*.

Now you have to actually **turn to those pages**. Four separate flips through a 900-page book, each one hunting for a page number. That flipping is the expensive part — the index lookup itself took four seconds; the flipping takes a minute.

Three things change the arithmetic:

**If the index says 400 pages instead of 4**, you stop flipping one at a time. You'd sort the page numbers, go through the book front to back once, and grab all 400 as you pass. That's a **bitmap heap scan**.

**If the index entry itself contained the answer** — "Chola: founded 300 BCE" right there in the index — you'd never open the book at all. That's an **index-only scan**, and it's the fastest thing a database can do that isn't a cache hit.

**And one thing you'd never think about with a book:** each page you turn to might have been *edited since the index was printed*. So you must check every page you land on to confirm it's still valid. That's the **MVCC visibility check** — and it's why PostgreSQL can't just trust its own index.

---

## Where this fits in the big picture

```
   10 what an index is ──▶ 11 B-tree structure
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │ 12 LOOKUP END TO END         │ ← YOU ARE HERE
                    │ index → heap → visibility    │
                    └───────────────┬──────────────┘
                                    │
          ┌─────────────────┬───────┴────────┬──────────────┐
          ▼                 ▼                ▼              ▼
    13 index types    15 selectivity    17 when they    18 the planner
    (INCLUDE)         (which scan?)     hurt            (the cost model)
```

Topic 11 got you to the leaf page holding TIDs. **This topic is everything that happens after** — and it's where most real index performance lives.

---

## What is this?

The complete path from a `WHERE` clause to a returned row, through an index. It has three distinct phases:

1. **Index scan** — descend the B-tree, collect TIDs (Topic 11).
2. **Heap fetch** — read the actual row from the table file using each TID.
3. **Visibility check** — confirm this row version is visible to your transaction.

Phase 2 is usually the expensive one. Phases 2 and 3 can sometimes be **skipped entirely** — and knowing when is the difference between a 0.4 ms query and a 40 ms one.

---

## Why does it matter for a backend developer?

Because "I added an index and it's still slow" almost always means the heap fetch is the bottleneck, not the index.

- An index scan returning 50,000 rows does **50,000 random page reads**. At 100 µs each on NVMe, that's 5 seconds. The index part took 0.01 ms.
- **The same query with a covering index takes 40 ms** — because it never touches the heap.
- A bitmap heap scan on the same query takes 800 ms — because it sorts the TIDs and reads each page once, sequentially.

Three plans, same index, same rows, 125× spread. The planner chooses between them, and understanding the choice is how you read `EXPLAIN` and how you design indexes that actually help.

And there's a gotcha that catches everyone: **`Heap Fetches: 0` requires the visibility map to be current, which requires VACUUM.** An index-only scan can silently stop being index-only after a batch of updates, and your p99 quadruples with no code change.

---

## The physical reality

### What the index gives you, and what it doesn't

```
 LEAF PAGE of idx_orders_user_id
 ┌────────────────────────────────────────────────────────────┐
 │ key=4471 → TID (1204, 3)                                   │
 │ key=4471 → TID (2891,17)                                   │
 │ key=4471 → TID (4102, 5)                                   │
 └────────────────────────────────────────────────────────────┘
                          │
                          │  TID = (block_number, item_offset)
                          │  ⚠ This is a POINTER. It contains:
                          │     • no other column values
                          │     • no visibility information
                          ▼
 HEAP FILE base/16388/16390
 ┌────────────────────────────────────────────────────────────┐
 │ block 1204 (8 KB)                                          │
 │   line pointer [3] → offset 7920, len 105                  │
 │     ┌──────────────────────────────────────────┐           │
 │     │ t_xmin=91004  t_xmax=0  t_ctid=(1204,3)  │ ← MVCC     │
 │     ├──────────────────────────────────────────┤           │
 │     │ id=8891 │ user_id=4471 │ total=249900 │… │ ← data    │
 │     └──────────────────────────────────────────┘           │
 └────────────────────────────────────────────────────────────┘
```

**Two consequences, both critical:**

1. **The index has no visibility information.** An index entry exists for a row that might be deleted, might be from an uncommitted transaction, might be an old version superseded by an update. PostgreSQL *must* check the heap tuple to know. This is the price of PostgreSQL's MVCC design — and it's why index-only scans need a separate mechanism.

2. **TIDs are scattered.** Consecutive index entries point to wildly different heap blocks: 1204, 2891, 4102. Each is a **random** read. Random reads cost ~4× sequential reads (that's what `random_page_cost = 4.0` encodes).

### The visibility map — the fork that makes index-only scans possible

```
 base/16388/16390_vm    — TWO BITS per heap page

 ┌───────────────────────────────────────────────────────────┐
 │ heap page 0   │ ALL_VISIBLE=1  ALL_FROZEN=1               │
 │ heap page 1   │ ALL_VISIBLE=1  ALL_FROZEN=0               │
 │ heap page 2   │ ALL_VISIBLE=0  ALL_FROZEN=0   ← modified  │
 │ heap page 3   │ ALL_VISIBLE=1  ALL_FROZEN=0               │
 └───────────────────────────────────────────────────────────┘

 ALL_VISIBLE = "every tuple on this page is visible to EVERY
                current and future transaction"
              ⇒ if set, an index-only scan can trust the index entry
                and skip the heap read entirely.

 SIZE: 2 bits per 8192-byte page = 1 byte per 32 KB of table.
       A 320 MB table has a 10 KB visibility map. It is ALWAYS cached.

 WHO SETS IT:   only VACUUM (and autovacuum).
 WHO CLEARS IT: any INSERT, UPDATE, or DELETE touching that page —
                immediately, as part of the write.

 ★ THIS IS THE FRAGILE PART. One UPDATE clears the bit for a whole page.
   Your index-only scan silently becomes an index scan with heap fetches
   until autovacuum comes back.
```

### The three scan types, physically

```
 ═══ INDEX SCAN ═══════════════════════════════════════════════
   descend → leaf → TID → HEAP READ → visibility → emit
                    TID → HEAP READ → visibility → emit
                    TID → HEAP READ → visibility → emit
   One heap read per row, in KEY order → RANDOM I/O.
   Rows come out SORTED by the index key. ★ This preserves ORDER BY.
   Best when: few rows (< ~1–2% of the table), or ORDER BY matters.

 ═══ BITMAP HEAP SCAN ═════════════════════════════════════════
   PHASE 1 (Bitmap Index Scan):
     descend → scan ALL matching leaf entries → build a BITMAP of
     heap PAGES that contain at least one match.
       page 1204: yes    page 1205: no    page 1206: yes ...
   PHASE 2 (Bitmap Heap Scan):
     walk the bitmap IN PAGE ORDER, read each page ONCE,
     recheck the condition on each tuple.
   ⇒ Random I/O becomes near-sequential. Each page read at most once.
   ⇒ Rows come out in PHYSICAL order → ORDER BY needs a Sort node.
   Best when: medium selectivity (~1–20%), or combining multiple indexes.

   ★ LOSSY BITMAPS: if the matching pages exceed work_mem, the bitmap
     degrades from per-tuple to per-page granularity. EXPLAIN shows
     "Heap Blocks: exact=1204 lossy=88900" and a "Recheck Cond" that
     now actually runs on every tuple of those pages. Raise work_mem.

 ═══ INDEX ONLY SCAN ══════════════════════════════════════════
   descend → leaf → for each entry:
       check the VISIBILITY MAP for that TID's page
         ALL_VISIBLE=1 → emit from the INDEX. NO HEAP READ. ★
         ALL_VISIBLE=0 → fall back to a heap read + visibility check
   ⇒ Requires: every column the query needs is IN the index.
   ⇒ 10–100× fewer page reads. The single biggest index optimisation.
   EXPLAIN shows "Heap Fetches: N" — N should be ~0.
```

---

## How it works — step by step

### Trace A: Index Scan (few rows)

```
 SELECT id, total_paise FROM orders WHERE user_id = 4471;   -- 17 rows

  1. Planner: estimates 15 rows from pg_statistic. 15 / 3,000,000 =
     0.0005 selectivity → Index Scan chosen.

  2. Descend idx_orders_user_id:
       metapage → root → internal → leaf              [4 buffer requests,
                                                       3 usually hits]
  3. Position at the first entry with key ≥ 4471.

  4. LOOP, one row at a time (this is a STREAMING node):
       a) read the next index entry → TID (1204, 3)
       b) ReadBuffer(orders, block 1204)
            hit  → ~100 ns
            miss → ~100 µs RANDOM read              [1 heap page read]
       c) follow line pointer 3 → the tuple bytes
       d) VISIBILITY CHECK against the snapshot:
            • is t_xmin committed and < snapshot.xmin?
            • is t_xmax unset, or set by an uncommitted/aborted txn?
            • ⚠ if the HINT BITS aren't set, this reads pg_xact —
              and then WRITES the hint bit back, dirtying the page
       e) if visible → project (id, total_paise) → emit upward
       f) release the buffer pin, advance to the next index entry

  5. After the last matching key, stop.

  TOTAL: 4 index pages + up to 17 heap pages = ~21 page reads.
  Rows emitted IN user_id ORDER (there's only one value here, but for a
  range scan this matters enormously).
```

### Trace B: Bitmap Heap Scan (medium selectivity)

```
 SELECT * FROM orders WHERE total_paise BETWEEN 100000 AND 130000;
 -- 180,000 rows of 3,000,000 = 6%

  PHASE 1 — Bitmap Index Scan
  1. Descend idx_orders_total.
  2. Walk leaves collecting EVERY matching TID: 180,000 of them.
  3. Build a TID bitmap in work_mem:
       a bit per (page, offset), organised by page.
       memory: ~180,000 × ~2 bits + page overhead ≈ 200 KB
     ⚠ IF THIS EXCEEDS work_mem → the bitmap goes LOSSY: it records
       "page 1204 has matches" without recording WHICH tuples. Phase 2
       must then recheck every tuple on that page.
  4. Sort/organise by page number.  ← the whole point

  PHASE 2 — Bitmap Heap Scan
  5. For each set page, IN ASCENDING PAGE ORDER:
       a) ReadBuffer(orders, page)   ← sequential-ish access pattern;
                                       the OS readahead helps
       b) for each marked offset (or every tuple, if lossy):
            visibility check
            RECHECK the condition   ← required when lossy; also required
                                      because the bitmap is per-page
            emit
  6. Done.

  TOTAL: ~2,050 index pages + ~28,000 heap pages, but read IN ORDER.
  vs a plain Index Scan: the same ~28,000 pages read RANDOMLY, many
  of them MULTIPLE TIMES (once per matching row on that page).

  ★ THE TWO WINS:
    (1) each heap page is read AT MOST ONCE
    (2) pages are read in physical order → sequential I/O + readahead

  ⚠ THE LOSS: output is in PHYSICAL order, not key order.
    An ORDER BY now needs an explicit Sort node.
```

### Trace C: Index Only Scan

```
 SELECT user_id, count(*) FROM orders WHERE user_id BETWEEN 1000 AND 2000
 GROUP BY user_id;
 -- everything the query needs (user_id) is IN idx_orders_user_id

  1. Descend to the leaf containing key 1000.
  2. LOOP:
       a) read the next index entry → key=1042, TID (8891, 12)
       b) ★ CHECK THE VISIBILITY MAP for heap page 8891:
            ReadBuffer(orders_vm, page 8891/32768)   ← the VM is tiny
                                                       and always cached
            ALL_VISIBLE bit == 1?
              YES → emit key=1042 STRAIGHT FROM THE INDEX.
                    NO HEAP READ. NO VISIBILITY CHECK. ★
              NO  → fall back: read heap page 8891, check visibility,
                    then emit.        (counted as a "Heap Fetch")
       c) advance; walk to the sibling leaf when this one is exhausted
  3. Aggregate.

  TOTAL (VM fully set): ~10 index pages. ZERO heap pages.
  TOTAL (VM stale):     ~10 index pages + up to 1,000 heap pages.

  ⇒ The SAME PLAN can be 100× slower depending on whether VACUUM has
    run. This is the single most surprising performance cliff in
    PostgreSQL, and `Heap Fetches:` in EXPLAIN is how you see it.
```

### The covering index — `INCLUDE`

```sql
-- The query needs user_id (filter) and total_paise + status (output).
CREATE INDEX idx_orders_cover ON orders (user_id) INCLUDE (total_paise, status);
```

```
 WHAT `INCLUDE` DOES PHYSICALLY:

 Regular composite (user_id, total_paise, status):
   ┌──────────────────────────────────────────────────────────┐
   │ INTERNAL: separator keys contain ALL THREE columns       │  ← wide
   │ LEAF:     all three columns + TID                        │     keys,
   └──────────────────────────────────────────────────────────┘     lower
                                                                    fanout
 INCLUDE (user_id) INCLUDE (total_paise, status):
   ┌──────────────────────────────────────────────────────────┐
   │ INTERNAL: separator keys contain ONLY user_id            │  ← narrow
   │ LEAF:     user_id + total_paise + status + TID           │     keys,
   └──────────────────────────────────────────────────────────┘     high
                                                                    fanout
 ⇒ INCLUDE columns are stored ONLY in leaves. They are payload, not
   key. They cannot be searched or sorted on — but they can be
   RETURNED, which is all an index-only scan needs.

 ⇒ ADVANTAGES over a plain composite:
   • higher fanout → shallower tree, smaller index
   • a UNIQUE index can INCLUDE extra columns without them
     participating in the uniqueness. ★ This is genuinely useful:
       CREATE UNIQUE INDEX ON users (email) INCLUDE (id, name);
   • non-indexable types (e.g. some geometric types) can be included
```

---

## Concept breakdown

```
THE THREE PHASES OF ANY INDEX LOOKUP
│
├── 1. INDEX SCAN       descend + walk leaves → TIDs.        CHEAP.
├── 2. HEAP FETCH       TID → 8 KB page read.                EXPENSIVE.
└── 3. VISIBILITY       is this version visible to me?       CHEAP but
                                                             requires #2
   ⇒ ALL index optimisation is about reducing or eliminating #2.

THE FOUR SCAN STRATEGIES
│
├── Seq Scan         read every page. No index. Wins above ~10%.
├── Index Scan       1 heap read per ROW. Random I/O. Preserves order.
│                    ✓ few rows  ✓ ORDER BY  ✓ LIMIT (streams, stops early)
├── Bitmap Heap Scan 1 heap read per PAGE, in page order.
│                    ✓ medium selectivity  ✓ combining indexes
│                    ✗ loses ordering  ✗ can go lossy
└── Index Only Scan  0 heap reads (when the VM is set).
                     ✓ all needed columns in the index  ✓ VACUUMed table

BITMAP AND / BITMAP OR  — combining indexes
│
│   WHERE user_id = 4471 AND status = 'paid'
│     Bitmap Index Scan on idx_user_id   → bitmap A
│     Bitmap Index Scan on idx_status    → bitmap B
│     BitmapAnd(A, B)                    → intersect
│     Bitmap Heap Scan                   → read only the surviving pages
│
└── ⚠ This is why "one index per column" isn't crazy — the planner can
     AND them. But a COMPOSITE index on (user_id, status) is almost
     always better: it does the intersection during the tree descent
     instead of materialising two bitmaps. (Topic 14.)

VISIBILITY MAP
│
├── 2 bits per heap page: ALL_VISIBLE, ALL_FROZEN
├── SET by VACUUM only
├── CLEARED by any write to that page, immediately
└── ⇒ INDEX-ONLY SCANS DEPEND ON VACUUM. On an update-heavy table
     they degrade silently. `Heap Fetches:` is the number to watch.

RECHECK COND
│
└── Appears in bitmap plans. Two reasons:
    (a) the bitmap is per-page (lossy), so tuples must be re-tested
    (b) the index condition was only approximate (GIN, some GiST)
    A "lossy=NNNN" number in `Heap Blocks:` means work_mem was too small.

INDEX COND  vs  FILTER  — a critical distinction in EXPLAIN
│
├── Index Cond:  evaluated DURING the tree descent. Reduces rows read.  ✓
└── Filter:      evaluated AFTER the heap fetch. Rows were already read. ✗
    ⇒ A predicate in `Filter` did NOT save you any I/O. If you see a big
      `Rows Removed by Filter`, that column belongs in the index.
```

---

## Diagrams

**Diagram 1 — big picture: the three strategies side by side**

```
  INDEX SCAN                BITMAP HEAP SCAN           INDEX ONLY SCAN
  ─────────────             ────────────────           ───────────────
   [index]                   [index]                     [index]
      │                         │                           │
      │ TID                     │ ALL TIDs                  │ key + payload
      ▼                         ▼                           ▼
   heap pg 4102             build bitmap:               check VM bit
      │ visibility          pages {1204,1206,          ┌────┴────┐
      ▼                            4102, 8891...}      1         0
   heap pg 1204                    │                    │         │
      │ visibility                 ▼ sort by page       ▼         ▼
      ▼                     read 1204 ─▶ 1206 ─▶      EMIT     heap read
   heap pg 8891             ─▶ 4102 ─▶ 8891           (no I/O!)  (fallback)
      │                            │
      ▼                            ▼
    rows (in KEY order)         rows (in PAGE order)      rows

  RANDOM I/O                SEQUENTIAL-ISH I/O          ~NO I/O
  1 read per ROW            1 read per PAGE             0 reads
  preserves ORDER BY        needs a Sort                preserves ORDER BY
```

**Diagram 2 — data flow: where the cost actually is**

```
  QUERY: WHERE user_id = 4471  →  17 rows

  ┌──────────────────────────────────────────────────────────────┐
  │ INDEX DESCENT      ████                          4 pages     │
  │ HEAP FETCHES       ████████████████             17 pages     │
  │ VISIBILITY         ▪                             (in the     │
  │                                                   same pages)│
  └──────────────────────────────────────────────────────────────┘
                       ↑ 81% of the cost is the heap fetch

  QUERY: WHERE user_id BETWEEN 1 AND 20000  →  300,000 rows

  ┌──────────────────────────────────────────────────────────────┐
  │ INDEX DESCENT      ██                          620 pages     │
  │ HEAP FETCHES       ████████████████████████ 28,400 pages     │
  │                    (and with an Index Scan, many of these    │
  │                     pages are read MULTIPLE times)           │
  └──────────────────────────────────────────────────────────────┘
                       ↑ 98% of the cost is the heap fetch

  ⇒ ELIMINATE THE HEAP FETCH AND YOU ELIMINATE THE QUERY'S COST.
    That is what a covering index does.
```

**Diagram 3 — before/after: the visibility map cliff**

```
 AFTER VACUUM — visibility map fully set
 ┌──────────────────────────────────────────────────────────────┐
 │ VM: [1][1][1][1][1][1][1][1][1][1][1][1][1][1][1][1]         │
 │ Index Only Scan   Heap Fetches: 0   Buffers: shared hit=12   │
 │ Execution Time: 0.31 ms                                      │
 └──────────────────────────────────────────────────────────────┘

 AFTER `UPDATE orders SET status='shipped' WHERE created_at > ...`
 (touching 4% of rows, spread across 60% of pages)
 ┌──────────────────────────────────────────────────────────────┐
 │ VM: [0][1][0][0][1][0][1][0][0][0][1][0][0][1][0][0]         │
 │ Index Only Scan   Heap Fetches: 184,220   Buffers: read=9822 │
 │ Execution Time: 412 ms                       ← 1,330× slower │
 └──────────────────────────────────────────────────────────────┘
                                     ↑
              SAME PLAN. SAME QUERY. SAME INDEX.
              The only difference is whether VACUUM has run.

 AFTER autovacuum catches up
 ┌──────────────────────────────────────────────────────────────┐
 │ VM: [1][1][1][1][1][1][1][1][1][1][1][1][1][1][1][1]         │
 │ Heap Fetches: 0    Execution Time: 0.33 ms                   │
 └──────────────────────────────────────────────────────────────┘

 ⇒ If you rely on index-only scans, you MUST tune autovacuum for
   that table. This is not optional. (Topic 47.)
```

---

## Example 1 — basic

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
CREATE INDEX idx_orders_user_id ON orders (user_id);
VACUUM ANALYZE orders;
```

**Step 1 — Index Scan, and where the cost is.**

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = 4471;
```
```
Index Scan using idx_orders_user_id on orders
    (actual time=0.048..0.121 rows=17 loops=1)
  Index Cond: (user_id = 4471)
  Buffers: shared hit=21
Execution Time: 0.146 ms
```
21 buffers: ~4 index + 17 heap. **The heap fetch is 81% of it.**

**Step 2 — the same index, more rows: the plan flips.**

```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id BETWEEN 1 AND 20000;
```
```
Bitmap Heap Scan on orders  (actual time=48.2..241.9 rows=300412 loops=1)
  Recheck Cond: ((user_id >= 1) AND (user_id <= 20000))
  Heap Blocks: exact=28402
  Buffers: shared hit=1204 read=27818
  ->  Bitmap Index Scan on idx_orders_user_id  (actual time=42.1..42.1 rows=300412)
        Index Cond: ((user_id >= 1) AND (user_id <= 20000))
        Buffers: shared hit=824
Execution Time: 256.4 ms
```

Force a plain Index Scan and compare:

```sql
SET enable_bitmapscan = off;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id BETWEEN 1 AND 20000;
```
```
Index Scan using idx_orders_user_id on orders (actual time=0.04..1841.2 rows=300412)
  Buffers: shared hit=48210 read=252891         ← 301,101 buffer requests!
Execution Time: 1852.7 ms
SET enable_bitmapscan = on;
```

**29,022 buffers vs 301,101. 256 ms vs 1,853 ms.** The Index Scan read the *same* heap pages over and over — once per matching row — in random order. The bitmap read each page once, in order.

**Step 3 — eliminate the heap fetch.**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, count(*) FROM orders WHERE user_id BETWEEN 1 AND 20000 GROUP BY user_id;
```
```
GroupAggregate  (actual time=0.041..118.2 rows=20000 loops=1)
  ->  Index Only Scan using idx_orders_user_id on orders
        Index Cond: ((user_id >= 1) AND (user_id <= 20000))
        Heap Fetches: 0                    ← ★
        Buffers: shared hit=824
Execution Time: 121.4 ms
```
**824 buffers instead of 29,022. 35× less I/O.** The heap was never opened.

**Step 4 — a covering index for the real query.**

```sql
-- The query needs total_paise and status, which aren't in the index.
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, total_paise, status FROM orders WHERE user_id BETWEEN 1 AND 5000;
```
```
Bitmap Heap Scan on orders  (actual time=12.4..84.1 rows=75104 loops=1)
  Buffers: shared hit=1204 read=17888
Execution Time: 89.2 ms
```
```sql
CREATE INDEX idx_orders_cover ON orders (user_id) INCLUDE (total_paise, status);
VACUUM ANALYZE orders;
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, total_paise, status FROM orders WHERE user_id BETWEEN 1 AND 5000;
```
```
Index Only Scan using idx_orders_cover on orders (actual time=0.038..18.4 rows=75104)
  Index Cond: ((user_id >= 1) AND (user_id <= 5000))
  Heap Fetches: 0
  Buffers: shared hit=412
Execution Time: 21.1 ms
```
**19,092 buffers → 412. 89 ms → 21 ms.**

**Step 5 — break it, to see the cliff.**

```sql
UPDATE orders SET status = 'shipped' WHERE id % 25 = 0;   -- 4% of rows
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, total_paise, status FROM orders WHERE user_id BETWEEN 1 AND 5000;
```
```
Index Only Scan using idx_orders_cover on orders (actual time=0.06..204.8 rows=75104)
  Heap Fetches: 71204                   ← ⚠ 95% of rows now need the heap
  Buffers: shared hit=4102 read=14882
Execution Time: 209.6 ms
```

**Updating 4% of rows made 95% of the index-only scan fall back to heap reads** — because those 4% were spread across nearly every page, and *one* modified tuple clears `ALL_VISIBLE` for the whole page.

```sql
VACUUM orders;
EXPLAIN (ANALYZE, BUFFERS)
SELECT user_id, total_paise, status FROM orders WHERE user_id BETWEEN 1 AND 5000;
-- Heap Fetches: 0      Buffers: shared hit=419      Execution Time: 21.8 ms
```

**Step 6 — see BitmapAnd combine two indexes.**

```sql
CREATE INDEX idx_orders_status ON orders (status);
CREATE INDEX idx_orders_total ON orders (total_paise);
VACUUM ANALYZE orders;
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM orders WHERE status = 'paid' AND total_paise BETWEEN 400000 AND 401000;
```
```
Bitmap Heap Scan on orders  (actual time=8.1..14.2 rows=1502 loops=1)
  Recheck Cond: ((total_paise >= 400000) AND (total_paise <= 401000)
                 AND (status = 'paid'))
  Heap Blocks: exact=1487
  ->  BitmapAnd  (actual time=7.9..7.9 rows=0 loops=1)
        ->  Bitmap Index Scan on idx_orders_total (actual rows=6011)
        ->  Bitmap Index Scan on idx_orders_status (actual rows=750128)
```
Note the asymmetry: the `status` bitmap scanned 750,128 entries to contribute almost nothing. A composite index on `(total_paise, status)` would do far better (Topic 14).

**Step 7 — make the bitmap go lossy.**

```sql
SET work_mem = '64kB';
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM orders WHERE user_id < 100000;
```
```
Bitmap Heap Scan on orders  (actual time=88.1..612.4 rows=1500842 loops=1)
  Recheck Cond: (user_id < 100000)
  Rows Removed by Index Recheck: 1499158      ← ⚠ the cost of lossy
  Heap Blocks: exact=1204 lossy=27288         ← ⚠
Execution Time: 641.2 ms
```
```sql
SET work_mem = '64MB';
-- Heap Blocks: exact=28492 lossy=0    Execution Time: 288.1 ms   (2.2× faster)
RESET work_mem;
```

---

## Example 2 — production scenario

**The situation.** A dashboard endpoint: "orders for this merchant in the last 30 days, with totals." It was 60 ms for a year. Over three weeks it drifts to 2.8 s. No deploy. No schema change. Traffic flat.

```sql
SELECT o.id, o.user_id, o.total_paise, o.status, o.created_at
FROM orders o
WHERE o.merchant_id = $1
  AND o.created_at >= now() - interval '30 days'
ORDER BY o.created_at DESC
LIMIT 200;
```

**Step 1 — get the plan.**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, user_id, total_paise, status, created_at FROM orders
WHERE merchant_id = 8812 AND created_at >= now() - interval '30 days'
ORDER BY created_at DESC LIMIT 200;
```
```
Limit  (actual time=2814.2..2814.6 rows=200 loops=1)
  ->  Sort  (actual time=2814.2..2814.4 rows=200 loops=1)
        Sort Key: created_at DESC
        Sort Method: top-N heapsort  Memory: 78kB
        ->  Bitmap Heap Scan on orders  (actual time=142.1..2701.8 rows=184220)
              Recheck Cond: (merchant_id = 8812)
              Filter: (created_at >= (now() - '30 days'::interval))
              Rows Removed by Filter: 1204882          ← ⚠⚠
              Heap Blocks: exact=142904
              Buffers: shared hit=8420 read=136891     ← 1.1 GB
              ->  Bitmap Index Scan on idx_orders_merchant (actual rows=1389102)
Execution Time: 2815.1 ms
```

**Read it forensically:**

- `Bitmap Index Scan ... rows=1389102` — the index matched **1.39 million** rows for this merchant. They grew.
- `Rows Removed by Filter: 1204882` — the date predicate was applied **after** the heap fetch. Those 1.2M rows were read from disk and then thrown away.
- `Heap Blocks: exact=142904` — 1.1 GB of heap read.
- `Sort` — a **blocking** node. `LIMIT 200` saved nothing; all 184,220 surviving rows had to be sorted first.
- `read=136891` — nearly all cold, and this evicts the API's working set (Topic 07).

**Step 2 — why it drifted.** A year ago this merchant had 40,000 orders and the 30-day window was most of them. Now they have 1.39 million and the window is 13%. The plan never changed; the *data* did. The index on `(merchant_id)` alone cannot express "and only recent ones."

**Step 3 — the fix, one decision at a time.**

```sql
-- ATTEMPT: put the date in the index
CREATE INDEX idx_orders_merchant_created ON orders (merchant_id, created_at DESC);
VACUUM ANALYZE orders;
```
```
Limit  (actual time=0.058..2.104 rows=200 loops=1)
  ->  Index Scan using idx_orders_merchant_created on orders
        (actual time=0.056..2.081 rows=200 loops=1)
        Index Cond: ((merchant_id = 8812) AND (created_at >= ...))
        Buffers: shared hit=204 read=8
Execution Time: 2.14 ms
```

**2,815 ms → 2.14 ms. 1,315×.** Three separate things happened:

| Change | Mechanism |
|---|---|
| `Rows Removed by Filter` → 0 | `created_at` moved from `Filter` to `Index Cond` — evaluated during the descent, so those 1.2M rows were never fetched |
| `Sort` node disappeared | the index is already in `(merchant_id, created_at DESC)` order, so rows arrive sorted |
| `LIMIT 200` now stops early | with no blocking Sort, the pipeline streams and halts after 200 rows |

**Step 4 — go further with a covering index.**

```sql
DROP INDEX idx_orders_merchant_created;
CREATE INDEX idx_orders_merchant_created ON orders (merchant_id, created_at DESC)
  INCLUDE (id, user_id, total_paise, status);
VACUUM ANALYZE orders;
```
```
Limit  (actual time=0.041..0.301 rows=200 loops=1)
  ->  Index Only Scan using idx_orders_merchant_created on orders
        Index Cond: ((merchant_id = 8812) AND (created_at >= ...))
        Heap Fetches: 0
        Buffers: shared hit=9
Execution Time: 0.322 ms
```
**212 buffers → 9. 2.14 ms → 0.32 ms.**

**Step 5 — but check the cost you just took on.**

```sql
SELECT pg_size_pretty(pg_relation_size('idx_orders_merchant_created'));
```
```
 pg_size_pretty
----------------
 1842 MB           ← vs 412 MB for the non-covering version
```

**The trade, stated honestly:**

| | Non-covering | Covering |
|---|---|---|
| Query time | 2.14 ms | 0.32 ms |
| Index size | 412 MB | 1,842 MB |
| Buffer pool consumed | 412 MB | 1,842 MB |
| Write amplification | +1 entry/insert | +1 entry/insert, **4.5× wider** |
| Depends on VACUUM? | no | **yes** — `Heap Fetches` degrades without it |

**Is 1.8 ms worth 1.4 GB of RAM and a VACUUM dependency?** At 20,000 calls/sec on a latency-critical path: yes. At 20 calls/sec on a dashboard: no — take the 2.14 ms version.

**Step 6 — make the VACUUM dependency explicit.**

```sql
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.02,
  autovacuum_vacuum_insert_scale_factor = 0.05,   -- PG13+: vacuum on INSERTs too,
  autovacuum_vacuum_cost_delay = 2                -- which is what sets the VM
);
```

```sql
-- Monitor it. If this climbs, your covering index has quietly stopped working.
SELECT indexrelname, idx_scan,
       (SELECT n_dead_tup FROM pg_stat_user_tables t WHERE t.relid = s.relid) AS dead
FROM pg_stat_user_indexes s WHERE s.relname = 'orders';
```

**The prevention rule:** `EXPLAIN (ANALYZE)` on the top 20 queries, **monthly**, comparing `Rows Removed by Filter` and `Heap Fetches` over time. Both are leading indicators that degrade silently while the plan looks unchanged.

---

## Common mistakes

**1. Reading `Filter` as if it were `Index Cond`.**
- *Symptom:* "the index is being used" but the query is slow.
- *Engine-level why:* `Index Cond` reduces rows read during the descent. `Filter` runs *after* the heap fetch — those rows were already paid for. `Rows Removed by Filter: 1204882` means 1.2M rows were read from disk and discarded.
- *Diagnose:* look for `Rows Removed by Filter` > a few thousand.
- *Fix:* add that column to the index, in the right position (Topic 14).

**2. Assuming an index scan is always better than a seq scan.**
- *Symptom:* forced index usage is slower than the plan the planner wanted.
- *Engine-level why:* an Index Scan does one random heap read *per row*, and revisits pages. Above ~5–10% selectivity, sequential wins.
- *Fix:* trust the planner; if it's wrong, fix the statistics (Topic 15) or `random_page_cost` (Topic 07), not the plan.

**3. Expecting an index-only scan without VACUUM.**
- *Symptom:* `Heap Fetches: 184220` on a plan labelled `Index Only Scan`.
- *Engine-level why:* the visibility map is set only by VACUUM and cleared by any write. `Index Only Scan` is a plan *choice*; whether it actually avoids the heap is a *runtime* outcome.
- *Diagnose:* `Heap Fetches:` in `EXPLAIN ANALYZE`.
- *Fix:* aggressive autovacuum on that table, including `autovacuum_vacuum_insert_scale_factor` for append-only tables.

**4. Adding `INCLUDE` columns without pricing them.**
- *Symptom:* index 4× bigger; writes slower; buffer pool pressure; other queries get slower.
- *Engine-level why:* every included column is stored in every leaf entry, written on every insert, and occupies buffer pool that your table wanted.
- *Fix:* include only what the *hot* query needs. Measure the size before and after.

**5. Ignoring `lossy` in `Heap Blocks`.**
- *Symptom:* a bitmap scan slower than expected; large `Rows Removed by Index Recheck`.
- *Engine-level why:* the TID bitmap exceeded `work_mem` and degraded to page granularity, so every tuple on those pages must be rechecked.
- *Fix:* raise `work_mem` for that query (`SET LOCAL`), or make the predicate more selective.

**6. Expecting `ORDER BY` to be satisfied by a bitmap scan.**
- *Symptom:* an explicit `Sort` node appears despite a "matching" index.
- *Engine-level why:* bitmap scans emit rows in *physical* order by construction. Only a plain Index Scan preserves key order.
- *Fix:* if `ORDER BY` + `LIMIT` matters, you want an Index Scan — which means high selectivity or an index whose leading columns match both the filter and the sort.

**7. Forgetting that `SELECT *` kills index-only scans.**
- *Symptom:* the covering index works for one query and not the one next to it.
- *Engine-level why:* index-only requires *every* referenced column to be in the index. One extra column in the `SELECT` list forces the heap fetch.
- *Fix:* list the columns you need. This is the one place where `SELECT *` genuinely costs you (contrast Topic 04).

---

## Hands-on proof

**PROVE IT #1 — index scan cost is dominated by the heap.**
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id = 4471;
EXPLAIN (ANALYZE, BUFFERS) SELECT user_id FROM orders WHERE user_id = 4471;
```
Second is an Index Only Scan: ~4 buffers vs ~21.

**PROVE IT #2 — index scan vs bitmap, forced.**
```sql
SET enable_bitmapscan = off;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id BETWEEN 1 AND 20000;
SET enable_bitmapscan = on;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM orders WHERE user_id BETWEEN 1 AND 20000;
```

**PROVE IT #3 — the visibility map, directly.**
```sql
CREATE EXTENSION IF NOT EXISTS pg_visibility;
SELECT count(*) FILTER (WHERE all_visible) AS visible_pages,
       count(*) AS total_pages,
       round(100.0*count(*) FILTER (WHERE all_visible)/count(*),1) AS pct
FROM pg_visibility_map('orders');
```
```
 visible_pages | total_pages |  pct
---------------+-------------+-------
         28402 |       28402 | 100.0
```
```sql
UPDATE orders SET status='x' WHERE id % 100 = 0;    -- 1% of rows
SELECT round(100.0*count(*) FILTER (WHERE all_visible)/count(*),1) AS pct
FROM pg_visibility_map('orders');
-- 4.2      ← 1% of rows cleared 96% of the visibility map
VACUUM orders;
-- back to 100.0
```
**This single experiment explains more production mysteries than almost any other.**

**PROVE IT #4 — INCLUDE vs composite, in fanout.**
```sql
CREATE INDEX ix_comp ON orders (user_id, total_paise, status);
CREATE INDEX ix_incl ON orders (user_id) INCLUDE (total_paise, status);
SELECT c.relname, i.tree_level, pg_size_pretty(i.index_size::bigint)
FROM pg_class c, LATERAL pgstatindex(c.oid) i
WHERE c.relname IN ('ix_comp','ix_incl');
```
The `INCLUDE` version has narrower internal nodes → higher fanout → often one level shallower.

**PROVE IT #5 — `Index Cond` vs `Filter`.**
```sql
CREATE INDEX ix_u ON orders(user_id);
EXPLAIN (ANALYZE) SELECT * FROM orders
WHERE user_id BETWEEN 1 AND 5000 AND status='paid';
-- Index Cond: user_id ...        Filter: status='paid'
--   Rows Removed by Filter: ~56000    ← paid for, then thrown away

CREATE INDEX ix_us ON orders(user_id, status);
EXPLAIN (ANALYZE) SELECT * FROM orders
WHERE user_id BETWEEN 1 AND 5000 AND status='paid';
-- Index Cond: ((user_id >= 1) AND (user_id <= 5000) AND (status = 'paid'))
--   Rows Removed by Filter: 0
```

**PROVE IT #6 — lossy bitmaps.**
```sql
SET work_mem='64kB';
EXPLAIN (ANALYZE) SELECT count(*) FROM orders WHERE user_id < 100000;
-- Heap Blocks: exact=... lossy=...
SET work_mem='64MB';
EXPLAIN (ANALYZE) SELECT count(*) FROM orders WHERE user_id < 100000;
-- Heap Blocks: exact=... lossy=0
RESET work_mem;
```

---

## The design decision framework

```
AIM FOR AN INDEX ONLY SCAN WHEN:
  ✓ The query is hot (thousands of calls/sec)
  ✓ It needs few columns, and they're small
  ✓ The table is not update-heavy (or you can tune autovacuum for it)
  ✓ You can afford the index size and the write amplification
  → CREATE INDEX ... (filter_cols) INCLUDE (output_cols)
  ⚠ AND set aggressive autovacuum on that table. Non-negotiable.

ACCEPT AN INDEX SCAN WHEN:
  ✓ Selectivity is high (< ~1–2%)
  ✓ ORDER BY + LIMIT — only this scan type preserves order and streams
  ✓ The covering index would be too large to justify

ACCEPT A BITMAP HEAP SCAN WHEN:
  ✓ Medium selectivity (~1–20%)
  ✓ Combining multiple independent predicates (BitmapAnd/Or)
  ✓ Order doesn't matter
  ⚠ ensure work_mem is large enough that it doesn't go lossy

ACCEPT A SEQ SCAN WHEN:
  ✓ > ~10% of rows match
  ✓ The table is small
  ✓ You're aggregating most of the table anyway
  → and DON'T create the index. It would be pure write tax.

THE DECISION PROCEDURE — run this on any slow indexed query:
  1. EXPLAIN (ANALYZE, BUFFERS)
  2. Is there a `Filter` with a large `Rows Removed by Filter`?
       → that column belongs in the index. Biggest single win.
  3. Is there a `Sort` under a `Limit`?
       → an index matching the ORDER BY removes it AND enables early stop.
  4. Is `Heap Fetches` > 0 on an Index Only Scan?
       → autovacuum problem, not an index problem.
  5. Is `Heap Blocks` showing `lossy`?
       → raise work_mem.
  6. Is `Buffers: read` ≫ `hit`?
       → covering index, or narrower rows, or more RAM (Topic 07).

THE SIGNAL TO LOOK FOR:
      ratio = (rows returned) / (buffers read)
  • > 5 rows per buffer   → good. You're reading pages densely.
  • < 0.5 rows per buffer → you are reading ~2+ pages per row. Either
    the heap fetch is random and scattered (→ covering index) or the
    filter is post-fetch (→ move it into the index).
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
On the 3M-row `orders` table, write three queries that produce an Index Scan, a Bitmap Heap Scan, and an Index Only Scan respectively — using the same index. For each, record buffers read and execution time, and explain in one sentence why the planner chose it.

### Exercise 2 — medium (apply it)
Take the query `SELECT id, total_paise FROM orders WHERE user_id = $1 AND status = 'paid' ORDER BY created_at DESC LIMIT 50`.
(a) Design the single best index for it. Justify the column order and what belongs in `INCLUDE`.
(b) Measure before/after: buffers, time, `Rows Removed by Filter`, presence of a `Sort` node.
(c) Update 2% of the table, re-run, and report `Heap Fetches`. Explain the number.
(d) State the index's size and decide, with a stated call rate, whether the covering version is worth it.

### Exercise 3 — hard (production simulation)
A reporting endpoint runs 4,000 times/sec:

```sql
SELECT o.merchant_id, o.status, count(*), sum(o.total_paise)
FROM orders o
WHERE o.merchant_id = ANY($1)          -- 1–50 merchant ids
  AND o.created_at >= $2 AND o.created_at < $3
GROUP BY o.merchant_id, o.status;
```

`orders` is 900M rows, 340 GB. Current plan: Bitmap Heap Scan reading 40 GB, 8.2 s, `Heap Blocks: exact=2100 lossy=4800000`, `Rows Removed by Index Recheck: 41M`.

(a) Explain every element of that plan output. What specifically went wrong, and in what order?
(b) Design the index. Justify column order against the predicate shapes (`= ANY`, range, group-by).
(c) Compute the expected index size and state whether it fits in a 64 GB buffer pool alongside everything else.
(d) The table receives 40k inserts/sec and updates to `status` on ~30% of rows within 24 hours. Explain what this does to your index-only scan, quantify it, and give the mitigation.
(e) Propose an alternative design that avoids the index entirely, and state the conditions under which it wins.
(f) Write the two monitoring queries that would alert you *before* this regressed, and explain what each would have shown three weeks early.

---

## Mental model checkpoint

1. Name the three phases of an index lookup. Which is usually the most expensive, and why?
2. Why does an index scan returning 300,000 rows read some heap pages more than once? What plan fixes that, and how?
3. What is the visibility map, who sets it, who clears it, and why does an index-only scan depend on it?
4. Explain the difference between `Index Cond` and `Filter` in `EXPLAIN`. Which one saved you I/O?
5. What does `INCLUDE` do that a composite index doesn't? Name two advantages.
6. Why does a bitmap heap scan lose `ORDER BY`? What's the consequence for `LIMIT`?
7. `Heap Blocks: exact=1204 lossy=88900` — what happened, what does it cost, and what's the fix?

---

## Quick reference card

| Scan type | Heap reads | Order preserved | Best for |
|---|---|---|---|
| Seq Scan | all pages, sequential | no | > 10% of rows |
| Index Scan | 1 per **row**, random | **yes** | < 2%; ORDER BY + LIMIT |
| Bitmap Heap Scan | 1 per **page**, in page order | no | 1–20%; combining indexes |
| Index Only Scan | **0** (if VM set) | yes | hot queries, few columns |

**EXPLAIN fields that matter**

| Field | Means |
|---|---|
| `Index Cond` | evaluated during descent — **saved I/O** |
| `Filter` + `Rows Removed by Filter` | evaluated after fetch — **wasted I/O** |
| `Heap Fetches: N` | index-only scan fell back N times → VACUUM |
| `Heap Blocks: exact / lossy` | lossy > 0 → raise `work_mem` |
| `Rows Removed by Index Recheck` | the cost of a lossy bitmap |
| `Buffers: shared hit / read` | cache hits vs disk reads |

**Numbers**

| Thing | Value |
|---|---|
| Visibility map size | 2 bits/page ≈ 1 byte per 32 KB of table |
| Index scan break-even | ~1–2% of rows |
| Bitmap scan range | ~1–20% |
| Random vs sequential page cost | ~4× (`random_page_cost` 4.0; use **1.1** on SSD) |
| Covering index size penalty | roughly proportional to included column width |

**The three moves, in order of value**
1. Move `Filter` predicates into the index → stop fetching rows you'll discard.
2. Match the index order to `ORDER BY` → delete the Sort, enable early `LIMIT` stop.
3. `INCLUDE` the output columns → delete the heap fetch. *And tune autovacuum.*

---

## When would I use this at work?

1. **Any "I added an index and it's still slow" ticket.** You look for `Rows Removed by Filter` and a `Sort` under a `Limit`. Nine times out of ten one of those two is the answer, and the fix is a column order change rather than a new index.

2. **A p99 that degrades over weeks with no deploy.** `Heap Fetches` climbing on an index-only scan is a silent regression that no plan diff would show. Knowing to look at it — and that the fix is autovacuum, not indexing — saves days.

3. **Deciding whether a covering index is worth it.** You can price it precisely: index size, buffer pool consumed, write amplification, and the VACUUM dependency, against the measured latency win and the call rate. That's a design-doc paragraph rather than an argument.

---

## Connected topics

**Understand before this:** 04 (TIDs, pages, visibility map fork), 07 (random vs sequential I/O), 11 (the B-tree descent), 46 (MVCC — why visibility must be checked at all).

**This unlocks:**
- **13** — partial, expression, unique, and covering indexes as variants
- **14** — composite column order: the mechanism behind `Index Cond` vs `Filter`
- **15** — selectivity: how the planner picks between these scan types
- **17** — the write cost of the covering indexes this topic recommends
- **18** — the cost model that produces these choices
- **47** — VACUUM and the visibility map, in full
