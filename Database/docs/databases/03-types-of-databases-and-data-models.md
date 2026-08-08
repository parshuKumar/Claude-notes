# 03 — Types of Databases and Data Models
## Phase: Storage Internals

---

## ELI5 — The Simple Analogy

Think about how you store different things in your house.

**Spices** go in a rack with labelled jars, in rows and columns — you always want "the turmeric" and you want it in one reach. That's a **relational** table.
**A recipe** goes in a folder as one whole sheet — ingredients, steps, photo, all together, because you never want just the step-4 of a recipe without the rest. That's a **document**.
**Your keys** go on one specific hook by the door. One hook, one thing, grab and go, no searching. That's **key-value**.
**Your electricity meter readings** go in a notebook, one line per day, forever appended, never edited. That's **time-series**.
**Your family tree** on the wall is about the *lines between people*, not the people. That's a **graph**.
**A warehouse of 40,000 boxes** where you record, for each box, only the columns that box actually has — box 12 has 3 fields, box 900 has 200 — that's **wide-column**.
**The index at the back of a book** — "the word 'monsoon' appears on pages 4, 88, 210" — that's **search**.

Nobody argues that a spice rack is better than a key hook. They are shaped for different retrieval patterns. Database models are exactly the same, and the question is never "which is best" — it is **"what is the shape of my access pattern?"**

---

## Where this fits in the big picture

```
   01 What is a DB          02 Engine architecture
   (four problems)          (the components)
              \                  /
               \                /
                ▼              ▼
        03 DATA MODELS  ← YOU ARE HERE
        (the same components, arranged for different access patterns)
                │
                ├──→ 04–08  how the relational model lands on disk
                ├──→ 20–28  designing IN the relational model
                └──→ 70–76  designing in the OTHER models (Phase 8)
                              ↑
                     you will come back here after Phase 7,
                     armed enough to judge the trade-offs properly
```

This topic gives you the **vocabulary and the decision axes**. Phase 8 gives you the modelling skill inside each one.

---

## What is this?

A **data model** is the shape the database forces your data into, and therefore the set of questions it can answer cheaply. The model determines the physical layout; the physical layout determines which access patterns are one disk seek and which are a full scan.

There are six models that matter, plus search engines (which are not really databases). Every product is one of these with marketing on top.

---

## Why does it matter for a backend developer?

Because choosing the wrong model is the single most expensive mistake in a system's life. Every other mistake in this curriculum — a bad index, a missing constraint, an unnormalised table — is a day or a week to fix. A wrong data model is a rewrite.

Concretely:

- Put a **social graph** in a relational database and "friends of friends of friends" becomes a 3-way self-join over 200M rows that times out. Neo4j does it in 12ms because adjacency is stored as pointers.
- Put **IoT metrics** in a normalised OLTP table and at 50,000 inserts/sec your B-tree index spends all its time splitting pages, and `AVG(value) OVER last 7 days` reads 30GB.
- Put **financial transactions** in an eventually-consistent key-value store and you will pay someone twice and have no transaction to roll back.
- Put your **whole product catalogue** in Redis and the first time someone asks "which products have no image and were updated last week" you have to `SCAN` the entire keyspace.

The right question is never "SQL or NoSQL." It is: *what does one read look like, what does one write look like, and how many of each per second?*

---

## The physical reality

The models differ because the **bytes are laid out differently on disk**. Here is the same data — a user with two orders — in five models.

**RELATIONAL** — fixed-width rows in 8KB pages, one file per table, joined at read time:
```
users heap (16385), page 0                orders heap (16391), page 0
┌──────────────────────────────┐          ┌────────────────────────────────────┐
│ hdr │ ptr1 ptr2               │          │ hdr │ ptr1 ptr2                    │
│                              │          │                                    │
│ [7 | arjun@shop.in | Arjun]  │◀──┐      │ [91 | 7 | 249900 | 2026-08-01]    │
│ [8 | meera@shop.in | Meera]  │   └──────│ [92 | 7 | 129900 | 2026-08-03]    │
└──────────────────────────────┘  FK      └────────────────────────────────────┘
   read = 2 files + an index lookup, joined by the executor
```

