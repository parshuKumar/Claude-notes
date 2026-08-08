# 16 — Specialised Index Types
## Hash · GIN · GiST · SP-GiST · BRIN
## Phase: Indexes

---

## ELI5 — The Simple Analogy

The library again — but now people ask questions the alphabetical catalogue simply cannot answer.

**"Give me the book with ISBN 9780143421&nbsp;"** — you don't need alphabetical order at all. You need a formula that turns the ISBN straight into a shelf number. One calculation, one walk. That's a **hash index**.

**"Which books mention 'monsoon'?"** — the catalogue is sorted by *title*. It cannot help. What you need is the index at the *back* of every book, merged into one giant list: *monsoon → books 4, 88, 210, 4471*. One entry per **word**, pointing at many books. That's **GIN** — an inverted index.

**"Which meeting rooms are free between 2pm and 4pm on Thursday?"** — you're comparing *intervals*, not points. Sorting by start time doesn't work; a room booked 1pm–5pm sorts before 2pm but overlaps it. You need a structure that understands "overlaps." That's **GiST**.

**"Which of the 40 million shipping logs are from March?"** — the logs are in a warehouse, filed strictly by date. You don't need an index card per log. You need a note on each *shelf*: "this shelf holds 1–14 March." Skip 99% of shelves without opening them. That's **BRIN** — 40 kilobytes for a 400 gigabyte table.

Four questions a B-tree can't answer well. Four different structures.

---

## Where this fits in the big picture

```
   11 B-tree (the default, 95% of cases)
   12 lookup · 13 types · 14 order · 15 statistics
                          │
                          ▼
        ┌──────────────────────────────────────────┐
        │ 16 SPECIALISED INDEX TYPES               │ ← YOU ARE HERE
        │ when the B-tree is the wrong SHAPE       │
        └────────────────────┬─────────────────────┘
                             │
         ┌───────────────────┼────────────────────┐
         ▼                   ▼                    ▼
   24 constraints      73 time-series        74 graph & search
   (GiST EXCLUDE)      (BRIN)                (GIN, full-text)
```

Topics 13 and 14 were *modifiers* on a B-tree. **These are genuinely different data structures**, each solving a query shape a B-tree fundamentally cannot.

---

## What is this?

PostgreSQL's index framework supports pluggable **access methods**. Six ship in core:

| AM | Structure | Answers |
|---|---|---|
| **btree** | balanced sorted tree | `= < > BETWEEN` ORDER BY, prefix `LIKE` |
| **hash** | hash table | `=` only |
| **gin** | inverted index (term → doc list) | "contains": arrays, JSONB, full-text, trigrams |
| **gist** | balanced tree of *bounding predicates* | "overlaps/contains/near": ranges, geometry, exclusion |
| **spgist** | space-partitioned trees (quadtree, radix) | non-balanced partitioning: points, text prefixes, IPs |
| **brin** | per-block-range min/max summary | huge tables where physical order ≈ value order |

Choosing the right one is often a 100–10,000× difference, and frequently the difference between "possible" and "impossible."

---

## Why does it matter for a backend developer?

Because four extremely common product requirements are B-tree-hostile:

```
 1. SEARCH:     WHERE name ILIKE '%kurta%'
    B-tree: Seq Scan. A tree sorted by `name` has no prefix to descend on.
    GIN + pg_trgm: 0.4 ms.

 2. JSONB:      WHERE attributes @> '{"colour":"red","size":"M"}'
    B-tree: cannot index containment at all.
    GIN: 0.8 ms over 8M rows.

 3. BOOKING:    "no two reservations for this room may overlap"
    B-tree: impossible to enforce; application checks race.
    GiST EXCLUDE constraint: structurally impossible to violate.

 4. TIME-SERIES: 400 GB events table, always queried by time range
    B-tree on occurred_at: 12 GB index, doesn't fit in RAM.
    BRIN: 48 KB, always cached, skips 99% of blocks.
```

And the reverse mistake matters too: **hash indexes are almost never the right answer** despite sounding faster, and **BRIN is worse than useless on uncorrelated data**. Knowing when *not* to use these is half the value.

---

## The physical reality

### HASH — a bucket directory

```
 CREATE INDEX idx_h ON sessions USING hash (session_token);

 ┌─────────────────────────────────────────────────────────────┐
 │ METAPAGE  │ bucket count, high mask, low mask, overflow map  │
 ├─────────────────────────────────────────────────────────────┤
 │ BUCKET 0  │ (hash, TID) (hash, TID) ...                      │
 │ BUCKET 1  │ (hash, TID) ...            ──▶ OVERFLOW PAGE     │
 │ BUCKET 2  │ ...                                              │
 │ ...       │                                                  │
 │ BUCKET N  │                                                  │
 └─────────────────────────────────────────────────────────────┘

 LOOKUP:  h = hash(key) → bucket = h & mask → read ONE page → scan it
          ⇒ 1 page read, regardless of table size.  O(1).

 ★ Only the 32-bit HASH is stored, not the key. So:
   • the index is SMALL for wide keys (a 200-byte URL → 8 bytes)
   • it CANNOT do ranges, ORDER BY, prefixes, or uniqueness
   • it CANNOT be used for a UNIQUE constraint
 ⚠ Before PG10, hash indexes were not WAL-logged — unsafe and
   unreplicated. Everyone learned to avoid them, and the reputation
   stuck. They are crash-safe from PG10 onwards.
```

### GIN — the inverted index

```
 CREATE INDEX idx_gin ON products USING gin (attributes);

 A GIN index INVERTS the relationship. Instead of row → values,
 it stores value → rows.

 ┌────────────────────────────────────────────────────────────────┐
 │ ENTRY TREE (a B-tree over the extracted KEYS)                   │
 │   "colour:blue"  ──▶ posting list: (1204,3)(2891,7)(4102,1)     │
 │   "colour:red"   ──▶ POSTING TREE (its own B-tree of TIDs)      │
 │                       ↑ used when the list is too big for a page │
 │   "size:M"       ──▶ posting list: (88,2)(91,4)...              │
 │   "brand:Fabindia"──▶ posting list: ...                          │
 └────────────────────────────────────────────────────────────────┘

 ONE ROW PRODUCES MANY ENTRIES:
   {"colour":"red","size":"M","brand":"Fabindia"}  →  3 index entries
   'the quick brown fox' with to_tsvector          →  3 lexemes
   ARRAY[1,5,9,12]                                 →  4 entries

 QUERY `attributes @> '{"colour":"red","size":"M"}'`:
   1. look up "colour:red" → posting list A
   2. look up "size:M"     → posting list B
   3. INTERSECT A ∩ B
   4. recheck the surviving TIDs against the full condition
   ⇒ set intersection, not a tree descent

 ★ THE PENDING LIST — GIN's crucial write optimisation:
   Inserting a row with 8 keys means 8 B-tree inserts. That's brutal.
   So GIN buffers new entries in an unsorted PENDING LIST, and merges
   them in bulk during VACUUM or when the list exceeds gin_pending_list_limit.
     `fastupdate = on` (default): fast writes, but queries must ALSO
                                  scan the pending list linearly
     `fastupdate = off`:          slower writes, consistent read latency
   ⚠ A large pending list is a classic cause of "GIN queries got slow
     and nobody changed anything."
```

