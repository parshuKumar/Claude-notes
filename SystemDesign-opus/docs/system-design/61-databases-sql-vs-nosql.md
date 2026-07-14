# 61 — Databases — SQL vs NoSQL Deep Dive
## Category: HLD Components

---

## What is this?

A database is the part of your system that **remembers things after the process dies**. Everything else — your API servers, your caches, your load balancers — can be thrown away and rebuilt. The database cannot. It is the single most consequential choice in a system design, and the hardest one to reverse.

"SQL vs NoSQL" is the question of **what shape your memory has**. SQL (relational) databases store data as rigid tables with enforced relationships, like a government filing cabinet where every form must have every field filled in correctly. NoSQL is not one thing — it is **five different families** of database, each of which threw away a different guarantee in exchange for a different superpower.

---

## Why does it matter?

Pick wrong and you pay for years.

- Choose MongoDB for a banking ledger and you will spend two years hand-rolling transactions that Postgres gives you for free in one line.
- Choose Postgres for storing 40 billion IoT sensor readings and your `INSERT` throughput collapses because every write updates six B-tree indexes.
- Choose Cassandra because a conference talk said "web scale," then discover you cannot run the ad-hoc `JOIN` your CEO asks for on Monday morning.

**In interviews:** Every single HLD question has a "which database and why" moment. The interviewer is not testing whether you memorized that "Cassandra is write-optimized." They are testing whether you can derive the choice from the **access pattern**. A candidate who says "Postgres, because we need multi-row transactions on orders and the data is under 2 TB" beats a candidate who reflexively says "NoSQL for scale" every time.

**At work:** You will inherit a database someone else chose. Understanding *why* it was chosen — and whether that reason still holds — is how you avoid both a needless migration and a needed one that comes too late.

---

## The core idea — explained simply

### The Filing Cabinet vs The Warehouse Analogy

Imagine you run an organization and you must store information.

**The government filing cabinet (SQL / relational):**
Every document is a **form**. Every form has the same fields, in the same order, in the same drawer. Form 27B has a field "Employee ID" and it *must* match a real employee in the Employees drawer — the clerk physically will not accept the form otherwise. If you want "all employees in the Delhi office hired after 2020," a clerk can walk to the cabinet and get it, even though nobody planned for that question in advance. If you want to move a payment from one account to another, the clerk locks both drawers, does both edits, and unlocks — nobody can ever see a world where money left one account and never arrived at the other.

The cost: to add a new field to Form 27B, you must reprint and refile **every existing copy**. And there is only one cabinet, in one room. When the queue of clerks gets too long, you cannot just build a second room — the cabinets have to talk to each other, and that's where it gets ugly.

**The warehouse of labelled bins (NoSQL):**
Every item goes in a bin with a label on it. You want the item? Shout the label, a runner fetches the bin. It is *blindingly* fast, and you can add warehouses forever — bins with labels A–M in Mumbai, N–Z in Chennai. Nobody checks whether the contents of a bin make sense. You can put a shoe in a bin labelled "invoice." Nobody will stop you.

The cost: "all employees in the Delhi office hired after 2020" is now a nightmare — you'd have to open **every bin in every warehouse** to find out. Unless you *pre-planned* that question and kept a second set of bins organized by office-and-year. NoSQL makes you decide your questions **before** you store your data.

**Mapping it back:**

| Analogy | Technical concept |
|---|---|
| The form with mandatory fields | **Schema** — an enforced structure every row must obey |
| Clerk rejects a bad Employee ID | **Foreign key constraint** — referential integrity |
| Locking both drawers for a transfer | **Transaction / ACID** |
| Any question answerable by walking the cabinet | **Ad-hoc queries + JOINs** |
| Reprinting every form to add a field | **Schema migration** |
| One room, one cabinet | **Vertical scaling limit / single-writer bottleneck** |
| Shout a label, get a bin | **Key-value lookup, O(1)** |
| Warehouses split A–M / N–Z | **Sharding / horizontal scaling** |
| Nobody checks the bin contents | **Schemaless — the app enforces shape, or nobody does** |
| Keep a second set of bins for a second question | **Denormalization — one table per query pattern** |

The whole SQL-vs-NoSQL debate collapses into one sentence: **relational databases optimize for questions you haven't thought of yet; NoSQL databases optimize for questions you already know you'll ask, a million times a second.**

---

## Key concepts inside this topic

### 1. Relational databases — the default, and why

A relational database stores data in **tables**: rows (records) and columns (fields), with a declared **schema** — the name and type of every column, fixed ahead of time.