**DOCUMENT** — one self-contained BSON blob, everything for one entity in one place:
```
collection users, extent 0
┌───────────────────────────────────────────────────────────────┐
│ { _id: ObjectId("64f.."),                                     │
│   email: "arjun@shop.in", name: "Arjun",                      │
│   orders: [ { id: 91, total: 249900, at: ISODate("...") },    │
│             { id: 92, total: 129900, at: ISODate("...") } ] } │
└───────────────────────────────────────────────────────────────┘
   read = ONE seek. No join. But updating order 91 rewrites the whole doc.
```

**KEY-VALUE** — a hash table on disk; the key is the only access path:
```
  hash("user:7")        → offset 0x4A88 → {"email":"arjun@shop.in","name":"Arjun"}
  hash("user:7:orders") → offset 0x5C10 → [91,92]
  hash("order:91")      → offset 0x6100 → {"total":249900}
   read by key = O(1). "all orders over ₹2000" = impossible without scanning.
```

**WIDE-COLUMN** — data physically sorted and clustered by partition key, then clustering key:
```
SSTable on disk, partition key = user_id, clustering key = created_at DESC
┌────────────────────────────────────────────────────────────────────┐
│ partition 7 │ 2026-08-03 → {order:92,total:129900}                 │
│             │ 2026-08-01 → {order:91,total:249900}                 │
├────────────────────────────────────────────────────────────────────┤
│ partition 8 │ 2026-07-29 → {order:88,total: 45000}                 │
└────────────────────────────────────────────────────────────────────┘
   "user 7's last 10 orders" = ONE contiguous disk read, already sorted.
   "all orders over ₹2000" = full cluster scan. The schema IS the query.
```

**COLUMNAR** (analytics) — one file per column, so a scan of one column touches nothing else:
```
orders/user_id.parquet   [7,7,8,8,9,9,9,...]           ← compresses to almost nothing
orders/total.parquet     [249900,129900,45000,...]
orders/created_at.parquet[2026-08-01,2026-08-03,...]
   SUM(total) reads ONE file. Fetching one whole row reads N files. (Topic 06)
```

**GRAPH** — adjacency stored as physical pointers, not as a joinable key:
```
node 7 (User "Arjun")
  ├─ rel[PLACED]  ──▶ node 91 (Order)     ← a POINTER, ~15 bytes, O(1) to follow
  ├─ rel[PLACED]  ──▶ node 92 (Order)
  └─ rel[FRIENDS] ──▶ node 8  (User "Meera")
   3-hop traversal = 3 pointer dereferences, independent of total graph size.
   In SQL the same query is 3 joins whose cost grows with table size.
```

**TIME-SERIES** — append-only, chunked by time, column-compressed within a chunk:
```
metrics hypertable
  chunk_2026_08_01 ──┐
  chunk_2026_08_02   ├── each chunk is its own table + index
  chunk_2026_08_03 ──┘   old chunks compressed 10–20×, or dropped wholesale
   "last 24h" touches ONE chunk. Retention = DROP TABLE, not DELETE. (Topic 73)
```

---

## How it works — step by step

The same business question, traced through three models. Question: **"Show me user 7's 10 most recent orders with product names."**

```
RELATIONAL (PostgreSQL)
 1. Planner picks idx_orders_user_id_created_at
 2. Index scan: descend b-tree to (7, max_date), walk backwards 10 entries   [~4 page reads]
 3. Heap fetch: 10 random reads into the orders heap                          [~10 page reads]
 4. Nested loop into order_items via idx_order_items_order_id                 [~30 page reads]
 5. Hash join to products (products likely fully cached)                      [~5 page reads]
 6. Sort/limit
 TOTAL: ~49 page reads, 4 relations. Correct even if a product was renamed
        yesterday, because the name is stored in ONE place.

DOCUMENT (MongoDB, embedded design)
 1. Index lookup on {_id: 7}                                                  [~3 page reads]
 2. Fetch the one document — orders array is inside it                        [~2 page reads]
 3. Slice last 10 in memory
 TOTAL: ~5 page reads, 1 collection. BUT: the product name was copied into
        each order line at write time. Rename a product and 400,000 documents
        now hold a stale name. (Topic 70 — this is the whole embed/reference trade.)

WIDE-COLUMN (Cassandra, query-first design)
 Table designed as: PRIMARY KEY ((user_id), created_at DESC)
 1. Hash user_id 7 → node(s) owning that partition
 2. Read the partition's first 10 rows — already sorted on disk, contiguous   [~2 page reads]
 TOTAL: ~2 page reads, 1 table. Fastest of the three.
        BUT you cannot ask ANY other question of this table. "Orders by
        product" requires a SECOND table holding the same data, written to
        by your application, kept in sync by your application.
```