### GiST — a tree of bounding predicates

```
 CREATE INDEX idx_gist ON bookings USING gist (room_id, during);

 GiST is a FRAMEWORK. Each data type supplies:
   consistent()  — could this subtree contain a match?
   union()       — what predicate covers these children?
   penalty()     — cost of inserting here
   picksplit()   — how to split a full page

 For a range type, the "predicate" is a BOUNDING RANGE:

                  ┌──────────────────────────────────┐
                  │ ROOT: covers [Jan 1 .. Dec 31]   │
                  └───────┬──────────────────┬───────┘
              ┌───────────┘                  └──────────┐
              ▼                                         ▼
     ┌────────────────────┐                  ┌────────────────────┐
     │ [Jan 1 .. Jun 30]  │                  │ [Jul 1 .. Dec 31]  │
     └────┬──────────┬────┘                  └────────────────────┘
          ▼          ▼
   ┌──────────┐ ┌──────────┐
   │[Jan..Mar]│ │[Apr..Jun]│   leaves hold the actual ranges + TIDs
   └──────────┘ └──────────┘

 QUERY: which bookings overlap [Mar 12 14:00, Mar 12 16:00)?
   at each node, ask consistent(): does the child's bounding range
   overlap my query range?
     NO  → prune the entire subtree
     YES → descend
 ⇒ Unlike a B-tree, MULTIPLE children can match, so GiST may descend
   several branches. That's why it's slower than a B-tree for equality
   but able to answer questions a B-tree can't express at all.

 ★ NOT BALANCED BY VALUE, and the tree quality depends on picksplit().
   A poorly-clustered GiST index degrades badly — this is why
   `REINDEX` matters more for GiST than for B-tree.
```

### BRIN — block range summaries

```
 CREATE INDEX idx_brin ON events USING brin (occurred_at)
   WITH (pages_per_range = 128);

 The table's HEAP, 400 GB = 52,428,800 pages, grouped into ranges of 128:

  BLOCK RANGE 0    (pages     0–127)  min=2025-01-01  max=2025-01-01
  BLOCK RANGE 1    (pages   128–255)  min=2025-01-01  max=2025-01-02
  BLOCK RANGE 2    (pages   256–383)  min=2025-01-02  max=2025-01-02
  ...
  BLOCK RANGE 409k (pages ...      )  min=2026-08-07  max=2026-08-07

 THE INDEX IS JUST THAT SUMMARY:
   409,600 ranges × ~32 bytes = 13 MB          ← for a 400 GB table
   with pages_per_range=1024: 51,200 × 32 = 1.6 MB

 QUERY: WHERE occurred_at BETWEEN '2026-03-01' AND '2026-03-05'
   1. scan the tiny BRIN index (a few pages)
   2. for each range, does [min,max] overlap the query range?
        NO  → SKIP ALL 128 PAGES without reading them
        YES → add all 128 pages to a bitmap
   3. Bitmap Heap Scan the surviving pages, RECHECKING every tuple

 ⇒ LOSSY BY DESIGN. It never tells you which ROW — only which BLOCKS
   might contain one. Always a bitmap scan with a recheck.

 ★★★ THE ABSOLUTE REQUIREMENT: physical order must correlate with value
     order. If rows are inserted in time order, every range has a tight
     [min,max] and pruning is superb.
     If rows are randomly ordered, EVERY range has min=Jan 1, max=Dec 31,
     nothing can be pruned, and BRIN is worse than no index.
     → Check pg_stats.correlation. Above ~0.9: excellent. Below ~0.5: useless.
```

### SP-GiST — space-partitioned trees

```
 CREATE INDEX ON ip_ranges USING spgist (network inet_ops);
 CREATE INDEX ON places USING spgist (location);       -- quadtree
 CREATE INDEX ON urls USING spgist (url);              -- radix/suffix tree

 UNBALANCED, non-overlapping partitioning of the search space:

   QUADTREE for points          RADIX TREE for text
        ┌─────┬─────┐              root
        │  NW │ NE  │               ├─ "http"
        ├─────┼─────┤               │   ├─ "://shop.in/a"
        │  SW │ SE  │               │   └─ "://shop.in/b"
        └─────┴─────┘               └─ "ftp://..."

 vs GiST: partitions DON'T overlap, so a search descends ONE path.
   ✓ faster for point/prefix data with natural hierarchy
   ✗ unbalanced — degrades badly on skewed data
   ✗ no multicolumn support, no exclusion constraints
 → Niche. Reach for it for IP/network containment and text prefix
   search; otherwise GiST or GIN.
```

---

## How it works — step by step

### GIN query on JSONB

```
 SELECT * FROM products WHERE attributes @> '{"colour":"red","size":"M"}';

 1. EXTRACT QUERY KEYS from the operand:
      jsonb_ops (default)  → keys: "colour", "red", "size", "M"
      jsonb_path_ops       → keys: hash("colour"→"red"), hash("size"→"M")

 2. For each key, descend the ENTRY TREE (a B-tree) → posting list/tree.
      "colour" → 8,000,000 TIDs   ⚠ every row has a colour!
      "red"    →   400,112 TIDs
      "size"   → 8,000,000 TIDs   ⚠
      "M"      → 1,204,881 TIDs

 3. INTERSECT all four → ~62,000 TIDs.

 4. Build a bitmap, Bitmap Heap Scan, RECHECK the full @> condition
    (necessary because jsonb_ops keys don't preserve key→value pairing —
     a row with colour:blue, size:red would match all four keys!).

 ★ THIS IS WHY jsonb_path_ops IS USUALLY BETTER FOR @>:
     jsonb_ops:      indexes keys AND values separately
                     → bigger index, more false positives, but supports
                       ?, ?|, ?& (key-existence) operators
     jsonb_path_ops: indexes hashed key→value PATHS
                     → ~40% smaller, far fewer false positives,
                       but ONLY supports @>
   Measured on 8M rows: jsonb_ops 1,240 MB / 41 ms
                        jsonb_path_ops 712 MB / 8 ms
```

