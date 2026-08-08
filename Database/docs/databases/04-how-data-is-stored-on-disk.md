# 04 — How Data Is Stored on Disk
## Phase: Storage Internals

---

## ELI5 — The Simple Analogy

A hospital filing room.

Records are never stored as loose sheets. They go into **standard-size folders**, and every folder is exactly the same thickness so they slot into the shelves predictably. Each folder has a **cover sheet** listing what's inside and where in the folder each document sits — so you can find "the X-ray" without flipping through everything. New documents go in from the **back** of the folder; the cover-sheet index grows from the **front**. When they meet, the folder is full and you start a new one.

Crucially: **the clerk never fetches one sheet. He fetches the whole folder.** Even if you only want one line from one form, a whole folder comes to your desk. That is the single most important fact about database storage, and it explains index design, row width, caching, and half of query performance.

The folder is a **page**. It is 8 kilobytes. Always.

---

## Where this fits in the big picture

```
   01 What is a DB ──▶ 02 Engine architecture
                              │
                              ▼
                    ┌─────────────────────┐
                    │ 04 PAGES ON DISK    │ ← YOU ARE HERE
                    │ (the folder)        │
                    └──────────┬──────────┘
                               │
              ┌────────────────┼────────────────┐
              ▼                ▼                ▼
      05 Tuple layout   07 Buffer pool   11 B-tree pages
      (one sheet)       (folders in RAM) (index folders)
                               │
                               ▼
                    Everything in Phase 2 is
                    "how do I touch fewer folders?"
```

Every performance topic from here on reduces to one number: **how many 8KB pages did we have to read?**

---

## What is this?

A table is stored in a **heap file** — a plain file on disk, divided into fixed-size **pages** (also called blocks) of 8,192 bytes. Rows (**tuples**) are packed into pages. All I/O between disk and memory happens in whole pages.

"Heap" means *unordered*: rows sit wherever there was space, not in any sorted order. That is why finding a row without an index means reading every page.

---

## Why does it matter for a backend developer?

Because page count is the currency of database performance, and row design controls page count.

- A table with 200-byte rows fits ~40 rows per page. The same table with 60-byte rows fits ~130. **The narrow table is 3× faster to scan and takes 3× less RAM to cache** — same data, same queries, just different column choices.
- `SELECT id FROM orders` and `SELECT * FROM orders` read **exactly the same number of pages** on a heap scan. The row is fetched whole regardless. This surprises people constantly.
- A `DELETE` frees no disk space at all. The page keeps the dead bytes until VACUUM comes. Your 400MB table can occupy 4GB.
- An `UPDATE` of one boolean column writes a whole new copy of the row — potentially onto a different page, and re-writes every index entry.

If you don't know how many pages your query touches, you cannot reason about its cost. `EXPLAIN (ANALYZE, BUFFERS)` tells you, and this topic teaches you to read it.

---

## The physical reality

### The file layout

```
base/16388/16390        ← the heap file for `orders`
│
├── page 0   bytes      0 –  8191
├── page 1   bytes   8192 – 16383
├── page 2   bytes  16384 – 24575
│   ...
└── page N

⚠ A single file is capped at 1 GB. Beyond that PostgreSQL creates SEGMENTS:
    16390        first 1 GB
    16390.1      second 1 GB
    16390.2      third 1 GB
   A 12 GB table is 12 files. This is transparent to you, but explains why
   `ls` shows a table "smaller" than pg_relation_size says.

Companion forks — same relfilenode, different suffix:
    16390_fsm    Free Space Map      — a tree of per-page free-space estimates
    16390_vm     Visibility Map      — 2 bits per page (all-visible, all-frozen)
    16390_init   only for UNLOGGED tables
```

### Inside one 8KB page — the exact layout

```
byte 0                                                              byte 8191
┌────────────────────────────────────────────────────────────────────────┐
│ PageHeaderData — 24 bytes                                              │
│   pd_lsn        (8)  LSN of the last WAL record touching this page     │
│   pd_checksum   (2)  page checksum (if data checksums enabled)         │
│   pd_flags      (2)  has free line pointers? all visible?              │
│   pd_lower      (2)  offset to END of the item-pointer array   ──┐     │
│   pd_upper      (2)  offset to START of tuple data             ──┼──┐  │
│   pd_special    (2)  offset to special space (indexes use this)  │  │  │
│   pd_pagesize_version (2)                                        │  │  │
│   pd_prune_xid  (4)  hint: is there prunable garbage here?       │  │  │
├──────────────────────────────────────────────────────────────────┼──┼──┤
│ ItemIdData ARRAY — 4 bytes per line pointer, grows DOWNWARD ─────▶│  │  │
│   [1] off=8100 len=88  flags=NORMAL                              │  │  │
│   [2] off=8012 len=88  flags=NORMAL                              │  │  │
│   [3] off=0    len=0   flags=DEAD      ← a deleted row's slot    │  │  │
│   [4] off=7920 len=92  flags=NORMAL                              │  │  │
│◀── pd_lower ─────────────────────────────────────────────────────┘  │  │
├─────────────────────────────────────────────────────────────────────┼──┤
│                                                                     │  │
│                    F R E E   S P A C E                              │  │
│              (this is what the FSM tracks)                          │  │
│                                                                     │  │
│◀── pd_upper ────────────────────────────────────────────────────────┘  │
├────────────────────────────────────────────────────────────────────────┤
│ TUPLE 4   (92 bytes)                                                   │
│ TUPLE 2   (88 bytes)          tuples grow UPWARD from the bottom       │
│ TUPLE 1   (88 bytes)                                                   │
├────────────────────────────────────────────────────────────────────────┤
│ SPECIAL SPACE — 0 bytes for heap pages;                                │
│                 16 bytes for b-tree pages (sibling pointers) — Topic 11│
└────────────────────────────────────────────────────────────────────────┘
```