**Notice what actually varies:** not "speed" in the abstract, but *how many places one fact is stored*, and therefore *how many questions are cheap* and *what happens when a fact changes*.

---

## Concept breakdown

```
THE FOUR AXES THAT DECIDE EVERYTHING
│
├── 1. ACCESS PATTERN SHAPE
│      point lookup by key       → key-value, document
│      range scan on a sort key  → wide-column, time-series, relational
│      arbitrary ad-hoc filters  → relational (only model that's good at this)
│      traversal of relationships→ graph
│      aggregate over one column → columnar
│      "find text like this"     → search
│
├── 2. WHERE THE JOIN HAPPENS
│      at read time, by the engine     → relational  (flexible, costs CPU per read)
│      at write time, by you           → document, wide-column (fast reads, you own consistency)
│      never (pointers, precomputed)   → graph
│
├── 3. CONSISTENCY MODEL
│      strong / transactional          → relational, and modern document DBs
│      tunable per query               → wide-column
│      eventual                        → most distributed KV, search indexes
│
└── 4. SCHEMA ENFORCEMENT
       enforced by the engine          → relational (bad data is REJECTED)
       enforced optionally             → document with validators
       enforced by nobody              → key-value ("schema-on-read" = "schema-in-every-
                                          consumer, and one of them is wrong")


"NoSQL" — the word means nothing
│
└── It is a 2009 marketing term bundling four unrelated models that share only
    "not a 1990s RDBMS." Never make a design decision using this word. Say
    "document store" or "wide-column store" and the decision gets easier.


OLTP  vs  OLAP    (the other axis that matters)
│
├── OLTP  Online Transaction Processing
│         many small reads/writes, latency-critical, row-oriented, indexed
│         "give me order 91" · "insert this payment"
└── OLAP  Online Analytical Processing
          few huge scans, throughput-critical, column-oriented, compressed
          "revenue by category by month for 3 years"
   Trying to serve both from one table is the most common architecture mistake
   in mid-size companies. (Topics 06, 55, 58.)
```

---

## Diagrams

**Diagram 1 — the model map, by access pattern**

```
                        WHAT DOES ONE READ LOOK LIKE?
                                    │
        ┌───────────────────────────┼────────────────────────────┐
        │                           │                            │
   "I know the exact          "I have filters,        "I care about the
    identifier"                sorts, and joins"       connections"
        │                           │                            │
    ┌───┴────┐              ┌───────┴────────┐                   │
    │        │              │                │                   ▼
 one small  one whole   arbitrary       fixed, known          GRAPH
  value     entity      ad-hoc          in advance
    │        │              │                │
    ▼        ▼              ▼                ▼
 KEY-VALUE  DOCUMENT   RELATIONAL      WIDE-COLUMN
 (Redis,    (MongoDB)  (PostgreSQL)    (Cassandra)
  DynamoDB)

        SPECIAL SHAPES (time and text override everything above)
        ├── every row has a timestamp, you query ranges → TIME-SERIES
        ├── you aggregate one column over billions rows → COLUMNAR / OLAP
        └── you search unstructured text with ranking   → SEARCH ENGINE
```

**Diagram 2 — data flow: where the join work happens**

```
   RELATIONAL: pay at READ time                DOCUMENT: pay at WRITE time
   ────────────────────────────────            ──────────────────────────────
   write:  INSERT order        (cheap)         write: update product name
           INSERT order_items  (cheap)                 → rewrite 400,000 docs
                                                       → (expensive, and racy)
   read:   SELECT ... JOIN JOIN JOIN
           executor does the work                read:  findOne({_id})
           (CPU per request)                            one seek (very cheap)

   ⇒ Neither is "faster". You choose WHICH SIDE pays.
     Read-heavy + rarely-changing facts → pay at write.
     Write-heavy + frequently-changing facts → pay at read.
```

**Diagram 3 — before / after choosing wrong (a real trajectory)**

