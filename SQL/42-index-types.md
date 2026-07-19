# Topic 42 — Index Types in PostgreSQL
### SQL Mastery Curriculum — Phase 7: Indexes and Query Optimisation

---

## 1. ELI5 — The Analogy That Makes It Click

Imagine a giant library with ten million books and no organisation whatsoever — books are just placed on shelves in the order they arrived. Now someone asks you five different kinds of questions, and each kind needs a *different* kind of finding-aid:

- **"Find the book titled exactly 'Dune'."** — You want an alphabetical card catalogue. You flip to D, then to Du, then Dun… found it. That's a **B-tree**: sorted, so you can binary-search, and you can also answer "all books whose title is between 'D' and 'F'" by walking the drawer. It handles equality *and* ranges *and* gives you results already in order.

- **"Is there a book with ISBN 978-0441172719? Yes or no, fastest possible."** — You don't care about ranges, you just want a yes/no lookup. A **hash table** on the wall: hash the ISBN, go straight to the bucket. Blazing fast for exact match, useless for "ISBNs between X and Y".

- **"Find every book that mentions the word 'sandworm' anywhere in its text."** — One book contains thousands of words; a title index is useless. You need an index that maps *each word* to *the list of books containing it*. That's a **GIN** index — an inverted index, built for "this container holds this element": full-text, arrays, JSONB keys.

- **"Find every book whose story takes place within 50 miles of this map coordinate."** or **"Find the reservation whose time-range overlaps 2pm–3pm."** — Now you're indexing *shapes and ranges* and asking about overlap, containment, and nearest-neighbour. That's a **GiST** index — a balanced tree of bounding boxes.

- **"Find books shelved between the 4-millionth and 5-millionth arrival."** — If books were physically shelved in arrival order, you don't need a per-book index at all. You just need a tiny note on each *shelf* saying "this shelf holds arrivals 4.0M–4.1M". Skip the shelves that can't contain your answer. That's **BRIN** — one tiny summary per block range, for data that's already physically ordered.

There is no universal finding-aid. **The index type is chosen to match the *shape of the question*, not the shape of the data alone.** A principal engineer picks the index by first asking: *what operator will the query use?* — `=`, `<`, `@>`, `<@`, `&&`, `<->`, `@@` — and then picks the index type that can accelerate *that operator*.

---

## 2. Connection to SQL Internals

An index in PostgreSQL is a **separate on-disk data structure** that stores a mapping from indexed key values to physical row locations (TIDs — tuple identifiers, a `(block_number, item_offset)` pair pointing into the heap). Every index type shares that contract but differs radically in the internal structure that does the mapping.

The engine concepts each index type touches:

- **Access Method (AM)**: PostgreSQL's index machinery is pluggable. `pg_am` lists the access methods: `btree`, `hash`, `gin`, `gist`, `spgist`, `brin`. Each AM implements a standard interface (build, insert, bulk-delete, and — crucially — `amcostestimate` so the planner can cost it). This is why extensions like `pg_trgm`, `btree_gin`, and `bloom` can add capability: they plug into an AM.

- **Operator Classes** (`pg_opclass`): An index doesn't know what `<` means for your type — the *operator class* tells it. `int4_ops`, `text_pattern_ops`, `jsonb_path_ops`, `gist_trgm_ops` are operator classes. Choosing the right opclass is often the difference between an index being used and ignored (the classic `LIKE 'foo%'` needs `text_pattern_ops` on non-C locales).

- **The heap and TIDs**: Almost all index types (except covering data in an index-only scan) ultimately hand back a set of TIDs, which the executor uses to fetch rows from the heap. This is why MVCC matters — the index may point at row versions that are dead to your snapshot; the executor re-checks visibility against the heap (except in index-only scans validated by the visibility map).

- **MVCC and bloat**: Because PostgreSQL keeps old row versions until VACUUM, indexes accumulate pointers to dead tuples. B-tree has microvacuum and page-level dedup; GIN has a *pending list* to batch inserts; BRIN barely cares because it summarises ranges, not rows. Index bloat is an internal-storage concern that differs per AM.

- **WAL and build cost**: Every index write is WAL-logged. GIN and GiST inserts are expensive (multiple pages touched, potential tree restructuring), which is why write-heavy tables pay a real penalty for these index types. `fastupdate` (GIN pending list) exists precisely to amortise this.

- **The planner's cost model**: The planner picks an index (or a Seq Scan) by estimating pages read and rows returned using `pg_statistic`. Each AM's cost estimator encodes its access pattern — random vs sequential I/O, correlation (`pg_stats.correlation`, which is what makes BRIN and a clustered B-tree cheap), and selectivity. Understanding index *types* is inseparable from understanding how the planner *costs* them (the subject of Topic 43, Index Scan Types).

Put simply: an index type is a **(data structure + operator class + cost estimator)** triple, and mastering indexes means knowing which triple accelerates which operator.

---

## 3. Logical Execution Order Context

Indexes are not a clause in the logical query order (`FROM → JOIN → WHERE → GROUP BY → HAVING → SELECT → DISTINCT → window → ORDER BY → LIMIT`). They are a *physical* implementation detail the planner injects to satisfy several of those logical phases more cheaply. But *where* in that pipeline an index helps depends on the index type:

- **FROM / JOIN**: An index on the join key lets the planner choose a Nested Loop with an inner Index Scan (Topic 11) or a Merge Join fed by an already-sorted Index Scan. B-tree is the workhorse here because joins are equality on ordered keys.

- **WHERE**: This is where index *type* matters most. The predicate's *operator* decides which index type can serve it:
  - `col = x`, `col < x`, `col BETWEEN` → B-tree (or hash for `=` only)
  - `tags @> ARRAY['x']`, `jsonb @> '{...}'`, `tsv @@ query` → GIN
  - `geom && box`, `range && range`, `ORDER BY geom <-> point` → GiST
  - `col = x` on a huge, naturally-ordered table → BRIN can pre-filter block ranges

- **GROUP BY / DISTINCT**: A B-tree can supply pre-sorted input, letting the planner use a cheap Group Aggregate or Unique instead of a Hash Aggregate + Sort. No other index type provides this ordering.

- **ORDER BY**: Only **B-tree** (ordered) and **GiST** (via distance operators for `ORDER BY geom <-> point` — nearest-neighbour) can satisfy an ORDER BY directly from the index. Hash, GIN, SP-GiST (mostly), and BRIN cannot produce ordered output.

- **LIMIT**: An ordered index (B-tree) plus LIMIT is the magic combination that lets `ORDER BY … LIMIT 10` read only the first few index entries instead of sorting the whole table (Topic 41, Top-N Per Group, leans on exactly this).

So the mental model is: **the logical query names the operators; the operators determine the eligible index types; the planner costs them and picks one.** You cannot reason about which index to create without knowing which logical phase you are trying to accelerate and with which operator.

---

## 4. What Is an Index Type?

An **index type** (access method) is the internal data structure and algorithm PostgreSQL uses to accelerate lookups for a particular *class of operators*. PostgreSQL ships six general-purpose index access methods, each optimised for a different query shape. You choose the type when you create the index; if you omit it, you get B-tree.

```sql
CREATE [UNIQUE] INDEX [CONCURRENTLY] [index_name]
    ON table_name
    USING method              -- btree | hash | gin | gist | spgist | brin
    ( column_or_expression [ opclass ] [ ASC | DESC ] [ NULLS {FIRST|LAST} ] [, ...] )
    [ INCLUDE (column [, ...]) ]        -- covering columns (payload, not searchable)
    [ WITH ( storage_parameter = value [, ...] ) ]   -- e.g. fillfactor, fastupdate, pages_per_range
    [ WHERE predicate ];               -- partial index: only rows matching predicate
```

Annotated breakdown:

```sql
CREATE UNIQUE INDEX CONCURRENTLY idx_orders_customer_created
    │       │      │             │
    │       │      │             └── index name (optional; auto-generated if omitted)
    │       │      └── CONCURRENTLY: build without an ACCESS EXCLUSIVE lock;
    │       │          slower, two heap passes, but no write downtime. Cannot run in a txn block.
    │       └── UNIQUE: enforce uniqueness as a side effect (B-tree only; hash/GIN/GiST/BRIN cannot be UNIQUE
    │           — except GiST via an EXCLUDE constraint, which is different)
    └── CREATE INDEX
    ON orders
       │
       └── target table (or materialized view)
    USING btree
          │
          └── access method. Omit → defaults to btree.
    ( customer_id, created_at DESC )
      │            │          │
      │            │          └── per-column sort direction (B-tree only; meaningless for hash/GIN/BRIN)
      │            └── second key column → makes this a composite (multicolumn) index
      └── first key column; leftmost-prefix rule applies for B-tree
    INCLUDE ( total_amount, status )
             │
             └── payload columns stored in the leaf but NOT part of the search key;
                 enable index-only scans without bloating the searchable key (B-tree & GiST)
    WHERE status = 'completed';
          │
          └── partial index predicate — index only the subset of rows that match,
              shrinking the index and making it cheaper to scan and maintain
```

