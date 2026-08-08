# 11 — B-tree Indexes in Depth
## Phase: Indexes

---

## ELI5 — The Simple Analogy

A library with 50 million books, and one rule: **you must find any book in four steps.**

You can't have one giant alphabetical list — walking it would take hours. So the librarian builds a *hierarchy of signposts*.

At the entrance, one board: *"A–F → Wing 1. G–M → Wing 2. N–S → Wing 3. T–Z → Wing 4."* One glance, one decision.

In Wing 2, another board: *"Ga–Gk → Aisle 1. Gl–Gz → Aisle 2. Ha–Hm → Aisle 3…"* Another glance.

In Aisle 1, a shelf label. And on the shelf itself: the actual books, in order, with a **string tied to the next shelf** so you can walk sideways through "everything from Gandhi to Ghosh" without going back to the entrance.

Three signposts, then the books. Every book, every time, exactly the same effort. Add 50 million more books and it becomes *four* signposts — not eight, not eighty. That flatness under growth is the entire point of a B-tree, and it's why every relational database has used one for fifty years.

---

## Where this fits in the big picture

```
   04 pages ────┐
   08 B-tree vs LSM ──┐
                      ▼
   10 what an index is
            │
            ▼
   ┌──────────────────────────┐
   │ 11 B-TREE IN DEPTH       │  ← YOU ARE HERE
   │ the structure itself     │
   └────────────┬─────────────┘
                │
   ┌────────────┼─────────────┬──────────────┐
   ▼            ▼             ▼              ▼
 12 lookup   14 composite  17 bloat &     22 primary
 end-to-end  column order  page splits    keys (UUIDv7)
```

Topic 10 told you an index maps keys to TIDs. **This topic is the data structure that makes that mapping fast, and the physics of why it degrades.**

---

## What is this?

PostgreSQL's default index is a **B+tree** (everyone calls it "B-tree"; the `+` matters and we'll see why). It is a balanced, sorted, multi-way search tree where:

- **Every node is one 8 KB page.**
- **Internal nodes hold only separator keys and child pointers** — no TIDs.
- **All data (keys + TIDs) lives in the leaf level**, and only the leaf level.
- **All leaves are at the same depth** — the tree is always perfectly balanced.
- **Leaves are linked to their siblings**, so a range scan walks sideways without re-descending.

The "B" is not "binary" — it's usually taken to mean "balanced." Each node has hundreds of children, not two.

---

## Why does it matter for a backend developer?

Because the structure explains four things you will hit repeatedly, and nothing else does:

1. **Why lookups stay fast as data grows.** 1,000 rows and 1,000,000,000 rows are 2 and 5 page reads. Logarithmic with a base of ~500 is nearly flat.
2. **Why random primary keys destroy insert performance.** UUIDv4 causes page splits everywhere; `bigserial` and UUIDv7 append to one hot rightmost page. This is a *structural* property of the tree, not a tuning issue. It is the whole argument in Topic 22.
3. **Why indexes bloat and how to tell.** Page splits leave leaves ~50% full. `avg_leaf_density` is the number that reveals it.
4. **Why `ORDER BY` can be free.** The leaf level *is* a sorted linked list. If your index order matches your query's order, the sort node disappears — which converts a blocking pipeline into a streaming one (Topic 09).

And it explains why the answer to "why not a hash table?" is: hash tables can't do ranges, can't do `ORDER BY`, and can't do prefix matching. Which is most of what you actually query.

---

## The physical reality

### The tree, drawn to scale

```
 idx_orders_user_id  on orders(user_id)  — 3,000,000 rows

 LEVEL 2 (root)   1 page
 ┌───────────────────────────────────────────────────────────────────┐
 │ META → this page                                                  │
 │ [ptr]  9,281  [ptr]  38,402  [ptr]  71,004  [ptr] ... ~500 entries│
 └───┬───────────────┬───────────────────┬──────────────┬────────────┘
     │               │                   │              │
 LEVEL 1 (internal)  ~500 pages
     ▼               ▼                   ▼              ▼
 ┌─────────┐   ┌─────────┐         ┌─────────┐    ┌─────────┐
 │[p] 1204 │   │[p]11002 │         │[p]40119 │    │[p]72330 │  ~500 keys each
 │[p] 2401 │   │[p]12880 │         │  ...    │    │  ...    │
 └──┬──────┘   └──┬──────┘         └──┬──────┘    └──┬──────┘
    │             │                   │              │
 LEVEL 0 (leaves) ~6,150 pages — ALL the keys and ALL the TIDs
    ▼             ▼                   ▼              ▼
 ┌────────────┐ ┌────────────┐   ┌────────────┐  ┌────────────┐
 │k=1→(0,1)   │◀│k=487→(88,2)│◀─▶│k=4471→..   │◀▶│k=9280→..   │
 │k=2→(0,7)   │▶│k=488→(91,4)│   │k=4471→..   │  │...         │
 │... ~500    │ │... ~500    │   │... ~500    │  │... ~500    │
 └────────────┘ └────────────┘   └────────────┘  └────────────┘
      ◀────────── doubly-linked sibling pointers ──────────▶

 HEIGHT = 3 levels for 3 MILLION rows.
 Any lookup = 3 page reads (the top two are ~always cached → ~1 real read).
```

### One index page, byte by byte

```
 ┌────────────────────────────────────────────────────────────────┐
 │ PageHeaderData (24 B)   pd_lower, pd_upper, pd_lsn ...          │
 ├────────────────────────────────────────────────────────────────┤
 │ ItemIdData array (4 B each) — one per index tuple               │
 ├────────────────────────────────────────────────────────────────┤
 │                                                                 │
 │            F R E E   S P A C E                                  │
 │       (this is what fillfactor reserves — default 90 for btree) │
 │                                                                 │
 ├────────────────────────────────────────────────────────────────┤
 │ INDEX TUPLES, in KEY ORDER, growing upward:                     │
 │   ┌────────────────────────────────────────────────┐            │
 │   │ IndexTupleData (8 B):                          │            │
 │   │   t_tid   (6 B)  ← the heap TID (block,offset) │            │
 │   │   t_info  (2 B)  ← size + null/varlena flags   │            │
 │   ├────────────────────────────────────────────────┤            │
 │   │ KEY DATA (8 B for a bigint)                    │            │
 │   └────────────────────────────────────────────────┘            │
 │   = 16 bytes per entry for a bigint key                         │
 ├────────────────────────────────────────────────────────────────┤
 │ BTPageOpaqueData — SPECIAL SPACE (16 B)                         │
 │   btpo_prev   (4 B)  ← LEFT sibling block number                │
 │   btpo_next   (4 B)  ← RIGHT sibling block number               │
 │   btpo_level  (4 B)  ← 0 = leaf                                 │
 │   btpo_flags  (2 B)  ← LEAF / ROOT / DELETED / HALF_DEAD        │
 │   btpo_cycleid(2 B)                                             │
 └────────────────────────────────────────────────────────────────┘

 ★ The SPECIAL SPACE is the difference between a heap page (Topic 04,
   0 bytes special) and an index page. Those two sibling pointers are
   what make range scans cheap.
```