### Why two arrays growing toward each other?

Because a row must be **movable within its page without changing its address**.

Your indexes point at rows using a **TID** — `(block_number, item_offset)`, e.g. `(1204, 3)`. That is "page 1204, line pointer 3." The line pointer, not the byte offset. So when VACUUM compacts a page and physically slides tuples around to close gaps, it updates the line pointers, and **every index entry pointing at that row remains valid**. Without this indirection, defragmenting a page would require rewriting every index.

```
BEFORE PAGE COMPACTION              AFTER PAGE COMPACTION
line ptr [3] → byte 7920            line ptr [3] → byte 8020   ← pointer updated
index entry  → TID (1204, 3)        index entry  → TID (1204, 3)  ← UNCHANGED
```

### Why exactly 8KB?

```
Too small (e.g. 512B)                Too large (e.g. 64KB)
 ✗ header overhead per page is        ✗ read one 90-byte row → pull 64KB
   a large % of the page              ✗ one changed row dirties 64KB → 64KB
 ✗ more pages → bigger FSM,             of WAL full-page-image on first
   bigger index fanout cost             write after a checkpoint
 ✗ fewer rows per page → more         ✗ worse buffer pool granularity:
   I/Os per scan                        caching 64KB to use 90 bytes

8 KB sits at the balance point:
  • ≥ typical row (so most rows fit; oversized ones TOAST out — Topic 05)
  • a multiple of the 4KB OS page and typical filesystem block
  • fits comfortably in a single SSD read
  • keeps b-tree fanout high (~250–400 keys per internal page → 3–4 levels
    is enough for hundreds of millions of rows — Topic 11)

It is a COMPILE-TIME constant (`--with-blocksize`). You cannot change it on a
running cluster. Treat 8192 as a law of physics.
```

### Extents and segments — the allocation vocabulary

Different engines use different words for the same idea. Know all three so you can read any documentation:

```
          PostgreSQL              Oracle / SQL Server / InnoDB
          ──────────              ────────────────────────────
 smallest  PAGE (8 KB)             PAGE / BLOCK (8–16 KB)
 group of  (grown one page at a    EXTENT — a contiguous run of pages
           time, or 8 at a time     (e.g. 64 pages = 1 MB) allocated at once
           under contention)        to reduce fragmentation
 file      SEGMENT (1 GB chunk     SEGMENT / TABLESPACE FILE
           of one relation fork)
```

PostgreSQL keeps this simple: it extends a relation by appending pages to the end of the file. Under heavy concurrent insert it grows in batches to avoid a lock convoy on the extension operation.

---

## How it works — step by step

### Trace A: `INSERT INTO orders (user_id, total_paise) VALUES (7, 249900);`

```
 1. Tuple assembled in the backend's private memory: header + column bytes.
    Say it comes to 88 bytes (Topic 05 explains the byte-by-byte breakdown).

 2. FSM CONSULTED: "which page of relation 16390 has ≥ 92 bytes free?"
    (88 for the tuple + 4 for its line pointer.)
    READS: 16390_fsm  — a small 3-level tree, almost always cached.
    ANSWER: page 1204.

 3. BUFFER MANAGER: pin page 1204.
    HIT  → pointer returned in ~100 ns.
    MISS → evict a victim buffer, read 8192 bytes from 16390 at offset
           1204 × 8192 = 9,863,168.  ~100 µs on SSD.

 4. EXCLUSIVE PAGE LATCH taken (a lightweight lock, microseconds — NOT the
    heavyweight row lock; that's a different mechanism, Topic 45).

 5. WAL RECORD built and appended to the WAL buffer:
       xl_info = XLOG_HEAP_INSERT
       target  = rel 16390, block 1204, offset 41
       payload = the 88 tuple bytes
    ⚠ If this is the FIRST write to page 1204 since the last checkpoint,
      the ENTIRE 8KB page is written to WAL (a "full page image") to protect
      against torn writes. This is why WAL volume spikes right after a
      checkpoint. (Topic 41.)

 6. PAGE MODIFIED IN MEMORY:
       pd_upper  -= 88        → tuple bytes copied to the new pd_upper
       new ItemIdData written at pd_lower → (off=pd_upper, len=88, NORMAL)
       pd_lower  += 4
       pd_lsn     = the LSN of the WAL record from step 5
    Buffer marked DIRTY. The file on disk is still unchanged.

 7. LATCH RELEASED. FSM updated with the page's new free space (lazily).

 8. COMMIT → WAL fsync'd. Durable. (The heap page is still only in RAM.)

 9. LATER: checkpointer or bgwriter writes the 8KB page back to 16390.
```