### GiST exclusion constraint

```
 ALTER TABLE bookings ADD CONSTRAINT no_overlap
   EXCLUDE USING gist (room_id WITH =, during WITH &&);

 INSERT INTO bookings (room_id, during)
   VALUES (12, '[2026-03-12 14:00, 2026-03-12 16:00)');

 1. Build the key: (room_id=12, during=[14:00,16:00)).
 2. Descend the GiST index, calling consistent() at each node:
      "does any child's bounding box have room_id=12 AND an
       overlapping time range?"
 3. Collect candidate rows; for each, evaluate the actual operators:
      existing.room_id = 12  AND  existing.during && [14:00,16:00)
 4. If ANY match is VISIBLE or from an IN-PROGRESS transaction:
      in-progress → WAIT on it, then re-check
      committed   → ERROR: conflicting key value violates exclusion
                    constraint "no_overlap"
 5. Otherwise insert.

 ★ ALL OF THIS HAPPENS INSIDE THE INDEX INSERT, under the appropriate
   page locks. There is NO WINDOW between check and insert — which is
   precisely why this beats any application-level check under
   concurrency. (Case study 02.)
```

### BRIN scan

```
 SELECT count(*) FROM events WHERE occurred_at >= '2026-03-01'
                               AND occurred_at <  '2026-03-08';

 1. Read the BRIN index (13 MB, always cached): 409,600 range summaries.
 2. For each: does [min_val, max_val] overlap the query range?
      range 0     [2025-01-01, 2025-01-01]  → no  → skip 128 pages
      ...
      range 291k  [2026-02-28, 2026-03-01]  → YES → add 128 pages
      range 291k+1[2026-03-01, 2026-03-01]  → YES → add 128 pages
      ...
      range 298k  [2026-03-08, 2026-03-09]  → YES → add 128 pages
      ...
      range 409k  [2026-08-07, 2026-08-07]  → no  → skip
 3. Bitmap: ~7,000 ranges × 128 pages = 896,000 pages of 52,428,800.
    ⇒ 98.3% of the table SKIPPED.
 4. Bitmap Heap Scan reads those pages and RECHECKS every tuple
    (BRIN is lossy — it only knows blocks, never rows).

 ⇒ 400 GB table, 13 MB index, 7 GB read instead of 400 GB.
   A B-tree index on the same column would be ~12 GB — a thousand
   times larger — and give exact rows rather than blocks.
   ★ THE TRADE: BRIN is 1000× smaller and reads more heap. On a table
     far larger than RAM, that trade is overwhelmingly worth it.

 ⚠ SUMMARISATION IS NOT AUTOMATIC FOR NEW ROWS.
   New pages beyond the last summarised range are ALWAYS scanned
   (treated as "might match"). Fix:
     • autosummarize = on  (PG10+, per-index option — NOT the default)
     • or run brin_summarize_new_values('idx') periodically
```

---

## Concept breakdown

```
HASH
├── O(1) equality only. Stores the hash, not the key.
├── ✓ small for wide keys; ✓ crash-safe since PG10
├── ✗ no ranges, no ORDER BY, no UNIQUE, no multicolumn, no covering
└── ⇒ Marginal. A B-tree does equality in 3–4 cached page reads and does
     everything else too. Use hash only for very wide equality-only keys.

GIN — Generalised INverted iNdex
├── One row → MANY index entries (one per extracted key)
├── ✓ arrays, JSONB, full-text, trigrams, hstore
├── ✓ superb for "contains" and multi-term intersection
├── ✗ SLOW WRITES (N entries per row) — mitigated by the pending list
├── ✗ no ordering, no range scans
└── ⇒ The default choice for any containment/search query.

GiST — Generalised Search Tree
├── A FRAMEWORK: types supply consistent/union/penalty/picksplit
├── ✓ ranges (&&, @>), geometry, nearest-neighbour (ORDER BY <->)
├── ✓ THE ONLY WAY to build an EXCLUSION CONSTRAINT
├── ✗ lossy — always needs a recheck; slower than B-tree for equality
└── ⇒ Reach for it when the question is about OVERLAP, CONTAINMENT,
     or DISTANCE.

SP-GiST — Space-Partitioned GiST
├── Unbalanced, NON-overlapping partitioning (quadtree, radix, k-d tree)
├── ✓ IP/network containment (inet_ops), text prefix, point data
└── ⇒ Niche. Try it when GiST is slow on point or prefix data.

BRIN — Block Range INdex
├── min/max (or bloom, or inclusion) summary per range of heap pages
├── ✓ TINY — kilobytes for terabytes
├── ✓ near-zero write cost
├── ★ REQUIRES physical ↔ value correlation (pg_stats.correlation)
├── ✗ lossy: identifies BLOCKS, not rows; always a bitmap scan + recheck
├── ✗ useless (worse than nothing) on uncorrelated data
└── ⇒ Append-only, time-ordered, huge tables. The time-series index.

THE DECISION IN ONE LINE PER TYPE
   "is this value equal to X?"            → btree
   "does this document CONTAIN X?"        → gin
   "does this range OVERLAP X?"           → gist
   "is this point NEAR X?"                → gist (KNN)
   "which BLOCKS could contain X?"        → brin
   "is this IP inside this subnet?"       → spgist
```

---

## Diagrams

**Diagram 1 — big picture: the decision tree**

```
                    WHAT SHAPE IS THE QUESTION?
                              │
    ┌─────────────┬───────────┼────────────┬──────────────┐
    ▼             ▼           ▼            ▼              ▼
 "equals /    "contains"  "overlaps /  "near"        "which blocks?"
  between /                contains
  ordered"                 a range"
    │             │           │            │              │
    ▼             ▼           ▼            ▼              ▼
 ┌──────┐    ┌────────┐  ┌────────┐  ┌────────┐   ┌────────────┐
 │BTREE │    │  GIN   │  │  GiST  │  │  GiST  │   │   BRIN     │
 │      │    │        │  │        │  │  (KNN) │   │            │
 │ 95%  │    │ JSONB  │  │ ranges │  │ <-> op │   │ huge +     │
 │ of   │    │ arrays │  │ geo    │  │        │   │ correlated │
 │ cases│    │ FTS    │  │ EXCLUDE│  │        │   │            │
 │      │    │ trgm   │  │        │  │        │   │            │
 └──────┘    └────────┘  └────────┘  └────────┘   └────────────┘
                                                          │
                                                    is correlation
                                                      > 0.9 ?
                                                    NO → don't.
```