```sql
CREATE TABLE users (
  id          BIGSERIAL PRIMARY KEY,
  email       VARCHAR(255) NOT NULL UNIQUE,
  name        VARCHAR(120) NOT NULL,
  created_at  TIMESTAMPTZ  NOT NULL DEFAULT now()
);

CREATE TABLE orders (
  id          BIGSERIAL PRIMARY KEY,
  user_id     BIGINT       NOT NULL REFERENCES users(id),  -- foreign key
  total_cents BIGINT       NOT NULL CHECK (total_cents >= 0),
  status      VARCHAR(20)  NOT NULL DEFAULT 'PENDING',
  created_at  TIMESTAMPTZ  NOT NULL DEFAULT now()
);
```

`REFERENCES users(id)` is the magic. The database itself will **refuse** to insert an order pointing at a user who doesn't exist, and will refuse to delete a user who still has orders. That guarantee is enforced in one place — the database — not scattered across every service that writes.

**Normalization** — storing each fact exactly once — is the organizing principle. Three levels are worth knowing:

- **1NF (First Normal Form):** no repeating groups, no arrays crammed into a column. Bad: a `users` row with `phone1, phone2, phone3`. Good: a separate `phones` table with one row per phone.
- **2NF:** every non-key column depends on the *whole* primary key. If the key is `(order_id, product_id)` and you store `product_name` in that table, that's a violation — `product_name` depends only on `product_id`. Move it to `products`.
- **3NF:** no non-key column depends on another non-key column. If `orders` stores `user_id` **and** `user_email`, that's a violation — email depends on user_id, not on order_id. Store it once, in `users`.

The point of all this is **exactly one place to update**. If a user changes their email and it lives in one table, one `UPDATE` fixes everything. If it was copy-pasted into 4 million order rows, you now have a data migration and a bug report.

**JOINs** are how you reassemble the pieces at read time:

```sql
-- "Revenue per user in the last 30 days, top 10 spenders"
SELECT u.id, u.name, u.email,
       COUNT(o.id)        AS order_count,
       SUM(o.total_cents) AS revenue_cents
FROM users u
JOIN orders o ON o.user_id = u.id
WHERE o.created_at > now() - INTERVAL '30 days'
  AND o.status = 'PAID'
GROUP BY u.id, u.name, u.email
ORDER BY revenue_cents DESC
LIMIT 10;
```

Nobody designed the schema for this query. It just works. That is the superpower — and it is the thing you throw away first when you go NoSQL.

**Why relational is still the default choice for most systems:**

1. **Your data is relational.** Users have orders, orders have items, items belong to products. That is a graph of relationships, and relational databases model it natively.
2. **You don't know your queries yet.** Product will ask for a report you didn't anticipate. SQL answers it in an afternoon.
3. **Transactions are free.** Multi-row correctness is one `BEGIN`/`COMMIT` away.
4. **One machine is enough.** A single well-tuned Postgres box on modern hardware handles **tens of thousands of QPS** and **terabytes** of data. Most companies never outgrow it — and Postgres has JSONB columns, full-text search, and geospatial indexes, so it also does 80% of what people reach for NoSQL to get.

**Products:** PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, SQLite.

---

### 2. The five NoSQL families

NoSQL is not a database. It's a category containing five *very* different things. Learn them as five, never as one.

#### a) Key-Value — "a giant `Map` that survives reboot"

**What it is:** The simplest possible database. You store a value under a key. You get it back by key. That's it.

**Data model:** `key -> opaque blob`. The database does not know or care what's inside the value.

```
"session:8f3a"  ->  {"userId": 42, "role": "admin", "exp": 1751500000}
"cart:42"       ->  [{"sku":"A1","qty":2}]
"ratelimit:1.2.3.4:60s" -> 17
```

**When to use it:** Sessions, caches, rate-limit counters, feature flags, leaderboards, shopping carts — anywhere the *only* question you will ever ask is "give me the value for this exact key."

**When NOT to use it:** The moment you want "all sessions for admins" you're stuck. There is no query; there is only lookup.

**Real products:** **Redis** (in-memory, sub-millisecond, also does lists/sets/sorted-sets — this is what powers most leaderboards and rate limiters). **DynamoDB** (AWS, disk-backed, single-digit-millisecond at essentially unlimited scale — it powers the Amazon.com shopping cart).

```javascript
// Redis as a session store — the entire access pattern is get/set by key.
import { createClient } from 'redis';
const redis = await createClient({ url: process.env.REDIS_URL }).connect();

async function saveSession(sessionId, data) {
  // TTL is a first-class feature — the DB expires the key for you.
  await redis.set(`session:${sessionId}`, JSON.stringify(data), { EX: 3600 });
}

async function loadSession(sessionId) {
  const raw = await redis.get(`session:${sessionId}`);
  return raw ? JSON.parse(raw) : null;
}
```

#### b) Document — "JSON blobs you can query inside"

**What it is:** Key-value, but the database *understands* the value. Values are documents (JSON/BSON), and you can query and index on fields *inside* the document.