The six methods at a glance:

| Method | Structure | Operators it accelerates | Ordered output? | UNIQUE? | Multicolumn? |
|--------|-----------|--------------------------|-----------------|---------|--------------|
| **btree** | Balanced sorted tree | `= < <= > >= BETWEEN IN` , `LIKE 'x%'`, `IS NULL` | Yes | Yes | Yes (leftmost-prefix) |
| **hash** | Hash table | `=` only | No | No | No (single column) |
| **gin** | Inverted index | `@> <@ && ? ?& ?|` (arrays/jsonb), `@@` (FTS) | No | No | Yes (multi-key columns) |
| **gist** | Balanced tree of predicates/bboxes | `&& @> <@` (geo/range), `<->` (KNN), `@@`, `~=` | KNN only | Via EXCLUDE | Yes |
| **spgist** | Space-partitioned trie/quadtree | `= << >> <@` (points, text prefixes, ranges) | No (mostly) | No | Limited |
| **brin** | Per-block-range summaries | `= < <= > >= BETWEEN` on correlated columns | No | No | Yes |

---

## 5. Why Index Type Mastery Matters in Production

1. **The wrong type means no index at all.** Create a B-tree on a `jsonb` column and query it with `@>` — PostgreSQL will Seq Scan every time, silently. The index exists, `\d` shows it, but the planner never uses it because B-tree has no operator class for `@>`. Teams routinely ship "we added an index" only to discover queries never touched it.

2. **Query latency differs by orders of magnitude.** A tag-search on a 50M-row table is a 12-second Seq Scan with a B-tree and a 3-millisecond bitmap scan with GIN. A geospatial "within radius" query is impossible to serve at scale without GiST. These aren't 2× differences — they're 1000× differences that decide whether a feature ships.

3. **Index size and write cost are budget line items.** A GIN index on large JSONB can be *larger than the table*. A BRIN index on a 100M-row time-series table is a few hundred kilobytes — literally 10,000× smaller than the equivalent B-tree — while still pruning 99% of blocks. On write-heavy tables, GIN/GiST insert cost can halve your ingest throughput. Choosing the type is a storage and throughput decision, not just a latency one.

4. **Some capabilities are *only* possible with the right type.** Nearest-neighbour (`ORDER BY location <-> point LIMIT 10`) is only servable by GiST/SP-GiST. Exclusion constraints ("no two bookings overlap for the same room") require GiST. Efficient full-text ranking requires GIN. You cannot substitute a B-tree.

5. **Interviewers use this to separate senior from principal.** "You have a 200M-row append-only events table queried by time range and a tags array — what indexes?" The senior says "B-tree on time, GIN on tags." The principal says "BRIN on time (it's naturally ordered — the B-tree would be 4GB and add write latency; BRIN is 200KB and prunes ranges), GIN with `fastupdate` on tags, and I'd check the `correlation` stat before committing to BRIN." That reasoning is exactly what this topic builds.

---

## 6. Deep Technical Content

### 6.1 B-tree — The Default Workhorse

A B-tree is a self-balancing, sorted, multi-way tree. Internal (branch) pages hold separator keys and child pointers; leaf pages hold the indexed keys in sorted order plus the TIDs, and leaves are doubly-linked so range scans walk sideways without revisiting the root.

**What it accelerates:**
- Equality: `col = 42`
- Range: `col > 100`, `col BETWEEN 5 AND 9`, `col <= x`
- `IN (list)` (treated as multiple equality probes)
- Sorted retrieval: `ORDER BY col` (forward or backward — B-tree is bidirectional)
- `IS NULL` / `IS NOT NULL` (NULLs are indexed, sorted last by default)
- Prefix `LIKE 'abc%'` and `~ '^abc'` — **but only** with `text_pattern_ops`/`varchar_pattern_ops` opclass in non-C locales, because default locale collation breaks prefix ordering.
- Uniqueness enforcement (`UNIQUE`, primary keys)

```sql
CREATE INDEX idx_users_email ON users (email);                    -- equality/range/sort
CREATE INDEX idx_orders_created ON orders (created_at);           -- range + ORDER BY … LIMIT
CREATE INDEX idx_users_email_pat ON users (email text_pattern_ops); -- for LIKE 'foo%'
```

**Composite / multicolumn B-trees and the leftmost-prefix rule.** A `(a, b, c)` index is usable for predicates on `a`, `(a,b)`, or `(a,b,c)` — a prefix of the columns — and can also satisfy `a = x AND b > y`. It is *not* usable to seek on `b` alone. Column order is a design decision: put equality columns before range columns, and put the ORDER BY column last if you want the index to also serve the sort.

```sql
-- Serves: WHERE customer_id = ? ORDER BY created_at DESC LIMIT 20  (no sort needed)
CREATE INDEX idx_orders_cust_created ON orders (customer_id, created_at DESC);
```

**Covering indexes (`INCLUDE`).** Non-key payload columns stored only in the leaf let the planner answer a query entirely from the index (index-only scan) without a heap fetch, provided the visibility map says the page is all-visible.

```sql
-- Index-only scan candidate: no heap access for total_amount
CREATE INDEX idx_orders_cust_incl ON orders (customer_id) INCLUDE (total_amount, status);
```

**Deduplication (PG 13+).** Leaf pages with many duplicate keys are stored as a key plus a posting list of TIDs, dramatically shrinking indexes on low-cardinality columns. This is automatic; you can disable it with `WITH (deduplicate_items = off)`.

**When B-tree is the wrong choice:** very-low-cardinality columns on huge tables (a boolean on 100M rows — the index is enormous and rarely more selective than a Seq Scan; use a partial index instead), and containment/overlap operators it cannot serve.

### 6.2 Hash — Equality Only

A hash index stores 32-bit hashes of the key in buckets. It supports exactly one operator: `=`.

```sql
CREATE INDEX idx_sessions_token_hash ON sessions USING hash (session_token);
```

**Key facts:**
- **Only `=`.** No ranges, no sorting, no `LIKE`, no multicolumn, no UNIQUE, no `IS NULL` probing.
- **Crash-safe and WAL-logged since PostgreSQL 10.** Before 10 they were unlogged and not replicated — the reason hash indexes had a bad reputation for a decade. That reputation is now outdated but persists in old advice.
- **When it can beat B-tree:** for equality on *large* keys (long text, big values), the hash is fixed-width and the index can be smaller and the probe cheaper than descending a wide B-tree. The wins are narrow.
- **When to skip it:** almost always in practice, a B-tree on the same column serves `=` *and* everything else, so teams default to B-tree. Reach for hash only when you have measured a specific equality-only workload on a large key and confirmed the size/latency win.

### 6.3 GIN — Generalized Inverted Index

GIN is an **inverted index**: for each *element* (a word, an array item, a JSONB key/value, a trigram) it stores a sorted list (posting list/tree) of the rows that contain that element. It's the "which documents contain this term" structure.