**Diagram 2 — data flow: B-tree vs GIN for the same table**

```
  TABLE products, row: {"colour":"red","size":"M","brand":"Fabindia"}

  B-TREE on attributes                GIN on attributes
  ────────────────────────            ────────────────────────────
  ONE entry per row:                  THREE entries per row:
   [whole jsonb value] → TID           "colour"    → …(1204,3)…
                                       "red"       → …(1204,3)…
  ⇒ only usable for                    "size"      → …(1204,3)…
    attributes = '<exact>'             "M"         → …(1204,3)…
    (the entire document)              "brand"     → …(1204,3)…
                                       "Fabindia"  → …(1204,3)…
  ⇒ USELESS for @>
                                      ⇒ @> works: intersect posting lists
                                      ⇒ WRITE COST: 6 B-tree inserts per row
                                        (buffered in the pending list)
```

**Diagram 3 — before/after: BRIN and correlation**

```
 CORRELATED TABLE (inserted in time order, correlation = 0.999)
 heap pages:   [Jan][Jan][Feb][Feb][Mar][Mar][Apr][Apr][May][May]
 BRIN ranges:  └min=Jan┘   └min=Feb┘   └min=Mar┘  └min=Apr┘ └min=May┘
                max=Jan     max=Feb     max=Mar    max=Apr   max=May

 WHERE occurred_at IN March  →  skip 8 of 10 ranges. 80% pruned. ✓✓✓


 UNCORRELATED TABLE (random insert order, correlation = 0.001)
 heap pages:   [Mar][Jan][May][Feb][Apr][Jan][Mar][May][Feb][Apr]
 BRIN ranges:  └min=Jan┘   └min=Jan┘   └min=Jan┘  └min=Jan┘ └min=Jan┘
                max=May     max=May     max=May    max=May   max=May

 WHERE occurred_at IN March  →  EVERY range overlaps. 0% pruned.
                                 Full scan PLUS the index read. ✗✗✗

 ★ SAME index definition. SAME data. The ONLY difference is physical
   row order — which is why `SELECT correlation FROM pg_stats` is the
   first thing to check before creating a BRIN index.
```

---

## Example 1 — basic

```sql
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE EXTENSION IF NOT EXISTS btree_gist;

CREATE TABLE products (
  id bigserial PRIMARY KEY,
  name text NOT NULL,
  description text NOT NULL,
  attributes jsonb NOT NULL,
  tags text[] NOT NULL,
  price_paise bigint NOT NULL
);

INSERT INTO products (name, description, attributes, tags, price_paise)
SELECT
  (ARRAY['Cotton Kurta','Silk Saree','Denim Jeans','Leather Jacket','Wool Shawl'])[1+(i%5)]
    || ' ' || i,
  'A comfortable garment made from natural fibres, item ' || i,
  jsonb_build_object(
    'colour', (ARRAY['red','blue','green','black','white'])[1+(i%5)],
    'size',   (ARRAY['S','M','L','XL'])[1+(i%4)],
    'brand',  (ARRAY['Fabindia','W','Biba','Manyavar'])[1+(i%4)]),
  ARRAY[(ARRAY['ethnic','casual','formal'])[1+(i%3)],
        (ARRAY['sale','new','bestseller'])[1+(i%3)]],
  (random()*500000)::bigint
FROM generate_series(1,8000000) i;
VACUUM ANALYZE products;
```

**Step 1 — B-tree cannot do substring search.**

```sql
CREATE INDEX idx_name_btree ON products (name);
VACUUM ANALYZE products;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM products WHERE name LIKE 'Cotton%';
--  Index Scan using idx_name_btree ...  Buffers: shared hit=1204   ✓ prefix works

EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM products WHERE name ILIKE '%Kurta%';
```
```
Seq Scan on products  (actual time=0.4..3841.2 rows=1600000 loops=1)
  Filter: (name ~~* '%Kurta%'::text)
  Buffers: shared hit=2104 read=188902
Execution Time: 3902.1 ms
```

```sql
CREATE INDEX idx_name_trgm ON products USING gin (name gin_trgm_ops);
VACUUM ANALYZE products;
EXPLAIN (ANALYZE, BUFFERS) SELECT * FROM products WHERE name ILIKE '%Kurta%' LIMIT 20;
```
```
Limit  (actual time=8.2..8.4 rows=20 loops=1)
  ->  Bitmap Heap Scan on products (actual rows=20 loops=1)
        Recheck Cond: (name ~~* '%Kurta%'::text)
        ->  Bitmap Index Scan on idx_name_trgm (actual time=8.1..8.1 rows=1600000)
  Buffers: shared hit=1841
Execution Time: 8.5 ms
```
**3,902 ms → 8.5 ms.** Trigram GIN splits `'Cotton Kurta'` into `' co','cot','ott','tto','ton','on ',...` and indexes each — so any substring of ≥3 characters is findable.

**Step 2 — JSONB containment, and the two operator classes.**

```sql
CREATE INDEX idx_attr_ops  ON products USING gin (attributes);                 -- jsonb_ops
CREATE INDEX idx_attr_path ON products USING gin (attributes jsonb_path_ops);
VACUUM ANALYZE products;

SELECT relname, pg_size_pretty(pg_relation_size(oid)) FROM pg_class
WHERE relname IN ('idx_attr_ops','idx_attr_path');
```
```
    relname     | pg_size_pretty
----------------+----------------
 idx_attr_ops   | 1240 MB
 idx_attr_path  | 712 MB           ← 43% smaller
```
```sql
DROP INDEX idx_attr_path;
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM products
WHERE attributes @> '{"colour":"red","size":"M"}';
--  Execution Time: 41.2 ms   Buffers: shared hit=18402

CREATE INDEX idx_attr_path ON products USING gin (attributes jsonb_path_ops);
DROP INDEX idx_attr_ops;
VACUUM ANALYZE products;
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM products
WHERE attributes @> '{"colour":"red","size":"M"}';
--  Execution Time: 8.1 ms    Buffers: shared hit=4102
```
**5× faster, 43% smaller.** The trade: `jsonb_path_ops` supports only `@>`, not `?`/`?|`/`?&`.

**A third option worth knowing:** if you always query one key, an **expression B-tree** beats GIN entirely.

```sql
CREATE INDEX idx_colour ON products ((attributes->>'colour'));
SELECT pg_size_pretty(pg_relation_size('idx_colour'));    -- 184 MB
EXPLAIN (ANALYZE) SELECT count(*) FROM products WHERE attributes->>'colour' = 'red';
--  Execution Time: 2.1 ms
```
**184 MB and 2.1 ms** vs 712 MB and 8.1 ms. GIN's value is *flexibility* — arbitrary key combinations. If you know the key, a B-tree expression index wins on every axis.