```
MONTH 1 — "MongoDB, we'll move fast"
   users { _id, name, address, orders:[ {items:[{sku,name,price}]} ] }
   ✓ product page: 1 query, 4ms
   ✓ no migrations

MONTH 14 — the questions changed
   "revenue per category last quarter"       → $unwind over 40M docs, 90s
   "which users bought BOTH X and Y"         → aggregation pipeline, 4 min
   "we renamed a product"                    → 2.1M documents to rewrite
   "refund this one line item"               → rewriting a 400KB doc for a 12-byte change
   user doc for a power seller               → 14 MB, approaching the 16 MB cap

   The data was fine. The MODEL could not express the questions the
   business started asking. Nothing about indexes or hardware fixes that.
```

---

## Example 1 — basic

The same entity, modelled three ways, so you can *see* the trade.

```sql
-- RELATIONAL: one fact in one place
CREATE TABLE products (id bigserial PRIMARY KEY, name text, price_paise bigint);
CREATE TABLE orders   (id bigserial PRIMARY KEY, user_id bigint, created_at timestamptz);
CREATE TABLE order_items (
  order_id bigint REFERENCES orders(id),
  product_id bigint REFERENCES products(id),
  qty int,
  PRIMARY KEY (order_id, product_id)
);
-- Rename a product: ONE row updated. Every order instantly shows the new name.
UPDATE products SET name = 'Basmati Rice 5kg' WHERE id = 88;
```

```js
// DOCUMENT: the order is self-contained
db.orders.insertOne({
  _id: 91, userId: 7, createdAt: new Date(),
  items: [ { productId: 88, name: "Basmati Rice", price: 54900, qty: 2 } ]
});
// Rename a product: you must decide.
//  (a) rewrite every order       → 2M writes, and now it says the wrong
//                                   historical price on an invoice
//  (b) leave it                  → orders show the OLD name (often CORRECT for
//                                   invoices! this is deliberate denormalisation)
```

```
# KEY-VALUE: only the exact key is retrievable
SET order:91 '{"userId":7,"total":109800}'
GET order:91                 # O(1), microseconds
# "all orders for user 7"? You must have maintained the index yourself:
LPUSH user:7:orders 91
LRANGE user:7:orders 0 9     # you are the index maintainer now
```

The lesson: **the document model isn't wrong to copy the name — for an invoice line, the historical name and price are the correct thing to store.** The mistake is copying a fact that is supposed to change everywhere at once. That distinction is the whole of Topic 70.

---

## Example 2 — production scenario

**The system.** Your e-commerce backend, 8M users, 40M orders/year, Node.js + PostgreSQL. Four new requirements land in one quarter:

| Requirement | Access pattern | Right model | Why |
|---|---|---|---|
| Product search: "red cotton kurta under ₹2000" with typo tolerance and relevance ranking | full-text, ranked, faceted | **Search engine** (OpenSearch) | Relational `ILIKE '%kurta%'` cannot use a b-tree, cannot rank, cannot handle "kurtaa". An inverted index does all three. |
| Session store: 200k concurrent, 30-min TTL, read on every request | point lookup by key, expiring | **Key-value** (Redis) | 200k rows churning every 30 min in PostgreSQL = enormous MVCC dead-tuple pressure (Topic 47). Redis expires them for free. |
| "Customers who bought this also bought" | 2-hop traversal over co-purchase edges | **Graph** — or a precomputed table | In SQL: a self-join of `order_items` to itself over 120M rows, per product. As a nightly-materialised table it's fine (Topic 55). As a live query it is not. |
| Real-time revenue dashboard: 3 years by category by day | scan + aggregate one column | **Columnar / OLAP** (ClickHouse, or PG + a rollup) | Row store reads all 14 columns to sum one. Column store reads one file, compressed 20×. |

**And critically — what stays in PostgreSQL:** `users`, `orders`, `order_items`, `payments`, `inventory`. Everything where **money must be correct** and **two facts must change atomically**. You never move those to a store without transactions to chase a benchmark.

**The architecture that results:**

```
                     ┌──────────────┐
   writes ──────────▶│  POSTGRESQL  │  ← THE SOURCE OF TRUTH. Always.
                     │ orders,      │
                     │ payments,    │
                     │ inventory    │
                     └──────┬───────┘
                            │ change data capture from the WAL (Topic 76)
          ┌─────────────────┼──────────────────┬───────────────────┐
          ▼                 ▼                  ▼                   ▼
   ┌────────────┐   ┌──────────────┐   ┌──────────────┐   ┌──────────────┐
   │ OpenSearch │   │  ClickHouse  │   │    Redis     │   │  rollup tbl  │
   │  (search)  │   │  (analytics) │   │(sessions,    │   │ (also-bought)│
   │            │   │              │   │ cache)       │   │              │
   └────────────┘   └──────────────┘   └──────────────┘   └──────────────┘
      derived           derived           ephemeral          derived
      rebuildable       rebuildable       losable            rebuildable
```