### Fanout — the number that makes it all work

```
 usable bytes per page = 8192 − 24 (header) − 16 (special) ≈ 8152
 with fillfactor 90    ≈ 7,337 bytes actually used

 KEY TYPE           ENTRY SIZE      FANOUT      HEIGHT FOR 1 BILLION ROWS
 ─────────────────────────────────────────────────────────────────────────
 int4 (4 B)         16 B (aligned)  ~458        4  (458⁴ = 44 billion)
 bigint (8 B)       16 B            ~458        4
 uuid (16 B)        24 B            ~305        4  (305⁴ = 8.6 billion)
 text avg 40 B      48 B            ~152        5  (152⁵ = 81 billion)
 text avg 200 B     208 B           ~35         6  ⚠
 composite (8+8+8)  32 B            ~229        5

 ⇒ WIDE KEYS ARE EXPENSIVE. They reduce fanout, which increases height,
   which increases page reads per lookup — AND makes the index bigger,
   so less of it fits in the buffer pool.
   This is the concrete reason to avoid indexing long text columns
   directly, and to prefer a hash or a surrogate key.

 ⚠ HARD LIMIT: an index entry cannot exceed ~1/3 of a page (2,704 bytes).
   `ERROR: index row size 3104 exceeds btree version 4 maximum 2704`
   Fix: index an expression like md5(col) or use a GIN/hash index.
```

### Deduplication (PostgreSQL 13+) — why low-cardinality indexes shrank

```
 BEFORE dedup (PG ≤12), an index on `status` with 4 distinct values:
   'paid' → (1204,3)
   'paid' → (1204,7)
   'paid' → (1205,1)     ... 750,000 identical keys, 16 bytes each

 WITH DEDUPLICATION (PG 13+), leaf entries become POSTING LISTS:
   ┌────────────────────────────────────────────────────────┐
   │ key='paid'  │ TID array: (1204,3)(1204,7)(1205,1)...   │
   │             │ 6 bytes per TID, key stored ONCE          │
   └────────────────────────────────────────────────────────┘

 ⇒ 16 B/row → ~6 B/row for duplicate keys. Indexes on low-cardinality
   columns are ~2.5× smaller than they used to be.
 ⚠ This makes them SMALLER, not USEFUL. The selectivity argument from
   Topic 10 is unchanged — the planner still won't use an index that
   matches 25% of the table.
 Control with: CREATE INDEX ... WITH (deduplicate_items = off);
```

---

## How it works — step by step

### Search: `WHERE user_id = 4471`

```
 1. Read the METAPAGE (block 0).
    It holds: root block number, root level, "fast root" (an optimisation
    when the top levels have been emptied by deletions).
    → root = block 412, level = 2.
    [1 page read — always cached]

 2. Read block 412 (root).
    Binary search its ~500 separator keys for the largest key ≤ 4471.
       [ptr₀] 9,281 [ptr₁] 38,402 [ptr₂] ...
       4471 < 9281  → follow ptr₀ → block 88
    ⚠ THE B+TREE RULE: internal nodes contain NO TIDs. A separator key
      here is just a routing decision. The actual key 4471 (if it exists)
      is guaranteed to be in the LEAF level, never here.
    [1 page read — almost always cached]

 3. Read block 88 (internal, level 1).
    Binary search again → child block 3,902.
    [1 page read — usually cached]

 4. Read block 3,902 (LEAF, level 0).
    Binary search ~500 entries → first entry with key ≥ 4471.
    [1 page read — this is the real I/O]

 5. Scan FORWARD from that position while key = 4471, collecting TIDs:
       (1204,3) (2891,17) (4102,5) ... 17 of them.
    If the matches run past the end of this leaf, follow `btpo_next`
    to the sibling page — NO re-descent needed.

 6. Hand the TIDs to the executor for heap fetches. (Topic 12.)

 TOTAL: 4 index page reads (3 usually cached) + N heap reads.
```

### Range scan: `WHERE user_id BETWEEN 4471 AND 4600`

```
 Steps 1–4 identical: descend to the leaf containing 4471.
 Then simply WALK RIGHT via btpo_next until key > 4600.

 ┌────────┐   ┌────────┐   ┌────────┐   ┌────────┐
 │ 4400.. │──▶│ 4489.. │──▶│ 4552.. │──▶│ 4611.. │  stop
 │  4488  │   │  4551  │   │  4610  │   │        │
 └────────┘   └────────┘   └────────┘   └────────┘
      ↑ descend here          walk right, sequentially

 ★ THIS IS WHY B+TREES BEAT B-TREES (the non-plus kind) FOR DATABASES.
   In a plain B-tree, data lives in internal nodes too, so a range scan
   must go up and down the tree repeatedly. In a B+tree, all data is at
   the leaf level in a linked list, so a range is one descent plus a
   sequential walk.

 ★ AND THIS IS WHY `ORDER BY user_id` CAN BE FREE:
   the leaf level IS the sorted order. The executor gets rows already
   sorted, so the planner emits no Sort node. Combined with LIMIT, it
   can stop after N rows. (Topic 09's blocking-vs-streaming point.)
```

### Insert with a page split — where all the pain comes from