**Before / after `pd_lower` and `pd_upper`:**

```
BEFORE                                AFTER
pd_lower = 184  (40 line pointers)    pd_lower = 188  (41 line pointers)
pd_upper = 4600                       pd_upper = 4512
free     = 4600 - 184 = 4416 bytes    free     = 4512 - 188 = 4324 bytes
                                                 (4416 - 92 = 4324 ✓)
```

### Trace B: `DELETE FROM orders WHERE id = 91;`

```
 1. Locate the tuple (index scan → TID (1204, 3)).
 2. WAL record: XLOG_HEAP_DELETE.
 3. The tuple's header field `t_xmax` is set to the deleting transaction's XID.
    *** THE BYTES ARE NOT REMOVED. NOTHING IS FREED. ***
    The row is now invisible to transactions starting after this one commits,
    but it still occupies its 88 bytes and its line pointer.
 4. Commit.
 5. Table size on disk: UNCHANGED.
 6. LATER, VACUUM:
       - marks the line pointer DEAD, then (after index cleanup) UNUSED
       - reclaims the 88 bytes into the page's free space
       - updates the FSM so a future INSERT can use that space
    The FILE still does not shrink — the space is reusable, not returned to
    the OS. Only VACUUM FULL / CLUSTER rewrites the file smaller, and that
    takes an ACCESS EXCLUSIVE lock. (Topic 47.)
```

### Trace C: `UPDATE orders SET status = 'shipped' WHERE id = 91;`

```
   An UPDATE is a DELETE + an INSERT. There is no in-place update.

   1. Old tuple at (1204,3): t_xmax = current XID.
   2. NEW tuple written — on page 1204 if there's room, otherwise on a
      different page entirely.
   3. Index entries: if the new tuple is on a DIFFERENT page, or if any
      indexed column changed, EVERY index gets a new entry pointing at the
      new TID.
      If the new tuple fits on the SAME page and NO indexed column changed,
      PostgreSQL uses a HOT update (Heap-Only Tuple): the old line pointer
      is redirected to the new tuple, and NO index is touched. This is a
      huge optimisation — and it's why leaving `fillfactor` headroom on
      hot tables matters. (Topics 17, 46, 47.)

   ⇒ Updating one boolean rewrites the ENTIRE row, plus possibly every index.
     This is the storage-level reason "wide tables with hot columns" is an
     antipattern (Topic 27).
```

---

## Concept breakdown

```
HEAP FILE
│  │
│  └── File: an ordinary OS file (plus 1GB segments), named by relfilenode
└── Heap: UNORDERED. Rows live wherever space was found. There is no
     "natural order" — a SELECT without ORDER BY may return rows in any
     order, and that order can change after an UPDATE. (Never rely on it.)

PAGE  (= block)
│
├── The unit of ALL I/O — disk↔buffer pool, and WAL full-page images
├── Fixed 8192 bytes, compile-time constant
└── Structure: header(24) │ line pointers(4 each, grow down) │ free │ tuples(grow up)

TID / CTID — Tuple Identifier
│
├── (block_number, item_offset)  e.g. (1204, 3)
├── The physical address of a row. Every index entry stores one.
└── NOT STABLE: an UPDATE changes a row's ctid. Never store a ctid.

LINE POINTER (ItemIdData) — 4 bytes
│
├── (offset, length, flags)
├── The indirection that lets VACUUM defragment a page without touching indexes
└── flags: NORMAL · REDIRECT (HOT chain) · DEAD · UNUSED

FSM — Free Space Map (16390_fsm)
│
├── A 3-level tree; one byte per heap page holding an approximate free-space
│   category (granularity ~32 bytes)
└── Answers "where can I put a 92-byte tuple?" without scanning the table

VM — Visibility Map (16390_vm)
│
├── TWO BITS per heap page: ALL_VISIBLE, ALL_FROZEN
├── ALL_VISIBLE lets an INDEX-ONLY SCAN skip the heap fetch entirely (Topic 12)
└── ALL_FROZEN lets VACUUM skip the page completely

FILLFACTOR
│
└── % of a page INSERT may fill (default 100 for heaps, 90 for b-trees).
    Set to 80–90 on update-heavy tables so HOT updates can stay on-page.
```

---

## Diagrams

**Diagram 1 — big picture: the containment hierarchy**

