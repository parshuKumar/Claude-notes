# 62 — Database Indexing — How Indexes Work Under the Hood
## Category: HLD Components

---

## What is this?

An **index** is a separate, sorted data structure that a database keeps *alongside* your table so it can find rows without reading the whole table. Think of the index at the back of a textbook: instead of flipping through 900 pages looking for "photosynthesis", you jump to the index, find "photosynthesis — p. 412", and turn straight to that page.

That analogy gets you in the door, but it hides the interesting part. The book index is alphabetical and flat. A database index is a **tree** — and the *shape* of that tree was chosen not because it's elegant, but because reading from disk is brutally expensive and the tree is engineered to touch as few disk pages as humanly possible. Everything in this document flows from that one fact.

---

## Why does it matter?

Because indexing is the single highest-leverage performance knob in almost every backend you will ever build, and it is the thing that quietly kills products.

**What breaks without it:** A query that runs in 2ms on your laptop with 5,000 seed rows runs in 8 seconds in production with 20 million rows. Nothing in your code changed. The query is the same. The database is now reading every single row on every single request, your connection pool saturates, requests queue, the API gateway times out, and your on-call phone rings. The fix is usually a single `CREATE INDEX` statement — and knowing *which* one to write is the skill.

**What breaks with too much of it:** A table with 12 indexes takes 12 extra writes on every `INSERT`. Your write throughput collapses, your bulk import that used to take 4 minutes takes 40, and your disk bill doubles. Indexes are not free. They are a trade: **you buy read speed with write speed and storage.**

**Interview angle:** "Your query is slow. Walk me through what you'd do." The expected answer is: run `EXPLAIN`, look at whether it's a sequential scan, understand *why* the planner chose that, add the right index (usually composite, with the right column order), and `EXPLAIN` again to prove it worked. Interviewers also love: "Why do databases use B+ trees instead of binary search trees?" — a question that separates people who memorized "indexes make things fast" from people who actually understand the machine.

**Real-work angle:** Every schema review, every slow-query alert, every "the dashboard page takes 11 seconds" ticket. This is daily work, not exotica.

---

## The core idea — explained simply

### The Warehouse Analogy

Imagine a warehouse with **10 million cardboard boxes** stacked on shelves. Each box has a slip of paper inside with a customer's order details. The boxes were placed on the shelves in the order they arrived — box #1, box #2, ... box #10,000,000. There is no other organization.

Your boss says: **"Find me the order for customer email `ada@lovelace.dev`."**

**Option A — the full scan.** You start at shelf 1, box 1. Open it. Read the slip. Not Ada. Next box. Not Ada. You do this 10 million times. Even at 10,000 boxes per second you're there for **~17 minutes**. And crucially — you have to do this *even if Ada's box is the very first one*, because you don't know there isn't a *second* Ada box further down. You must open every box to be sure.

**Option B — the card catalog.** Now imagine a clerk maintains a card catalog by the door. Every card has an email and a shelf/box number, and the cards are kept in **sorted order**. Even better: the catalog is organized into drawers, and there's a small directory taped to the front telling you which drawer covers which email range.

You walk to the directory (`a–f → drawer 1`), open drawer 1, find a divider card inside (`aa–ad → section 3`), flip to section 3, and there's the card: `ada@lovelace.dev → shelf 4,182, box 77`. You walk to that shelf, grab the box, done. **Four lookups instead of ten million.**

That card catalog is an index. The directory-of-drawers-of-sections structure is a **B+ tree**. And the reason it isn't just one giant sorted list of ten million cards is the same reason the drawers exist: you can't hold ten million cards in your hand. You can only pull out **one drawer at a time** — and a drawer holds a few hundred cards.

**That drawer is a disk page.** Hold that thought — it is the whole insight.

### Mapping the analogy back

| Warehouse thing | Database thing |
|---|---|
| The boxes on shelves, in arrival order | The **table** (heap), rows in insertion order |
| Opening every box one by one | **Full table scan** / `Seq Scan` — O(n) |
| The card catalog | The **index** |
| One card (email → shelf/box) | One **index entry**: key → row pointer |
| A drawer you pull out and read | A **disk page** (~8 KB), a **node** of the B+ tree |
| The directory taped to the front | The **root node** of the tree |
| The divider cards inside a drawer | **Internal nodes** |
| The actual cards with box numbers | **Leaf nodes** |
| Walking to the shelf to fetch the box | The **bookmark lookup** / heap fetch |
| The clerk updating the catalog on every new box | **Index maintenance cost on every write** |
| The catalog taking up floor space | **Index storage cost** |

### The worked number that makes it real

Table: `orders`, **10,000,000 rows**, each row ~200 bytes. Total ~2 GB on disk.

```sql
SELECT * FROM orders WHERE customer_email = 'ada@lovelace.dev';
```

**Without an index (Seq Scan):**
- The DB must read all ~2 GB from disk into memory and compare `customer_email` on every row.
- 2 GB at ~500 MB/s (a decent SSD, sequential) = **~4 seconds**. On a network disk or a cold cache, worse.
- Rows examined: **10,000,000**. Rows returned: **1**. You did 10 million units of work to produce 1 unit of result.