```
 INSERT key = 4500 into a leaf that is FULL.

 BEFORE
 ┌──────────────────────────────────────┐
 │ LEAF block 3902  (level 0)           │
 │ 4400 4401 ... 4488   ← 500 entries   │
 │ btpo_prev=3901  btpo_next=3903       │
 └──────────────────────────────────────┘
        parent block 88: [... ptr→3902 | 4489 | ptr→3903 ...]

 SPLIT PROCEDURE
  1. Allocate a NEW page (block 7741).
  2. Choose a split point. PostgreSQL is smart here:
       • RIGHTMOST leaf (an append pattern) → split 90/10, leaving the
         new page nearly empty for more appends. ★ This is the
         "fastpath" that makes bigserial inserts cheap.
       • Interior leaf (random insert)      → split ~50/50.
  3. Move the upper half of the entries to block 7741.
  4. Fix sibling pointers: 3902.next=7741, 7741.next=3903,
     3903.prev=7741.
  5. Insert a new separator key into the PARENT (block 88).
  6. ⚠ IF THE PARENT IS ALSO FULL → split it too. Recursively.
     If the ROOT splits, a NEW ROOT is created and the tree grows
     one level taller. (This is the only way height increases.)

 AFTER (random insert, 50/50 split)
 ┌────────────────────┐    ┌────────────────────┐
 │ block 3902         │◀──▶│ block 7741         │
 │ 4400 ... 4444      │    │ 4445 ... 4488 4500 │
 │ ~250 entries (50%) │    │ ~251 entries (50%) │
 └────────────────────┘    └────────────────────┘
        parent: [... ptr→3902 | 4445 | ptr→7741 | 4489 | ptr→3903 ...]

 ⇒ COST OF ONE SPLIT: 2 page writes + 1 parent write + WAL for all three
   (with full-page images if it's the first touch since a checkpoint).
   ⇒ AND both pages are now only 50% full → the index is now 2× the
     size it needs to be, forever, until REINDEX.
```

### Why insertion order changes everything

```
 SEQUENTIAL KEYS (bigserial, UUIDv7, ULID, timestamp)
 ───────────────────────────────────────────────────────────
   Every insert goes to the RIGHTMOST leaf.
   • That page is ALWAYS in the buffer pool (just touched)
   • PostgreSQL caches the rightmost leaf block explicitly (fastpath):
     no tree descent at all for the common case
   • Splits are 90/10, so pages end up ~90% full
   • Splits happen once per ~450 inserts

   ┌──┐┌──┐┌──┐┌──┐┌██┐   ██ = hot page, always cached
   │90││90││90││90││10│   all others sealed at ~90% full
   └──┘└──┘└──┘└──┘└──┘

 RANDOM KEYS (UUIDv4, hash, random int)
 ───────────────────────────────────────────────────────────
   Every insert goes to a RANDOM leaf out of 6,150.
   • Almost certainly NOT in the buffer pool → a random 8 KB read
   • Read-modify-write, then the page is dirty and must be written
   • Splits are 50/50, so pages settle at ~50–70% full
   • Once the index exceeds RAM, EVERY insert is a random disk read

   ┌──┐┌──┐┌──┐┌──┐┌──┐
   │66││52││71││58││63│   all pages half-empty, all pages cold
   └──┘└──┘└──┘└──┘└──┘

 MEASURED (3M rows):
   bigserial : 8.4 s,   64 MB index, avg_leaf_density 90%
   uuid v4   : 41.2 s, 130 MB index, avg_leaf_density 66%,
                                     leaf_fragmentation 48%
   ⇒ 4.9× slower, 2.0× bigger. Purely structural.
```

---

## Concept breakdown

```
B+TREE
│  ├── B     "balanced" — every leaf is at the same depth, ALWAYS
│  ├── +     data lives ONLY in leaves; internal nodes are pure routing
│  └── TREE  height grows logarithmically, base ≈ fanout (~500)
│
├── METAPAGE (block 0)  root block number, root level, fast root
├── ROOT                one page; splits here grow the tree taller
├── INTERNAL NODES      separator keys + child pointers. NO TIDs.
├── LEAF NODES          keys + TIDs + prev/next sibling pointers
└── HIGH KEY            each page's first item is an upper bound on its
                        contents — used to detect that a concurrent split
                        moved your target, so you follow btpo_next
                        instead of restarting. (This is the Lehman & Yao
                        algorithm, and it's why B-tree reads don't block
                        writers.)

FANOUT
│   entries per page ≈ (8192 − 24 − 16) × fillfactor / entry_size
└── Drives HEIGHT: height ≈ ceil(log_fanout(N))
    ⇒ Every byte you add to a key costs you fanout, which costs height,
      which costs a page read on EVERY lookup.

PAGE SPLIT
│
├── Trigger      a leaf is full and an insert must go there
├── Rightmost    90/10 split (append optimisation)
├── Interior     50/50 split
├── Cascading    parent full → parent splits → possibly up to the root
└── Consequence  pages left ~50% full = permanent bloat until REINDEX

FILLFACTOR (btree default = 90)
│   Reserves 10% of each leaf at BUILD time so later inserts fit without
│   splitting. Only applies to CREATE INDEX / REINDEX, not to a growing
│   index's new pages.
├── Append-only workload (time-ordered keys) → set 100. No splits happen
│   anyway; you're just wasting 10% of every page.
└── Random-insert workload → keep 90, or lower to 70–80.

WHY NOT A HASH TABLE?
│
├── ✗ no range queries        WHERE created_at > X
├── ✗ no ORDER BY support     the leaf level of a B-tree IS the sort
├── ✗ no prefix matching      LIKE 'Basm%'
├── ✗ no multi-column prefix  (a,b) can't serve a query on just a
└── ✓ O(1) instead of O(log n)... but log₅₀₀(10⁹) = 4. Constant-ish
                                   already. The gain is negligible;
                                   the losses are enormous. (Topic 16.)

WHY NOT A BINARY TREE?
│
└── A binary tree of 1B keys is 30 levels deep. Each level is a page
    read. 30 page reads vs 4. The whole point of a B-tree is to match
    the node size to the DISK BLOCK SIZE so one read makes ~500
    decisions instead of one.
```

---

## Diagrams

**Diagram 1 — big picture: how height stays flat**