```
 DATABASE  shop
    │
    └── TABLE  orders  (relfilenode 16390)
          │
          ├── MAIN FORK      base/16388/16390      ← rows
          │     ├── segment 16390     pages 0 … 131071      (1 GB)
          │     ├── segment 16390.1   pages 131072 …        (1 GB)
          │     │
          │     └── each PAGE = 8192 bytes
          │           ├── header (24 B)
          │           ├── line pointers (4 B each)
          │           ├── free space
          │           └── tuples
          │
          ├── FSM FORK       base/16388/16390_fsm  ← free space per page
          ├── VM  FORK       base/16388/16390_vm   ← 2 bits per page
          └── TOAST TABLE    base/16388/16395      ← oversized values (Topic 05)
```

**Diagram 2 — data flow: a sequential scan vs an index scan, in pages**

```
 SEQUENTIAL SCAN  (no index)
 ─────────────────────────────────────────────────────────────
  executor ──"give me page 0"──▶ buffer mgr ──▶ disk
           ◀── 8KB, ~126 tuples ──
           filter all 126, keep 0
  ...repeat for pages 1 … 12,470...
  TOTAL: 12,471 page reads ≈ 97 MB of I/O to find 17 rows.


 INDEX SCAN
 ─────────────────────────────────────────────────────────────
  executor ──▶ b-tree root page        [1 read]
            ──▶ internal page          [1 read]
            ──▶ leaf page → TIDs:      [1 read]
                (1204,3) (2891,17) (4102,5) ... 17 of them
            ──▶ heap page 1204         [1 read]  ← one page per matching row
            ──▶ heap page 2891         [1 read]
                ...
  TOTAL: ~20 page reads ≈ 160 KB.

  ⚠ Note the shape: index scan does RANDOM reads. If 40% of the table
    matched, 5,000 random reads beats nothing — a sequential scan wins,
    because sequential I/O is far cheaper per page. This is why the planner
    sometimes correctly ignores your index. (Topics 15, 17, 18.)
```

**Diagram 3 — before / after a DELETE and a VACUUM**

```
 STATE 1: page 1204 with 3 live rows
 ┌─────────────────────────────────────────────────┐
 │ hdr │ [1]→8100 [2]→8012 [3]→7920 │  free 4300B  │
 │                        │ T3 │ T2 │ T1 │         │
 └─────────────────────────────────────────────────┘
   size on disk: 8192 B · live: 3 · dead: 0

 STATE 2: after DELETE of row 2 (COMMITTED)
 ┌─────────────────────────────────────────────────┐
 │ hdr │ [1]→8100 [2]→8012 [3]→7920 │  free 4300B  │
 │                        │ T3 │ T2 │ T1 │         │
 └─────────────────────────────────────────────────┘
              ↑ T2 still physically present, t_xmax = 5001
   size on disk: 8192 B · live: 2 · dead: 1 · FREE SPACE UNCHANGED
   ⇒ "I deleted 10 million rows and the table didn't shrink" — this is why.

 STATE 3: after VACUUM
 ┌─────────────────────────────────────────────────┐
 │ hdr │ [1]→8104 [2]=UNUSED [3]→8016 │ free 4388B │
 │                             │ T3 │ T1 │         │
 └─────────────────────────────────────────────────┘
   size on disk: 8192 B · live: 2 · dead: 0 · free space RECLAIMED for reuse
   ⇒ file still 8192 B. Space is reusable, not returned to the OS.

 STATE 4: after VACUUM FULL (rewrites the whole relation; ACCESS EXCLUSIVE lock)
   file physically smaller. Do not run this on a live production table
   without a maintenance window. (Topic 47.)
```

---

## Example 1 — basic

Watch a table grow page by page, and prove the numbers.

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

CREATE TABLE orders (
  id          bigserial PRIMARY KEY,
  user_id     bigint      NOT NULL,
  total_paise bigint      NOT NULL,
  status      text        NOT NULL DEFAULT 'pending',
  created_at  timestamptz NOT NULL DEFAULT now()
);

INSERT INTO orders (user_id, total_paise)
SELECT i, i * 100 FROM generate_series(1, 1000) i;

SELECT relpages, reltuples, pg_size_pretty(pg_relation_size('orders')) FROM pg_class
WHERE relname = 'orders';
```
```
 relpages | reltuples | pg_size_pretty
----------+-----------+----------------
       10 |      1000 | 80 kB
```
1,000 rows in 10 pages = **100 rows per page**. Each row therefore occupies about `8192 / 100 ≈ 82` bytes including its line pointer. (Topic 05 accounts for every one of those bytes.)

Now look inside page 0:

```sql
SELECT lp, lp_off, lp_len, lp_flags, t_xmin, t_xmax, t_ctid
FROM heap_page_items(get_raw_page('orders', 0)) LIMIT 5;
```
```
 lp | lp_off | lp_len | lp_flags | t_xmin | t_xmax |  t_ctid