**Step 3 — arrays.**

```sql
CREATE INDEX idx_tags ON products USING gin (tags);
VACUUM ANALYZE products;
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM products
WHERE tags @> ARRAY['ethnic','sale'];
--  Bitmap Index Scan on idx_tags   Execution Time: 88.4 ms  (vs 3,100 ms Seq Scan)
```

**Step 4 — GiST exclusion constraint.**

```sql
CREATE TABLE room_bookings (
  id bigserial PRIMARY KEY,
  room_id bigint NOT NULL,
  during tstzrange NOT NULL,
  booked_by bigint NOT NULL,
  EXCLUDE USING gist (room_id WITH =, during WITH &&)
);

INSERT INTO room_bookings (room_id, during, booked_by)
  VALUES (12, '[2026-03-12 14:00, 2026-03-12 16:00)', 7);      -- ok
INSERT INTO room_bookings (room_id, during, booked_by)
  VALUES (12, '[2026-03-12 16:00, 2026-03-12 18:00)', 8);      -- ok — touching, not overlapping
INSERT INTO room_bookings (room_id, during, booked_by)
  VALUES (12, '[2026-03-12 15:00, 2026-03-12 17:00)', 9);
```
```
ERROR:  conflicting key value violates exclusion constraint
        "room_bookings_room_id_during_excl"
DETAIL:  Key (room_id, during)=(12, ["2026-03-12 15:00:00+00","2026-03-12 17:00:00+00"))
         conflicts with existing key
         (room_id, during)=(12, ["2026-03-12 14:00:00+00","2026-03-12 16:00:00+00")).
```

Note `[...)` — half-open ranges. `[14:00, 16:00)` and `[16:00, 18:00)` do **not** overlap. That's why every range in this curriculum uses `'[)'` bounds: it makes adjacency work correctly with no off-by-one.

**Step 5 — BRIN, and the correlation requirement.**

```sql
CREATE TABLE events_ordered AS
  SELECT i AS id, now() - (i || ' seconds')::interval AS occurred_at, repeat('x',100) AS pad
  FROM generate_series(1, 40000000) i;

CREATE TABLE events_random AS
  SELECT * FROM events_ordered ORDER BY random();

ANALYZE events_ordered; ANALYZE events_random;
SELECT tablename, correlation FROM pg_stats WHERE attname='occurred_at';
```
```
    tablename    | correlation
-----------------+-------------
 events_ordered  |      -1.000     ← perfect (descending)
 events_random   |       0.0004    ← none
```

```sql
CREATE INDEX idx_bo ON events_ordered USING brin (occurred_at);
CREATE INDEX idx_br ON events_random  USING brin (occurred_at);
CREATE INDEX idx_bt ON events_ordered (occurred_at);          -- for comparison

SELECT relname, pg_size_pretty(pg_relation_size(oid)) FROM pg_class
WHERE relname IN ('idx_bo','idx_br','idx_bt');
```
```
 relname | pg_size_pretty
---------+----------------
 idx_bo  | 176 kB          ← BRIN
 idx_br  | 176 kB          ← BRIN (same size, useless)
 idx_bt  | 857 MB          ← B-tree: 4,900× larger
```

```sql
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM events_ordered
WHERE occurred_at BETWEEN now() - interval '10 days' AND now() - interval '9 days';
```
```
Aggregate  (actual time=88.4..88.4 rows=1 loops=1)
  ->  Bitmap Heap Scan on events_ordered (actual rows=86400 loops=1)
        Rows Removed by Index Recheck: 6944
        Heap Blocks: lossy=1408
        ->  Bitmap Index Scan on idx_bo (actual time=0.9..0.9 rows=14080)
  Buffers: shared hit=1421
Execution Time: 89.1 ms
```
```sql
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM events_random
WHERE occurred_at BETWEEN now() - interval '10 days' AND now() - interval '9 days';
```
```
Aggregate  (actual time=8412.1..8412.1 rows=1 loops=1)
  ->  Bitmap Heap Scan on events_random (actual rows=86400 loops=1)
        Rows Removed by Index Recheck: 39913600           ← ⚠ ALL of them
        Heap Blocks: lossy=740741
  Buffers: shared hit=2104 read=738712
Execution Time: 8413.4 ms
```
**Same index, same query, same data. 89 ms vs 8,413 ms — 94×.** In the random case BRIN pruned nothing and added its own read on top of a full scan.

**Step 6 — BRIN summarisation of new data.**

```sql
INSERT INTO events_ordered SELECT 40000000+i, now() + (i||' seconds')::interval, repeat('x',100)
FROM generate_series(1,2000000) i;

EXPLAIN (ANALYZE) SELECT count(*) FROM events_ordered WHERE occurred_at > now();
--  Heap Blocks: lossy=740741        ← the new pages aren't summarised: all scanned

SELECT brin_summarize_new_values('idx_bo');
EXPLAIN (ANALYZE) SELECT count(*) FROM events_ordered WHERE occurred_at > now();
--  Heap Blocks: lossy=36000         ← now pruned

-- and make it automatic:
ALTER INDEX idx_bo SET (autosummarize = on);
```

---

## Example 2 — production scenario

**The situation.** A logistics platform. Three requirements land in one quarter, and each one is B-tree-hostile.

```
 R1: Parcel search — operators type any fragment of a tracking number,
     recipient name, or address. 4,000 searches/s. 900M parcels.
 R2: Vehicle scheduling — "no vehicle may be assigned two overlapping
     routes". Currently enforced in application code; 12 double-bookings
     last month.
 R3: Scan events table — 8 billion rows, 3.1 TB, always queried by
     time range. The B-tree on `scanned_at` is 240 GB and no longer
     fits in the 128 GB buffer pool.
```

**R1 — search.**

```sql
-- current: ILIKE with a leading wildcard
EXPLAIN (ANALYZE,BUFFERS) SELECT * FROM parcels
WHERE tracking_no ILIKE '%4471%' OR recipient_name ILIKE '%sharma%' LIMIT 50;
--  Seq Scan, 900M rows, 41 s, 3.1 TB read.  4,000/s of this is not a system.
```

Three options, evaluated:

| Option | Index size | Latency | Trade-off |
|---|---|---|---|
| **GIN + pg_trgm on both columns** | 88 GB | 12 ms | Big index, slow writes (10 trigrams/row/column), but works inside PostgreSQL with full transactional consistency |
| **PostgreSQL full-text (`tsvector` + GIN)** | 18 GB | 4 ms | Much smaller and faster, but token-based — won't match `'4471'` *inside* `'IN44712025'`. Wrong for tracking numbers, right for names |
| **OpenSearch, fed by CDC** | separate cluster | 3 ms | Best relevance and faceting; new operational surface; eventual consistency (Topic 76) |