```
 ROWS          HEIGHT   PAGE READS PER LOOKUP   INDEX SIZE (bigint key)
 ───────────────────────────────────────────────────────────────────────
 1,000            1              1                    16 KB
 100,000          2              2                     2 MB
 3,000,000        3              3                    64 MB
 500,000,000      4              4                    11 GB
 100,000,000,000  5              5                   2.2 TB

     rows ────────────────────────────────────────────▶
       │  ██
 reads │  ██ ██
       │  ██ ██ ██
       │  ██ ██ ██ ██
       │  ██ ██ ██ ██ ██
       └───────────────────────────────────────────────
         10³ 10⁵ 10⁶  10⁸  10¹¹

 ★ 100,000,000× more data costs 5× the reads. And the top 2–3 levels
   are always in the buffer pool, so the REAL disk cost goes 1 → 2.
```

**Diagram 2 — data flow: point lookup vs range scan**

```
  POINT LOOKUP  user_id = 4471          RANGE SCAN  user_id 4471..4600
  ───────────────────────────────       ───────────────────────────────
        [meta]                                [meta]
          │                                     │
        [root]  binary search                 [root]
          │                                     │
      [internal] binary search             [internal]
          │                                     │
        [leaf] ──▶ 17 TIDs                    [leaf]───▶[leaf]───▶[leaf]
          │                                     └── walk right via
          ▼                                         btpo_next, no
      heap fetch ×17                                re-descent
                                                    │
                                                    ▼
                                              heap fetch ×N
                                              (already in key order —
                                               so ORDER BY is FREE)
```

**Diagram 3 — before/after: the split, and what it costs you forever**

```
 SEQUENTIAL INSERTS (bigserial) — 1M rows
 ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
 │ 90%│ 90%│ 90%│ 90%│ 90%│ 90%│ 90%│ 90%│ 90%│ 10%│  ← only the last
 └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘     is hot
   2,270 leaf pages · 18 MB · avg_leaf_density 90%
   splits: ~2,270 total, all cheap 90/10 on a cached page

 RANDOM INSERTS (uuid v4) — same 1M rows
 ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
 │ 63%│ 51%│ 72%│ 48%│ 66%│ 55%│ 70%│ 49%│ 61%│ 58%│  ← ALL cold
 └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
   4,410 leaf pages · 36 MB · avg_leaf_density 58%
   splits: ~4,400, each a random read + 3 page writes

 THE PERMANENT COSTS OF THE RIGHT-HAND CASE
   • 2× the disk
   • 2× the buffer pool consumed (so less room for the table)
   • ~1 extra tree level once you pass ~500M rows
   • 4.9× the insert time
   • 2× the WAL, shipped to every replica, forever
   ⇒ and the ONLY difference was the shape of the primary key.
```

---

## Example 1 — basic

Dissect a real B-tree page by page.

```sql
CREATE EXTENSION IF NOT EXISTS pageinspect;

CREATE TABLE orders (
  id bigserial PRIMARY KEY,
  user_id bigint NOT NULL,
  created_at timestamptz NOT NULL DEFAULT now()
);
INSERT INTO orders (user_id, created_at)
SELECT (random()*200000)::bigint, now() - (random()*365)::int * interval '1 day'
FROM generate_series(1, 3000000);
CREATE INDEX idx_orders_user_id ON orders (user_id);
VACUUM ANALYZE orders;
```

**Step 1 — the metapage.**

```sql
SELECT * FROM bt_metap('idx_orders_user_id');
```
```
 magic  | version | root | level | fastroot | fastlevel | last_cleanup_num_delpages
--------+---------+------+-------+----------+-----------+---------------------------
 340322 |       4 |  412 |     2 |      412 |         2 |                         0
```
`level = 2` → three levels (2, 1, 0). Root is block 412.

**Step 2 — the root page.**

```sql
SELECT * FROM bt_page_stats('idx_orders_user_id', 412);
```
```
 blkno | type | live_items | dead_items | avg_item_size | page_size | free_size | btpo_level
-------+------+------------+------------+---------------+-----------+-----------+------------
   412 | r    |         28 |          0 |            15 |      8192 |      7716 |          2
```
Type `r` = root. 28 children — the tree is wide but not full at the top, which is normal.

```sql
SELECT itemoffset, ctid AS child_block, data FROM bt_page_items('idx_orders_user_id', 412) LIMIT 5;
```
```
 itemoffset | child_block |          data
------------+-------------+-------------------------
          1 | (3,0)       |                              ← leftmost, no key
          2 | (1751,0)    | 9c 1c 00 00 00 00 00 00      ← 7324
          3 | (3502,0)    | 6f 39 00 00 00 00 00 00      ← 14703
          4 | (5253,0)    | a1 55 00 00 00 00 00 00      ← 21921
```
`ctid` in an internal node is a **child block pointer**, not a heap TID. Item 1 has no key — the leftmost child needs no lower bound.

**Step 3 — a leaf page.**

```sql
SELECT * FROM bt_page_stats('idx_orders_user_id', 3);
```
```
 blkno | type | live_items | avg_item_size | free_size | btpo_prev | btpo_next | btpo_level
-------+------+------------+---------------+-----------+-----------+-----------+------------
     3 | l    |        367 |            16 |       268 |         0 |         4 | 0
```
Type `l` = leaf. **367 entries × 16 bytes**, 268 bytes free, next sibling is block 4.

```sql
SELECT itemoffset, ctid AS heap_tid, itemlen, data
FROM bt_page_items('idx_orders_user_id', 3) LIMIT 4;
```
```
 itemoffset |  heap_tid  | itemlen |          data
------------+------------+---------+-------------------------
          1 | (12,4)     |      16 | 00 00 00 00 00 00 00 00
          2 | (18820,29) |      16 | 00 00 00 00 00 00 00 00
          3 | (5711,45)  |      16 | 01 00 00 00 00 00 00 00
```
Here `ctid` **is** a heap TID. Note how scattered they are — `(12,4)` then `(18820,29)`. That scatter is exactly why index scans do random I/O.

**Step 4 — the overall health.**

```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstatindex('idx_orders_user_id');
```
```
 version | tree_level | index_size | root_block_no | internal_pages | leaf_pages |
       4 |          2 |   67108864 |           412 |             28 |       8163 |
 empty_pages | deleted_pages | avg_leaf_density | leaf_fragmentation
           0 |             0 |            89.87 |               0.00
```
`avg_leaf_density 89.87%` — healthy, because the keys arrived in random order but the index was **built in one pass by `CREATE INDEX`**, which sorts first and packs leaves to fillfactor. Watch what happens when the same keys arrive as *inserts*:

```sql
CREATE TABLE grown (k bigint);
CREATE INDEX idx_grown ON grown (k);          -- index FIRST, then insert
INSERT INTO grown SELECT (random()*10000000)::bigint FROM generate_series(1,3000000);
SELECT avg_leaf_density, leaf_fragmentation, pg_size_pretty(index_size::bigint)
FROM pgstatindex('idx_grown');
```
```
 avg_leaf_density | leaf_fragmentation | pg_size_pretty
------------------+--------------------+----------------
            64.21 |              41.83 | 92 MB
```
**64% density, 42% fragmentation, 92 MB instead of 64 MB.** Same data. The only difference: entries arrived one at a time in random order, causing 50/50 splits.

```sql
REINDEX INDEX idx_grown;
SELECT avg_leaf_density, pg_size_pretty(index_size::bigint) FROM pgstatindex('idx_grown');
-- 89.94 | 64 MB     ← rebuilt in sorted order, packed to fillfactor
```

**Step 5 — prove the range scan uses sibling pointers.**

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM orders WHERE user_id BETWEEN 4471 AND 4600;
```
```
Aggregate  (actual time=0.412..0.413 rows=1 loops=1)
  ->  Index Only Scan using idx_orders_user_id on orders
        Index Cond: ((user_id >= 4471) AND (user_id <= 4600))
        Heap Fetches: 0
        Buffers: shared hit=11
```
**11 pages for 130 distinct user_ids covering ~1,950 rows** — 3 for the descent, 8 leaf pages walked sideways. No re-descent per key.

**Step 6 — prove `ORDER BY` is free.**

```sql
EXPLAIN (ANALYZE) SELECT user_id FROM orders ORDER BY user_id LIMIT 20;
```
```
Limit  (actual time=0.030..0.034 rows=20 loops=1)
  ->  Index Only Scan using idx_orders_user_id on orders
        Heap Fetches: 0
Execution Time: 0.048 ms
```
```sql
EXPLAIN (ANALYZE) SELECT created_at FROM orders ORDER BY created_at LIMIT 20;
```
```
Limit  (actual time=1284.2..1284.2 rows=20 loops=1)
  ->  Sort  (actual time=1284.2..1284.2 rows=20 loops=1)
        Sort Method: top-N heapsort  Memory: 27kB
        ->  Seq Scan on orders (actual rows=3000000)
Execution Time: 1284.9 ms
```
**0.048 ms vs 1,285 ms — 26,700×.** Identical `LIMIT 20`. The first streams off the sorted leaf level; the second must read all 3M rows before it can know which 20 come first.

---

## Example 2 — production scenario

**The situation.** A messaging platform. `messages` table, 2.1 billion rows, `PRIMARY KEY (id uuid)` generated with `uuid_generate_v4()`. Over 14 months, insert latency p99 has climbed from 3 ms to 480 ms. Write rate is flat at 8,000/s. Nobody changed anything.

**Step 1 — measure the index.**

```sql
SELECT pg_size_pretty(pg_relation_size('messages')) AS heap,
       pg_size_pretty(pg_relation_size('messages_pkey')) AS pkey;
```
```
  heap   |  pkey
---------+---------
 780 GB  | 412 GB    ⚠ the primary key index is HALF the size of the table
```
```sql
SELECT tree_level, leaf_pages, avg_leaf_density, leaf_fragmentation
FROM pgstatindex('messages_pkey');
```
```
 tree_level | leaf_pages | avg_leaf_density | leaf_fragmentation
------------+------------+------------------+--------------------
          4 |   51000000 |            57.31 |              52.44
```

**Read those numbers:**
- `avg_leaf_density 57%` — nearly half of the 412 GB is empty space left by 50/50 splits.
- `leaf_fragmentation 52%` — leaves are physically out of order, so even a range scan does random I/O.
- `tree_level 4` — a 5-level tree. At 305 fanout (uuid keys), a well-packed index would be 4 levels.

**Step 2 — why it degraded over time, not from day one.**

```
 MONTH 1:  index = 12 GB.  Buffer pool = 64 GB.
           ⇒ The ENTIRE index is cached. A random insert touches a
             random leaf, but that leaf is in RAM. Cost: ~0 disk reads.
             p99 = 3 ms. Everything is fine.

 MONTH 8:  index = 180 GB. Buffer pool = 64 GB.
           ⇒ ~35% of the index fits. A random insert has a 65% chance
             of a cold leaf → one random 8 KB disk read, then a dirty
             page to write back.
             p99 = 45 ms.

 MONTH 14: index = 412 GB. Buffer pool = 64 GB.
           ⇒ 15% cached. Nearly EVERY insert is:
                1 random read (find the leaf)
              + 1 random read (its parent, maybe)
              + 1–3 page writes (the leaf, and its split partner,
                                 and the parent)
             At 8,000 inserts/s that is ~24,000 random IOPS just for
             the primary key.
             p99 = 480 ms.

 ★ THIS IS A CLIFF, NOT A SLOPE. Nothing changed in the code. The
   index crossed the RAM boundary, and random access to a structure
   larger than RAM is a completely different performance regime.
```

**Step 3 — confirm it's the index, not the heap.**

```sql
SELECT relname, heap_blks_read, idx_blks_read,
       round(100.0*idx_blks_hit/nullif(idx_blks_hit+idx_blks_read,0),1) AS idx_hit_pct
FROM pg_statio_user_tables WHERE relname='messages';
```
```
  relname  | heap_blks_read | idx_blks_read | idx_hit_pct
-----------+----------------+---------------+-------------
 messages  |     1200000000 |   41000000000 |        14.2
```
**Index reads are 34× heap reads, at a 14% hit rate.** The heap is fine (inserts append). The B-tree is the entire problem.

**Step 4 — the fix, and the honest trade-offs.**

| Option | What it does | Effect | Cost |
|---|---|---|---|
| **1. REINDEX CONCURRENTLY** | Rebuild packed to 90% | 412 GB → 236 GB; density 57% → 90%; buys ~8 months | Immediate, no downtime; but the *cause* is untouched and it will drift back |
| **2. Switch new IDs to UUIDv7** | Time-ordered → inserts append to the rightmost leaf | Inserts return to fastpath: no random read, 90/10 splits. p99 → ~4 ms for new rows | Existing 2.1B rows keep v4 keys; the index stays mixed for a long time. Still the right move — do it today (Topic 22) |
| **3. PARTITION by month** | Each partition has its own small index | **The active partition's index is ~4 GB — it fits in RAM.** Old partitions are never inserted into, so their indexes are read-only and cold-but-fine | The real fix. Requires a migration, and the PK must include the partition key (Topic 59) |
| **4. Bigger instance** | 64 GB → 512 GB RAM | Buys ~14 more months | £££/month forever, and the cliff returns |

**What you actually ship, in order:**

```sql
-- WEEK 1 — stop the bleeding, no schema change
REINDEX INDEX CONCURRENTLY messages_pkey;
-- 412 GB → 236 GB. p99 480 ms → 120 ms. Buys time.