----+--------+--------+----------+--------+--------+---------
  1 |   8112 |     73 |        1 |    892 |      0 | (0,1)
  2 |   8032 |     73 |        1 |    892 |      0 | (0,2)
  3 |   7952 |     73 |        1 |    892 |      0 | (0,3)
  4 |   7872 |     73 |        1 |    892 |      0 | (0,4)
  5 |   7792 |     73 |        1 |    892 |      0 | (0,5)
```
Read this carefully — it is the page structure from the diagram, live:
- `lp_off` decreases by 80 each time → tuples grow **upward from the bottom** (73 bytes rounded up to an 8-byte boundary = 80).
- `lp_flags = 1` → NORMAL.
- `t_xmax = 0` → nobody has deleted it. All rows visible.
- `t_ctid = (0,1)` → each tuple points at itself; no update chain.

And the page header:
```sql
SELECT lower, upper, pagesize, lsn FROM page_header(get_raw_page('orders', 0));
```
```
 lower | upper | pagesize |    lsn
-------+-------+----------+-----------
   428 |   440 |     8192 | 0/1B4A2C8
```
`lower=428` → 24-byte header + 101 line pointers × 4 = 428. `upper=440`. Free space = 440 − 428 = **12 bytes**. This page is full.

Now delete and observe:

```sql
DELETE FROM orders WHERE id <= 500;
SELECT pg_size_pretty(pg_relation_size('orders'));    -- 80 kB — UNCHANGED
SELECT count(*) FROM heap_page_items(get_raw_page('orders',0)) WHERE t_xmax <> 0;
```
```
 count
-------
   101     ← every tuple on page 0 is dead, and still there
```
```sql
VACUUM orders;
SELECT lp, lp_off, lp_len, lp_flags FROM heap_page_items(get_raw_page('orders',0)) LIMIT 3;
```
```
 lp | lp_off | lp_len | lp_flags
----+--------+--------+----------
  1 |      0 |      0 |        0     ← lp_flags 0 = UNUSED, space reclaimed
  2 |      0 |      0 |        0
  3 |      0 |      0 |        0
```
```sql
SELECT pg_size_pretty(pg_relation_size('orders'));    -- STILL 80 kB
```
The space is reusable but the file did not shrink. This is the mechanism behind every bloat incident you will ever debug.

---

## Example 2 — production scenario

**The situation.** Your `orders` table is 42 GB. It holds 60 million rows. Checkout p99 is fine, but the nightly reconciliation job — a full scan — takes 51 minutes and saturates the disk, and the buffer pool hit ratio drops to 71% during it, which slows the API.

**Step 1: how much of that 42 GB is actually data?**

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstattuple('orders');
```
```
 table_len   | 45097156608     -- 42 GB
 tuple_count | 60000000
 tuple_len   | 19200000000     -- 17.9 GB of LIVE tuple bytes
 tuple_percent | 42.57
 dead_tuple_count | 41000000
 dead_tuple_len   | 13120000000  -- 12.2 GB of DEAD tuples
 dead_tuple_percent | 29.09
 free_percent | 26.1
```

**Diagnosis: 57% of every page read is garbage.** The scan reads 42 GB to process 17.9 GB of real data. The cause: this table has a `status` column updated 3–4 times per order (`pending → paid → packed → shipped`), and each update writes a whole new 320-byte tuple. Autovacuum has not kept up.

**Step 2: why are the rows 320 bytes?**

```sql
SELECT a.attname, t.typname, a.attlen, a.attnum
FROM pg_attribute a JOIN pg_type t ON t.oid = a.atttypid
WHERE a.attrelid = 'orders'::regclass AND a.attnum > 0 ORDER BY a.attnum;
```

The table has 31 columns, including `shipping_address_json jsonb`, `raw_gateway_response text`, and `notes text`. Those three account for ~240 of the 320 bytes — and **the reconciliation job reads none of them.**

**Step 3: the three fixes, in order of leverage**

| Fix | Mechanism | Effect on the scan |
|---|---|---|
| **1. VACUUM / tune autovacuum** | reclaim the 12.2 GB of dead tuples for reuse | 42 GB → ~30 GB of pages actually read |
| **2. Vertical split**: move `raw_gateway_response`, `notes`, `shipping_address_json` to `order_details` (1:1) | rows drop 320 B → ~90 B; ~90 rows/page → ~300 rows/page | 30 GB → ~8 GB. Also: `status` updates now rewrite 90 bytes, not 320, so future bloat generation drops 3.5× |
| **3. Covering index for the job** | `CREATE INDEX ON orders (created_at) INCLUDE (id, user_id, total_paise, status)` → index-only scan, heap never touched | the job reads a ~2 GB index instead of the table, and stops evicting the API's hot pages from `shared_buffers` |

**Step 4: prevent recurrence**

```sql
-- more aggressive autovacuum on this one hot table
ALTER TABLE orders SET (
  autovacuum_vacuum_scale_factor = 0.02,   -- default 0.2 = 12M dead rows before it runs!
  autovacuum_vacuum_cost_delay   = 2,
  fillfactor = 85                          -- leave room for HOT updates on the same page
);
```