**With a B+ tree index on `customer_email`:**
- The tree is **4 levels deep** (we'll prove why in a moment).
- Root page → internal page → internal page → leaf page = **4 page reads**, then 1 more read to fetch the actual row = **5 page reads total**.
- 5 pages × 8 KB = **40 KB** read. At ~0.1ms per random SSD page read (and the top levels are almost always cached in RAM anyway): **~0.5ms**.

**~4,000ms → ~0.5ms. That's an 8,000x improvement from one line of DDL.** That is the entire pitch for indexing, and it is not an exaggeration.

O(n) vs O(log n). But *which* log — and that's where B+ trees come in.

---

## Key concepts inside this topic

### 1. B-trees and B+ trees — the heart of it

A **binary search tree** (BST) is the data structure you learned in school: each node has a key and two children, left = smaller, right = bigger. Lookup is O(log₂ n). For 10 million rows that's log₂(10,000,000) ≈ **23 levels**. In a balanced BST you visit ~23 nodes to find your row.

23 comparisons is *nothing* for a CPU. So why don't databases use BSTs?

**Because the nodes are on disk, not in RAM.** Each node is a separate little object that could be *anywhere* on the disk. Visiting 23 nodes means **23 random disk reads**. On a spinning disk at ~10ms per seek, that's 230ms. On an SSD at ~0.1ms, it's 2.3ms — better, but still 23 round trips through the I/O stack, and a real BST-on-disk with pointer overhead is closer to 33 levels once you account for the fact that a node holding one 40-byte key wastes an entire 8 KB page.

The database doesn't optimize for CPU comparisons. **It optimizes for the number of pages it has to fetch.** And here's the lever: *the disk gives you an entire 8 KB page whether you want it or not.* Reading 1 byte and reading 8,192 bytes cost the same. So if you're going to pay for a page, **fill it with as many keys as you possibly can.**

That is the B-tree idea: a node is **exactly one disk page** (8 KB in PostgreSQL, 16 KB in InnoDB), and it holds **hundreds of keys** with hundreds+1 child pointers. The number of children per node is the **fanout**.

**Fanout math, concretely.** PostgreSQL page = 8,192 bytes. Say your index key is a 4-byte integer, plus a 6-byte pointer, plus overhead — call it ~16 bytes per entry. Minus page headers, you fit roughly:

```
8,192 bytes / 16 bytes per entry  ≈  500 entries per page
```

So **fanout ≈ 500**. Now how many rows can a tree of depth `d` address?

```
Depth 1 (root only):        500          rows
Depth 2:              500 × 500  =  250,000
Depth 3:        500 × 500 × 500  =  125,000,000
Depth 4:  500 × 500 × 500 × 500  =  62,500,000,000   (62.5 BILLION)
```

**A 4-level B+ tree indexes 62 billion rows.** Your 10-million-row table needs a tree of depth **3** (125M capacity ≥ 10M). Three page reads. And the root — and usually the entire second level — is permanently cached in RAM because it's read on every single query. So in practice you pay for **one, maybe two, actual disk reads**.

Compare:

| Structure | Levels for 10M rows | Disk reads per lookup |
|---|---|---|
| Binary search tree | ~23–33 | 23–33 |
| B+ tree (fanout 500) | **3** | **1–3** (top levels cached) |

**This is the punchline of the entire document:** *indexes are shaped by the cost of disk I/O, not by the cost of CPU comparisons.* Every weird thing about B+ trees — the fat nodes, the shallow depth, the page alignment — is a direct consequence of "a disk read is 100,000x slower than a RAM read, so minimize the count of them."

**B-tree vs B+ tree — the difference that matters.**

- In a **B-tree**, data (or row pointers) can live in *any* node, including internal ones.
- In a **B+ tree**, **all data lives in the leaves**. Internal nodes hold *only* keys used for routing — they're pure signposts. And **the leaves are linked together in a doubly-linked list.**

Two consequences, both huge:
1. **Internal nodes get more fanout** (no data payload to store) → the tree is even shallower.
2. **Range scans become trivial.** `WHERE created_at BETWEEN '2026-01-01' AND '2026-03-31'` walks down to the first matching leaf *once*, then just follows the leaf-level linked list sideways. No re-traversal from the root. Same for `ORDER BY` — the leaves are *already in sorted order*, so the database can return sorted results with **zero sorting work**. That's why an index can eliminate a `Sort` node from your query plan entirely.

Essentially every relational database index you'll meet — PostgreSQL's default, MySQL/InnoDB, SQL Server, Oracle — is a **B+ tree**. (People say "B-tree" colloquially, including in PostgreSQL's own syntax, `USING btree`. It's a B+ tree.)

### 2. Clustered vs non-clustered (secondary) indexes

This distinction confuses more engineers than any other part of the topic. It's simple once you see it.

**Clustered index: the index IS the table.**

The rows themselves are stored *inside the leaves of the B+ tree*, physically sorted by the index key. There is no separate heap of rows. Because the physical rows can only be sorted one way, **you get exactly one clustered index per table.**

In **MySQL/InnoDB**, the clustered index is always the **primary key** (if you don't declare one, InnoDB invents a hidden 6-byte rowid). So the table *is* a B+ tree keyed by PK, and looking up by PK is maximally fast — you land on the leaf and the whole row is right there. Zero extra reads.

**Non-clustered (secondary) index: the index points AT the table.**

The leaves contain the indexed key plus a **pointer back to the row**. To get columns not in the index, the database must follow that pointer — a second lookup, called a **bookmark lookup** (SQL Server's term) or a **heap fetch** (PostgreSQL's term).

And here's the InnoDB detail interviewers dig for: **in InnoDB, a secondary index leaf stores the primary key, not a physical row address.** So a lookup by a secondary index is *two tree traversals*:

```
SELECT * FROM users WHERE email = 'ada@lovelace.dev';

1. Walk the `email` secondary B+ tree      → find leaf → get PK = 42     (~3 page reads)
2. Walk the clustered (PK) B+ tree for 42  → find leaf → get the full row (~3 page reads)
```

Why store the PK instead of a physical address? Because rows *move* — a page split or a row update can relocate a row physically. If secondary indexes stored physical addresses, every row move would require updating every secondary index. Storing the PK makes secondary indexes immune to physical relocation. The price is that extra traversal.

**Practical consequence:** keep your primary key **small**. In InnoDB, the PK is copied into *every* secondary index. A `BIGINT` PK (8 bytes) across 5 secondary indexes on a 50M-row table costs 8 × 5 × 50M = **2 GB** just in PK copies. Use a random 36-character UUID string as your PK and you pay 36 bytes × 5 × 50M = **9 GB**, *plus* you destroy insert locality (random PKs mean inserts land on random pages instead of appending to the rightmost page, causing constant page splits). This is why "don't use UUIDv4 as a clustered PK" is real advice, and why UUIDv7 (time-ordered) exists.

PostgreSQL is different: it stores rows in an unordered **heap** and *all* indexes are secondary, holding a physical `ctid` (page, offset). `CLUSTER` exists but is a one-shot physical reorder, not maintained.

| | Clustered | Non-clustered (secondary) |
|---|---|---|
| Where the row lives | In the index leaf | Elsewhere (heap or clustered index) |
| How many per table | **One** | Many |
| Lookup cost | 1 traversal | 1 traversal **+ bookmark lookup** |
| Range scan on the key | Extremely fast (rows are physically adjacent) | Fast at the index level, but random I/O to fetch rows |
| InnoDB | = the PRIMARY KEY | leaf stores the **PK**, not a row address |
| PostgreSQL | none by default (heap table) | leaf stores a `ctid` |

### 3. Composite indexes and the leftmost-prefix rule

A **composite** (multi-column) index is one B+ tree whose key is a *tuple*, sorted by the first column, then the second within the first, then the third within the second.

```sql
CREATE INDEX idx_orders_cust_status_date
  ON orders (customer_id, status, created_at);
```

The entries are ordered like this:

```
(1001, 'paid',    2026-01-04)
(1001, 'paid',    2026-02-19)
(1001, 'shipped', 2026-01-30)
(1002, 'pending', 2026-03-01)
(1002, 'shipped', 2026-01-11)
...
```

### The Phone Book Analogy

A phone book is sorted by **(last_name, first_name)**.

- "Find Lovelace" → easy. Flip to L, done.
- "Find Lovelace, Ada" → easy. Flip to L, scan to Ada.
- "Find everyone named **Ada**" → **useless.** The Adas are scattered across every letter of the book. You'd have to read the whole thing.

That is the **leftmost-prefix rule**, and it is exactly, mechanically why: the tree is sorted by the *first* column first. Skip it, and the values you want are scattered across every leaf.

**HIT vs MISS table** for `INDEX (customer_id, status, created_at)`:

| Query | Uses index? | Why |
|---|---|---|
| `WHERE customer_id = 7` | **HIT** (full) | Leftmost prefix `(a)` |
| `WHERE customer_id = 7 AND status = 'paid'` | **HIT** (full) | Prefix `(a, b)` |
| `WHERE customer_id = 7 AND status = 'paid' AND created_at > '2026-01-01'` | **HIT** (full) | All three, range last |
| `WHERE customer_id = 7 ORDER BY status, created_at` | **HIT** (+ free sort) | Leaves already in that order |
| `WHERE customer_id = 7 AND created_at > '2026-01-01'` | **PARTIAL** | Seeks on `customer_id`, then must filter `created_at` inside that range — `status` gap breaks the seek |
| `WHERE status = 'paid'` | **MISS** | No leftmost column |
| `WHERE created_at > '2026-01-01'` | **MISS** | No leftmost column |
| `WHERE status = 'paid' AND created_at > '2026-01-01'` | **MISS** | Still no `customer_id` |
| `WHERE customer_id > 5 AND status = 'paid'` | **PARTIAL** | A **range** on `a` stops `b` from being used for seeking |

**Two rules that fall out of this:**

1. **Column order matters enormously.** `(customer_id, status)` and `(status, customer_id)` are *different indexes* serving *different queries*. Getting the order wrong is the most common indexing mistake in the wild.
2. **Equality columns first, range/sort columns last.** Once the tree hits a range predicate (`>`, `<`, `BETWEEN`, `LIKE 'abc%'`), everything to its right can no longer be used for seeking — only for filtering rows the DB has already fetched.

Also: a composite index on `(a, b, c)` makes a *separate* index on `(a)` **redundant** — delete it. It does **not** make `(b)` redundant.

### 4. Covering indexes (index-only scans)

If an index contains **every column the query touches** — in `SELECT`, `WHERE`, `ORDER BY`, all of it — the database can answer the query **entirely from the index and never touch the table at all.** No bookmark lookup. This is a **covering index**, and the plan node is called an **Index Only Scan**.

**Before — 1 index seek + 50,000 bookmark lookups:**

```sql
-- index: (customer_id)
SELECT customer_id, status, total_cents
FROM orders
WHERE customer_id = 7;
```
The index finds the 50,000 matching entries fast, but `status` and `total_cents` are *not* in the index — so for each of the 50,000 entries, the DB does a **random read** into the table. 50,000 random reads ≈ **hundreds of milliseconds**.

**After — index only, zero table reads:**

```sql
CREATE INDEX idx_orders_covering
  ON orders (customer_id) INCLUDE (status, total_cents);
-- PostgreSQL 11+. In MySQL, just put them in the key: (customer_id, status, total_cents)
```

Now every column is in the leaf. The DB walks down 3 levels, then reads the matching leaves **sequentially** (remember: leaves are linked). **~10–30ms.** A clean **10x+**, sometimes far more when the table is cold.

`INCLUDE` columns are stored only in the **leaves**, not in internal nodes — so they don't reduce fanout, they don't help you *search*, they just come along for the ride so the DB doesn't have to go get them. That's the right mental model: **key columns are for finding; included columns are for carrying.**

Caveat for PostgreSQL: an Index Only Scan still has to check the **visibility map** to confirm the row isn't a dead MVCC tuple. If the table hasn't been vacuumed recently, you'll see `Heap Fetches: 48291` in your plan and you didn't actually get the win. Run `VACUUM` and check again.

### 5. The other index types

B+ trees are ~95% of what you'll use. Know the rest exist and know exactly when they win.

**Hash index** — hashes the key into a bucket. **O(1)** equality lookup, genuinely faster than a B-tree for `=`. And **completely useless** for anything else: no ranges (`>`, `<`, `BETWEEN`), no `ORDER BY`, no `LIKE 'ab%'`, no leftmost-prefix. Hashing destroys ordering — that's the whole point of a hash. In PostgreSQL: `CREATE INDEX ... USING hash (col)`. In InnoDB you can't create one manually; it builds an **adaptive hash index** in memory automatically for hot keys. Use a hash index when you have a huge key (a long URL, a token) that you only ever look up by exact match. Otherwise the B-tree is more versatile and nearly as fast.

**Bitmap index** — one bitmap per distinct value: for `status`, a bit-vector saying which rows are `'paid'`, another for `'shipped'`. Brilliant for **low-cardinality** columns in **analytics** workloads, because you can `AND`/`OR` bitmaps together at CPU-word speed to combine many predicates. Terrible for OLTP: a single-row update has to rewrite big chunks of bitmap and takes heavy locks. Native in Oracle and in column stores (ClickHouse, Redshift). PostgreSQL has no bitmap *index*, but it builds bitmaps *on the fly* — that's what `Bitmap Heap Scan` in a plan means (see §7).

**Inverted index (full-text)** — for `WHERE body LIKE '%database%'` a B-tree is worthless (leading wildcard = no prefix = no seek). An inverted index maps **each word → list of documents containing it**, which is the exact opposite direction of a normal index. PostgreSQL: `GIN`. This is the foundation of Elasticsearch and Lucene. Covered properly in [79 — Full-Text Search & Inverted Indexes](./79-search-systems.md).

**GiST / geospatial** — "find all restaurants within 2km of me" can't be answered by a 1-D sorted tree, because 2-D proximity has no total ordering. GiST is a generalized tree that indexes *bounding boxes*; R-trees and PostGIS build on it. Also used for range types and nearest-neighbour search.

| Type | Best for | Fails at |
|---|---|---|
| **B+ tree** | Everything: `=`, ranges, `ORDER BY`, prefixes | Leading-wildcard `LIKE`, 2-D geo |
| **Hash** | Pure equality on a big key | Ranges, sorting, prefixes |
| **Bitmap** | Low-cardinality analytics, combining predicates | Any write-heavy OLTP table |
| **Inverted (GIN)** | Word/token search inside text, JSONB keys | Range queries on numbers |
| **GiST / R-tree** | Geospatial, multidimensional, nearest-neighbour | Nothing else, really |

### 6. When indexes HURT — the section everyone forgets

Indexes are a **read optimization paid for with writes, storage, and memory.** Every doc gushes about the read win. Here's the bill.

**a) Every write must update every index.**

An `INSERT` into a table with 6 indexes is **1 row write + 6 index writes** — 6 more B+ trees to descend, 6 more pages to dirty, possibly 6 page splits, 6 more WAL records. Real, measurable effect:

| Indexes on table | Rough INSERT throughput |
|---|---|
| 1 (just the PK) | ~50,000/sec |
| 5 | ~15,000/sec |
| 12 | ~5,000/sec |

(Illustrative shape, not a benchmark — but the *shape* is what happens, and it's roughly linear in index count.) An `UPDATE` is worse: if you change an indexed column, the DB must **delete the old index entry and insert a new one** in a different part of the tree. Updating a non-indexed column may cost nothing extra (PostgreSQL's HOT updates) — updating an indexed one costs a full index churn.

**Corollary:** on a high-write table (an events/audit/telemetry table doing 20k inserts/sec), **every index is a tax on your hottest path.** Justify each one.

**b) Indexes consume disk and RAM.**