**The rule that makes this safe:** every non-PostgreSQL store here is **derived, rebuildable, or losable**. If OpenSearch is wiped, you reindex from PostgreSQL. If Redis dies, users log in again. Nothing in that row of boxes is the only copy of a fact. The moment one of them *is* the only copy, you have a distributed-transaction problem (Topics 51, 76).

---

## Common mistakes

**1. Choosing a model by team enthusiasm rather than access pattern.**
- *Symptom:* 14 months in, the business asks a normal question and it takes 4 minutes to answer.
- *Engine-level why:* the physical layout was optimised for one retrieval shape; other shapes require a full scan, and no index can rescue a layout that doesn't cluster the data you need together.
- *Diagnose:* list your top 10 queries by call count and by total time (`pg_stat_statements` / MongoDB profiler). If more than two of them require scanning the whole dataset, the model is wrong for the workload.
- *Fix:* add a derived store for the mismatched pattern; don't migrate the source of truth.

**2. "Schemaless" mistaken for "no schema."**
- *Symptom:* `user.email` is a string in 90% of documents, an array in 8%, missing in 2%, and your Node.js code crashes on `.toLowerCase()` in production only.
- *Engine-level why:* the schema still exists — it moved from one enforced definition to N unenforced assumptions spread across every consumer. There is no component checking them.
- *Diagnose:* `db.users.aggregate([{$group:{_id:{$type:"$email"}, n:{$sum:1}}}])`
- *Fix:* JSON Schema validators on the collection, or Zod/TypeBox at the write boundary — but at *one* boundary, not per-handler.

**3. Using a relational database as a queue.**
- *Symptom:* `SELECT ... WHERE status='pending' ORDER BY created_at LIMIT 1 FOR UPDATE` — 40 workers, massive lock contention, table bloats to 20GB from constant status updates.
- *Engine-level why:* every status update is an MVCC row rewrite (Topic 46) plus every index entry rewritten; and 40 workers contending on the same oldest row serialise completely.
- *Fix:* `FOR UPDATE SKIP LOCKED` makes it survivable at moderate scale; a real broker (SQS, Kafka, Redis Streams) is correct above that. Note this is a *legitimate* PG pattern up to a few thousand jobs/min — know where the line is.

**4. Using a document store to avoid writing migrations.**
- *Symptom:* 6 shapes of `address` in one collection because nobody backfilled; `if (u.address?.line1 ?? u.addr?.street ?? u.address_line)` in the code.
- *Engine-level why:* the migration didn't disappear — it became a permanent runtime branch executed on every request forever.
- *Fix:* version your documents (`schemaVersion: 3`), migrate lazily on read, and *actually* run the backfill.

**5. Putting money in an eventually-consistent store.**
- *Symptom:* a double-spend, a balance that goes negative, a refund that doesn't stick.
- *Engine-level why:* without serialisable isolation, read-modify-write on a balance from two nodes both read the old value. Last-write-wins silently discards one.
- *Fix:* money lives in a database with real transactions. Non-negotiable. Cache reads of it anywhere you like; write it in exactly one place.

---

## Hands-on proof

**PROVE IT #1 — the relational model can answer questions you didn't plan for.**
```sql
-- No new index, no schema change, a question nobody designed for:
SELECT p.name, count(*) AS times_bought
FROM order_items oi JOIN products p ON p.id = oi.product_id
JOIN orders o ON o.id = oi.order_id
WHERE o.created_at > now() - interval '7 days'
GROUP BY p.name ORDER BY times_bought DESC LIMIT 10;
```
It may be slow — but it is *possible*, and one index makes it fast. In Cassandra this query requires a table that does not exist, and creating it means backfilling all history.

**PROVE IT #2 — see why `ILIKE '%...%'` cannot use a b-tree.**
```sql
CREATE INDEX idx_products_name ON products(name);
EXPLAIN SELECT * FROM products WHERE name LIKE 'Basmati%';   -- uses the index
EXPLAIN SELECT * FROM products WHERE name LIKE '%Basmati%';  -- Seq Scan!
```
A b-tree sorts by prefix. A leading wildcard has no prefix to descend on. This is *the* reason search engines exist (Topic 74).