-- WEEK 1 — stop making it worse
ALTER TABLE messages ALTER COLUMN id SET DEFAULT uuidv7();
-- new rows append. (PG18 has uuidv7() built in; before that, an
-- extension or application-side ULID.)

-- WEEK 2–6 — the actual fix
CREATE TABLE messages_new (
  id          uuid        NOT NULL DEFAULT uuidv7(),
  sent_at     timestamptz NOT NULL,
  conversation_id bigint  NOT NULL,
  sender_id   bigint      NOT NULL,
  body        text        NOT NULL,
  PRIMARY KEY (sent_at, id)          -- ★ partition key MUST be in the PK
) PARTITION BY RANGE (sent_at);
-- monthly partitions; backfill with pg_partman + batched copy;
-- dual-write during migration; cut over; drop the old table.
```

**Result:** active-partition index = 4.2 GB, fully cached. Insert p99 = 2.8 ms — better than month 1, at 20× the data. Retention becomes `DROP PARTITION` instead of a `DELETE` that would have generated 2 billion dead tuples.

**The lesson:** the problem was never "PostgreSQL can't handle 2 billion rows." It was **one B-tree, larger than RAM, receiving random inserts.** Two of the three fixes are about making the *active* portion of the tree small enough to stay cached.

---

## Common mistakes

**1. Random UUIDs as primary keys at scale.**
- *Symptom:* insert latency degrades gradually then cliffs; index 2× expected size.
- *Engine-level why:* random insertion → cold random leaf → read-modify-write → 50/50 split → 57% density. Once the index exceeds RAM, every insert is a random disk read.
- *Diagnose:* `SELECT avg_leaf_density, leaf_fragmentation FROM pgstatindex('...');` — below 70% and above 30% respectively.
- *Fix:* UUIDv7/ULID for new rows; partition; `REINDEX CONCURRENTLY` to reclaim. (Topic 22.)

**2. Indexing wide text columns.**
- *Symptom:* huge index, extra tree level, occasionally `ERROR: index row size exceeds maximum`.
- *Engine-level why:* entry size drives fanout drives height. A 200-byte key gives fanout ~35 and a 6-level tree.
- *Diagnose:* `SELECT avg_item_size FROM bt_page_stats('idx', <leaf_block>);`
- *Fix:* index `md5(col)` or `left(col, 32)` for equality; use GIN + `pg_trgm` for substring search; or a surrogate integer key.

**3. Expecting `CREATE INDEX` and incremental inserts to give the same index.**
- *Symptom:* a rebuilt index is 40% smaller than the one that grew.
- *Engine-level why:* `CREATE INDEX` sorts all keys first and packs leaves to fillfactor. Incremental inserts split pages 50/50.
- *Fix:* for bulk loads, **insert first, create indexes after**. It's faster *and* produces a better index. And schedule periodic `REINDEX CONCURRENTLY` on random-key indexes.

**4. Setting `fillfactor = 100` on an index that receives random inserts.**
- *Symptom:* insert performance worse than default.
- *Engine-level why:* no free space in any leaf → every insert into an existing range causes a split.
- *Fix:* 100 is correct only for append-only, time-ordered indexes. Random-insert indexes want 70–90.

**5. Assuming an index scan returns rows in physical order.**
- *Symptom:* an index scan on 200k rows is slower than a seq scan on 3M.
- *Engine-level why:* leaf order is *key* order; the TIDs point all over the heap. 200k random page reads at ~4× the cost of a sequential read ≈ 800k sequential-equivalents > 40k pages.
- *Fix:* this is exactly what a **bitmap heap scan** solves — and the planner picks it automatically. If it isn't, check `random_page_cost` (Topic 07) and your statistics (Topic 15).

**6. Not realising `ORDER BY` direction must match — or be exactly reversed.**
- *Symptom:* an index on `(a ASC, b ASC)` doesn't help `ORDER BY a ASC, b DESC`.
- *Engine-level why:* the leaf list can be walked forward or backward, giving `(a ASC, b ASC)` or `(a DESC, b DESC)`. Mixed directions are neither.
- *Fix:* `CREATE INDEX ON t (a ASC, b DESC);` (Topic 14.)

---

## Hands-on proof

**PROVE IT #1 — height is logarithmic.**
```sql
CREATE TABLE h1 (k bigint); INSERT INTO h1 SELECT generate_series(1,1000);
CREATE TABLE h2 (k bigint); INSERT INTO h2 SELECT generate_series(1,1000000);
CREATE TABLE h3 (k bigint); INSERT INTO h3 SELECT generate_series(1,50000000);
CREATE INDEX ON h1(k); CREATE INDEX ON h2(k); CREATE INDEX ON h3(k);
SELECT 'h1' t, level FROM bt_metap('h1_k_idx')
UNION ALL SELECT 'h2', level FROM bt_metap('h2_k_idx')
UNION ALL SELECT 'h3', level FROM bt_metap('h3_k_idx');
```
```
 t  | level
----+-------
 h1 |     0        ← 1,000 rows: root IS the leaf
 h2 |     2        ← 1M rows: 3 levels
 h3 |     3        ← 50M rows: 4 levels
```
**50,000× the data, 3 extra page reads.**

**PROVE IT #2 — fanout depends on key width.**
```sql
CREATE TABLE k_int  (k int);     INSERT INTO k_int  SELECT generate_series(1,1000000);
CREATE TABLE k_uuid (k uuid);    INSERT INTO k_uuid SELECT gen_random_uuid() FROM generate_series(1,1000000);
CREATE TABLE k_text (k text);    INSERT INTO k_text SELECT repeat(md5(i::text),3) FROM generate_series(1,1000000) i;
CREATE INDEX ON k_int(k); CREATE INDEX ON k_uuid(k); CREATE INDEX ON k_text(k);