An index on a `BIGINT` column of a 100M-row table is roughly `100M × ~20 bytes ≈ 2 GB`. Ten indexes and your indexes are larger than your table. Worse: they compete with the table for **buffer cache**. A rarely-used index that occasionally gets read evicts hot table pages from RAM, and your cache hit ratio quietly degrades everywhere.

**c) Low-cardinality columns are nearly useless.**

**Cardinality** = number of distinct values. An index on `is_active BOOLEAN` (2 values) or `gender` (a handful) on a 10M-row table: if 50% of rows match, using the index means 5M index entries → 5M random bookmark lookups into the heap. **Random I/O is ~10–100x slower per row than sequential I/O.** So reading 50% of the table via random access is *slower than just reading 100% of it sequentially*, and the query planner knows this — it will look at your index, do the arithmetic, and **ignore it**. You get all the write cost and none of the read benefit.

The rough rule: an index earns its keep when the predicate is **selective** — it filters down to roughly **<5–10%** of rows. `email` (unique) is perfectly selective. `status` where 97% of rows are `'completed'` is useless *for finding completed orders* — but a **partial index** makes it excellent for finding the other 3%:

```sql
-- Tiny index. Only contains the ~3% of rows you actually query for.
CREATE INDEX idx_orders_pending ON orders (created_at)
  WHERE status = 'pending';
```