**Data model:** collections of documents. Documents in the same collection need not have the same fields.

```javascript
// One document holds the whole aggregate — no JOIN needed to render a product page.
{
  _id: ObjectId("..."),
  sku: "TSHIRT-BLK-M",
  title: "Black T-Shirt",
  price: 1999,
  attributes: { size: "M", color: "black", material: "cotton" }, // varies per category!
  reviews: [
    { user: "amy", stars: 5, text: "great" },
    { user: "bob", stars: 3, text: "shrank" }
  ]
}
```

**When to use it:** When your entity is naturally a **self-contained tree** that you read and write as a whole, and its shape genuinely varies between records. Product catalogs (a laptop has `ram_gb`, a t-shirt has `size`), CMS content, event/audit payloads, user-generated form data.

**When NOT to use it:** When entities are heavily interlinked. Modelling "orders reference users reference addresses" in Mongo means either duplicating data (and updating it in six places) or doing JOINs in application code — which is worse than the JOINs you were avoiding.

**Real product:** **MongoDB**. (Note: modern MongoDB *does* have multi-document ACID transactions since v4.0 — the old "NoSQL has no transactions" line is outdated. But they're slower and less ergonomic than in Postgres, so the design pressure is still to avoid needing them.)

#### c) Wide-Column — "rows where every row can have different columns, sharded to infinity"

**What it is:** The hardest one to picture. It looks like a table, but think of it as a **nested map**: `partition key -> (clustering key -> columns)`. Data with the same partition key lives together on the same node, sorted by the clustering key.

**Data model:**

```
Partition key: user_id       (decides WHICH machine the data lives on)
Clustering key: message_ts   (decides the SORT ORDER within that machine)

user_42 ──▶ [ 2026-07-12T10:00 | text="hi"    ]
            [ 2026-07-12T10:01 | text="hello" ]
            [ 2026-07-12T10:05 | text="bye"   ]     <- all on one node, pre-sorted
```

```sql
-- Cassandra CQL. Notice: you design the table FOR the query.
CREATE TABLE messages_by_conversation (
  conversation_id UUID,
  message_ts      TIMEUUID,
  sender_id       UUID,
  body            TEXT,
  PRIMARY KEY ((conversation_id), message_ts)   -- partition key, then clustering key
) WITH CLUSTERING ORDER BY (message_ts DESC);
```

That table answers exactly one question — "the latest N messages in conversation X" — and answers it at millions of writes per second across hundreds of nodes with **no single master**. Any node accepts a write. That's the superpower.

**When to use it:** Enormous write volume, time-ordered data, a known access pattern, and a need for linear horizontal scaling with no coordinator. Chat messages, activity feeds, event logs, IoT telemetry.

**When NOT to use it:** Ad-hoc queries. `WHERE body LIKE '%refund%'` is essentially impossible. You cannot JOIN. If you want a second access pattern, you write a **second table** and dual-write to it. That's not a hack — that's the intended design.

**Real products:** **Cassandra** (used by Discord for message history; by Netflix for viewing history). **HBase** (the same idea on top of Hadoop/HDFS). **ScyllaDB** (a C++ rewrite of Cassandra).

#### d) Graph — "the relationships ARE the data"

**What it is:** Nodes and edges are first-class. Traversing an edge is a pointer hop, not a JOIN.

**Data model:** `(Person)-[:FOLLOWS]->(Person)`, `(Person)-[:BOUGHT {on: date}]->(Product)`.

**When to use it:** When the *interesting* questions are about paths of unknown depth. "Friends of friends of friends who like jazz." "Which accounts are within 4 hops of this known fraudster?" "Shortest route through the road network."

Here's the key insight: in SQL, a 4-hop friend-of-friend query is a **self-join four times deep**, and the row count explodes — it will take seconds. In a graph DB it's a traversal that touches only the nodes it needs, and it takes milliseconds. The advantage grows with depth.

```cypher
// Neo4j Cypher — "people 2-3 hops from me who work at Google"
MATCH (me:Person {id: 42})-[:KNOWS*2..3]-(p:Person)-[:WORKS_AT]->(:Company {name:'Google'})
RETURN DISTINCT p.name LIMIT 20;
```

**When NOT to use it:** Everything else. Graph DBs are a specialist tool. Do not store your orders in Neo4j.

**Real product:** **Neo4j** (fraud rings at banks, recommendation engines, network/IT topology). Also Amazon Neptune, and Postgres with recursive CTEs for lighter cases.

#### e) Time-Series — "append-only, timestamped, aggregated over windows"

**What it is:** A database that assumes every row has a timestamp, that writes are almost always *appends at the current time*, and that reads are almost always *aggregations over a time window*.

**Data model:** `(measurement, tags, timestamp) -> fields`.

```
cpu_usage,host=web-01,region=us-east  value=73.2  1751500000
cpu_usage,host=web-02,region=us-east  value=61.8  1751500000
```

Because it knows the shape, it can do things a general DB cannot: **columnar compression** (10-20x smaller than Postgres for the same metrics), automatic **downsampling** (keep 1-second resolution for a day, 1-minute for a month, 1-hour for a year), and **retention policies** that drop old data automatically.

**When to use it:** Metrics, monitoring, IoT sensors, stock ticks, application telemetry. Anything where you ask "average CPU per host, in 5-minute buckets, over the last 6 hours."

**Real products:** **InfluxDB** (purpose-built). **TimescaleDB** (a Postgres extension — you get time-series superpowers *and* keep SQL, joins, and ACID; often the right answer). Also Prometheus for monitoring specifically.

---

### 3. ACID — property by property, with a bank transfer

ACID is the set of four guarantees a relational database gives you for a **transaction** — a group of operations treated as one indivisible unit.

The canonical example: Alice transfers ₹500 to Bob.

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 'alice' AND balance >= 500;
  UPDATE accounts SET balance = balance + 500 WHERE id = 'bob';
  INSERT INTO ledger (from_id, to_id, amount, at) VALUES ('alice','bob',500, now());
COMMIT;
```

**A — Atomicity: all or nothing.**
The server can crash, the power can die, the process can be `kill -9`'d between the two UPDATEs. It does not matter. On restart, the database will have applied **all three statements or none of them**. There is no universe in which ₹500 leaves Alice and never reaches Bob. Without atomicity, you would have to write compensating-transaction logic by hand — and get it right under every crash interleaving.

**C — Consistency: the database never violates its own rules.**
If you declared `CHECK (balance >= 0)` and a foreign key from `ledger.from_id` to `accounts.id`, then no transaction can ever commit a state that breaks those rules. The transfer that would drive Alice negative is rejected, not silently applied. (Note: this "C" is about *constraints*, and is a totally different word from the "C" in CAP theorem, which is about *replicas agreeing*. This confusion is worth a full sentence in an interview.)

**I — Isolation: concurrent transactions don't see each other's half-done work.**
Alice's transfer to Bob and Bob's transfer to Carol run at the same time. Isolation guarantees the result is the same as if they had run **one after the other** in *some* order. Without it, a classic disaster:

```
Time  Transfer A (Alice→Bob)        Transfer B (Alice→Carol)
 t1   read alice.balance = 1000
 t2                                 read alice.balance = 1000
 t3   write alice.balance = 500
 t4                                 write alice.balance = 500   <- A's deduction VANISHED
```

Alice sent ₹1000 total but her balance dropped by only ₹500. That's a **lost update**, and it's how you print money by accident. Isolation levels (read committed, repeatable read, serializable) are the dial that controls how much of this you prevent versus how much concurrency you keep — that's the subject of [66 — Transactions and Isolation Levels](./66-database-transactions-and-isolation.md).

**D — Durability: once it says "committed," it's on disk.**
The `COMMIT` does not return until the change is in a write-ahead log flushed to durable storage. Pull the power cord one nanosecond after the client gets its OK — the money is still moved when the machine comes back. Not "probably." Guaranteed.

---

### 4. BASE — the contrast

Distributed NoSQL systems typically cannot offer ACID across nodes without paying a coordination cost on every write (that's essentially the CAP theorem — see [09 — CAP Theorem](./09-cap-theorem.md): under a network partition you must choose consistency or availability). So they offer **BASE** instead:

- **Ba**sically **A**vailable — the system always answers. It may give you a slightly stale answer, but it does not go down. A node that can't reach its peers still serves reads and accepts writes.
- **S**oft state — the state of the system may change over time even with no new writes coming in, because replicas are still catching up with each other in the background.
- **E**ventual consistency — if writes stop, all replicas *converge* to the same value, eventually. "Eventually" is typically milliseconds; occasionally, under partition, seconds or minutes.

| | ACID | BASE |
|---|---|---|
| Guarantee | Correct **now**, everywhere | Correct **soon**, everywhere |
| On network partition | Refuse writes (stay correct) | Accept writes (stay up) |
| Read after write | Always sees your write | May not see your write |
| Typical home | Postgres, MySQL | Cassandra, DynamoDB (default mode) |
| Right for | Money, inventory, bookings | Likes, view counts, feeds, presence |

The honest framing: **ACID vs BASE is not "safe vs reckless." It's a question of what a wrong answer costs.** If a like-count is stale by 300ms, nobody is harmed. If a bank balance is stale by 300ms, you can double-spend. Choose per-dataset, not per-company.

---

### 5. The REAL decision framework

Forget the hype. Answer these five questions about your **access pattern**. Each one moves the needle.

| Question | If YES → | If NO → | Why it matters |
|---|---|---|---|
| **Do you need multi-row / multi-entity transactions?** (money, inventory, bookings) | **Relational.** Strongly. | Free to consider NoSQL. | Hand-rolling atomicity across documents is where careers go to die. |
| **Is your schema stable and uniform?** | **Relational** — let the DB enforce it. | **Document** — genuinely varying shapes (product catalogs) are what it's for. | "Schemaless" doesn't mean no schema; it means the schema moved into your app code, unenforced. |
| **Do you need flexible ad-hoc queries / reports?** | **Relational.** JOINs and a query planner, for free. | Wide-column is fine — you can design one table per known query. | Product *will* ask for a query nobody planned. Bet on it. |
| **What's the read:write ratio, and what's the write volume?** | Write-dominant, >50k writes/sec sustained, append-only → **Wide-column** (Cassandra) or **time-series**. | Read-dominant or moderate writes → **Relational + read replicas + a cache**. | A single Postgres primary is a single writer. That's the real ceiling. |
| **Will the working set fit on one machine (with replicas)?** | **Relational.** One box, done. | You need built-in sharding → **DynamoDB / Cassandra / Mongo sharded / CockroachDB**. | "One machine" today means ~64 TB of NVMe and 1 TB of RAM. That is a *lot* of business. |

Two more, for the interview:

- **What are the query shapes?** Point lookups by key → key-value. Range scans over time within a partition → wide-column/time-series. Deep multi-hop traversals → graph. Anything else → relational.
- **How bad is a stale read?** Catastrophic → strong consistency (relational, or Spanner). Annoying → eventual consistency is fine.

Say this out loud in an interview and you have already outperformed most candidates.

---

### 6. Myth-busting

**Myth: "NoSQL is faster."**
For a **single-key lookup**, yes — Redis will beat Postgres because it's in memory. But Postgres doing a primary-key lookup on a warm page is also sub-millisecond. The speed difference people attribute to "NoSQL" is almost always one of three things that have nothing to do with SQL: (a) it was in memory, (b) the data was denormalized so no JOIN was needed, (c) the SQL query had no index on it. You can denormalize *in* Postgres. You can put a cache *in front of* Postgres. Neither requires abandoning transactions.

**Myth: "SQL can't scale."**
This is the big one, and it's simply false as stated. A single PostgreSQL instance on commodity hardware routinely serves **tens of thousands of QPS** and multiple terabytes. Add read replicas and read throughput scales near-linearly. Add partitioning and a single table handles billions of rows. Real, verifiable examples: Instagram ran on sharded PostgreSQL well past 100 million users. GitHub runs on MySQL. Shopify runs on sharded MySQL through Black Friday. Stack Overflow famously served its entire traffic from a handful of SQL Server boxes.

What *is* true: a single relational **primary** is a single point of write scaling, and scaling writes past that means sharding — which relational databases don't do for you automatically the way DynamoDB or Cassandra do. That's the real limit. It is much, much further away than people think.

**Myth: "We'll need web scale, so let's start with NoSQL."**
Most companies that reached for NoSQL never needed it. They took on eventual consistency, lost JOINs, and hand-wrote transaction logic — to solve a scale problem they never had. The pattern that actually works: **start relational. Measure. Move the specific hot dataset that genuinely doesn't fit** — the message history, the event log, the session store — to the specialized store it belongs in. Which brings us to:

---

### 7. Polyglot persistence, and NewSQL

**Polyglot persistence** means: stop asking "which database?" and start asking "which database *for this data*?" A real production system usually looks like:

| Data | Store | Why |
|---|---|---|
| Users, orders, payments | PostgreSQL | Transactions, relations, reporting |
| Sessions, rate limits, hot cache | Redis | Sub-ms key lookups, TTLs |
| Chat message history | Cassandra | Massive append-only writes, partition by conversation |
| Product catalog / search | Elasticsearch | Full-text relevance ranking |
| Images, video | S3 (object store) | Cheap bytes, not a database problem |
| App metrics | Prometheus / Timescale | Time-window aggregation |

The cost is real: more systems to operate, more failure modes, no cross-store transactions, and engineers who must know five query languages. **Don't start here.** Arrive here, one justified store at a time.

**NewSQL** is the attempt to stop choosing. **Google Spanner** and **CockroachDB** (and TiDB, YugabyteDB) give you SQL, JOINs, and full ACID transactions — but *distributed*, with automatic sharding and horizontal write scaling. Spanner does it partly with atomic clocks (TrueTime) in Google's datacenters; Cockroach does it with a consensus protocol (Raft) per data range.

The catch: cross-shard transactions require coordination, so writes have higher latency than a single-node Postgres (tens of ms, not sub-ms), and the operational complexity is significant. Reach for NewSQL when you genuinely need **both** relational semantics **and** multi-region write scaling — not before.

---

## Visual / Diagram description

### Diagram 1: The five NoSQL families, in one picture

```
┌────────────────────────────────────────────────────────────────────────┐
│                          THE DATA MODEL ZOO                            │
└────────────────────────────────────────────────────────────────────────┘

 RELATIONAL                            KEY-VALUE
 ┌──────────────────────────┐          ┌────────────────────────┐
 │ users                    │          │  "session:8f3a" ──▶ {…}│
 │ ┌────┬───────┬─────────┐ │          │  "cart:42"      ──▶ [{…│
 │ │ id │ name  │ email   │ │          │  "rl:1.2.3.4"   ──▶ 17 │
 │ ├────┼───────┼─────────┤ │          └────────────────────────┘
 │ │ 1  │ Amy   │ a@x.com │ │           opaque values, O(1) by key
 │ └─┬──┴───────┴─────────┘ │           Redis, DynamoDB
 │   │ FK                   │
 │ ┌─▼──┬─────────┬───────┐ │          DOCUMENT
 │ │ id │ user_id │ total │ │          ┌────────────────────────┐
 │ │ 9  │    1    │ 1999  │ │          │ { _id: 1,              │
 │ └────┴─────────┴───────┘ │          │   sku: "TSHIRT",       │
 │ orders                   │          │   attrs:{size:"M"},    │◀─ query
 └──────────────────────────┘          │   reviews:[{…},{…}] }  │   INSIDE
   schema + JOINs + ACID               └────────────────────────┘
   Postgres, MySQL                      MongoDB — a whole tree per doc

 WIDE-COLUMN                            GRAPH
 ┌──────────────────────────┐          ┌────────────────────────┐
 │ conv_A ▶ [t1|"hi"  ]     │          │   (Amy)──FOLLOWS──▶(Bo)│
 │          [t2|"yo"  ]     │          │     │                ▲ │
 │          [t3|"bye" ]     │          │  BOUGHT           KNOWS│
 │ ────── (node 1) ──────   │          │     ▼                │ │
 │ conv_B ▶ [t1|"hey" ]     │          │  (iPhone)◀─BOUGHT──(Cy)│
 │ ────── (node 2) ──────   │          └────────────────────────┘
 └──────────────────────────┘           edges are pointers, not JOINs
  partition key picks the NODE          Neo4j
  clustering key sorts WITHIN it
  Cassandra, HBase                      TIME-SERIES
                                        ┌────────────────────────┐
                                        │ cpu{host=web-01}       │
                                        │  10:00 ▏▎▍▌▋▊▉█ 73.2   │
                                        │  10:01 ▏▎▍▌▋    61.8   │
                                        │  auto-downsample+expire│
                                        └────────────────────────┘
                                         InfluxDB, TimescaleDB
```

**What this shows:** the *shape* of the data dictates the family. Opaque blob → key-value. Self-contained tree → document. Huge sorted append-only partitions → wide-column. Relationships-as-the-point → graph. Timestamped measurements → time-series. Everything else → relational.

### Diagram 2: Polyglot persistence in a real e-commerce system

```
                        ┌─────────────────┐
                        │   API Gateway   │
                        └────────┬────────┘
             ┌───────────────────┼───────────────────┬───────────────┐
             ▼                   ▼                   ▼               ▼
      ┌────────────┐      ┌────────────┐      ┌────────────┐  ┌────────────┐
      │   Order    │      │  Catalog   │      │   Session  │  │  Activity  │
      │  Service   │      │  Service   │      │  Service   │  │  Service   │
      └─────┬──────┘      └─────┬──────┘      └─────┬──────┘  └─────┬──────┘
            │                   │                   │               │
            ▼                   ▼                   ▼               ▼
   ┌─────────────────┐  ┌──────────────┐    ┌────────────┐  ┌──────────────┐
   │   PostgreSQL    │  │Elasticsearch │    │   Redis    │  │  Cassandra   │
   │                 │  │              │    │            │  │              │
   │ users, orders,  │  │ product      │    │ sessions,  │  │ user activity│
   │ payments,       │  │ search index │    │ carts,     │  │ event stream │
   │ inventory       │  │ (full-text)  │    │ rate limit │  │ (append-only)│
   │                 │  └──────▲───────┘    └────────────┘  └──────────────┘
   │  ACID. The      │         │
   │  source of      │  ┌──────┴───────┐
   │  TRUTH.         │──│ CDC / indexer│  Postgres stays the source of truth;
   └─────────────────┘  └──────────────┘  everything else is a derived view.
```

**What this shows — and the rule to remember:** exactly **one** store is the source of truth (Postgres). Every other store is a **derived, rebuildable projection** of it. If Elasticsearch burns down, you reindex from Postgres and lose nothing. If Redis burns down, users re-login. If Postgres burns down, you're out of business. Keep the ACID guarantees where the money is; use the specialized stores for everything downstream.

---

## Real world examples

### 1. Discord — Cassandra (then ScyllaDB) for messages, Postgres for everything else

Discord stores **trillions** of messages. They publicly documented moving message storage from MongoDB to Cassandra, and later from Cassandra to ScyllaDB. The reason is a textbook wide-column fit: messages are append-heavy, always read as "the last N messages in channel X," and never need a JOIN. Their partition key is a `(channel_id, bucket)` pair — bucketing by time so a single hot channel's partition doesn't grow unbounded, which is the classic wide-column failure mode. Meanwhile the *rest* of Discord — users, guilds, permissions — is relational, because that data is small, highly interlinked, and needs consistency.

### 2. Instagram — sharded PostgreSQL, well past 100M users

Instagram scaled to hundreds of millions of users on **PostgreSQL**, sharded by user ID across many logical shards mapped onto physical machines. They didn't switch to a NoSQL store for their core data; they sharded the relational one and used Memcached aggressively in front. It is the clearest counterexample to "SQL can't scale," and a good story to have ready when an interviewer implies that relational is a toy.

### 3. Amazon — DynamoDB, born from a real relational failure

DynamoDB's ancestor (Dynamo) was built because the Amazon shopping cart could **not** be allowed to go down — and a relational primary that refuses writes during a partition means a customer can't add to cart, which means lost revenue. So they chose availability: the cart is a key-value store, writes always succeed, and conflicting versions of a cart are reconciled later (historically by merging — which is why an item you deleted could famously reappear). That's BASE in production, and it's a *deliberate* trade: a resurrected cart item costs almost nothing; a cart that rejects writes costs millions.

---

## Trade-offs

| Dimension | Relational (Postgres/MySQL) | NoSQL (varies by family) |
|---|---|---|
| **Schema** | Enforced by the DB — bad data cannot get in | Enforced by your app, or by nobody |
| **Transactions** | Multi-row ACID, trivially | Single-key usually; multi-key is limited/slow/absent |
| **Queries** | Ad-hoc, JOINs, aggregations, a query planner | Only what you designed the table/keys for |
| **Write scaling** | One primary; sharding is manual work | Built-in, near-linear, add nodes |
| **Read scaling** | Read replicas (with replication lag) | Built-in |
| **Schema change** | `ALTER TABLE` — can be slow/locking on huge tables | Just write the new field; old docs keep the old shape |
| **Operational cost** | Well-understood, mature tooling, easy to hire for | More moving parts, fewer experts, subtler failure modes |
| **Correctness under partition** | Consistent (may refuse writes) | Available (may serve stale data) |

| Choice | You gain | You give up |
|---|---|---|
| Postgres for everything | Correctness, flexibility, simplicity, hiring | Automatic sharding; a hard write ceiling far away |
| Mongo for everything | Schema flexibility, easy horizontal scale | JOINs; ergonomic transactions; DB-enforced integrity |
| Cassandra for the write-heavy dataset | Enormous write throughput, no single master | Ad-hoc queries, JOINs, strong consistency by default |
| Polyglot persistence | Each dataset in its best home | Ops complexity, no cross-store transactions, more to learn |
| NewSQL (Spanner/Cockroach) | SQL + ACID + horizontal write scale | Higher write latency, cost, operational complexity |

**Rule of thumb:** **Start with PostgreSQL. Always.** Move a *specific dataset* off it only when you have a *measured* reason — a write rate one machine can't sustain, a data shape that genuinely isn't relational, or a query type (full-text, deep traversal, time-window rollup) that a specialist does 100x better. "We might need scale someday" is not a measured reason.

---

## Common interview questions on this topic

### Q1: "SQL or NoSQL for this system — and why?"
**Hint:** Never answer from vibes. Derive it: *What are the access patterns? Do we need multi-row transactions? Is the schema uniform? What's the write rate? Will it fit on one machine?* Then answer per-dataset, not per-system: "Postgres for users/orders because we need ACID on payments; Cassandra for the message history because it's 200k appends/sec, always read by conversation, and never joined."

### Q2: "Explain ACID. Then explain how it differs from CAP's C."
**Hint:** ACID's **C** = *the database never violates its declared constraints* (a single-node integrity guarantee). CAP's **C** = *all replicas return the same value* (a distributed agreement guarantee). Different words, different problems. Nailing this distinction signals real depth — most candidates conflate them.

### Q3: "Everyone says NoSQL scales and SQL doesn't. Is that true?"
**Hint:** Mostly false. A single Postgres primary handles tens of thousands of QPS and terabytes; Instagram and Shopify ran huge on sharded relational. What's *true* is that relational databases don't shard writes **automatically** — you do it yourself. NoSQL's real advantage is built-in horizontal write scaling, not raw speed. Most companies that reached for NoSQL never hit the relational ceiling.

### Q4: "You're at 50,000 writes/sec on a Postgres table and it's dying. What do you do — before switching databases?"
**Hint:** In order: (1) check your indexes — every index multiplies write cost; drop unused ones. (2) Batch inserts instead of one-per-row. (3) Move the write off the hot path with a queue. (4) Partition the table by time. (5) Move *just this table* to a store built for it (Cassandra/Timescale). Switching your whole database is the last resort, not the first.

### Q5: "When would you actually reach for a graph database?"
**Hint:** When the queries are variable-depth traversals — fraud rings, friend-of-friend-of-friend, dependency/permission chains, routing. The tell is a SQL query with 4+ self-joins that gets slower with each hop. If your graph is small or shallow, Postgres recursive CTEs are fine and you should not add a fifth database.

---

## Practice exercise

### The Datastore Bake-Off

You're the architect for a ride-hailing app (a mini-Uber). Here are five datasets:

1. **Riders and drivers** — profiles, documents, verification status, bank details.
2. **Trips** — a trip has a rider, a driver, a fare, a payment. Fare must be charged to the rider and credited to the driver as one indivisible operation.
3. **Live driver locations** — every online driver pings GPS every 4 seconds. 500,000 drivers online at peak.
4. **Trip history events** — every state change of every trip ("requested", "accepted", "started", "ended"), append-only, retained for 3 years for compliance. ~50 million events/day.
5. **Surge-pricing metrics** — per-city demand/supply ratios, computed every 30 seconds, queried as "average surge in Bangalore over the last 2 hours in 5-minute buckets."

**Produce:**
- For **each** of the five, name the database family and a concrete product.
- For each, justify with **at least two** of the five framework questions (transactions? stable schema? ad-hoc queries? read:write ratio + write volume? fits on one machine?). Show the arithmetic where volume matters — e.g. do the writes/sec math for dataset 3 (500,000 drivers ÷ 4 seconds = ? writes/sec) and say whether one Postgres primary can take it.
- Name the **one** store that is your source of truth, and explain what you'd lose if each of the others were wiped.
- Finally: write the SQL for dataset 2's fare settlement as a single transaction, with the `CHECK` constraint that makes it impossible to overdraw.

Time-box: 30 minutes. Write it out. If you can't justify a choice with numbers, you chose from hype.

---

## Quick reference cheat sheet

- **NoSQL is five things, not one:** key-value, document, wide-column, graph, time-series. Never say "NoSQL" without naming the family.
- **Key-value** (Redis, DynamoDB): O(1) by key, nothing else. Sessions, carts, rate limits, caches.
- **Document** (MongoDB): query *inside* the JSON. Best for self-contained trees with genuinely varying shapes.
- **Wide-column** (Cassandra, HBase): partition key picks the node, clustering key sorts within it. Massive append-only writes, one table per query.
- **Graph** (Neo4j): edges are pointers. Wins on variable-depth traversals, loses on everything else.
- **Time-series** (InfluxDB, TimescaleDB): timestamped appends, window aggregations, auto-downsampling and retention.
- **ACID** = Atomicity (all or nothing), Consistency (constraints never break), Isolation (concurrent txns behave as if serial), Durability (committed = on disk).
- **ACID's C ≠ CAP's C.** Constraints vs replica agreement. Know this cold.
- **BASE** = Basically Available, Soft state, Eventual consistency. Stay up, converge later.
- **The framework:** multi-row transactions? stable schema? ad-hoc queries? read:write ratio and write volume? fits on one machine? Answer those five and the DB picks itself.
- **"NoSQL is faster" is a myth** — the speedup is usually memory, denormalization, or a missing index, all of which you can have in SQL.
- **"SQL can't scale" is a myth** — Instagram, GitHub, Shopify. The real limit is *automatic write sharding*, and it's far away.
- **Polyglot persistence:** one source of truth (usually Postgres); every other store is a rebuildable derived view.
- **NewSQL** (Spanner, CockroachDB) = SQL + ACID + horizontal write scale, at the cost of write latency and ops complexity.
- **Default answer, in life and in interviews: start with PostgreSQL,** and move a dataset off it only with a measured reason.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [60 — CDN](./60-cdn.md) — the last of the request-path components, before we hit storage |
| **Next** | [62 — Database Indexing](./62-database-indexing.md) — once you've picked a database, indexes are what make it fast |
| **Related** | [09 — CAP Theorem](./09-cap-theorem.md) — the theorem that forces the ACID-vs-BASE choice under partition |
| **Related** | [66 — Database Transactions and Isolation Levels](./66-database-transactions-and-isolation.md) — the "I" in ACID, turned into a dial you control |