`fillfactor = 85` is the subtle one: it deliberately wastes 15% of each page so that a `status` update can place the new tuple **on the same page**, enabling a HOT update that touches **zero indexes**. On a table with 6 indexes, that turns one status change from 7 writes into 1.

**The result:** 51-minute scan → 6 minutes, and the API's buffer hit ratio stops collapsing at night. Not one line of application code changed. All of it follows from knowing what a page is.

---

## Common mistakes

**1. "I deleted the rows, why is the disk still full?"**
- *Symptom:* `DELETE FROM events WHERE created_at < '2025-01-01'` removes 200M rows; `pg_relation_size` is unchanged; disk alerts continue.
- *Engine-level why:* DELETE only sets `t_xmax`. The bytes remain until VACUUM, and even then the file doesn't shrink — free space is returned to the *table*, not the OS.
- *Diagnose:* `SELECT * FROM pgstattuple('events');` — look at `dead_tuple_percent` and `free_percent`.
- *Fix:* `VACUUM` to make the space reusable. To actually shrink: `VACUUM FULL` (exclusive lock, needs 2× space) or `pg_repack` (online). **Prevention:** for time-based deletion, use partitioning and `DROP TABLE` the old partition — instant, no bloat, no VACUUM (Topic 59).

**2. Assuming `SELECT one_column` is cheaper than `SELECT *` on a heap scan.**
- *Symptom:* team "optimises" queries by listing fewer columns; nothing gets faster.
- *Engine-level why:* the heap fetch retrieves the entire tuple in the entire page. Column projection happens *after* the page is in memory. The page count is identical.
- *When it DOES help:* (a) network transfer of wide columns, (b) if a **covering index** makes an index-only scan possible, (c) avoiding TOAST de-toasting of large values (Topic 05). Otherwise, no.
- *Fix:* if you want fewer pages read, you need a narrower table or a covering index — not a shorter SELECT list.

**3. Very wide tables with a few hot columns.**
- *Symptom:* a table with 60 columns where 3 are updated constantly; enormous bloat and WAL volume.
- *Engine-level why:* MVCC copies the *whole* row on every update. Updating a 4-byte counter on a 2 KB row writes 2 KB of new tuple plus 2 KB of WAL, plus every index entry if it can't do a HOT update.
- *Diagnose:* `pg_stat_user_tables.n_tup_upd` vs `n_tup_hot_upd` — if HOT ratio is low, you're rewriting indexes constantly.
- *Fix:* split hot columns into a 1:1 side table; set `fillfactor` to 85–90; drop unused indexes.