**d) Unused indexes are pure cost.** They slow every write, eat disk, eat cache, and return nothing. Find and kill them:

```sql
-- PostgreSQL: indexes that have never been scanned
SELECT relname AS table, indexrelname AS index,
       idx_scan AS times_used, pg_size_pretty(pg_relation_size(indexrelid)) AS size
FROM pg_stat_user_indexes
WHERE idx_scan = 0 AND indexrelid NOT IN (SELECT conindid FROM pg_constraint)
ORDER BY pg_relation_size(indexrelid) DESC;
```

**e) Redundant indexes.** If you have `(a, b, c)`, then a separate index on `(a)` and on `(a, b)` are both **redundant** (leftmost prefixes are already served). Drop them. This is astonishingly common in codebases where three engineers each added "the index they needed."

### 7. Reading a query plan — `EXPLAIN ANALYZE`

**Never add an index on a hunch. `EXPLAIN` before, `EXPLAIN` after.** `EXPLAIN` shows the plan; `EXPLAIN ANALYZE` actually *runs* the query and shows real timings and real row counts.

**Before the index:**

```sql
EXPLAIN ANALYZE
SELECT id, total_cents FROM orders
WHERE customer_id = 42 AND status = 'pending';
```
```
Seq Scan on orders  (cost=0.00..245831.00 rows=118 width=12)
                    (actual time=0.412..3871.905 rows=112 loops=1)
  Filter: ((customer_id = 42) AND (status = 'pending'::text))
  Rows Removed by Filter: 9999888
Planning Time: 0.121 ms
Execution Time: 3872.410 ms
```