**The shipped answer is a hybrid**, because the two columns have different query shapes:

```sql
-- tracking numbers: substring matching, so trigram
CREATE INDEX CONCURRENTLY idx_parcels_tracking_trgm
  ON parcels USING gin (tracking_no gin_trgm_ops);

-- names/addresses: word matching with stemming, so full-text
ALTER TABLE parcels ADD COLUMN search_vec tsvector
  GENERATED ALWAYS AS (
    to_tsvector('simple', coalesce(recipient_name,'') || ' ' || coalesce(address,''))
  ) STORED;
CREATE INDEX CONCURRENTLY idx_parcels_search ON parcels USING gin (search_vec);
```
```
 tracking substring:  41,000 ms → 12 ms
 name search:         41,000 ms →  4 ms
 index total:         62 GB  (vs 88 GB for trigrams on both)
 insert throughput:   −18%  ← the real cost of GIN, and it's acceptable here
```

⚠ **Watch the pending list**, because at 3,000 inserts/s with ~14 trigrams each:

```sql
SELECT * FROM pgstatginindex('idx_parcels_tracking_trgm');
--  pending_pages | pending_tuples
--            412 |         88204        ← every query scans these linearly

ALTER INDEX idx_parcels_tracking_trgm SET (gin_pending_list_limit = '8MB');
ALTER TABLE parcels SET (autovacuum_vacuum_scale_factor = 0.02);   -- merges the list
```

**R2 — vehicle scheduling.**

```sql
-- BEFORE: application check, with the classic race
--   SELECT 1 FROM routes WHERE vehicle_id=$1 AND tstzrange(...) && during;
--   if none: INSERT
-- Two dispatchers, same vehicle, same second → both see none → both insert.

CREATE EXTENSION IF NOT EXISTS btree_gist;   -- needed to mix = and && in one index

ALTER TABLE routes ADD CONSTRAINT no_vehicle_overlap
  EXCLUDE USING gist (vehicle_id WITH =, during WITH &&)
  WHERE (status IN ('scheduled','in_progress'));
```
```
 double-bookings/month: 12 → 0
 index size: 2.1 GB on 40M routes
 insert cost: +0.4 ms/route (GiST is more expensive than B-tree — acceptable
              at 200 routes/s)
```

The partial `WHERE` matters: cancelled routes must not block a rebooking of the same slot (the pattern from Topic 13).

**R3 — the scan-events table.**

```sql
SELECT correlation FROM pg_stats WHERE tablename='scan_events' AND attname='scanned_at';
--  0.9987     ← append-only, inserted in time order. BRIN is viable.
```

```sql
CREATE INDEX CONCURRENTLY idx_scan_brin ON scan_events
  USING brin (scanned_at) WITH (pages_per_range = 64, autosummarize = on);

SELECT pg_size_pretty(pg_relation_size('idx_scan_brin'));      -- 1848 kB
SELECT pg_size_pretty(pg_relation_size('idx_scan_btree'));     -- 240 GB
```

```sql
EXPLAIN (ANALYZE,BUFFERS) SELECT count(*) FROM scan_events
WHERE scanned_at >= '2026-03-01' AND scanned_at < '2026-03-02';
```
```
 BEFORE (B-tree):  Buffers: shared hit=41204 read=8912004   Execution: 88,402 ms
                   (the index doesn't fit in RAM; every descent is disk)
 AFTER  (BRIN):    Buffers: shared hit=204 read=412008      Execution:  4,102 ms
```
**21× faster, and 240 GB of buffer-pool pressure removed** — which made *every other query on the system* faster too. That second-order effect is often larger than the direct win.

```sql
DROP INDEX CONCURRENTLY idx_scan_btree;
```

⚠ **What you give up:** point lookups. `WHERE scan_id = 4471` was served by a different index; but `WHERE scanned_at = '<exact timestamp>'` now reads 64 pages instead of 1. Check your access-pattern table before dropping a B-tree for a BRIN — BRIN is a *range* index, not a lookup index.

**Summary of the quarter:**

| | Before | After |
|---|---|---|
| Parcel search | 41 s (impossible) | 4–12 ms |
| Vehicle double-bookings | 12/month | 0 |
| Scan-event range query | 88 s | 4.1 s |
| Index storage | 240 GB + none | 62 + 2.1 + 0.002 GB |
| Buffer pool freed | — | ~180 GB of pressure |

---

## Common mistakes

**1. BRIN on uncorrelated data.**
- *Symptom:* the BRIN index is used and the query is *slower* than a seq scan.
- *Engine-level why:* every block range's `[min,max]` spans the whole domain, so nothing is pruned — and you pay for the index read plus a full recheck of every tuple.
- *Diagnose:* `SELECT correlation FROM pg_stats WHERE tablename=... AND attname=...;` and `Heap Blocks: lossy=<huge>` in `EXPLAIN`.
- *Fix:* only use BRIN above ~0.9 correlation. To *create* correlation: `CLUSTER` (one-off, takes an exclusive lock) or partition by the same column.

**2. Forgetting BRIN doesn't summarise new pages automatically.**
- *Symptom:* BRIN works, then gradually degrades on recent data.
- *Fix:* `WITH (autosummarize = on)` at creation, or `SELECT brin_summarize_new_values('idx');` on a schedule. **`autosummarize` is off by default** — this catches people constantly.

**3. Using `jsonb_ops` when `jsonb_path_ops` would do.**
- *Symptom:* GIN index is 40% larger and queries are 5× slower than necessary.
- *Engine-level why:* `jsonb_ops` indexes keys and values as separate entries, producing many false positives that the recheck must discard.
- *Fix:* if you only use `@>`, use `jsonb_path_ops`. If you need `?`/`?|`/`?&`, you need `jsonb_ops`.

**4. Not knowing about the GIN pending list.**
- *Symptom:* GIN queries slow down over hours and recover after a VACUUM.
- *Engine-level why:* `fastupdate=on` buffers new entries in an unsorted list that every query must scan linearly.
- *Diagnose:* `SELECT * FROM pgstatginindex('idx');` — check `pending_pages`.
- *Fix:* lower `gin_pending_list_limit`, vacuum more aggressively, or set `fastupdate=off` (slower writes, consistent reads).