**4. Storing large blobs in a row and expecting scans to be fast.**
- *Symptom:* a `documents` table with a `content text` column averaging 40 KB; `SELECT id, title FROM documents` is slow.
- *Engine-level why:* values over ~2 KB are TOASTed out-of-line, which is *good* — but if they're between the inline limit and the TOAST threshold, or compression keeps them inline, rows-per-page collapses to 1–2 and a scan reads enormous numbers of pages.
- *Diagnose:* `SELECT pg_relation_size('documents'), pg_total_relation_size('documents');` — a big gap means TOAST is doing its job; a small gap with a big table means the blobs are inline.
- *Fix:* keep blobs out of hot tables entirely; put bytes in object storage and a key in the row (Topic 01's framework).

**5. Storing or relying on `ctid`.**
- *Symptom:* a cache or external index keyed on `ctid` returns the wrong row after a while.
- *Engine-level why:* `ctid` is a *physical* address. Any UPDATE moves the row and changes it; VACUUM FULL changes all of them.
- *Fix:* use the primary key. `ctid` is a debugging tool only.

---

## Hands-on proof

**PROVE IT #1 — the page size is fixed.**
```sql
SHOW block_size;            -- 8192
SELECT current_setting('block_size')::int * relpages = pg_relation_size(oid) AS exact
FROM pg_class WHERE relname='orders';   -- t
```

**PROVE IT #2 — find the file, on the actual filesystem.**
```sql
SELECT pg_relation_filepath('orders');       -- base/16388/16390
```
```bash
docker exec pg-lab ls -l /var/lib/postgresql/data/base/16388/16390*
# 16390       ← main fork
# 16390_fsm   ← free space map
# 16390_vm    ← visibility map (appears after the first VACUUM)
```

**PROVE IT #3 — the two arrays growing toward each other.**
```sql
SELECT lower, upper, upper - lower AS free_bytes, pagesize
FROM page_header(get_raw_page('orders', 0));
```
Insert more rows into a page with room and re-run: `lower` grows by 4 per row, `upper` shrinks by the tuple size.

**PROVE IT #4 — an UPDATE writes a new tuple and leaves the old one.**
```sql
CREATE TABLE t (id int, v text);
INSERT INTO t VALUES (1, 'a');
SELECT lp, t_xmin, t_xmax, t_ctid FROM heap_page_items(get_raw_page('t',0));
-- lp=1  t_xmax=0   t_ctid=(0,1)

UPDATE t SET v = 'b' WHERE id = 1;
SELECT lp, t_xmin, t_xmax, t_ctid FROM heap_page_items(get_raw_page('t',0));
```
```
 lp | t_xmin | t_xmax |  t_ctid
----+--------+--------+---------
  1 |    901 |    902 | (0,2)     ← OLD tuple: xmax set, ctid POINTS FORWARD
  2 |    902 |      0 | (0,2)     ← NEW tuple
```
Two physical rows for one logical row. That forward-pointing `t_ctid` is the update chain — the foundation of MVCC (Topic 46).

**PROVE IT #5 — HOT updates depend on page free space.**
```sql
CREATE TABLE hot_demo (id int PRIMARY KEY, n int, pad text) WITH (fillfactor = 90);
INSERT INTO hot_demo SELECT i, 0, repeat('x', 100) FROM generate_series(1,1000) i;
UPDATE hot_demo SET n = n + 1;
SELECT relname, n_tup_upd, n_tup_hot_upd,
       round(100.0*n_tup_hot_upd/nullif(n_tup_upd,0),1) AS hot_pct
FROM pg_stat_user_tables WHERE relname='hot_demo';
```
```
 relname  | n_tup_upd | n_tup_hot_upd | hot_pct
----------+-----------+---------------+---------
 hot_demo |      1000 |           962 |    96.2
```
Now repeat with `fillfactor = 100` and watch `hot_pct` collapse. Every non-HOT update rewrites every index entry.

**PROVE IT #6 — rows per page controls scan cost.**
```sql
CREATE TABLE narrow AS SELECT i AS id, i*2 AS n FROM generate_series(1,500000) i;
CREATE TABLE wide   AS SELECT i AS id, i*2 AS n, repeat('x',300) AS pad
                       FROM generate_series(1,500000) i;
SELECT relname, relpages FROM pg_class WHERE relname IN ('narrow','wide');
```
```
 relname | relpages
---------+----------
 narrow  |     2213    ← 226 rows/page
 wide    |    22728    ← 22 rows/page   — 10× the I/O for the SAME 500k rows
```
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM narrow;   -- Buffers: read=2213
EXPLAIN (ANALYZE, BUFFERS) SELECT count(*) FROM wide;     -- Buffers: read=22728
```
**Same row count. Same query. 10× the pages.** Column choice *is* performance.

---

## The design decision framework

```
KEEP ROWS NARROW WHEN:
  ✓ The table is scanned or range-scanned frequently
  ✓ The table is large enough that it won't fit entirely in shared_buffers
  ✓ Some columns are updated far more often than others
  ✓ Some columns are read far less often than others (blobs, raw payloads, notes)
  → Split them into a 1:1 side table. This is a legitimate, deliberate
    denormalisation in the OTHER direction (Topic 55: vertical partitioning).

ACCEPT WIDE ROWS WHEN:
  ✓ Access is almost always by primary key, one row at a time
  ✓ The table is small enough to stay fully cached
  ✓ All columns are genuinely read together
  ✗ AVOID splitting when it would turn every read into a join for no measured win

SET fillfactor < 100 WHEN:
  ✓ n_tup_hot_upd / n_tup_upd is low AND the table is update-heavy
  ✓ The table has several indexes (so non-HOT updates are expensive)
  Start at 85–90. It costs disk to save write amplification.

LEAVE fillfactor = 100 WHEN:
  ✓ Append-only tables (logs, events, orders that are never updated)
  → wasting page space on a table that never updates is pure loss.

DELETE vs PARTITION-DROP:
  ✓ Deleting > ~10% of a large table on a schedule → PARTITION and DROP instead.
    DROP TABLE is O(1) and creates zero bloat. DELETE is O(n) and creates
    n dead tuples that VACUUM must then process. (Topic 59.)

THE SIGNAL TO LOOK FOR:
  Run this on any table you suspect:
      SELECT relpages, reltuples, reltuples/nullif(relpages,0) AS rows_per_page
      FROM pg_class WHERE relname = '<table>';
  • rows_per_page < 20 on a frequently-scanned table → your rows are too wide
  • pgstattuple dead_tuple_percent > 20 → autovacuum is losing; tune it
  • n_tup_hot_upd / n_tup_upd < 50% on an update-heavy table → lower fillfactor
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Create a table with a single `int` column and insert 10,000 rows. Using `pg_class`, compute rows-per-page. Then use `page_header()` and `heap_page_items()` on page 0 to show exactly how the 8,192 bytes are accounted for: header + line pointers + tuples + free. Your numbers should add up to 8192. Explain any discrepancy.

### Exercise 2 — medium (apply it)
Take the `wide` table from PROVE IT #6. Without deleting any data, make `SELECT count(*) FROM wide WHERE n > 900000` read fewer than 3,000 pages. You may change the schema, add objects, or both. Prove it with `EXPLAIN (ANALYZE, BUFFERS)` before and after, and explain the mechanism in one paragraph.

### Exercise 3 — hard (production simulation)
An `events` table is 180 GB. It receives 40M inserts/day and a nightly `DELETE FROM events WHERE created_at < now() - interval '90 days'`. The delete takes 6 hours, autovacuum never finishes, table size grows every week despite the delete, and the disk fills every 6 weeks requiring an emergency `VACUUM FULL` with 40 minutes of downtime.

(a) Explain, at the page level, why the DELETE + VACUUM cycle cannot keep up. Be specific about what work each dead tuple creates.
(b) Explain why `VACUUM FULL` needs roughly 2× the table size in free disk and why it takes an ACCESS EXCLUSIVE lock.
(c) Design the fix so that removing 90-day-old data takes under one second and generates zero dead tuples. Describe the schema change, the retention job, and the one thing about indexes on this design that will surprise you.
(d) What monitoring query would have caught this 5 months earlier?

---

## Mental model checkpoint

1. Draw an 8KB heap page from memory, labelling the header, line pointers, free space, and tuples. Which direction does each array grow, and why do they grow toward each other?
2. What is a TID, and why do indexes store one instead of a byte offset?
3. After `DELETE FROM orders WHERE id = 5; COMMIT;`, what has physically changed on disk? What has not?
4. Why does `SELECT id FROM orders` read the same number of pages as `SELECT * FROM orders` in a sequential scan? Name three situations where that is not true.
5. What is a HOT update, what two conditions must hold for one, and why does `fillfactor` affect it?
6. Why is 8KB the page size? Give one argument against 512 bytes and one against 64 KB.
7. Your table is 40 GB with 12 GB of live tuples. Name three separate costs you are paying for the other 28 GB.

---

## Quick reference card

| Term | Meaning |
|---|---|
| Page / block | 8,192 bytes; the unit of all I/O |
| Heap file | The unordered file of a table's rows |
| Segment | A 1 GB chunk of one relation fork |
| Line pointer | 4-byte (offset, length, flags) — the indirection indexes point at |
| TID / ctid | (block, offset) — a row's *physical*, unstable address |
| FSM | Free space map fork — where can a new tuple go |
| VM | Visibility map fork — 2 bits/page; enables index-only scans |
| Fillfactor | % of a page INSERT may fill; headroom for HOT updates |
| HOT update | Same-page update that touches zero indexes |
| Bloat | Dead tuples + free space occupying pages you still read |

**Numbers to memorise**

| Thing | Value |
|---|---|
| Page size | **8192 bytes** |
| Page header | 24 bytes |
| Line pointer | 4 bytes |
| Max segment file | 1 GB |
| Heap fillfactor default | 100 |
| B-tree fillfactor default | 90 |
| Max heap tuple (inline) | ~8160 bytes; larger → TOAST |
| Buffer hit | ~100 ns · SSD page read ~100 µs · HDD seek ~10 ms |

**Diagnostic queries to keep**

```sql
SELECT relpages, reltuples, reltuples/nullif(relpages,0) AS rows_per_page
FROM pg_class WHERE relname = 'orders';

SELECT * FROM pgstattuple('orders');                       -- bloat truth

SELECT relname, n_tup_upd, n_tup_hot_upd, n_dead_tup, last_autovacuum
FROM pg_stat_user_tables ORDER BY n_dead_tup DESC LIMIT 10;

SELECT * FROM heap_page_items(get_raw_page('orders', 0));  -- raw page contents
SELECT * FROM page_header(get_raw_page('orders', 0));      -- header
```

---

## When would I use this at work?

1. **Disk alerts that deletes don't fix.** You immediately reach for `pgstattuple`, see `dead_tuple_percent = 44`, and know the answer is autovacuum tuning plus a partitioning plan — not a bigger volume. That's a 20-minute diagnosis instead of a week of guessing.

2. **A schema review.** A teammate adds a `last_seen_at timestamptz` to the 40-column `users` table, updated on every request. You can point out that this makes every page-view rewrite a 400-byte row and 5 index entries, propose a separate `user_activity` table or a Redis-backed write-behind, and show the `n_tup_hot_upd` number that proves it.

3. **Explaining a slow report.** A batch job reads 42 GB. You show that only 18 GB is live data and only 3 of 31 columns are used, then propose a covering index for an index-only scan. The report drops from 51 minutes to 6, and you can explain exactly which pages stopped being read.

---

## Connected topics

**Understand before this:** 01 (databases are files), 02 (the buffer manager and storage layer boxes).

**This unlocks:**
- **05 — Row storage format**: every byte inside the tuple, plus TOAST
- **07 — The buffer pool**: how these pages live in RAM
- **11 — B-tree indexes**: the same page structure, with special space and sibling pointers
- **12 — Index lookup end to end**: TIDs, heap fetches, and the visibility map in action
- **46/47 — MVCC and VACUUM**: why dead tuples exist and how they're reclaimed
- **59 — Partitioning**: the correct answer to large-scale deletion