**PROVE IT #3 — measure the row-store penalty for an analytics query.**
```sql
EXPLAIN (ANALYZE, BUFFERS) SELECT sum(total_paise) FROM orders;
```
```
Aggregate  (actual time=812.4..812.4 rows=1)
  Buffers: shared hit=2100 read=10371          ← read ALL 12,471 pages
```
It read every byte of every row to sum one 8-byte column. A columnar store reads only that column's file. Topic 06.

**PROVE IT #4 — PostgreSQL can *be* a document store, with a caveat.**
```sql
CREATE TABLE events (id bigserial PRIMARY KEY, payload jsonb);
INSERT INTO events (payload) VALUES ('{"type":"checkout","cart":{"items":[1,2,3]}}');
CREATE INDEX idx_events_payload ON events USING GIN (payload);
SELECT * FROM events WHERE payload @> '{"type":"checkout"}';
```
JSONB + GIN gives you document storage *with* transactions and joins. For many teams this removes the reason to run MongoDB at all. The caveat: you lose per-field update efficiency — updating one key rewrites the whole JSONB value. Topics 16 and 25.

**PROVE IT #5 — key-value has exactly one access path.**
```bash
redis-cli SET user:7 '{"name":"Arjun"}'
redis-cli GET user:7          # instant
redis-cli KEYS 'user:*'       # works on 100 keys; NEVER run on production
                              # — O(N) over the whole keyspace, blocks the server
```

---

## The design decision framework