**5. Reaching for a hash index because "O(1) beats O(log n)."**
- *Symptom:* a hash index that gives no measurable improvement and can't do anything else.
- *Engine-level why:* a B-tree lookup on a billion rows is 4 page reads, of which 3 are cached. The theoretical difference is ~1 page read. Meanwhile you lose ranges, ordering, uniqueness, multicolumn, and `INCLUDE`.
- *Fix:* use a B-tree unless the key is very wide *and* you only ever do equality.

**6. Using GIN where an expression B-tree is better.**
- *Symptom:* a 700 MB GIN index serving queries that only ever filter on one JSONB key.
- *Fix:* `CREATE INDEX ON t ((data->>'status'));` — smaller, faster, supports ranges and ordering. Use GIN only when the *set* of queried keys is open-ended.

**7. Mixing `=` and `&&` in an exclusion constraint without `btree_gist`.**
- *Symptom:* `ERROR: data type bigint has no default operator class for access method "gist"`.
- *Fix:* `CREATE EXTENSION btree_gist;` — it teaches GiST to handle scalar equality so you can combine it with range overlap.

---

## Hands-on proof

**PROVE IT #1 — trigram vs B-tree for substring.** (Example 1, step 1.)

**PROVE IT #2 — the three JSONB strategies, measured.**
```sql
CREATE INDEX i_ops  ON products USING gin (attributes);
CREATE INDEX i_path ON products USING gin (attributes jsonb_path_ops);
CREATE INDEX i_expr ON products ((attributes->>'colour'));
SELECT relname, pg_size_pretty(pg_relation_size(oid)) FROM pg_class
WHERE relname IN ('i_ops','i_path','i_expr');
```

**PROVE IT #3 — GIN produces many entries per row.**
```sql
CREATE EXTENSION IF NOT EXISTS pgstattuple;
SELECT * FROM pgstatginindex('i_path');
SELECT show_trgm('Cotton Kurta');
--  {"  c"," co","cot","ott","tto","ton","on "," ku","kur","urt","rta","ta "}
--  12 index entries for ONE column of ONE row.
```

**PROVE IT #4 — the exclusion constraint has no race window.**
```sql
-- session 1
BEGIN;
INSERT INTO room_bookings (room_id, during, booked_by)
  VALUES (99, '[2026-04-01 10:00, 2026-04-01 12:00)', 1);
-- do NOT commit
-- session 2
INSERT INTO room_bookings (room_id, during, booked_by)
  VALUES (99, '[2026-04-01 11:00, 2026-04-01 13:00)', 2);   -- ⏸ BLOCKS
-- session 1
COMMIT;
-- session 2 immediately: ERROR: conflicting key value violates exclusion constraint
```

**PROVE IT #5 — BRIN correlation.** (Example 1, step 5 — run both tables.)

**PROVE IT #6 — BRIN `pages_per_range` trade-off.**
```sql
CREATE INDEX b16   ON events_ordered USING brin (occurred_at) WITH (pages_per_range=16);
CREATE INDEX b128  ON events_ordered USING brin (occurred_at) WITH (pages_per_range=128);
CREATE INDEX b1024 ON events_ordered USING brin (occurred_at) WITH (pages_per_range=1024);
SELECT relname, pg_size_pretty(pg_relation_size(oid)) FROM pg_class
WHERE relname LIKE 'b1%' OR relname='b16';
-- then EXPLAIN the same range query with each (drop the others) and compare
-- `Heap Blocks: lossy=` — smaller ranges prune more precisely but the index grows.
```

**PROVE IT #7 — GIN write cost.**
```sql
CREATE TABLE g0 (id bigserial, d jsonb);
CREATE TABLE g1 (id bigserial, d jsonb);
CREATE INDEX ON g1 USING gin (d jsonb_path_ops);
\timing on
INSERT INTO g0 (d) SELECT jsonb_build_object('a',i,'b',i,'c',i,'d',i)
  FROM generate_series(1,500000) i;
INSERT INTO g1 (d) SELECT jsonb_build_object('a',i,'b',i,'c',i,'d',i)
  FROM generate_series(1,500000) i;
```

---

## The design decision framework

```
START WITH B-TREE. Only move away when the QUERY SHAPE forbids it.

USE GIN WHEN:
  ✓ `@>` containment on JSONB or arrays
  ✓ full-text search (tsvector)
  ✓ substring/fuzzy matching (pg_trgm) — `ILIKE '%x%'`, similarity
  ✓ the set of queried keys is OPEN-ENDED
  ✗ AVOID when you always query ONE known key → expression B-tree wins
    on size, speed, and it supports ranges and ordering
  ⚠ COST: N index entries per row; watch the pending list; slow writes
  → jsonb_path_ops for `@>` only; jsonb_ops if you need `?`/`?|`/`?&`

USE GiST WHEN:
  ✓ range overlap (`&&`) or containment (`@>`) on ranges
  ✓ geometric/geographic queries (with PostGIS)
  ✓ nearest-neighbour: `ORDER BY location <-> point LIMIT 10`
  ✓ ★ EXCLUSION CONSTRAINTS — the only access method that supports them
  ⚠ needs `btree_gist` to combine scalar `=` with range `&&`
  ⚠ lossy: always a recheck; more expensive writes than B-tree

USE BRIN WHEN:  ★ all four must hold
  ✓ the table is very large (> ~50 GB — below that a B-tree is fine)
  ✓ pg_stats.correlation on the column is > 0.9
  ✓ queries are RANGE scans, not point lookups
  ✓ the table is append-only or insert-ordered
  ⚠ set autosummarize = on. It is OFF by default.
  ⚠ tune pages_per_range: smaller = better pruning, bigger index
  ✗ NEVER on randomly-ordered data — it is worse than no index

USE SP-GiST WHEN:
  ✓ IP/network containment (`inet_ops`)
  ✓ text prefix search at scale (radix tree)
  ✓ point data with natural quadtree partitioning
  → niche; benchmark against GiST

USE HASH WHEN:
  ✓ equality only, AND the key is very wide (long URLs, tokens)
  ✓ and you don't need uniqueness, ordering, or multicolumn
  → rare. A B-tree is almost always the better default.

THE SIGNAL TO LOOK FOR:
  Write your predicate down and identify the OPERATOR:
      =  <  >  BETWEEN  LIKE 'x%'   → btree
      @>  ?  @@  %  ILIKE '%x%'     → gin
      &&  @>  <->  (ranges/geometry) → gist
      range scan on a huge correlated table → brin
      << (network containment)      → spgist

  Then verify with:
      SELECT amname FROM pg_am;                        -- available AMs
      SELECT * FROM pg_opclass WHERE opcmethod =
        (SELECT oid FROM pg_am WHERE amname='gin');     -- what GIN supports

  ★ AND BEFORE ANY BRIN INDEX:
      SELECT correlation FROM pg_stats
      WHERE tablename='t' AND attname='c';
    Below 0.9 → stop. Above 0.9 → proceed.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Build the 8M-row `products` table. Create three indexes on `attributes`: `gin (attributes)`, `gin (attributes jsonb_path_ops)`, and an expression B-tree on `(attributes->>'colour')`. Report each one's size, and the `EXPLAIN (ANALYZE, BUFFERS)` output for both `attributes @> '{"colour":"red"}'` and `attributes->>'colour' = 'red'`. Explain which index each query chose and why.

### Exercise 2 — medium (apply it)
Build two 20M-row tables with identical data — one inserted in timestamp order, one shuffled. Create a BRIN index on the timestamp column of each. Then:
(a) report `pg_stats.correlation` for both,
(b) run the same range query on each and report `Heap Blocks: lossy=` and execution time,
(c) vary `pages_per_range` across 16/128/1024 on the ordered table and plot index size against pruning effectiveness,
(d) run `CLUSTER` on the shuffled table using a B-tree on that column, re-`ANALYZE`, and re-measure. Explain exactly what changed and what it cost.

### Exercise 3 — hard (production simulation)
A compliance archive: `documents` table, 400M rows, 2.8 TB. Requirements:

```
 Q1 (2,000/s): full-text search over `body` (avg 40 KB), with ranking
 Q2 (   800/s): WHERE metadata @> '{"dept":"finance","year":2026}'
                — but the set of metadata keys is open-ended (400+ keys seen)
 Q3 (   400/s): WHERE created_at BETWEEN $1 AND $2   (always a range,
                never a point lookup); the table is strictly insert-ordered
 Q4 (    50/s): "documents whose retention window overlaps [now, now+30d)"
                where retention is a tstzrange
 Q5 ( writes ):  4,000 inserts/s, no updates, no deletes