SELECT c.relname, i.tree_level, pg_size_pretty(i.index_size::bigint) AS size,
       round(i.leaf_pages::numeric, 0) AS leaves
FROM pg_class c, LATERAL pgstatindex(c.oid) i
WHERE c.relname IN ('k_int_k_idx','k_uuid_k_idx','k_text_k_idx');
```
```
   relname     | tree_level |  size   | leaves
---------------+------------+---------+--------
 k_int_k_idx   |          1 | 21 MB   |   2745
 k_uuid_k_idx  |          2 | 44 MB   |   4885
 k_text_k_idx  |          2 | 194 MB  |  24010
```
**Same row count. 9× the index size for a 96-byte key, and an extra level.**

**PROVE IT #3 — sequential vs random insert, with density.**
```sql
CREATE TABLE seq_i (k bigint); CREATE INDEX ON seq_i(k);
CREATE TABLE rnd_i (k bigint); CREATE INDEX ON rnd_i(k);
\timing on
INSERT INTO seq_i SELECT generate_series(1,2000000);
INSERT INTO rnd_i SELECT (random()*1000000000)::bigint FROM generate_series(1,2000000);

SELECT c.relname, i.avg_leaf_density, i.leaf_fragmentation,
       pg_size_pretty(i.index_size::bigint)
FROM pg_class c, LATERAL pgstatindex(c.oid) i
WHERE c.relname IN ('seq_i_k_idx','rnd_i_k_idx');
```
```
   relname    | avg_leaf_density | leaf_fragmentation | pg_size_pretty
--------------+------------------+--------------------+----------------
 seq_i_k_idx  |            89.94 |               0.00 | 43 MB
 rnd_i_k_idx  |            63.87 |              46.21 | 61 MB
```

**PROVE IT #4 — watch a page split happen.**
```sql
CREATE TABLE sp (k int); CREATE INDEX ON sp(k);
INSERT INTO sp SELECT generate_series(1, 366);         -- fills one leaf
SELECT blkno, type, live_items, free_size FROM bt_page_stats('sp_k_idx', 1);
-- 1 | l | 366 | 12          ← nearly full

INSERT INTO sp VALUES (183);                            -- forces an interior split
SELECT blkno, type, live_items, free_size, btpo_prev, btpo_next
FROM bt_page_stats('sp_k_idx', 1)
UNION ALL SELECT blkno, type, live_items, free_size, btpo_prev, btpo_next
FROM bt_page_stats('sp_k_idx', 2);
```
```
 blkno | type | live_items | free_size | btpo_prev | btpo_next
-------+------+------------+-----------+-----------+-----------
     1 | l    |        184 |      3660 |         0 |         2
     2 | l    |        183 |      3676 |         1 |         0
```
**One page became two, each ~50% full, linked by sibling pointers. The split, observed.**

**PROVE IT #5 — internal nodes hold no TIDs (the "+" in B+tree).**
```sql
SELECT bt.itemoffset, bt.ctid, bt.data
FROM bt_metap('idx_orders_user_id') m,
     LATERAL bt_page_items('idx_orders_user_id', m.root) bt LIMIT 3;
-- ctid values here are (block, 0) — CHILD POINTERS, offset always 0.
-- Compare a leaf: offsets are real heap positions like (18820,29).
```

**PROVE IT #6 — the ORDER BY win (Example 1, step 6).** Run both `LIMIT 20` queries and compare. This is the single most valuable practical consequence of the leaf-level linked list.

---

## The design decision framework

```
USE A B-TREE WHEN:  (it is the default for a reason)
  ✓ Equality lookups            WHERE user_id = 4471
  ✓ Range queries               WHERE created_at > X AND < Y
  ✓ Sorting                     ORDER BY created_at DESC
  ✓ Prefix matching             WHERE name LIKE 'Basm%'
  ✓ Multi-column with a leading prefix
  ✓ Uniqueness enforcement
  ✓ MIN/MAX (one descent to an edge)
  → i.e. ~95% of all indexing needs

CHOOSE A DIFFERENT ACCESS METHOD WHEN:
  ✗ Only equality, and the key is huge      → hash    (Topic 16)
  ✗ Containment in arrays/JSONB/text        → GIN     (Topic 16)
  ✗ Geometry, ranges, exclusion constraints → GiST    (Topics 16, 24)
  ✗ Huge, naturally ordered, low-precision  → BRIN    (Topic 16)

KEY DESIGN — the four rules
  1. TIME-ORDERED KEYS for anything high-volume.
     bigserial · UUIDv7 · ULID.   Never uuid v4 as a PK at scale.
  2. NARROW KEYS. Every byte reduces fanout. Prefer bigint to text;
     hash long text if you must index it.
  3. INDEX AFTER BULK LOAD, not before. Sorted build → 90% density,
     and it's faster.
  4. PUT THE COLUMN YOU RANGE ON LAST in a composite. (Topic 14.)

FILLFACTOR
  100  → append-only, time-ordered keys (no splits happen anyway)
   90  → default; fine for most
   70–80 → heavy random inserts into an existing key range

MAINTENANCE
  REINDEX CONCURRENTLY when:
    ✓ avg_leaf_density < 70%
    ✓ leaf_fragmentation > 30%
    ✓ index_size ≫ (rows × entry_size × 1.15)
  Schedule it for random-key indexes; append-only ones rarely need it.

THE SIGNAL TO LOOK FOR:
      SELECT c.relname, i.tree_level, i.avg_leaf_density,
             i.leaf_fragmentation, pg_size_pretty(i.index_size::bigint)
      FROM pg_class c, LATERAL pgstatindex(c.oid) i
      WHERE c.relkind='i' AND pg_relation_size(c.oid) > 100*1024*1024
      ORDER BY i.avg_leaf_density ASC;

  • avg_leaf_density < 70%  → bloat. REINDEX, and ask why (random keys?)
  • tree_level ≥ 4 AND index > RAM → you are approaching the cliff.
    PARTITION before it arrives, not after.
  • The cliff question, asked explicitly:
        index_size  vs  shared_buffers
    An index larger than RAM receiving RANDOM writes is a different
    performance regime. Design so the ACTIVE index stays smaller
    than RAM — that is what partitioning buys you.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Create indexes on `int`, `bigint`, `uuid`, and `text` (60 chars) columns over 1M rows each. For each, report from `bt_metap` and `pgstatindex`: tree level, leaf pages, index size, `avg_leaf_density`. Then compute the theoretical fanout for each key type by hand and check it against `leaf_pages × avg_item_size`. Explain any discrepancy.