```
USE RELATIONAL WHEN:
  ✓ Multiple facts must change atomically (money, inventory, bookings)
  ✓ Your query patterns will change and you cannot enumerate them today
  ✓ Data has genuine relationships you'll traverse in more than one direction
  ✓ Correctness matters more than the last 20% of read latency
  ✓ You are under ~10 TB and under ~50k writes/sec  (i.e. almost everyone)

USE DOCUMENT WHEN:
  ✓ The aggregate is the natural unit — always read whole, usually written whole
  ✓ Fields genuinely vary per record (product attributes across categories)
  ✓ Read patterns are dominated by "fetch this one entity by id"
  ✗ AVOID when: you need cross-document transactions on hot paths, or your
    access patterns are ad-hoc and analytical

USE KEY-VALUE WHEN:
  ✓ Access is 100% by exact key
  ✓ Values are small and independent
  ✓ Data is ephemeral, regenerable, or cached
  ✗ AVOID when: you will ever need "all X where Y"

USE WIDE-COLUMN WHEN:
  ✓ Write volume genuinely exceeds a single node (>100k writes/sec sustained)
  ✓ Query patterns are FIXED and known before you design the schema
  ✓ You accept maintaining one table per query pattern, in application code
  ✗ AVOID when: you're not certain you need it. The operational cost is real.

USE TIME-SERIES WHEN:
  ✓ Rows are append-only, timestamped, and never updated
  ✓ Queries are time-range + aggregate
  ✓ You need retention/downsampling as a first-class feature

USE GRAPH WHEN:
  ✓ Queries are variable-depth traversals ("shortest path", "3 hops out")
  ✓ The relationships carry as much meaning as the nodes
  ✗ AVOID when: your "graph" is 2 joins deep — a relational DB wins there

USE SEARCH WHEN:
  ✓ Ranked relevance, typo tolerance, faceting, or natural-language matching
  ✓ ...and it is a DERIVED index, never the source of truth

THE SIGNAL TO LOOK FOR:
  Write down your top 10 queries BEFORE choosing.
  • If you can't enumerate them → relational. It is the only model that
    tolerates not knowing.
  • If you can enumerate them AND you're past single-node write capacity
    → a specialised model becomes worth its operational cost.
  • If you're choosing a model to avoid learning SQL → you are choosing wrong.

THE DEFAULT: start relational. Add a derived specialised store when a specific,
measured access pattern proves the relational one can't serve it. Never move the
source of truth off a transactional engine to chase read latency.
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
For each of these, name the model and the reason in one sentence: (a) user sessions with a 20-minute expiry, (b) an audit log written 8,000 times/sec and read as "everything for user X last month", (c) "which of my contacts also know this person", (d) product prices that must never be wrong, (e) autocomplete for a search box.

### Exercise 2 — medium (apply it)
You have a `products` table where different categories have wildly different attributes (a laptop has RAM and CPU; a shirt has size and fabric). Three options: (a) a wide table with 200 mostly-NULL columns, (b) an EAV table, (c) a `jsonb attributes` column with a GIN index. For each: state the storage cost, the query cost for "all shirts of size M", the integrity guarantee, and one failure mode. Then pick one and defend it.

### Exercise 3 — hard (production simulation)
Your relational e-commerce DB is at 2.4 TB. The analytics team runs 40 queries an hour, each scanning 18 months of `orders`, and these queries are now causing p99 latency spikes on checkout. A vendor proposes migrating everything to a wide-column store.

(a) Explain at the storage-engine level *why* the analytics queries hurt checkout — be specific about which shared resource they contend for.
(b) Give three solutions in increasing order of cost/complexity, none of which is "migrate everything."
(c) State the exact condition under which the vendor would actually be right.
(d) Name the one thing that must never move off PostgreSQL in this system, and why.

---

## Mental model checkpoint

1. Name the four axes that decide a data model. Which one do people most often skip?
2. In the relational model, when is the join paid for? In the document model? What does that imply about which is right for a write-heavy vs read-heavy workload?
3. Why can't a B-tree index serve `LIKE '%kurta%'`? What structure can, and why?
4. Give a case where copying a product's name into an order line is *correct*, and a case where it is a bug. What distinguishes them?
5. What does "schemaless" actually mean about where the schema lives?
6. Why is a row store a bad fit for `SUM(one_column)` over 40M rows? Quantify roughly.
7. What single property must every derived store in a polyglot architecture have?

---

## Quick reference card

| Model | Physical layout | Cheap | Expensive |
|---|---|---|---|
| Relational | rows in 8KB pages, separate index files | ad-hoc filters, joins, transactions | very high write rates; single-column scans |
| Document | one self-contained blob per entity | fetch whole entity by id | cross-entity queries; updating a copied fact |
| Key-value | hash → offset | exact-key get/set | anything that isn't an exact key |
| Wide-column | sorted, clustered by partition+clustering key | the one query you designed for | any other query |
| Columnar | one file per column | aggregates over few columns | fetching whole rows; point updates |
| Graph | pointers between nodes | variable-depth traversal | aggregate reporting |
| Time-series | time-chunked, compressed | time-range + aggregate | updates, non-time queries |
| Search | inverted index (term → doc ids) | ranked text match, facets | being a source of truth |

**Numbers to keep in mind**

| Threshold | Meaning |
|---|---|
| ~10 TB / ~50k writes/sec | roughly where one PostgreSQL node starts needing help |
| 16 MB | MongoDB document size limit |
| >2 joins deep, variable depth | the point where graph starts beating SQL |
| 99% of companies | never exceed single-node relational capacity |

**The one-line rule:** *Start relational. Derive everything else from it. Never let a derived store become the only copy of a fact.*

---

## When would I use this at work?

1. **A new feature kicks off.** Someone says "this should go in Mongo." You ask for the top 5 queries. Half the time, writing them down reveals they're ad-hoc and relational is correct; the other half, it reveals a genuine document shape. Either way the decision takes 15 minutes and is defensible in a design doc.

2. **Search gets slow.** `ILIKE '%term%'` is doing sequential scans. Instead of adding hardware, you recognise this as a model mismatch — a b-tree fundamentally cannot serve a leading wildcard — and propose either PostgreSQL full-text (`tsvector` + GIN) or a search engine, with the trade-offs written out.

3. **Analytics is killing production.** Reporting queries scan 18 months of orders and evict the entire hot working set from `shared_buffers`, so checkout starts hitting disk. You can explain the mechanism (buffer pool contention) and propose a read replica or a columnar rollup rather than a full re-platform.

---

## Connected topics

**Understand before this:** 01 (the four problems), 02 (the engine components).

**This unlocks:**
- **04–08** — how the relational model specifically lands on disk
- **06** — row vs column store, the OLTP/OLAP split in detail
- **70–76** — Phase 8: actually *modelling* in each of these non-relational models
- **75** — the full SQL vs NoSQL decision, revisited with everything you'll know by then
- **76** — polyglot persistence and keeping derived stores in sync via CDC