```

(a) Choose an access method for each query. Justify from the operator shape, not from familiarity.
(b) Estimate each index's size. `body` averages 40 KB — compute the GIN index size and explain why full-text GIN is much smaller than the raw text.
(c) Q5 is 4,000 inserts/s and Q1 needs GIN. Compute the write amplification and explain the pending-list mechanism, including what you'd set and why.
(d) For Q3, prove BRIN is appropriate with a specific query, and state the exact condition that would invalidate that choice six months from now.
(e) Q4 needs an exclusion-style overlap query but *not* an exclusion constraint. Explain the difference and give the index.
(f) One index in your design is a liability if the access pattern shifts slightly. Identify it, name the shift, and give the fallback.
(g) Give the full deployment procedure for a live 2.8 TB table, including how you'd build a 200 GB GIN index without blocking writes and how you'd detect a failed build.

---

## Mental model checkpoint

1. Why can't a B-tree serve `ILIKE '%kurta%'`? What structure can, and how does it work?
2. What does "inverted index" mean? How many index entries does one row with 5 JSONB keys produce?
3. Name the single hard requirement for BRIN to be useful. What query tells you whether it holds?
4. Why is BRIN "lossy," and what does that mean for the plan you'll see in `EXPLAIN`?
5. What is the GIN pending list, what problem does it solve, and what problem does it create?
6. Which access method can support an exclusion constraint, and why can't the others?
7. When is an expression B-tree better than a GIN index on JSONB? When is it worse?

---

## Quick reference card

| AM | Operators | Size | Writes | Use for |
|---|---|---|---|---|
| **btree** | `= < > BETWEEN`, `LIKE 'x%'`, ORDER BY | medium | fast | 95% of everything |
| **hash** | `=` only | small for wide keys | fast | rare; wide equality-only keys |
| **gin** | `@> ? @@ %` `ILIKE '%x%'` | **large** | **slow** | JSONB, arrays, FTS, trigrams |
| **gist** | `&& @> <->` | medium | medium | ranges, geometry, KNN, **EXCLUDE** |
| **spgist** | `<< @>` prefix | small | medium | IP/network, text prefix, points |
| **brin** | range predicates | **tiny** | ~free | huge + correlated + append-only |

**Key facts**

| | |
|---|---|
| `jsonb_path_ops` vs `jsonb_ops` | ~43% smaller, ~5× faster, but `@>` only |
| GIN pending list | `fastupdate=on` default; check `pgstatginindex` |
| BRIN `autosummarize` | **off by default** — turn it on |
| BRIN correlation threshold | **> 0.9**, else worse than nothing |
| `btree_gist` | required to mix `=` with `&&` in one GiST index |
| Exclusion constraints | GiST only |
| Range bounds | always `'[)'` — half-open makes adjacency correct |

**Extensions you'll need**

```sql
CREATE EXTENSION pg_trgm;     -- trigram similarity / ILIKE '%x%'
CREATE EXTENSION btree_gist;  -- scalar = inside a GiST index
CREATE EXTENSION btree_gin;   -- scalar = inside a GIN index
CREATE EXTENSION postgis;     -- geospatial (GiST/SP-GiST)
```

**Before creating a BRIN index, always:**
```sql
SELECT correlation FROM pg_stats WHERE tablename='t' AND attname='c';
```

---

## When would I use this at work?

1. **A search feature ships and `ILIKE '%term%'` melts the database.** You recognise it instantly as a structural mismatch — a B-tree cannot serve a leading wildcard — and can present three costed options (trigram GIN, full-text GIN, external search engine) with sizes and latencies rather than guessing.

2. **A booking or scheduling double-write bug.** Instead of adding a lock or a better check, you add a GiST exclusion constraint and the entire class of bug becomes impossible. That's a five-line migration replacing a permanent source of incidents.

3. **A time-series table whose index no longer fits in RAM.** Checking `correlation` takes ten seconds and tells you whether swapping a 240 GB B-tree for a 2 MB BRIN is viable — which frees enough buffer pool to speed up every other query on the instance.

---

## Connected topics

**Understand before this:** 11 (B-tree, for contrast), 12 (bitmap scans and rechecks — BRIN and GIN always produce them), 15 (`correlation`, the BRIN prerequisite).

**This unlocks:**
- **17** — the write cost of GIN and GiST in production
- **24** — exclusion constraints in full (GiST is the mechanism)
- **25** — JSONB storage, and when to index it
- **59** — partitioning, which creates the correlation BRIN needs
- **73** — time-series modelling, where BRIN is standard
- **74** — search and graph modelling, where GIN is the foundation
- **Case study 02** — the GiST exclusion constraint in a production design