### Exercise 2 — medium (apply it)
Build two 2M-row tables — one with a `bigserial` PK, one with a `uuid v4` PK — inserting rows one batch at a time (not `CREATE INDEX` afterwards). Then:
(a) report insert time, index size, `avg_leaf_density`, `leaf_fragmentation` for both,
(b) `REINDEX` both and report again; explain why one changed far more,
(c) set `fillfactor = 100` on the sequential one, rebuild, and explain what happens and why it's safe there but not for the random one,
(d) predict what UUIDv7 would give and justify from the split algorithm.

### Exercise 3 — hard (production simulation)
A `events` table has 4.2 billion rows and a `PRIMARY KEY (id uuid)` using v4. The index is 780 GB; the server has 128 GB RAM (`shared_buffers = 32 GB`). Insert p99 is 620 ms at 12,000 inserts/s. Query patterns are: (a) `WHERE id = ?` at 40k/s, (b) `WHERE occurred_at BETWEEN ? AND ?` at 200/s, (c) retention delete of data older than 180 days, nightly.

(a) Compute the tree height, expected `avg_leaf_density`, and the number of random IOPS the insert workload requires. Show your working.
(b) Explain precisely why p99 was 4 ms at 400M rows and is 620 ms at 4.2B, when the insert rate never changed. Your answer must identify the exact threshold that was crossed.
(c) Design the fix. Specify: the new key strategy, the partitioning scheme, what the PK becomes and why, the active index size, and the new IOPS estimate.
(d) The nightly retention delete currently takes 6 hours and generates 2 billion dead index entries. Explain what your design does to it, and why.
(e) You cannot take downtime and the table is 6 TB. Describe the migration: how you backfill, how you dual-write, how you cut over, and how you roll back if the new path misbehaves.
(f) Query (a) is a point lookup by random UUID. After partitioning by time, that query no longer knows which partition to look in. Explain the problem and give two solutions with their trade-offs.

---

## Mental model checkpoint

1. Draw a 3-level B+tree from memory. What is in an internal node? What is in a leaf? What is the extra 16 bytes of "special space" for?
2. Why is it a B**+**tree and not a B-tree? Name the two operations that get dramatically cheaper because of the `+`.
3. Compute the fanout for a `bigint` key. Now for a 100-byte `text` key. What happens to tree height at 1 billion rows?
4. Walk through a leaf page split. When is it 90/10 and when is it 50/50? Why does PostgreSQL distinguish?
5. Why does a UUIDv4 primary key produce a ~57% dense index while `bigserial` produces ~90%?
6. Explain, mechanically, why insert p99 can go from 4 ms to 480 ms over a year with a completely flat write rate.
7. Why can't a hash index replace a B-tree? Name four capabilities lost, and say why the O(1) vs O(log n) advantage barely matters.

---

## Quick reference card

| Component | Contents |
|---|---|
| Metapage (block 0) | root block, root level, fast root |
| Root | separator keys + child pointers |
| Internal node | separator keys + child pointers, **no TIDs** |
| Leaf | keys + **TIDs** + prev/next sibling pointers |
| Special space (16 B) | btpo_prev, btpo_next, btpo_level, btpo_flags |
| High key | upper bound; enables lock-free concurrent reads |

**Numbers to memorise**

| Thing | Value |
|---|---|
| Index tuple header | 8 bytes |
| Entry size, bigint key | 16 bytes |
| Fanout, bigint key | ~450–500 |
| Height, 1M rows | 3 |
| Height, 1B rows | 4–5 |
| Max index entry size | ~2,704 B (⅓ page) |
| btree `fillfactor` default | 90 |
| Healthy `avg_leaf_density` | > 70% (90% after REINDEX) |
| Healthy `leaf_fragmentation` | < 30% |
| Rightmost split ratio | 90/10 |
| Interior split ratio | 50/50 |

**The diagnostic queries**

```sql
SELECT * FROM bt_metap('idx_name');                    -- height, root
SELECT * FROM bt_page_stats('idx_name', <blk>);        -- one page
SELECT * FROM bt_page_items('idx_name', <blk>);        -- entries
SELECT * FROM pgstatindex('idx_name');                 -- density, fragmentation
REINDEX INDEX CONCURRENTLY idx_name;                   -- rebuild, no lock
```

**The three rules that matter most**
1. Time-ordered keys (bigserial / UUIDv7 / ULID). Never random UUIDs as a PK at scale.
2. Narrow keys — every byte costs fanout, height, and RAM.
3. Keep the **active** index smaller than RAM. That's what partitioning buys.

---

## When would I use this at work?

1. **Choosing a primary key in a design doc.** You can show, with `avg_leaf_density` numbers from your own database, that UUIDv4 costs 2× the index size and 5× the insert time — and that UUIDv7 gets you global uniqueness *and* the append behaviour. That ends the debate in one slide.

2. **Diagnosing gradual write degradation.** Insert p99 climbing over months with flat traffic is a signature. `pgstatindex` gives you density and level in ten seconds, and the index-size-vs-RAM comparison tells you whether you're approaching a cliff or already past it.

3. **Planning a bulk import.** Knowing that `CREATE INDEX` after loading is both faster *and* produces a 90%-dense index (vs 64% for incremental) changes the import procedure and saves 40% of the index size permanently.

---

## Connected topics

**Understand before this:** 04 (pages, special space), 07 (buffer pool — the RAM cliff), 08 (B-tree vs LSM), 10 (what an index is).

**This unlocks:**
- **12** — the lookup end to end: heap fetches, bitmap scans, index-only scans
- **13** — partial, expression, unique, covering indexes — all B-trees underneath
- **14** — composite indexes: why `(a,b)` and `(b,a)` are different trees
- **17** — index bloat and write amplification in production
- **18** — how the planner costs a B-tree descent
- **22** — primary keys: this topic *is* the argument for UUIDv7/ULID
- **59** — partitioning: keeping the active tree small enough to cache