Read it like this:
- **`Seq Scan`** — the alarm bell. It read the whole table.
- **`cost=0.00..245831.00`** — the planner's *estimate* in arbitrary units. Startup cost .. total cost. Only useful for comparing plans, not as a time.
- **`rows=118`** — the planner *estimated* 118 rows. **`actual ... rows=112`** — it really got 112. Estimate ≈ actual, so **your statistics are healthy**. If estimate said 118 and actual was 900,000, your stats are stale → `ANALYZE orders;` — a bad estimate is the #1 cause of a *good* index being ignored.
- **`Rows Removed by Filter: 9999888`** — **this is the smoking gun.** It touched 10 million rows to keep 112.
- **`Execution Time: 3872ms`**.

**Add the index:**

```sql
CREATE INDEX CONCURRENTLY idx_orders_cust_status
  ON orders (customer_id, status) INCLUDE (total_cents);
-- CONCURRENTLY: builds without taking a write lock. ALWAYS use this in production.
```

**After:**

```
Index Only Scan using idx_orders_cust_status on orders
     (cost=0.56..12.94 rows=112 width=12)
     (actual time=0.038..0.194 rows=112 loops=1)
  Index Cond: ((customer_id = 42) AND (status = 'pending'::text))
  Heap Fetches: 0
Planning Time: 0.208 ms
Execution Time: 0.241 ms
```

**3872ms → 0.24ms. ~16,000x.** `Heap Fetches: 0` proves the covering index worked — the table was never touched.

**The scan-node vocabulary you must know:**