**What it accelerates:**
- Arrays: `tags @> ARRAY['sql']` (contains), `tags && ARRAY['sql','pg']` (overlaps), `tags <@ ...` (contained by)
- JSONB: `data @> '{"status":"active"}'` (containment), `data ? 'key'`, `data ?& array['a','b']`, `data ?| array['a','b']`
- Full-text search: `to_tsvector('english', body) @@ to_tsquery('english', 'sandworm')`
- Trigram fuzzy/substring search (with `pg_trgm`'s `gin_trgm_ops`): `col LIKE '%needle%'`, `col % 'similr'`

```sql
-- Array containment
CREATE INDEX idx_articles_tags ON articles USING gin (tags);

-- JSONB: default jsonb_ops supports @>, ?, ?&, ?|   (keys AND values, larger index)
CREATE INDEX idx_events_data ON events USING gin (data);

-- JSONB: jsonb_path_ops supports ONLY @> but is smaller and faster for it
CREATE INDEX idx_events_data_path ON events USING gin (data jsonb_path_ops);

-- Full-text
CREATE INDEX idx_docs_fts ON documents USING gin (to_tsvector('english', body));

-- Substring/fuzzy search — makes LIKE '%x%' indexable
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_users_name_trgm ON users USING gin (name gin_trgm_ops);
```

**Internal mechanics that matter in production:**
- **Slow, expensive inserts** — a single row with 20 tags updates 20 posting lists. To amortise this, GIN keeps a **pending list** (`fastupdate = on`, the default): new entries are appended to an unsorted pending list and merged into the main structure during VACUUM or when the list exceeds `gin_pending_list_limit`. This speeds inserts but *slows reads* (the pending list must be scanned) until a merge happens. On read-heavy tables you may set `fastupdate = off`; on write-heavy tables you tune the limit and VACUUM cadence.
- **No ordering, no `<`/range.** GIN answers "which rows contain X"; it cannot sort or do inequality.
- **`jsonb_ops` vs `jsonb_path_ops`:** `jsonb_ops` (default) indexes every key and value, supports the full operator set, larger index. `jsonb_path_ops` indexes hashes of full paths, supports only `@>`, ~2–3× smaller and faster for containment. Pick `jsonb_path_ops` if `@>` is all you need.
- **Multi-key GIN via `btree_gin`:** the `btree_gin` extension adds opclasses so a scalar column can share a single GIN index with an array/jsonb column — handy for mixed predicates.

### 6.4 GiST — Generalized Search Tree

GiST is a *framework* for building balanced trees where each internal node stores a **predicate that is true for everything below it** (typically a bounding box). Searching descends only into subtrees whose predicate could match. It is the engine behind geometry, ranges, and nearest-neighbour.

**What it accelerates:**
- Geometry / PostGIS: `geom && box` (bbox overlap), `ST_DWithin`, `@>`, `<@`
- Range types: `tsrange && tsrange` (overlap), `@>` (contains a point/range), `<@`
- **Nearest-neighbour (KNN):** `ORDER BY location <-> point('(0,0)') LIMIT 10` — GiST can return rows *in distance order* by descending the tree best-first. No other general index does this.
- Exclusion constraints: "no two rows overlap" via `EXCLUDE USING gist (room WITH =, during WITH &&)`
- Trigram similarity (`gist_trgm_ops`) as an alternative to GIN for `%`

```sql
-- Range overlap: prevent double-booking a room
CREATE EXTENSION IF NOT EXISTS btree_gist;
ALTER TABLE bookings
  ADD CONSTRAINT no_overlap
  EXCLUDE USING gist (room_id WITH =, during WITH &&);

-- Geospatial nearest-neighbour (PostGIS)
CREATE INDEX idx_stores_geom ON stores USING gist (geom);
-- SELECT * FROM stores ORDER BY geom <-> ST_MakePoint(-122.4,37.8) LIMIT 5;

-- Range containment
CREATE INDEX idx_reservations_during ON reservations USING gist (during);
```

**Internal mechanics:**
- **Lossy by design.** Internal predicates are approximations (bounding boxes), so GiST may return false-positive candidates that a *recheck* against the actual value filters out. EXPLAIN shows `Rows Removed by Index Recheck`. This is normal.
- **Balanced but overlap-prone.** Because bounding boxes can overlap, a search may descend multiple subtrees. Build quality (and the specific opclass) affects how much overlap there is.
- **Slower point-equality than B-tree.** For plain scalar equality/range, B-tree wins. Use GiST when you need overlap, containment, or KNN semantics that B-tree cannot express.
- **`btree_gist`** adds scalar opclasses so you can combine `room_id WITH =` (scalar) and `during WITH &&` (range) in one GiST index — required for the exclusion constraint above.

### 6.5 SP-GiST — Space-Partitioned GiST

SP-GiST supports **non-balanced, space-partitioning** trees: quadtrees, k-d trees, radix tries. These shine when the data partitions naturally into non-overlapping regions.

**What it accelerates:**
- Points with quadtree/k-d tree: equality, `<<` `>>` `<@` (point-in-box), and KNN
- Text with radix trie (`text_ops`): prefix matching, `LIKE 'abc%'`, longest-prefix
- Non-overlapping range types (`range_ops`) and IP addresses (`inet_ops`)

```sql
-- Prefix search on a huge dictionary/URL table
CREATE INDEX idx_urls_prefix ON urls USING spgist (url);
-- WHERE url LIKE 'https://example.com/%'

-- Points (non-overlapping partitions suit quadtree)
CREATE INDEX idx_places_point ON places USING spgist (location);   -- point column
```

**When SP-GiST over GiST:** when the data is naturally non-overlapping and hierarchical — clustered points, IP ranges, string prefixes. The partitioning avoids GiST's overlapping-bbox descent, often yielding a smaller, faster index. When data overlaps heavily (arbitrary polygons, overlapping time ranges), GiST is usually better. SP-GiST is the least-used of the six; reach for it for prefix-trie text search and point/inet workloads specifically.

### 6.6 BRIN — Block Range Index

BRIN stores, for each *block range* (default 128 heap pages, ~1MB), a tiny summary — for the default `minmax` opclass, the min and max value of the indexed column in that range. To answer `WHERE ts BETWEEN a AND b`, BRIN scans the summaries, discards every range whose [min,max] can't overlap [a,b], and returns the surviving ranges' blocks as a bitmap for the heap to scan and recheck.

**What it accelerates:**
- `=`, `<`, `<=`, `>`, `>=`, `BETWEEN` on columns whose physical row order **correlates** with the value — timestamps on append-only tables, sequential IDs, geographically clustered data.

```sql
-- 200M-row append-only events table, inserted in time order
CREATE INDEX idx_events_ts_brin ON events USING brin (created_at);

-- Tune granularity: fewer pages per range = more precise, slightly larger
CREATE INDEX idx_events_ts_brin32 ON events USING brin (created_at)
    WITH (pages_per_range = 32);

-- minmax_multi (PG 14+) handles moderate disorder / outliers better than plain minmax
CREATE INDEX idx_events_ts_brin_mm ON events USING brin (created_at timestamptz_minmax_multi_ops);
```

**Why it's a superpower — and its one hard requirement:**
- **Microscopic size.** A BRIN index on 200M rows can be a few hundred KB vs several GB for a B-tree. It fits in cache entirely.
- **Near-zero write cost.** Inserts usually just extend the current range's max; no tree restructuring.
- **The requirement: correlation.** BRIN only prunes if rows with similar values are physically near each other. Check `SELECT correlation FROM pg_stats WHERE tablename='events' AND attname='created_at';` — near ±1 is ideal; near 0 means BRIN will prune nothing and every query degrades to a Seq Scan with recheck overhead. If correlation is poor but the table is *mostly* ordered with some outliers, `minmax_multi` (PG 14+) stores several min/max intervals per range and tolerates the disorder.
- **Not for point lookups where you need one row fast.** BRIN returns a *range* of blocks (thousands of rows) that then get rechecked. It's for pruning huge scans, not single-row seeks. Pair it with a partial B-tree if you also need pinpoint lookups.

### 6.7 Cross-Cutting Features (apply across types)

- **Expression indexes** work on most types: `CREATE INDEX ON users (lower(email));` then query `WHERE lower(email) = $1`. The expression in the query must match the index expression exactly.
- **Partial indexes** (`WHERE`) work on all types and are the right answer for low-cardinality filters: `CREATE INDEX ON orders (created_at) WHERE status='pending';` indexes only pending orders — tiny, hot, and cheap to maintain.
- **`CREATE INDEX CONCURRENTLY`** avoids locking writes but takes longer, does two heap passes, cannot run inside a transaction, and leaves an `INVALID` index behind if it fails (which you must `DROP` and retry). Essential for production DDL.
- **Only B-tree and GiST support `INCLUDE`** covering columns. GiST INCLUDE arrived in PG 12.
- **`UNIQUE` is B-tree only.** For uniqueness on non-btree semantics, use an `EXCLUDE` constraint (GiST).

### 6.8 Operator Classes — The Hidden Half of the Decision

An index type alone doesn't determine behaviour; the **operator class** (opclass) does the fine-tuning within a type. The same `USING gin` or `USING btree` can behave completely differently depending on opclass. The ones you actually reach for:

| Type | Opclass | Enables |
|------|---------|---------|
| btree | `text_pattern_ops` / `varchar_pattern_ops` | `LIKE 'x%'`, `~ '^x'` in non-C locales |
| btree | `text_ops` (default) | equality/range under the DB collation |
| gin | `jsonb_ops` (default) | `@> ? ?& ?|` (keys + values) |
| gin | `jsonb_path_ops` | `@>` only, smaller/faster |
| gin | `gin_trgm_ops` (pg_trgm) | `LIKE '%x%'`, `%` similarity |
| gin | `array_ops` (default for arrays) | `@> <@ &&` |
| gist | `gist_trgm_ops` (pg_trgm) | trigram `%` + KNN on text |
| gist | `range_ops` (default for ranges) | `&& @> <@` on ranges |
| gist/spgist | `inet_ops` | subnet containment on `inet`/`cidr` |
| brin | `*_minmax_ops` (default) | min/max per range |
| brin | `*_minmax_multi_ops` (PG14+) | multiple intervals per range → tolerates disorder |
| brin | `*_bloom_ops` (PG14+) | equality on *uncorrelated* columns via a per-range Bloom filter |

The single most common "my index isn't used" bug after wrong *type* is wrong *opclass* — classically a default-locale B-tree that can't serve `LIKE 'foo%'` because the collation ordering doesn't match byte-prefix ordering. The fix is `text_pattern_ops`, not a different index type.

**BRIN bloom is the quiet exception to the correlation rule:** `brin (col int8_bloom_ops)` builds a small Bloom filter per block range, letting BRIN answer *equality* on columns that are *not* physically ordered — trading the range-pruning superpower for probabilistic equality pruning at tiny size. It's the middle ground between "BRIN needs correlation" and "B-tree is huge."

### 6.9 Index-Only Scans and the Visibility Map

Any index type that stores the queried columns (B-tree/GiST key or `INCLUDE` payload) can potentially serve an **index-only scan** — returning results without touching the heap at all. The catch is MVCC: the index doesn't store row visibility, so PostgreSQL consults the **visibility map** (a bitmap, one bit per heap page, set by VACUUM when every tuple on the page is visible to all transactions). If the page is all-visible, the heap fetch is skipped; if not, it falls back to a heap visibility check.

```sql
CREATE INDEX idx_orders_cust_incl ON orders (customer_id) INCLUDE (total_amount, status);
-- SELECT customer_id, total_amount FROM orders WHERE customer_id = 42;
-- → Index Only Scan  ... Heap Fetches: 0   (if pages are all-visible)
```

Practical consequence: an index-only scan on a write-heavy table can silently degrade to many `Heap Fetches` if autovacuum can't keep the visibility map warm. The `Heap Fetches:` line in EXPLAIN is how you diagnose it. This is why index-*type* choices interact with VACUUM health — a covering B-tree only pays off if the visibility map is maintained.

### 6.10 Combining Indexes — BitmapAnd / BitmapOr

You rarely build one index that satisfies every predicate. Instead, the planner can scan **several single-column indexes of different types** and combine their result bitmaps with `BitmapAnd` / `BitmapOr`. This is why the right answer to "filter by tags AND by time range" is often two indexes (a GIN and a BRIN), not one impossible composite.

```sql
-- GIN on tags, BRIN on created_at; planner ANDs the two bitmaps.
SELECT id FROM events
WHERE tags @> ARRAY['error']
  AND created_at >= '2026-07-01';
-- BitmapAnd
--   ->  Bitmap Index Scan on idx_events_tags_gin
--   ->  Bitmap Index Scan on idx_events_created_brin
```

Only bitmap-producing scans combine this way (B-tree, GIN, GiST, BRIN can all feed a bitmap). A plain ordered Index Scan cannot be BitmapAnd-ed. This changes how you design an index *set*: prefer several targeted single-purpose indexes the planner can mix over one over-specified composite that only serves one query.

### 6.11 Decision Table — Which Type for Which Question

| The query does… | Operator | Index type |
|-----------------|----------|-----------|
| Exact match on a scalar | `=` | B-tree (or hash if equality-only, large key) |
| Range / sort / `LIKE 'x%'` | `< > BETWEEN ORDER BY` | B-tree |
| Array/JSONB containment | `@> <@ && ?` | GIN |
| Full-text search | `@@` | GIN |
| Substring `LIKE '%x%'` / fuzzy | `LIKE`, `%` | GIN (`gin_trgm_ops`) |
| Geometric / range overlap | `&& @> <@` | GiST |
| Nearest-neighbour | `<->` `ORDER BY` | GiST (or SP-GiST) |
| No-overlap constraint | `EXCLUDE … &&` | GiST |
| String prefix at scale / points / IPs | `LIKE 'x%'`, point ops | SP-GiST |
| Huge naturally-ordered table, range scan | `< > BETWEEN` | BRIN |

---

## 7. EXPLAIN — Index Types in the Plan

Each index type surfaces differently in `EXPLAIN ANALYZE`. Learning to read these is how you confirm the index you created is actually being used *the way you intended*.

### B-tree Index Scan (ordered, LIMIT)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, total_amount FROM orders
WHERE customer_id = 4711
ORDER BY created_at DESC
LIMIT 20;
```

```
Limit  (cost=0.43..18.7 rows=20 width=12) (actual time=0.03..0.09 rows=20 loops=1)
  ->  Index Scan Backward using idx_orders_cust_created on orders
        (cost=0.43..142.6 rows=157 width=12) (actual time=0.02..0.08 rows=20 loops=1)
        Index Cond: (customer_id = 4711)
  Buffers: shared hit=5
Planning Time: 0.14 ms
Execution Time: 0.11 ms
```

Reading it: `Index Scan Backward` shows the B-tree serving both the `customer_id =` filter *and* the `ORDER BY created_at DESC` from the same `(customer_id, created_at DESC)` index — **no Sort node**, and LIMIT stops after 20 leaf entries. Only 5 buffers touched. This is the ideal Top-N plan from Topic 41.

### GIN Bitmap Scan (array containment)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT id, title FROM articles
WHERE tags @> ARRAY['postgres','indexing'];
```

```
Bitmap Heap Scan on articles  (cost=44.0..820.5 rows=210 width=40)
                              (actual time=0.6..2.1 rows=188 loops=1)
  Recheck Cond: (tags @> '{postgres,indexing}'::text[])
  Heap Blocks: exact=176
  ->  Bitmap Index Scan on idx_articles_tags  (cost=0.00..43.9 rows=210 width=0)
                                              (actual time=0.5..0.5 rows=188 loops=1)
        Index Cond: (tags @> '{postgres,indexing}'::text[])
Buffers: shared hit=182
Planning Time: 0.2 ms
Execution Time: 2.3 ms
```

Reading it: GIN never produces a plain "Index Scan" — it feeds a **Bitmap Index Scan** that builds a bitmap of candidate heap blocks, then a **Bitmap Heap Scan** fetches and rechecks them. `Recheck Cond` is expected for GIN. 2.3ms vs a multi-second Seq Scan without the index.

### GiST KNN (nearest-neighbour)

```sql
EXPLAIN (ANALYZE)
SELECT id, name FROM stores
ORDER BY geom <-> ST_MakePoint(-122.4, 37.8)
LIMIT 5;
```

```
Limit  (cost=0.28..2.10 rows=5 width=36) (actual time=0.09..0.13 rows=5 loops=1)
  ->  Index Scan using idx_stores_geom on stores
        (cost=0.28..364.0 rows=1000 width=36) (actual time=0.08..0.12 rows=5 loops=1)
        Order By: (geom <-> '0101000000...'::geometry)
Execution Time: 0.16 ms
```

Reading it: the `Order By:` line (not `Index Cond:`) is the tell — GiST is walking the tree in distance order and LIMIT stops at 5. No Sort node, no full scan. Only GiST/SP-GiST can produce this.

### BRIN Bitmap Scan (huge ordered table)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT count(*) FROM events
WHERE created_at BETWEEN '2026-07-01' AND '2026-07-02';
```

```
Aggregate  (cost=21540..21541 rows=1 width=8) (actual time=41.2..41.2 rows=1 loops=1)
  ->  Bitmap Heap Scan on events  (cost=48.1..21230 rows=124000 width=0)
                                  (actual time=1.1..33.8 rows=121880 loops=1)
        Recheck Cond: (created_at >= '2026-07-01' AND created_at <= '2026-07-02')
        Rows Removed by Index Recheck: 3120
        Heap Blocks: lossy=1536
        ->  Bitmap Index Scan on idx_events_ts_brin
              (cost=0.00..17.1 rows=124000 width=0) (actual time=0.4..0.4 rows=15360 loops=1)
              Index Cond: (created_at >= '2026-07-01' AND created_at <= '2026-07-02')
  Buffers: shared hit=1540
Planning Time: 0.2 ms
Execution Time: 41.4 ms
```

Reading it: `Heap Blocks: lossy=1536` — BRIN returns whole *block ranges*, so the heap scan is lossy and every candidate row is rechecked (`Rows Removed by Index Recheck: 3120`). The BRIN index scan itself touched almost nothing (the tiny summary). 41ms over a 200M-row table using a ~300KB index; a B-tree would be gigabytes.

### What to Look For

| Plan node / line | Index type it implies | Meaning |
|------------------|----------------------|---------|
| `Index Scan` / `Index Scan Backward` | B-tree (or GiST for `Order By:`) | Direct ordered access |
| `Index Only Scan` | B-tree/GiST with covering cols | No heap fetch (page all-visible) |
| `Bitmap Index Scan` → `Bitmap Heap Scan` | GIN, BRIN, or multi-index B-tree | Candidate bitmap + recheck |
| `Recheck Cond` present | GIN/GiST/BRIN (lossy) | False positives filtered — normal |
| `Order By:` (not `Index Cond:`) | GiST/SP-GiST KNN | Distance-ordered walk |
| `Heap Blocks: lossy=…` | BRIN (or bitmap overflow) | Whole block ranges returned |
| Seq Scan despite an index existing | Wrong opclass/type, or not selective | Index can't serve the operator |

---

## 8. Query Examples

### Example 1 — Basic: B-tree for Equality + Range + Sort

```sql
-- Single composite B-tree serves the filter and the ordering.
CREATE INDEX idx_orders_status_created
    ON orders (status, created_at DESC);

-- Uses the index for `status =` (equality prefix) and the DESC ordering (no Sort).
SELECT id, customer_id, total_amount, created_at
FROM orders
WHERE status = 'completed'
  AND created_at >= CURRENT_DATE - INTERVAL '30 days'
ORDER BY created_at DESC
LIMIT 50;
```

### Example 2 — Intermediate: GIN for JSONB Containment + Full-Text

```sql
-- audit_logs.payload is JSONB; we only ever query it with @> containment.
-- jsonb_path_ops is smaller/faster than default jsonb_ops for @>-only workloads.
CREATE INDEX idx_audit_payload
    ON audit_logs USING gin (payload jsonb_path_ops);

-- Find every audit event for a failed login by a given actor.
SELECT id, actor_id, created_at, payload
FROM audit_logs
WHERE payload @> '{"action":"login","result":"failure"}'
  AND created_at >= CURRENT_DATE - INTERVAL '7 days'
ORDER BY created_at DESC
LIMIT 100;

-- Full-text over a generated tsvector column, indexed with GIN.
ALTER TABLE products ADD COLUMN IF NOT EXISTS search_tsv tsvector
    GENERATED ALWAYS AS (to_tsvector('english', coalesce(name,'') || ' ' || coalesce(description,''))) STORED;
CREATE INDEX idx_products_search ON products USING gin (search_tsv);

SELECT id, name, ts_rank(search_tsv, query) AS rank
FROM products, to_tsquery('english', 'wireless & (mouse | keyboard)') query
WHERE search_tsv @@ query
ORDER BY rank DESC
LIMIT 20;
```

### Example 3 — Production Grade: BRIN + GIN + Partial B-tree on a 200M-Row Events Table

```sql
-- Table: audit_logs, 200M rows, append-only, inserted in strict created_at order.
-- Access patterns:
--   (a) time-range analytics scans  (huge ranges)      → BRIN on created_at
--   (b) filter by JSONB attributes  (@> containment)   → GIN on payload
--   (c) pinpoint lookups of the rare 'critical' events → partial B-tree
--
-- Sizes at 200M rows (measured, approximate):
--   B-tree on created_at ......... ~4.3 GB   (rejected: huge + write cost)
--   BRIN  on created_at .......... ~0.3 MB   (chosen)
--   GIN   on payload (path_ops) .. ~2.1 GB
--   partial B-tree (critical) .... ~9 MB     (only ~250k rows match)
--
-- Perf expectation:
--   (a) 1-day range count: ~40 ms (BRIN bitmap, ~1500 blocks) vs ~90 s Seq Scan
--   (b) containment lookup: ~3-8 ms (GIN bitmap)
--   (c) critical lookup: <1 ms (tiny partial index, index-only capable)

CREATE INDEX idx_audit_created_brin
    ON audit_logs USING brin (created_at) WITH (pages_per_range = 64);

CREATE INDEX idx_audit_payload_gin
    ON audit_logs USING gin (payload jsonb_path_ops) WITH (fastupdate = on);

CREATE INDEX idx_audit_critical
    ON audit_logs (created_at DESC)
    WHERE (payload ->> 'severity') = 'critical';

-- Query (a): time-range aggregation — served by BRIN
SELECT date_trunc('hour', created_at) AS hour, count(*)
FROM audit_logs
WHERE created_at >= '2026-07-17' AND created_at < '2026-07-18'
GROUP BY 1 ORDER BY 1;

-- Its EXPLAIN (abbreviated):
--   HashAggregate
--     ->  Bitmap Heap Scan on audit_logs   (Recheck Cond ...; Heap Blocks: lossy=1180)
--           ->  Bitmap Index Scan on idx_audit_created_brin
-- Execution Time: ~38 ms

-- Query (b): containment — served by GIN
SELECT id, created_at, payload
FROM audit_logs
WHERE payload @> '{"resource":"billing","action":"refund"}'
  AND created_at >= '2026-07-01'
ORDER BY created_at DESC LIMIT 50;
```

---

## 9. Wrong → Right Patterns

### Wrong 1: B-tree on JSONB expecting `@>` to use it

```sql
-- WRONG: B-tree has no operator class for @> on jsonb.
CREATE INDEX idx_events_data_bad ON events (payload);   -- defaults to btree

SELECT * FROM events WHERE payload @> '{"status":"active"}';
-- Result: Seq Scan on events. The index EXISTS but is never used.
-- WHY: at the execution level the planner looks for an index whose opclass
-- supports the @> operator; btree/jsonb_ops has none, so it's ineligible.
```

```sql
-- RIGHT: GIN with a jsonb operator class.
CREATE INDEX idx_events_data ON events USING gin (payload jsonb_path_ops);
-- Now @> is served by a Bitmap Index Scan.
```

### Wrong 2: B-tree for `LIKE '%needle%'`

```sql
-- WRONG: leading-wildcard LIKE cannot use a normal B-tree.
CREATE INDEX idx_users_name ON users (name);
SELECT * FROM users WHERE name LIKE '%smith%';
-- Result: Seq Scan. B-tree ordering only helps anchored prefixes ('smith%').
-- WHY: a %-prefix means the search can start anywhere in the ordered keyspace,
-- so the tree's ordering gives no pruning.
```

```sql
-- RIGHT: trigram GIN makes arbitrary-substring search indexable.
CREATE EXTENSION IF NOT EXISTS pg_trgm;
CREATE INDEX idx_users_name_trgm ON users USING gin (name gin_trgm_ops);
SELECT * FROM users WHERE name LIKE '%smith%';   -- now a Bitmap Index Scan
```

### Wrong 3: B-tree on a low-correlation huge column instead of BRIN — or BRIN on uncorrelated data

```sql
-- WRONG (variant A): a 4GB B-tree on created_at of a 200M-row append-only table,
-- adding write latency and cache pressure for range scans that BRIN handles in 300KB.
CREATE INDEX idx_events_created_btree ON events (created_at);

-- WRONG (variant B): BRIN on a column with correlation ≈ 0 (e.g. a random UUID).
CREATE INDEX idx_events_uuid_brin ON events USING brin (id_uuid);
SELECT * FROM events WHERE id_uuid = '...';
-- Result: BRIN prunes nothing (every block range's [min,max] spans the whole space),
-- so it degrades to a full scan PLUS recheck overhead — worse than no index.
```

```sql
-- RIGHT: BRIN for the correlated column, B-tree/hash for the random one.
SELECT correlation FROM pg_stats WHERE tablename='events' AND attname='created_at';  -- ~0.99
CREATE INDEX idx_events_created_brin ON events USING brin (created_at);   -- tiny, effective
CREATE INDEX idx_events_uuid_btree   ON events (id_uuid);                 -- point lookups
```

### Wrong 4: Composite B-tree with the columns in the wrong order

```sql
-- WRONG: index leads with the range column, so the equality can't seek efficiently
-- and the ORDER BY still needs a Sort.
CREATE INDEX idx_orders_created_cust ON orders (created_at, customer_id);
SELECT * FROM orders WHERE customer_id = 42 ORDER BY created_at DESC LIMIT 10;
-- Result: cannot use the index to seek customer_id=42 directly (it's the 2nd column);
-- planner may Bitmap-scan or Seq-scan then Sort.
```

```sql
-- RIGHT: equality column first, sort column second (matching the ORDER BY direction).
CREATE INDEX idx_orders_cust_created ON orders (customer_id, created_at DESC);
-- Now: Index Scan, Index Cond: customer_id=42, no Sort, LIMIT stops early.
```

### Wrong 5: Using default `jsonb_ops` when you only need `@>`

```sql
-- WRONG (wasteful): default jsonb_ops indexes every key AND value — 2-3x larger,
-- slower to build and maintain — when the app only ever uses @> containment.
CREATE INDEX idx_audit_payload ON audit_logs USING gin (payload);   -- jsonb_ops
```

```sql
-- RIGHT: jsonb_path_ops — supports only @> but is far smaller/faster for it.
CREATE INDEX idx_audit_payload ON audit_logs USING gin (payload jsonb_path_ops);
-- Only choose the default jsonb_ops if you also need ?, ?&, ?| key-existence operators.
```

---

## 10. Performance Profile

### Build Cost, Size, and Write Overhead by Type

| Type | Build speed | Index size (rel.) | Insert cost | Read for its operators |
|------|-------------|-------------------|-------------|------------------------|
| B-tree | Fast | Medium | Low | Excellent (log n) |
| Hash | Fast | Small (fixed-width) | Low | Excellent for `=` only |
| GIN | Slow | Large (can exceed table) | High (mitigated by pending list) | Excellent for containment/FTS |
| GiST | Medium | Medium | Medium–High | Good; only option for KNN/overlap |
| SP-GiST | Medium | Small–Medium | Medium | Good for prefix/points/inet |
| BRIN | Very fast | Tiny (KB) | Negligible | Good *only* with high correlation |

### Scaling at 1M / 10M / 100M Rows

- **B-tree** depth grows logarithmically: a lookup is ~3 page reads at 1M and ~4–5 at 100M — essentially flat latency. Size grows linearly (~roughly the indexed column width × rows plus overhead). At 100M rows a `timestamptz` B-tree is a few GB; this is where you weigh it against BRIN.
- **GIN** read latency scales with the number of matching rows and pending-list length, not table size directly. Its *size* scales with total number of distinct elements × occurrences — on large JSONB corpora it frequently exceeds the base table. Write throughput is the real ceiling: without `fastupdate`, high-volume inserts serialise on index maintenance.
- **GiST** search cost depends on bounding-box overlap quality; with well-separated data it stays near-logarithmic, but pathological overlap can force multi-subtree descent. KNN queries with small LIMITs stay fast regardless of table size.
- **BRIN** latency is dominated by the heap recheck of surviving block ranges, so it scales with *how many blocks survive pruning*, which for a tight range on well-correlated data is roughly constant regardless of total table size. This is why BRIN is the go-to for 100M+ append-only tables: 300KB index, sub-second range scans.

### Index Interactions and Optimisation Techniques

- **Bitmap AND/OR combining:** the planner can bitmap-scan two indexes (e.g. a GIN on tags and a BRIN on time) and AND their bitmaps — you often want *several* single-column indexes of different types rather than one impossible composite.
- **Partial indexes** shrink low-selectivity B-trees dramatically: `WHERE status='pending'` on a table that's 99% completed yields a tiny, hot index.
- **`fastupdate` tuning** for GIN: leave on for write-heavy, consider off + smaller `gin_pending_list_limit` for read-heavy-with-strict-latency.
- **`pages_per_range` / `minmax_multi`** for BRIN: lower `pages_per_range` = finer pruning at slightly larger size; `minmax_multi` (PG14+) tolerates moderate disorder.
- **Covering (`INCLUDE`)** on B-tree/GiST enables index-only scans — verify with `Index Only Scan` and keep `autovacuum` healthy so the visibility map stays warm.
- **CLUSTER / correlation:** physically ordering a table by a B-tree key (`CLUSTER`) both speeds range scans and makes BRIN on that column effective — but CLUSTER takes an exclusive lock and isn't maintained on new writes.

---

## 11. Node.js Integration

Index type is a schema/DDL concern, so with `pg` you typically create indexes in migrations and then simply run queries whose operators the index serves. The `pg` driver is fully transparent here — it sends SQL and returns rows; it neither knows nor cares which index the planner picked.

### 11.1 Creating indexes safely from a migration script

```javascript
import pg from 'pg';
const { Pool } = pg;
const pool = new Pool({ connectionString: process.env.DATABASE_URL });

async function createIndexes() {
  // CONCURRENTLY cannot run inside a transaction block — run each on its own.
  // Use a dedicated client so we don't hold a pooled connection in a weird state.
  const client = await pool.connect();
  try {
    await client.query(`CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_audit_created_brin
                        ON audit_logs USING brin (created_at) WITH (pages_per_range = 64)`);
    await client.query(`CREATE INDEX CONCURRENTLY IF NOT EXISTS idx_audit_payload_gin
                        ON audit_logs USING gin (payload jsonb_path_ops)`);
  } finally {
    client.release();
  }
}
```

### 11.2 Querying a GIN-indexed JSONB column with parameters

```javascript
// $1 is bound as jsonb; the @> containment uses the GIN index.
async function findAuditEvents(filter, sinceDays = 7) {
  const { rows } = await pool.query(
    `SELECT id, actor_id, created_at, payload
       FROM audit_logs
      WHERE payload @> $1::jsonb
        AND created_at >= NOW() - ($2 || ' days')::interval
      ORDER BY created_at DESC
      LIMIT 100`,
    [JSON.stringify(filter), sinceDays]   // filter e.g. { action: 'login', result: 'failure' }
  );
  return rows;
}
```

### 11.3 Array containment (GIN) with a parameterised array

```javascript
// tags @> $1 uses a GIN index on tags. pg maps a JS array to a Postgres array.
async function articlesWithAllTags(tags) {
  const { rows } = await pool.query(
    `SELECT id, title FROM articles WHERE tags @> $1 ORDER BY published_at DESC LIMIT 50`,
    [tags]   // e.g. ['postgres', 'indexing']  → text[]
  );
  return rows;
}
```

### 11.4 Verifying the planner used the index you expect

```javascript
// In tests/CI, assert the plan uses the intended access method.
async function assertUsesBrin() {
  const { rows } = await pool.query(
    `EXPLAIN (FORMAT JSON)
     SELECT count(*) FROM audit_logs
      WHERE created_at BETWEEN $1 AND $2`,
    ['2026-07-17', '2026-07-18']
  );
  const planText = JSON.stringify(rows[0]['QUERY PLAN']);
  if (!planText.includes('idx_audit_created_brin')) {
    throw new Error('Expected BRIN index scan; planner chose a different path');
  }
}
```

**ORM note:** No mainstream ORM chooses or reasons about index *type* for you — index creation is left to migrations/raw DDL, and query builders emit the operators (`@>`, `<->`, `@@`) via raw fragments. The planner does the rest. Getting the index *type* right is always a hand-authored DDL decision.

---

## 12. ORM Comparison

The key question for index types is not "can the ORM run the query?" but **"can the ORM (a) declare a non-B-tree index in its schema, and (b) emit the specialised operator (`@>`, `<->`, `@@`) that the index serves?"** Most ORMs are weak on both and fall back to raw SQL.

### Prisma

**Can it?** Partly. Prisma's schema supports index *type* via `type:` on `@@index` for `Hash`, `Gist`, `Gin`, `SpGist`, `Brin` (with the `postgresqlExtensions`/preview features enabled historically; now stable). But Prisma Client's query API has **no** operator for `@>`, `<->`, or `@@` — you must drop to `$queryRaw`.

```prisma
model AuditLog {
  id        Int      @id @default(autoincrement())
  payload   Json
  createdAt DateTime @default(now())
  @@index([payload], type: Gin)
  @@index([createdAt], type: Brin)
}
```

```typescript
// Querying the GIN index requires raw SQL — Client has no @> operator.
const rows = await prisma.$queryRaw`
  SELECT id, payload FROM "AuditLog"
  WHERE payload @> ${JSON.stringify({ action: 'login' })}::jsonb
  ORDER BY "createdAt" DESC LIMIT 100`;
```

**Where it breaks:** the entire specialised-operator surface. **Verdict:** declare the index type in schema, query via `$queryRaw`.

### Drizzle ORM

**Can it?** Yes for declaration — `index().using('gin', ...)`, `.using('brin', ...)`, and operator-class helpers. For queries, use the `sql` template for the operator.

```typescript
import { pgTable, index, jsonb, timestamp, serial } from 'drizzle-orm/pg-core';
import { sql } from 'drizzle-orm';

export const auditLogs = pgTable('audit_logs', {
  id: serial('id').primaryKey(),
  payload: jsonb('payload').notNull(),
  createdAt: timestamp('created_at').defaultNow(),
}, (t) => ({
  payloadGin: index('idx_payload_gin').using('gin', t.payload),
  createdBrin: index('idx_created_brin').using('brin', t.createdAt),
}));

// Query with the containment operator via sql template:
const rows = await db.select().from(auditLogs)
  .where(sql`${auditLogs.payload} @> ${JSON.stringify({ action: 'login' })}::jsonb`)
  .limit(100);
```

**Where it breaks:** operator classes like `jsonb_path_ops` need raw `sql` in the index definition. **Verdict:** best-in-class support; declares all six types, operators via `sql`.

### Sequelize

**Can it?** Index type via `indexes: [{ using: 'GIN', fields: [...] }]` in the model (`using` accepts `BTREE`, `HASH`, `GIN`, `GIST`, `SPGIST`, `BRIN`). Queries: operators like `@>` are available via `Op.contains` for arrays/ranges; JSONB containment and FTS need `sequelize.literal`/`Sequelize.fn`.

```javascript
const AuditLog = sequelize.define('AuditLog', { payload: DataTypes.JSONB }, {
  indexes: [{ using: 'GIN', fields: ['payload'] }],
});

// Array/range contains has an operator; jsonb @> often needs literal:
const rows = await AuditLog.findAll({
  where: sequelize.literal(`payload @> '{"action":"login"}'::jsonb`),
  limit: 100,
});
```

**Where it breaks:** operator classes and KNN (`<->`) — raw only. **Verdict:** declares types; specialised operators are patchy, expect `literal`.

### TypeORM

**Can it?** Index type is limited historically — `@Index` doesn't cleanly expose `USING gin/brin` across versions; teams commonly declare specialised indexes in a raw migration `queryRunner.query('CREATE INDEX ... USING gin ...')`. Queries use `.where('payload @> :p', {...})` raw fragments.

```typescript
// Migration (raw) — the reliable path:
await queryRunner.query(
  `CREATE INDEX idx_payload_gin ON audit_log USING gin (payload jsonb_path_ops)`);

// Query:
const rows = await repo.createQueryBuilder('a')
  .where(`a.payload @> :p`, { p: JSON.stringify({ action: 'login' }) })
  .limit(100).getMany();
```

**Where it breaks:** decorator-level type declaration; use raw migrations. **Verdict:** raw migrations for the index, raw fragments for the operator.

### Knex.js

**Can it?** Yes via `.index(cols, { indexType })` / `.specificType`, but for GIN/BRIN with operator classes most people use `knex.raw`. Queries use `.whereRaw` for the operators.

```javascript
await knex.raw(
  `CREATE INDEX idx_payload_gin ON audit_logs USING gin (payload jsonb_path_ops)`);

const rows = await knex('audit_logs')
  .whereRaw(`payload @> ?::jsonb`, [JSON.stringify({ action: 'login' })])
  .orderBy('created_at', 'desc').limit(100);
```

**Where it breaks:** nothing structurally — Knex is transparent; you just write the DDL and operator yourself. **Verdict:** most honest; raw DDL, raw operator, full control.

### ORM Summary Table

| ORM | Declare index type? | Emit `@>` / `<->` / `@@`? | Operator classes? | Verdict |
|-----|---------------------|---------------------------|-------------------|---------|
| Prisma | Yes (`type:` on `@@index`) | No — `$queryRaw` | No | Schema declares, raw queries |
| Drizzle | Yes (`.using(...)`) | Via `sql` template | Via `sql` | Best support |
| Sequelize | Yes (`using:`) | Partial (`Op.contains`) | No | Types yes, operators patchy |
| TypeORM | Weak — raw migration | Raw fragment | Raw migration | Raw DDL + raw fragments |
| Knex | Via raw | `.whereRaw` | Via raw | Transparent, full control |

Across the board: **the index *type* decision is yours to make in DDL; no ORM optimises it for you.**

---

## 13. Practice Exercises

### Exercise 1 — Easy

You have `products(id, name, description, price, category_id)`. Two query patterns dominate:
1. `WHERE category_id = $1 ORDER BY price ASC LIMIT 20`
2. `WHERE name ILIKE 'wireless%'` (anchored prefix, case-insensitive)

Write the `CREATE INDEX` statements that make both fast, choosing the correct type and (for pattern 2) the correct operator class or expression. Explain in one line why a plain `CREATE INDEX ON products(name)` would *not* serve pattern 2 on a default (non-C) locale.

```sql
-- Write your indexes here
```

### Exercise 2 — Medium (combines Topic 11 JOINs + indexing)

`orders(id, customer_id, status, created_at)` and `order_items(id, order_id, product_id)`. This query is slow:

```sql
SELECT o.id, count(*) AS items
FROM orders o
JOIN order_items oi ON oi.order_id = o.id
WHERE o.status = 'completed'
  AND o.created_at >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY o.id;
```

Design the indexes so that: (a) the `orders` filter is served by a single B-tree that also avoids a Sort if you later add `ORDER BY o.created_at`, and (b) the join's inner side is indexed. Then state which join algorithm you'd expect the planner to choose and why.

```sql
-- Write your indexes here
```

### Exercise 3 — Hard (production simulation; the naive answer is wrong)

`audit_logs(id bigserial, created_at timestamptz, actor_id int, payload jsonb)` has **300M rows**, is append-only in `created_at` order, and serves:
1. Hourly-bucketed counts over arbitrary day ranges (huge scans).
2. `payload @> '{"resource":"billing"}'` containment lookups scoped to a recent day.
3. Occasional pinpoint retrieval of the ~0.1% of rows where `payload->>'severity' = 'critical'`.

A junior proposes: `CREATE INDEX ON audit_logs(created_at); CREATE INDEX ON audit_logs(payload);` Explain why *both* of those are the wrong *type* for this table, then write the three indexes you would actually create (with types, operator classes, and any storage parameters), and justify each in one sentence. Include the `pg_stats` check you would run before committing to one of them.

```sql
-- Write your indexes and the pg_stats check here
```

### Exercise 4 — Interview Simulation

An interviewer says: *"We have a `stores(id, name, geom geometry(Point,4326))` table with 5M stores. Product wants a 'nearest 10 stores to me' feature and a 'stores within this map viewport' feature. What indexes, and can a B-tree do either?"* Write the DDL and a one-paragraph justification covering why B-tree cannot serve nearest-neighbour, which operator each feature uses, and what EXPLAIN line proves the KNN index is working.

```sql
-- Write your answer here
```

---

## 14. Interview Questions

### Q1 — "When would you choose a Hash index over a B-tree?"

**Junior answer:** "Hash is faster for equality." (True but incomplete, and often wrong in practice.)

**Principal answer:** "Almost never, and I'd justify it with a measurement. A B-tree already serves `=` well and also serves ranges, sorting, and uniqueness, so it's the safe default. Hash is *only* considered when the workload is strictly equality on a *large* key where the fixed-width hash yields a smaller index and cheaper probe than descending a wide B-tree — and only since PostgreSQL 10, when hash indexes became WAL-logged and crash-safe. Before 10 they weren't replicated or crash-safe, which is why the old advice was 'never use hash.' Even now, hash can't be UNIQUE, can't be multicolumn, can't do `IS NULL`, and can't order — so the moment a second access pattern appears, you're adding a B-tree anyway. I'd only ship a hash index after EXPLAIN + benchmark showed a real size/latency win on an equality-only column."

**Follow-up the interviewer asks:** "So what changed in PostgreSQL 10 specifically?" (Answer: hash indexes gained WAL logging → crash safety and replication support; before that they were effectively unusable in production.)

### Q2 — "You added a GIN index on a JSONB column but writes got much slower. Why, and what would you do?"

**Junior answer:** "GIN indexes are just slow to write." (Names the symptom, not the mechanism or the fix.)

**Principal answer:** "GIN is an inverted index: one row with N keys/values touches up to N posting lists, so each insert is far more work than a B-tree's single leaf insert. PostgreSQL mitigates this with the *pending list* (`fastupdate = on` by default): new entries are appended unsorted and merged into the main index later by autovacuum or when the list hits `gin_pending_list_limit`. If writes are slow, I'd check whether the pending list is being merged too eagerly or the limit is too small, tune `gin_pending_list_limit` up for write bursts, ensure autovacuum is keeping pace, and consider `jsonb_path_ops` instead of `jsonb_ops` — it indexes hashed paths only, so it's 2–3× smaller and cheaper to maintain when I only need `@>`. If the table is extremely write-heavy and containment queries are rare, I'd question whether the GIN index earns its keep at all."

**Follow-up:** "What's the read-side trade-off of a large pending list?" (Answer: reads must scan the unsorted pending list in addition to the main index, so read latency rises until a merge; `fastupdate=off` trades write speed for stable read latency.)

### Q3 — "Design indexes for a 200M-row append-only events table queried by time range and by a tags array."

**Junior answer:** "B-tree on the timestamp, GIN on tags."

**Principal answer:** "GIN on tags, yes. But for the timestamp I'd reach for **BRIN**, not B-tree, *because the table is append-only and therefore physically ordered by time*. A B-tree on `created_at` at 200M rows is multiple GB, adds real write latency, and evicts useful pages from cache — whereas a BRIN is a few hundred KB, has negligible write cost, and prunes block ranges for range scans. The one thing I'd verify first is `SELECT correlation FROM pg_stats WHERE attname='created_at'` — BRIN only works if that's near ±1, which append-only ingestion gives us. If there were some disorder (late-arriving events), I'd use `minmax_multi` (PG 14+). I'd tune `pages_per_range` to trade index size against pruning granularity. For tags, GIN with `fastupdate` tuned to the write pattern. And I'd note the planner can bitmap-AND the BRIN and GIN results for queries that filter on both."

**Follow-up:** "What breaks BRIN, and how would you detect it in production?" (Answer: low correlation — if the table gets updated/reordered or ingestion isn't time-ordered, BRIN prunes nothing; detect via `pg_stats.correlation` dropping and EXPLAIN showing `Heap Blocks: lossy=` covering most of the table with no reduction in scanned rows.)

---

## 15. Mental Model Checkpoint

1. A query filters a `jsonb` column with `@>`. You create `CREATE INDEX ON t(col)`. Why does EXPLAIN still show a Seq Scan, and what single word in the `CREATE INDEX` fixes it?

2. You need `WHERE customer_id = ? ORDER BY created_at DESC LIMIT 10` with no Sort node. What is the exact column order and direction of the B-tree, and why does reversing the two columns break it?

3. A colleague says "BRIN is a smaller B-tree." In one sentence, explain why that's fundamentally wrong at the structural level, and name the single table property that decides whether BRIN works at all.

4. Which two index types can satisfy an `ORDER BY` directly from the index, and what *kind* of ordering does each provide (they are not the same kind)?

5. You have a GIN index and reads are occasionally slow right after big insert batches. What internal GIN mechanism explains this, and what are the two knobs you'd consider?

6. For an exclusion constraint "no two bookings for the same room may overlap in time," which index type is mandatory, which extension do you need, and why can't a UNIQUE B-tree express this?

7. When would SP-GiST beat GiST for the *same* data (e.g. points or ranges), and what property of the data makes the difference?

---

## 16. Quick Reference Card

```sql
-- ── DEFAULTS & SYNTAX ────────────────────────────────────────────
CREATE INDEX name ON t USING method (cols) [INCLUDE(...)] [WHERE ...];
-- method omitted → btree

-- ── PICK BY OPERATOR ─────────────────────────────────────────────
--  =  <  >  BETWEEN  ORDER BY  LIKE 'x%' ....... btree
--  =  (equality only, large key) ................ hash
--  @>  <@  &&  ?  ?&  ?|  (array/jsonb) ......... gin
--  @@  (full-text) .............................. gin
--  LIKE '%x%'  /  fuzzy % ....................... gin (gin_trgm_ops)
--  &&  @>  <@  (geo/range overlap) .............. gist
--  <->  (nearest-neighbour, ORDER BY) ........... gist / spgist
--  EXCLUDE (... WITH &&) ........................ gist (+ btree_gist)
--  prefix at scale / points / inet ............. spgist
--  range scan on HUGE ordered table ............ brin

-- ── B-TREE COMPOSITE RULE ────────────────────────────────────────
-- equality cols FIRST, range/sort col LAST, match ORDER BY direction:
CREATE INDEX ON orders (customer_id, created_at DESC);   -- serves =, then sort, no Sort node

-- ── GIN JSONB OPCLASS CHOICE ─────────────────────────────────────
USING gin (payload)                 -- jsonb_ops:      @> ? ?& ?|   (bigger)
USING gin (payload jsonb_path_ops)  -- @> only, ~2-3x smaller/faster

-- ── BRIN — CHECK CORRELATION FIRST ───────────────────────────────
SELECT correlation FROM pg_stats WHERE tablename=? AND attname=?;  -- want ≈ ±1
CREATE INDEX ON events USING brin (created_at) WITH (pages_per_range=64);
-- disorder/outliers? → timestamptz_minmax_multi_ops (PG14+)

-- ── SUBSTRING / FUZZY ────────────────────────────────────────────
CREATE EXTENSION pg_trgm;
CREATE INDEX ON users USING gin (name gin_trgm_ops);   -- LIKE '%x%', %similarity

-- ── PARTIAL / EXPRESSION / COVERING ──────────────────────────────
CREATE INDEX ON orders (created_at) WHERE status='pending';   -- tiny, hot
CREATE INDEX ON users (lower(email));                          -- match expr in query
CREATE INDEX ON orders (customer_id) INCLUDE (total_amount);  -- index-only scan (btree/gist)

-- ── ONLINE DDL ───────────────────────────────────────────────────
CREATE INDEX CONCURRENTLY ...;   -- no write lock; not in a txn; DROP if it ends INVALID
```

**Perf rules of thumb**
- The index *type* must match the query's *operator*, or the index is dead weight.
- B-tree is the default because it serves the most operators; only leave it for a specific need.
- GIN: big index, expensive writes, unbeatable for containment/FTS. Tune `fastupdate`.
- GiST: the only path to KNN and overlap/exclusion. Expect lossy rechecks.
- BRIN: 10,000× smaller than B-tree on huge ordered tables — but worthless without correlation.
- Hash: rarely worth it; B-tree serves `=` plus everything else.

**Interview one-liners**
- "Wrong index type = Seq Scan with an index sitting unused." 
- "BRIN is a per-block-range summary, not a per-row tree — it needs physical/value correlation."
- "GIN is an inverted index; its pending list trades write speed for read latency until merge."
- "Only B-tree (sorted) and GiST (KNN distance) can feed an ORDER BY from the index."
- "Hash indexes became crash-safe/WAL-logged in PostgreSQL 10 — older advice is obsolete but their niche stayed tiny."

---

## Connected Topics

- **Topic 40 — B-tree Internals** (prior phase): the leaf/branch structure, deduplication, and fillfactor that this topic assumes.
- **Topic 41 — Top-N Per Group**: relies directly on an ordered B-tree + LIMIT to avoid sorting the whole partition — the ordered-index superpower named here.
- **Topic 43 — Index Scan Types** (next): how the planner turns these index *types* into Index Scan, Index Only Scan, and Bitmap Heap Scan, and how it costs each.
- **Topic 11 — INNER JOIN in Depth**: why an index on the inner join key turns a Nested Loop from catastrophic to instant.
- **Internals — MVCC & VACUUM**: why GIN has a pending list, why index-only scans need the visibility map, and why bloat differs per access method.
- **Internals — The Planner & `pg_statistic`**: `correlation` (the make-or-break stat for BRIN), selectivity, and the per-AM cost estimators that decide index vs Seq Scan.
- **Topic 44+ — Query Optimisation**: partial, expression, and covering indexes as tools for shrinking and specialising the types introduced here.