| Plan node | What it means | Verdict |
|---|---|---|
| **`Seq Scan`** | Read every row in the table | Bad on a big table with a selective filter. **Fine** on a small table (<~1000 rows — an index seek isn't worth it) or when you genuinely need most rows. |
| **`Index Scan`** | Walk the B+ tree, then **fetch each matching row from the heap** | Good. But every matching row = one random read. |
| **`Index Only Scan`** | Answered **entirely from the index**, table never read | **Best.** Check `Heap Fetches: 0`. |
| **`Bitmap Index Scan` + `Bitmap Heap Scan`** | Collect all matching row locations into an in-memory bitmap, **sort them by physical page**, then read the heap **in page order** | Clever middle ground: too many rows for pure random I/O, too few for a full scan. Turns random reads into semi-sequential ones. Often what you get with a medium-selectivity predicate, or when combining two indexes. |
| **`Nested Loop` / `Hash Join` / `Merge Join`** | Join strategies. A `Nested Loop` over an unindexed FK is a classic disaster: it re-scans the inner table once per outer row. | **Index your foreign keys.** |
| **`Sort`** | The DB is sorting in memory (or spilling to disk) | An index in the right order can delete this node entirely. |

**Reading tip:** plans are trees, read them **inside-out / bottom-up**. The most indented node runs first. And `loops=N` means the node ran N times — **the `actual time` shown is per loop**, so multiply by `loops` to get the real total. That's how a "0.03ms" node turns out to be the 9-second bottleneck.

---

## Visual / Diagram description

### Diagram 1 — a B+ tree, drawn properly

An index on `user_id`. Fanout is tiny here (3) so it fits on a page; in reality it's ~500.

```
                           ROOT (1 disk page)
                        ┌──────────────────┐
                        │   [ 30 | 70 ]    │   internal node: keys are SIGNPOSTS only
                        └───┬─────┬─────┬──┘   no data lives here
              <30 ─────────┘      │      └───────── >=70
                                >=30, <70
        ┌───────────────┐  ┌───────────────┐  ┌───────────────┐
        │  [ 10 | 20 ]  │  │  [ 40 | 55 ]  │  │  [ 80 | 92 ]  │  internal nodes
        └──┬────┬────┬──┘  └──┬────┬────┬──┘  └──┬────┬────┬──┘
           │    │    │        │    │    │        │    │    │
           ▼    ▼    ▼        ▼    ▼    ▼        ▼    ▼    ▼
  ┌────────────┐ ┌────────────┐ ┌────────────┐        ┌────────────┐
  │ 5,7 → row  │◄►│10,14 → row│◄►│20,25 → row│◄► ... ◄►│92,97 → row │   LEAF NODES
  │  pointers  │ │  pointers  │ │  pointers  │        │  pointers  │
  └────────────┘ └────────────┘ └────────────┘        └────────────┘
        ▲              ▲                                     ▲
        └──────────────┴──── doubly-linked list ─────────────┘
             ALL data lives here, in sorted order
```

What this shows:
- **Internal nodes hold no data** — only routing keys. That's the "+" in B+ tree, and it's why they fit more keys per page.
- **Every leaf is at the same depth.** The tree is *perfectly balanced*, always — B+ trees rebalance on insert via **page splits** (a full page splits in two and pushes a key up to the parent) and on delete via merges. So lookup cost is *identical* for every key. No worst case.
- **The linked leaves.** `WHERE user_id BETWEEN 20 AND 92`: descend to leaf `[20,25]` **once**, then just walk right along the arrows. This is why B+ trees crush range scans and `ORDER BY`.
- **Depth = disk reads.** Root, internal, leaf = 3 reads. Add one more level and you're addressing billions more rows for the price of *one* extra read.

### Diagram 2 — clustered vs secondary in InnoDB

```
  SECONDARY INDEX on (email)              CLUSTERED INDEX = THE TABLE (on PK id)
  ┌──────────────────────────┐            ┌───────────────────────────────────┐
  │  B+ tree keyed by email  │            │       B+ tree keyed by id         │
  └──────────────────────────┘            └───────────────────────────────────┘
             │                                            │
             ▼  leaves                                    ▼  leaves
  ┌────────────────────────────┐          ┌────────────────────────────────────┐
  │ ada@lovelace.dev  →  PK 42 │          │ 42 │ Ada │ ada@lovelace.dev │ 1815 │
  │ bob@example.com   →  PK 17 │          │ 43 │ Bob │ bob@example.com  │ 1990 │
  │ cleo@example.com  →  PK 88 │          │ 44 │ Cleo│ cleo@example.com │ 1988 │
  └────────────────────────────┘          └────────────────────────────────────┘
             │                                            ▲
             │        BOOKMARK LOOKUP                     │
             └────────── "now go find PK 42" ─────────────┘
                     (a SECOND full tree traversal)

  SELECT * FROM users WHERE email='ada@lovelace.dev';   → 2 traversals (~6 pages)
  SELECT id  FROM users WHERE email='ada@lovelace.dev'; → 1 traversal  (~3 pages)
                                                          ↑ covering! id is already
                                                            in the secondary leaf.
```

Notice the free lunch at the bottom: because InnoDB stores the PK in every secondary leaf, **any query selecting only the indexed column(s) + the PK is automatically covering.** That's why `SELECT id FROM users WHERE email = ?` is meaningfully cheaper than `SELECT * ...`, and one concrete reason to stop writing `SELECT *`.

### Diagram 3 — the decision you're actually making

```
   Query arrives ──► Planner asks: "how many rows will this predicate return?"
                                        │
                    ┌───────────────────┴────────────────────┐
                    ▼                                        ▼
          FEW rows (<~5%)                          MANY rows (>~20%)
                    │                                        │
                    ▼                                        ▼
          ┌──────────────────┐                     ┌──────────────────┐
          │  Index Scan      │                     │    Seq Scan      │
          │  random reads,   │                     │  sequential I/O, │
          │  but very few    │                     │  no index churn  │
          └──────────────────┘                     └──────────────────┘
                    │                                        │
                    └──────────────► middle ◄────────────────┘
                                        │
                              ┌──────────────────┐
                              │ Bitmap Heap Scan │
                              │ random reads made│
                              │ sequential-ish   │
                              └──────────────────┘
```

The planner is doing a **cost model**, not obeying you. When it "ignores your index," it is usually *right* — and when it's wrong, it's almost always because its row estimate is wrong (stale statistics). Fix the statistics, don't fight the planner.

---

## Real world examples

### GitHub — the `CONCURRENTLY` lesson at scale

GitHub runs a very large MySQL fleet, and their public engineering writing (and the tooling they open-sourced, `gh-ost`) exists precisely because **schema changes, including adding an index, lock tables and take down writes** on a busy table. Adding an index to a table with hundreds of millions of rows naively means the table is unavailable for writes for the duration of the build. Their answer is online schema migration: build a shadow copy with the new index, replay the binlog of changes onto it, then atomically swap. The lesson for you: `CREATE INDEX CONCURRENTLY` (PostgreSQL) or `gh-ost`/`pt-online-schema-change` (MySQL) is not a nicety — on a production table, a blocking `CREATE INDEX` **is an outage**.

### Discord — indexing shaped by the access pattern

Discord's public writing on their messages database describes storing messages partitioned/clustered by `(channel_id, message_id)` — because the *only* query that matters at volume is "give me the most recent N messages in this channel, and let me page backwards." Their storage is organized so that this exact access pattern is a **sequential read of physically adjacent data**, not a scatter of random lookups. That is clustered-index thinking taken to its conclusion: **don't ask "what indexes should I add to my table," ask "what is my dominant query, and can I make the data physically laid out in the order that query wants to read it?"** Snowflake-style IDs help here too — because they're time-ordered, sorting by ID *is* sorting by time, for free.

### Any e-commerce order table (representative)

The canonical composite-index story. `orders(id, customer_id, status, created_at, total_cents, ...)`, 500M rows. The two dominant queries are "my orders, newest first" and "the ops dashboard: all pending orders". The naive answer is four single-column indexes. The right answer is two purpose-built ones:

```sql
-- Q1: "my orders, newest first" — equality on customer, sort on date.
--     Serves the WHERE *and* the ORDER BY: zero sorting work, and it's covering.
CREATE INDEX idx_orders_cust_date ON orders (customer_id, created_at DESC)
  INCLUDE (status, total_cents);

-- Q2: ops dashboard. 'pending' is ~2% of rows, so a partial index is ~50x smaller
--     than a full index on status — and it fits entirely in RAM.
CREATE INDEX idx_orders_pending ON orders (created_at) WHERE status = 'pending';
```

---

## Trade-offs

**Adding an index**

| Pros | Cons |
|---|---|
| Reads go from O(n) → O(log n); 100x–10,000x on big tables | Every `INSERT`/`UPDATE`/`DELETE` must also update the index |
| Can eliminate `Sort` nodes entirely (`ORDER BY` for free) | Consumes disk — often 10–30% of table size *each* |
| Enforces `UNIQUE` constraints | Competes with the table for the buffer cache / RAM |
| Speeds up joins on foreign keys enormously | Building it on a live table can lock writes (use `CONCURRENTLY`) |
| Covering indexes remove heap access entirely | More indexes = more for the planner to consider = slower planning |

**Index type choices**

| Decision | You gain | You give up |
|---|---|---|
| B+ tree over hash | Ranges, `ORDER BY`, prefixes, composites | A little raw equality speed |
| Composite `(a,b)` over two separate indexes | One tree serves the combined query; less write cost | Only works if the query has `a`; wrong order = useless |
| Covering (`INCLUDE`) | No heap fetch — often 10x | Bigger index; more write amplification when included cols change |
| Partial index (`WHERE ...`) | Tiny index, fits in RAM, cheap to maintain | Only usable when the query's predicate matches the index's |
| Clustered on the PK | PK lookups are one traversal | Only one per table; random PKs cause page splits |
| Fewer indexes | Fast writes, small footprint | Some queries stay slow |

**Rule of thumb:** Index the columns you **filter on, join on, and sort by** — and nothing else. Start with your foreign keys (an unindexed FK makes joins and cascading deletes catastrophic). Prefer **one well-ordered composite index** over three single-column ones. Then measure: any index with `idx_scan = 0` after a month is a pure tax — drop it. And on a table doing >10k writes/sec, treat **every** index as something you must justify out loud.

**The sweet spot:** most OLTP tables land at **3–6 indexes**. If you're above ~8, you almost certainly have redundancy or dead weight; go look.

---

## Common interview questions on this topic

### Q1: "Why do databases use B+ trees instead of binary search trees or hash tables?"

**Hint:** Because the bottleneck is **disk I/O, not CPU comparisons**. A disk hands you a whole ~8 KB page whether you want 1 byte or 8,000, so you make the node *be* a page and cram ~500 keys into it. Fanout 500 → a 4-level tree addresses 62 *billion* rows in 4 page reads (and the top 2 levels are cached, so really 1–2 disk reads). A BST on the same data is ~23–33 levels = 23–33 random disk seeks. Not hash tables, because hashing destroys ordering — no ranges, no `ORDER BY`, no prefix matching. B+ trees specifically (data only in leaves, leaves linked) so range scans walk sideways instead of re-descending from the root.

### Q2: "You have an index on `(country, city, age)`. Which of these use it: `WHERE city='Paris'`? `WHERE country='FR' AND age > 30`? `WHERE country='FR' ORDER BY city`?"

**Hint:** (1) **No** — leftmost-prefix violated, `country` is missing; the Parises are scattered across every leaf, like looking for "Ada" in a phone book. (2) **Partially** — it seeks on `country='FR'`, then filters `age` within that range, but can't *seek* on `age` because `city` was skipped. (3) **Yes, beautifully** — it seeks on `country` and the results come out already sorted by `city`, so the planner deletes the `Sort` node entirely.

### Q3: "You added the perfect index and the planner still does a Seq Scan. Why?"

**Hint:** Three usual suspects. (a) **Low selectivity** — the predicate matches 40% of rows, and 40% of the table via random heap lookups is genuinely slower than reading 100% sequentially. The planner is *right*. (b) **Stale statistics** — it *estimates* 2 million rows when the truth is 50; run `ANALYZE`. Check the plan's `rows=` estimate against `actual rows=`; a big gap is the tell. (c) **The predicate isn't sargable** — you wrote `WHERE YEAR(created_at) = 2026` or `WHERE lower(email) = ?`, and wrapping the column in a function makes the index unusable. Rewrite as a range (`created_at >= '2026-01-01' AND created_at < '2027-01-01'`) or build a **functional index** (`CREATE INDEX ON users (lower(email))`). Also: on a tiny table, a Seq Scan is simply cheaper. That's fine.

### Q4: "Would you index a `gender` or `is_deleted` column?"

**Hint:** As a plain index — **no**. Cardinality is ~2, so any value matches a huge fraction of rows, and the planner will scan anyway. You'd pay the full write cost for zero read benefit. But two nuances that get you the offer: (1) it's a great **partial index** if the distribution is skewed — `CREATE INDEX ... WHERE is_deleted = false` gives you a small, hot index over exactly the rows you actually query. (2) It can be a fine *trailing* column in a **composite** index — `(customer_id, is_deleted)` is useful even though `(is_deleted)` alone is not.

### Q5: "What's the cost of a secondary-index lookup in InnoDB versus a primary-key lookup?"

**Hint:** PK lookup = **one** B+ tree traversal, and the full row is right there in the leaf (the clustered index *is* the table). A secondary-index lookup = **two** traversals: walk the secondary tree to get the **primary key**, then walk the clustered tree with that PK to get the row — the "bookmark lookup". InnoDB stores the PK rather than a physical address so that rows can move (page splits) without invalidating every secondary index. Two consequences: **keep the PK narrow** (it's copied into every secondary index — a UUID string PK bloats all of them and randomizes insert locality), and **a covering secondary index skips the second traversal entirely.**

---

## Practice exercise

### "Make the slow query fast — and prove it"

Set up a real table, break it, fix it, and *measure*. Roughly 30–40 minutes. Use PostgreSQL locally (Docker: `docker run -e POSTGRES_PASSWORD=pw -p 5432:5432 postgres`).

**Step 1 — Build a 2,000,000-row table.**

```sql
CREATE TABLE orders (
  id           BIGSERIAL PRIMARY KEY,
  customer_id  BIGINT      NOT NULL,
  status       TEXT        NOT NULL,
  total_cents  INT         NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL
);

INSERT INTO orders (customer_id, status, total_cents, created_at)
SELECT (random() * 50000)::bigint,
       (ARRAY['pending','paid','shipped','refunded'])[1 + (random()*3)::int],
       (random() * 20000)::int,
       now() - (random() * 365 || ' days')::interval
FROM generate_series(1, 2000000);

ANALYZE orders;
```

**Step 2 — Measure the pain.** Run `EXPLAIN ANALYZE` on each and **write down the plan node, the estimated rows, the actual rows, and the execution time**:

```sql
-- A
SELECT id, total_cents FROM orders WHERE customer_id = 12345;
-- B
SELECT * FROM orders WHERE customer_id = 12345 ORDER BY created_at DESC LIMIT 20;
-- C
SELECT count(*) FROM orders WHERE status = 'pending';
-- D
SELECT * FROM orders WHERE status = 'pending' AND created_at > now() - interval '7 days';
```

**Step 3 — Fix them, one index at a time**, re-running `EXPLAIN ANALYZE` after each. Aim for:
- **A** → an `Index Only Scan` with `Heap Fetches: 0` (hint: `INCLUDE`).
- **B** → an index that serves *both* the `WHERE` and the `ORDER BY`, so **no `Sort` node appears at all** (hint: `(customer_id, created_at DESC)`).
- **C** → try `CREATE INDEX ON orders (status)` and observe that the planner **still won't use it** (or only via a Bitmap scan). Understand why — 25% of rows are `pending`. This is the low-cardinality lesson, and you should *feel* it, not just read it.
- **D** → make it fly with a **partial index**. Compare `pg_relation_size` of the partial index vs. the full one on `status`.

**Step 4 — Feel the write cost.** Time `INSERT INTO orders SELECT ... FROM generate_series(1, 200000)` with **zero** indexes vs. with **all four** of yours. Write down both numbers.

**Deliverable:** a small table — query | plan before | ms before | index added | plan after | ms after — plus the two insert timings from step 4, and one sentence for query C explaining *why* the planner refused your index.

---

## Quick reference cheat sheet

- **An index is a sorted side-structure**, not magic. It trades write speed and disk for read speed.
- **B+ tree = the default index everywhere.** Node = one disk page (8–16 KB) holding ~hundreds of keys.
- **Fanout ~500 → depth 4 covers ~62 billion rows.** A lookup is 3–4 page reads. A BST would be ~23–33 seeks. **Disk I/O count, not CPU comparisons, is what indexes optimize.**
- **All data lives in the leaves; leaves are linked.** That's why range scans and `ORDER BY` are cheap.
- **Clustered index = the table itself**, physically sorted by the key. One per table. In InnoDB it's the **primary key**.
- **Secondary index → bookmark lookup.** In InnoDB the leaf stores the **PK**, so it's *two* tree traversals. **Keep your PK small.**
- **Leftmost-prefix rule:** `(a,b,c)` serves `a`, `a+b`, `a+b+c` — **never `b` alone.** It's a phone book sorted by last name.
- **Column order is a design decision:** equality columns first, range/sort columns last.
- **Covering index = index-only scan.** All needed columns in the index → the table is never touched. Often 10x. Check `Heap Fetches: 0`.
- **Low cardinality = useless index.** If a predicate matches >~10–20% of rows, the planner will scan anyway — and it's right. Reach for a **partial index** instead.
- **Every write pays for every index.** 12 indexes ≈ 13 writes per insert. Over-indexing kills write throughput.
- **`Seq Scan` bad (usually) → `Index Scan` good → `Index Only Scan` best.** `Bitmap Heap Scan` = the sensible middle.
- **Estimate vs actual rows in `EXPLAIN ANALYZE` is your stats health check.** Big gap → run `ANALYZE`.
- **`EXPLAIN ANALYZE` before AND after. Always. `CREATE INDEX CONCURRENTLY` in production. Drop indexes with `idx_scan = 0`.**

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [61 — SQL vs NoSQL](./61-databases-sql-vs-nosql.md) — the index is the machinery that makes a relational query planner viable in the first place |
| **Next** | [63 — Database Sharding](./64-database-sharding.md) — when one machine's indexes can no longer keep the working set in RAM, you split the data across machines |
| **Related** | [64 — Caching Strategies](./59-caching-in-depth.md) — a cache and an index solve the same problem (avoid expensive reads) at different layers; index first, cache second |
| **Related** | [79 — Full-Text Search & Inverted Indexes](./79-search-systems.md) — the index type that B+ trees cannot do: word → documents, the basis of Elasticsearch |
