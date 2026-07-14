# 75 — Data Partitioning Strategies
## Category: HLD Components

---

## What is this?

**Data partitioning** is the act of chopping one big pile of data into smaller piles, so each pile can live on a different machine. A **partition** is one of those piles. That's it — that's the whole idea.

Think of a library that has outgrown its building. You don't buy a bigger building forever; you open branch libraries and split the collection across them. The hard question is never *"should we split?"* — it's **"by what rule do we decide which book goes to which branch?"** That rule is your **partitioning strategy**, and picking it badly is one of the most expensive mistakes in system design, because it is nearly impossible to undo once you have real data and real traffic.

---

## Why does it matter?

Three things force you to partition, and they are different problems:

1. **The data doesn't fit.** 40 TB of messages will not fit on a machine with 2 TB of SSD. No amount of tuning fixes this.
2. **The writes don't fit.** A single Postgres primary tops out somewhere around 10k–50k writes/sec depending on hardware and row size. If you need 200k writes/sec, one machine physically cannot do it — you must spread the writes across many machines.
3. **Blast radius.** If all 100 million users live on one database and it dies, *everyone* is down. If they live on 20 partitions and one dies, 5% of users are down. Partitioning turns a total outage into a partial one. This is the reason people forget, and it's often the most valuable one.

**In interviews:** the moment you say "we'll shard the database," the very next question is *"shard by what?"* If you answer "by user_id" without being able to defend it — cardinality, access distribution, whether your top queries even contain a user_id — the interviewer knows you've memorised a word, not a concept. Partition key choice is where senior candidates separate from mid-level ones.

**At work:** you'll pick a partition key once, and live with it for years. Re-partitioning a live 50 TB dataset is a multi-quarter project involving dual writes, backfills, and shadow reads. Get it right the first time.

> Recall from **[64 — Database Sharding](./64-database-sharding.md)** that *sharding* is the specific case of splitting a **database** across machines. This doc is the general theory: the same strategies apply to caches (which Redis node holds this key?), queues (which Kafka partition does this event go to?), and file stores (which bucket holds this blob?). Sharding is one application of partitioning.

---

## The core idea — explained simply

### The Warehouse Aisles Analogy

You run a warehouse. Every incoming parcel must be put somewhere, and later a picker must find it fast. You have 4 aisles and one rule to write on the wall.

**Rule A — "By destination postcode: 1000–1999 → Aisle 1, 2000–2999 → Aisle 2..."**
Great when the boss says *"ship everything going to the 2000s today"* — one picker, one aisle, done. Terrible when it turns out 70% of your customers live in the 2000s: Aisle 2 is a stampede and Aisle 4 has cobwebs.

**Rule B — "Take the parcel's tracking number, run it through a scrambler, take the result mod 4."**
Now the parcels are beautifully even across the aisles. But *"everything going to the 2000s"* now means walking **all four aisles** and picking through everything. You traded query efficiency for balance.

**Rule C — "Keep a clipboard at the door listing exactly which aisle each customer's parcels go to."**
Total flexibility — the huge customer who ships 40% of your volume gets Aisle 3 all to themselves. But now every single parcel, in and out, must stop at the clipboard. Lose the clipboard and the warehouse is useless.

**Rule D — "Parcels for Europe go to the Berlin warehouse, parcels for Asia go to the Singapore warehouse."**
Fast local delivery and it satisfies the law that says European customer data stays in Europe. But a customer who moves from Berlin to Singapore is now a genuine headache.

Every partitioning scheme humans have invented is one of those four rules, or a combination.

| Warehouse | Technical name | Wins at | Loses at |
|---|---|---|---|
| Postcode ranges | **Range partitioning** | Range scans, time-series | Hotspots — uneven aisles |
| Scrambled tracking number | **Hash partitioning** | Even distribution | Range queries, resizing |
| Clipboard at the door | **Directory / lookup** | Flexibility, per-tenant moves | Extra hop, SPOF |
| Berlin vs Singapore | **Geographic** | Latency, data residency | Cross-region queries |
| Postcode *then* shelf | **Composite key** | Both balance and range scans | Requires careful key design |

---

## Key concepts inside this topic

### 1. Vertical partitioning — split by COLUMN (do this first)

Everyone jumps straight to "split the rows across machines." Stop. **Vertical partitioning splits by column or by feature**, needs no new distributed-systems knowledge, and is often enough on its own.

**Level 1 — split a fat table's columns.** A `users` table where every read pulls `id`, `email`, `name`, but the row also carries a 200 KB `profile_blob` and a `bio_html` column that's read once a month:

```
BEFORE — one fat table, 210 KB per row, ~5 rows fit in a page
┌──────────────────────────────────────────────────────────┐
│ users: id | email | name | last_login | profile_blob(200KB) | bio_html │
└──────────────────────────────────────────────────────────┘

AFTER — hot table is skinny; 200 rows fit in a page, cache hit rate soars
┌───────────────────────────────┐   ┌────────────────────────────────┐
│ users (hot, ~200 bytes/row)   │   │ user_profiles (cold)           │
│ id | email | name | last_login│──▶│ user_id | profile_blob | bio_html│
└───────────────────────────────┘   └────────────────────────────────┘
```

You didn't add a machine. You just made the hot table 1000× smaller, so far more of it fits in RAM. This alone has rescued plenty of systems.

**Level 2 — split by feature (the "service split").** The `users`, `orders`, `analytics_events` tables all live in one Postgres. Analytics writes 50k events/sec and its ugly aggregate queries lock up the box, taking checkout down with it. Move `analytics_events` to its own database (probably ClickHouse):

```javascript
// A tiny "vertical partition router" — pick a connection pool per feature.
// This is real code you can ship in an afternoon; no rebalancing, no key design.
const pools = {
  core:      new Pool({ host: 'pg-core.internal',      database: 'app' }),
  analytics: new Pool({ host: 'clickhouse.internal',   database: 'events' }),
  media:     new Pool({ host: 'pg-media.internal',     database: 'media' }),
};

// Every table is owned by exactly one feature-database. Cross-feature JOINs
// are now impossible — which is the point: it forces clean service boundaries.
const TABLE_OWNER = {
  users: 'core', orders: 'core', payments: 'core',
  analytics_events: 'analytics', page_views: 'analytics',
  uploads: 'media',
};

function poolFor(table) {
  const owner = TABLE_OWNER[table];
  if (!owner) throw new Error(`Unowned table: ${table}`);
  return pools[owner];
}
```

**Cost:** you lose JOINs across the split, and you lose transactions across the split. If `users` and `orders` land in different databases, "create order and decrement user credit atomically" is now a distributed transaction problem.

**Rule:** try vertical partitioning before horizontal. It buys you 12–24 months and costs a week.

---

### 2. Horizontal partitioning — split by ROW

Now the rows themselves must live on different machines. Same schema everywhere, different rows.

```
                     100M users, one table
                              │
        ┌─────────────────────┼─────────────────────┐
        ▼                     ▼                     ▼
   ┌──────────┐         ┌──────────┐         ┌──────────┐
   │ Node 1   │         │ Node 2   │         │ Node 3   │
   │ users    │         │ users    │         │ users    │
   │ (rows    │         │ (rows    │         │ (rows    │
   │  1–33M)  │         │ 34–66M)  │         │ 67–100M) │
   └──────────┘         └──────────┘         └──────────┘
        Same schema. Different rows. This is horizontal partitioning.
```

The only question left: **what rule maps a row to a node?** Four answers follow.

---

### 3. Strategy 1 — Range partitioning

Sort the key space, cut it into contiguous chunks, give one chunk to each node.

```
  KEY SPACE (usernames, sorted)
  ├── A ─── F ──┼── G ─── M ──┼── N ─── S ──┼── T ─── Z ──┤
  ▼             ▼             ▼             ▼
┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐
│ Shard 1 │  │ Shard 2 │  │ Shard 3 │  │ Shard 4 │
│  A–F    │  │  G–M    │  │  N–S    │  │  T–Z    │
└─────────┘  └─────────┘  └─────────┘  └─────────┘
```

```javascript
// Ranges are ordered; a router just walks them (or binary-searches for many shards).
const RANGES = [
  { max: 'G', shard: 'shard-1' }, // keys < 'G'
  { max: 'N', shard: 'shard-2' },
  { max: 'T', shard: 'shard-3' },
  { max: null, shard: 'shard-4' }, // catch-all tail
];

function shardForRange(key) {
  const k = key.toLowerCase();
  for (const r of RANGES) {
    if (r.max === null || k < r.max.toLowerCase()) return r.shard;
  }
}

// The superpower: a range query touches ONLY the shards that overlap the range.
function shardsForRangeQuery(fromKey, toKey) {
  return RANGES.filter(r => r.max === null || fromKey < r.max)
               .filter((_, i) => RANGES[i].max === null || fromKey <= toKey)
               .map(r => r.shard);
}
// "All users from 'ab' to 'ay'" → ['shard-1']. One node. One sequential disk scan.
```

**Why people love it:** range scans and time-series queries are *free*. "Give me every event in March 2026" hits exactly the shards holding March.

**THE FAILURE MODE: hotspots.** Range partitioning assumes your key space is uniformly used. It never is.

*Hotspot A — time as the key.* You partition an events table by day. Today is 2026-07-12.

```
WRITES RIGHT NOW (100% of them):
                                                 ▼▼▼▼▼▼▼▼▼▼▼▼
┌────────────┐  ┌────────────┐  ┌────────────┐  ┌────────────┐
│ Shard 1    │  │ Shard 2    │  │ Shard 3    │  │ Shard 4    │
│ Jan–Mar    │  │ Apr–May    │  │ Jun        │  │ Jul (TODAY)│
│ 0 writes/s │  │ 0 writes/s │  │ 0 writes/s │  │ 50k writes/s│
│  IDLE      │  │  IDLE      │  │  IDLE      │  │  ON FIRE   │
└────────────┘  └────────────┘  └────────────┘  └────────────┘
```
You bought 4 machines and you are using 1. Every write in the entire system lands on today's shard, because *time only moves forward*. This is the single most common partitioning mistake in the industry.

*Hotspot B — names as the key.* Split A–F / G–M / N–S / T–Z looks balanced. It isn't. In US surname data, **S** is the single most common initial (~9-10% of people), while **X**, **Q**, **U**, **Y**, **Z** together are well under 2%. So:

```
Shard 1 (A–F): ~28% of users
Shard 2 (G–M): ~30% of users
Shard 3 (N–S): ~27% of users
Shard 4 (T–Z): ~15% of users   ← half-empty, paid for in full
```
And that's before you notice that a *pure* alphabetical split of 26 letters across 26 shards would give the "S" shard ~5× the load of the "X" shard.

**Use range partitioning when:** the key is genuinely uniform (a pre-hashed token), or when range scans matter so much that you're willing to actively manage hotspots by splitting hot ranges (this is what HBase and DynamoDB do — see §8).

---

### 4. Strategy 2 — Hash partitioning

Run the key through a hash function; the hash decides the node.

```
   key = "user_8821"
        │
        ▼
  ┌───────────┐
  │  hash()   │  → 3_814_297_115
  └─────┬─────┘
        │  % 4
        ▼
      shard 3
┌────────┐┌────────┐┌────────┐┌────────┐
│Shard 0 ││Shard 1 ││Shard 2 ││Shard 3 │
│ 25.0%  ││ 25.1%  ││ 24.9%  ││ 25.0%  │   ← beautifully even
└────────┘└────────┘└────────┘└────────┘
```

```javascript
import crypto from 'node:crypto';

// Use a real hash, not JS's `hashCode`-style tricks. MD5/murmur are fine here —
// we need uniformity, not cryptographic strength. Take 4 bytes for a 32-bit int.
function hash32(key) {
  return crypto.createHash('md5').update(String(key)).digest().readUInt32BE(0);
}

const SHARDS = ['shard-0', 'shard-1', 'shard-2', 'shard-3'];

function shardForHash(key) {
  return SHARDS[hash32(key) % SHARDS.length];
}

// Prove the distribution is even before you trust it:
const counts = Object.fromEntries(SHARDS.map(s => [s, 0]));
for (let i = 0; i < 1_000_000; i++) counts[shardForHash(`user_${i}`)]++;
console.log(counts);
// → { 'shard-0': 250131, 'shard-1': 249802, 'shard-2': 250044, 'shard-3': 250023 }
// Within 0.1% of perfect. That is what a good hash buys you.
```

**Why people love it:** distribution is essentially perfect, and the router is stateless — any client can compute the shard with no lookup, no network hop.

**THE FAILURE MODE — two of them, and both hurt.**

**(a) Range queries are destroyed.** Hashing deliberately destroys ordering. `order_1000` and `order_1001` land on different shards by design. So *"give me all orders from last Tuesday"* cannot be answered by visiting one shard — you must **scatter-gather**: ask every shard, wait for all of them, merge. With 4 shards that's tolerable. With 200 shards, one query becomes 200 queries and your p99 is now the slowest of 200 machines (see §9).

**(b) Adding a shard rehashes almost everything.** This is the killer.

```
BEFORE: N = 4.  key "user_8821" → hash % 4 = 3  → shard 3
AFTER:  N = 5.  key "user_8821" → hash % 5 = 0  → shard 0   ← MOVED

How many keys move when N goes 4 → 5?  About 1 - (1/5) = 80% of ALL keys.
For a 40 TB dataset that is 32 TB of data crossing the network,
while the system is live, and every cache is cold.
```

This is exactly the pain that **[74 — Consistent Hashing](./74-consistent-hashing.md)** exists to solve: with consistent hashing, adding a 5th node moves only ~1/5 of the keys instead of 80%. **Never write `hash(key) % nodeCount` in a system you intend to grow.** (§8 gives the other fix: fixed logical partitions.)

---

### 5. Strategy 3 — Directory / lookup-based partitioning

Stop computing the shard. **Look it up.** A directory service holds an explicit `key → shard` map.

```
  Client
    │  1. "Where does tenant 'acme-corp' live?"
    ▼
┌───────────────────────────┐
│   Directory Service       │      lookup table
│   (replicated + cached)   │   ┌──────────────────────┐
│                           │──▶│ acme-corp   → shard-7│ ← whale: own shard
│   2. "shard-7"            │   │ tiny-startup→ shard-1│
└───────────┬───────────────┘   │ bobs-bakery → shard-1│
            │                   │ globex      → shard-2│
            │ 3. query          └──────────────────────┘
            ▼
    ┌──────────────┐
    │   shard-7    │
    └──────────────┘
```

```javascript
class ShardDirectory {
  constructor({ store, cache }) {
    this.store = store;   // durable map (e.g. its own replicated Postgres / etcd)
    this.cache = cache;   // Redis, plus an in-process LRU in front of that
    this.local = new Map();
  }

  async lookup(tenantId) {
    // Three tiers: process memory → Redis → durable store. The directory is on the
    // hot path of EVERY request, so a store round-trip per request is unacceptable.
    if (this.local.has(tenantId)) return this.local.get(tenantId);

    const cached = await this.cache.get(`shard:${tenantId}`);
    if (cached) { this.local.set(tenantId, cached); return cached; }

    const shard = await this.store.getShard(tenantId);
    if (!shard) throw new Error(`Unknown tenant ${tenantId}`);
    await this.cache.set(`shard:${tenantId}`, shard, { EX: 300 });
    this.local.set(tenantId, shard);
    return shard;
  }

  // The whole reason this strategy exists: you can MOVE a tenant at will.
  async migrateTenant(tenantId, toShard) {
    await this.store.setMigrating(tenantId, toShard); // writes dual-written from here
    await copyData(tenantId, await this.store.getShard(tenantId), toShard);
    await this.store.setShard(tenantId, toShard);
    await this.cache.del(`shard:${tenantId}`);
    this.local.delete(tenantId);
    // No rehashing. No data movement for any other tenant. Just this one.
  }
}
```

**Why people love it:** maximum flexibility. Your biggest customer generates 30% of load? Give them `shard-7` alone. A tenant in Germany needs EU residency? Point them at the Frankfurt shard. Nothing else moves.

**THE FAILURE MODE:** the directory is on the critical path of every single request. It is:
- **an extra network hop** (adds ~1–5 ms to *every* query — brutal if your DB query itself is 3 ms),
- **a bottleneck** (it must serve QPS equal to your entire system's QPS),
- **a single point of failure** (directory down = whole system down, even though every shard is healthy).

The fix is mandatory, not optional: **replicate it** (multiple read replicas, no single node), **cache it aggressively** (it changes rarely — TTLs of minutes are fine, and clients can hold the whole map in memory if it's small), and **make it fail-static** (if the directory is unreachable, clients keep using their last-known map rather than erroring).

---

### 6. Strategy 4 — Geographic partitioning

Partition by region. `region` is the partition key.

```
    ┌──────────────┐        ┌──────────────┐        ┌──────────────┐
    │  us-east-1   │        │  eu-west-1   │        │ ap-south-1   │
    │  US users    │        │  EU users    │        │  APAC users  │
    │  ~15 ms RTT  │        │  ~12 ms RTT  │        │  ~20 ms RTT  │
    └──────────────┘        └──────────────┘        └──────────────┘
         ▲                        ▲                       ▲
      US user                  EU user                 India user
                       (cross-region RTT: 80–250 ms)
```

**Why people do it:** two hard reasons, not one.
- **Latency.** A user in Mumbai hitting a Virginia database eats ~200 ms round-trip *per query*. Ten sequential queries = 2 seconds of pure network. Local partition → ~20 ms.
- **Law.** GDPR (and India's DPDP, and China's PIPL) push you toward keeping citizens' personal data physically inside the region. This isn't an optimisation — it's a compliance requirement, and it *forces* geographic partitioning whether or not you wanted it.

**THE FAILURE MODE:**
- **Cross-region queries.** "Show me global signups this week" now scatter-gathers across continents, and the slowest region sets your latency. Worse: "match this EU rider with a US driver" is a query that spans a boundary you're legally not allowed to cross freely.
- **Users who travel.** A German user flies to Tokyo. Their data is in Frankfurt. Every request now pays 250 ms. Do you migrate them? (Then GDPR complains.) Do you cache in Tokyo? (Then you're replicating EU personal data outside the EU.) There is no clean answer — you make a policy choice, e.g. "home region is fixed at signup; travelling users read from a nearby read-replica for non-personal data only."
- **Uneven regions.** US might be 60% of your traffic and APAC 5%. Geographic partitions are *never* balanced.

---

### 7. Composite / hierarchical keys — what real systems actually do

Here is the punchline of the whole topic. You don't have to choose between "even distribution" and "efficient range scans." **Use two keys.**

This is the Cassandra / DynamoDB model, and it is the design most real systems land on:

| Part | Cassandra name | DynamoDB name | What it decides |
|---|---|---|---|
| First part | **Partition key** | **Partition key (HASH key)** | **WHICH node** the row lives on (it gets hashed) |
| Second part | **Clustering key** | **Sort key (RANGE key)** | **The ORDER of rows within that node** |

```
PRIMARY KEY ((user_id), timestamp)
              ▲          ▲
              │          └── sort key: rows are stored physically sorted by
              │              timestamp INSIDE the partition → range scan is a
              │              single sequential read
              └── partition key: hashed → even spread across nodes
```

Worked example — a chat app, "get this user's last 20 messages":

```
   hash(user_42) → node B
                     │
                     ▼
┌──────────────────────────────────────────────────────────┐
│ NODE B                                                    │
│  partition: user_42                                       │
│  ┌────────────────────────────────────────────────────┐  │
│  │ ts=...09:01  msg  "hey"                             │  │
│  │ ts=...09:04  msg  "you around?"                     │  │  rows physically
│  │ ts=...09:30  msg  "cool"                            │  │  sorted on disk
│  │ ...                                                 │  │  by timestamp
│  │ ts=...17:58  msg  "see you"        ◀── newest       │  │
│  └────────────────────────────────────────────────────┘  │
│  partition: user_77   (a different, unrelated partition)  │
└──────────────────────────────────────────────────────────┘

Query: last 20 messages of user_42
  → hash(user_42) → node B          (ONE node — no scatter-gather)
  → seek to end of partition        (ONE seek — rows already sorted)
  → read 20 rows sequentially       (ONE sequential read)
Total: one node, one seek. This is as good as it gets in a distributed store.
```

```javascript
// Cassandra: DESC order means "newest first" is a read from the start of the partition.
// CREATE TABLE messages (
//   user_id     uuid,
//   ts          timeuuid,
//   sender_id   uuid,
//   body        text,
//   PRIMARY KEY ((user_id), ts)          -- (partition key), clustering key
// ) WITH CLUSTERING ORDER BY (ts DESC);

// DynamoDB equivalent — same idea, different vocabulary.
import { DynamoDBClient, QueryCommand } from '@aws-sdk/client-dynamodb';
const ddb = new DynamoDBClient({});

async function lastMessages(userId, limit = 20) {
  const out = await ddb.send(new QueryCommand({
    TableName: 'messages',
    // Note: we constrain the PARTITION key with '=', never with a range.
    // A Query MUST pin the partition key — that is what keeps it single-node.
    KeyConditionExpression: 'user_id = :u',
    ExpressionAttributeValues: { ':u': { S: userId } },
    ScanIndexForward: false, // walk the sort key backwards → newest first
    Limit: limit,
  }));
  return out.Items;
}
```

**The rule this teaches you:** *hash the thing that gives you spread, sort by the thing that gives you order.* Every efficient query in such a system pins the partition key and ranges over the sort key.

**And the trap:** if a query does *not* know the partition key, it degenerates into a full scatter-gather (a DynamoDB `Scan`) and everything you gained is gone. Which brings us to the most important decision in the doc.

---

### 8. Choosing a partition key — the most consequential, most irreversible decision

Three criteria. A key must pass **all three**.

**1. High cardinality** — enough distinct values to spread across all your nodes, now and in five years.
- `country` has ~200 values → caps you at 200 partitions, and 2 of them (US, India) hold half the data.
- `is_premium` has 2 values → catastrophic, you have 2 partitions forever.
- `user_id` has 100 million values → fine.

**2. Even access distribution** — not just *even data*, **even traffic**. A key can spread rows perfectly and still hotspot on reads. `celebrity_id` spreads celebrities evenly across nodes, but one of them has 200 million followers.

**3. It must be present in your most common queries.** This is the one people forget and it's the most punishing. If your hottest query is "get messages for a conversation" but you partitioned by `message_id`, then *every single read* becomes a scatter-gather across all shards. Your partition key must be the field your top queries already filter on.

**Worked examples:**

| System | Good key | Why | Bad key | Why it fails |
|---|---|---|---|---|
| **Chat app** | `conversation_id` (partition) + `timestamp` (sort) | Every read is "messages in this conversation, newest first" — pinned to one partition, sorted | `message_id` | High cardinality, perfectly even… and *useless*: no query ever asks for a single random message. Every read scatter-gathers. |
| **Chat app** | — | — | `timestamp` | All of today's writes hit one partition. Classic range hotspot. |
| **E-commerce (orders)** | `customer_id` (partition) + `order_date` (sort) | "My orders" and "my orders in date range" both hit one partition | `order_status` | Cardinality 5 (`pending`/`paid`/`shipped`/…). Also every order *changes* status → the row would have to move partitions. Never partition on a mutable field. |
| **E-commerce (orders)** | — | — | `product_id` | Fine for a product catalogue; disastrous for orders — a viral product makes one partition carry the whole Black Friday. |
| **Metrics / time-series** | `(metric_name, host_id, hour_bucket)` (partition) + `timestamp` (sort) | Bucketing the time into the *partition* key stops one partition growing forever, while `host_id` gives spread. Queries are "CPU for host-42, last hour" → one partition. | `timestamp` alone | Every metric in the fleet writing to one partition. The textbook hotspot. |
| **Metrics** | — | — | `metric_name` alone | `cpu.percent` is emitted by every host on earth. One giant partition, unbounded growth. |

**The immutability rule:** a partition key must never change for a given row. If it changes, the row has to physically move to another machine — which is a delete plus an insert plus a window where it exists twice or not at all. `order_status`, `user_tier`, `current_city` — all terrible partition keys for exactly this reason.

---

### 9. The hotspot / celebrity problem, and the three real fixes

You partitioned by `user_id`. Distribution is perfect. Then a celebrity with 300 million followers posts, and one partition takes 500k reads/sec while its neighbours idle. Hashing does not save you: **a single key cannot be split by a hash function, because a hash always sends the same key to the same place.**

```
                       ┌──────────────────────────────────┐
   500k req/s ────────▶│ Partition 7   (celebrity_9)      │  ◀── MELTING
                       └──────────────────────────────────┘
      120 req/s ──────▶│ Partition 8   (normal users)     │  ◀── bored
                       └──────────────────────────────────┘
```

**Fix 1 — Key salting (split the write hotspot).** Append a random suffix so one logical key becomes N physical keys.

```javascript
const SALT_BUCKETS = 16; // one hot key becomes 16 keys → 16 partitions share the load

// WRITE: pick a random bucket. Writes now spread across 16 partitions.
async function recordLike(celebrityId, userId) {
  const bucket = Math.floor(Math.random() * SALT_BUCKETS);
  const partitionKey = `${celebrityId}#${bucket}`;   // e.g. "celeb_9#11"
  await db.insert({ pk: partitionKey, sk: `${Date.now()}#${userId}`, userId });
}

// READ: you must now gather from ALL buckets and merge. This is the price.
async function recentLikes(celebrityId, limit = 50) {
  const buckets = Array.from({ length: SALT_BUCKETS }, (_, b) => `${celebrityId}#${b}`);
  const results = await Promise.all(
    buckets.map(pk => db.query({ pk, sortDesc: true, limit }))
  );
  return results.flat()
    .sort((a, b) => b.sk.localeCompare(a.sk)) // merge N sorted streams
    .slice(0, limit);
}
```
Salting trades **write scalability for read cost** (1 read → 16 reads). So **only salt the keys that are actually hot** — keep a small "hot key" set (updated by a streaming counter) and salt only those. Salting every key would make every read a 16-way fan-out for no reason.

**Fix 2 — Cache the hot key in front of the store.** Celebrity data is read-heavy and write-light. A Redis entry with a 10-second TTL absorbs 500k reads/sec and lets ~0.1 reads/sec reach the partition. For *read* hotspots this is the cheapest and best fix, and it's usually the right first move. (It does nothing for *write* hotspots.)

**Fix 3 — Give the whale a dedicated partition.** This is directory-based partitioning (§5) earning its keep: put `celebrity_9` — or your single enterprise customer generating 30% of load — on their own shard, with their own hardware. No salting, no fan-out, no noisy-neighbour effect. This is how B2B SaaS handles its biggest accounts.

---

### 10. Rebalancing — adding and removing nodes without a bad night

**The anti-pattern: `hash(key) % N` where N = number of nodes.** Covered in §4. Going from 4 → 5 nodes moves ~80% of your data. Do not do this.

**Approach A — Fixed number of logical partitions (the neatest answer).** Create far more partitions than nodes *up front* — say **1024** — and assign many partitions to each node. A partition is the unit of movement; it never splits or merges.

```
1024 logical partitions, 4 nodes → 256 partitions each

Node A: [p0   … p255 ]
Node B: [p256 … p511 ]
Node C: [p512 … p767 ]
Node D: [p768 … p1023]

Add Node E → target is 1024/5 ≈ 205 partitions each.
Move ~51 partitions from each existing node to E. That's it.
Only ~20% of data moves (the theoretical minimum), and the KEY→PARTITION
mapping NEVER changes — only the PARTITION→NODE mapping does.
```

```javascript
const NUM_PARTITIONS = 1024; // chosen once, at design time. Never changes.

// Key → partition: a pure function. Stable forever.
function partitionFor(key) {
  return hash32(key) % NUM_PARTITIONS;
}

// Partition → node: a small mutable map (kept in ZooKeeper/etcd, or gossiped).
// THIS is what changes when you add a node. It's ~1024 entries — tiny, cacheable.
let partitionToNode = { 0: 'node-a', 1: 'node-a', /* ... */ 1023: 'node-d' };

function nodeFor(key) {
  return partitionToNode[partitionFor(key)];
}
```
This is what **Riak** does (it calls them vnodes) and what **Elasticsearch** does (you fix `number_of_shards` at index creation and can never change it — now you know why that setting is immutable). Pick N high enough that you'll never outgrow it (partitions ≥ 10× your max expected nodes), but not so high that per-partition overhead hurts.

**Approach B — Dynamic splitting.** Start with one partition. When it exceeds a size threshold (HBase: ~10 GB; DynamoDB: ~10 GB **or** 3000 RCU / 1000 WCU of throughput), **split it in half** and hand one half to another node. When it shrinks, merge.

```
p1 [A ──────────────── Z]  grows to 10 GB
                │  split at the midpoint of the actual key distribution
                ▼
p1 [A ─── M]   p2 [N ─── Z]      → p2 moves to another node
```
This adapts to *actual* data distribution rather than assumed distribution, which is what makes range partitioning survivable. The cost: splits are operational events — they cause latency spikes and, in the worst case, a "split storm" while a fresh table warms up. (DynamoDB's adaptive capacity also does this for *throughput* hotspots, not just size.)

**Approach C — Consistent hashing.** The classic. See **[74 — Consistent Hashing](./74-consistent-hashing.md)**. Adding the Nth node moves only ~1/N of keys. Note that in practice, consistent hashing with virtual nodes and "fixed logical partitions" converge on the same idea: *decouple the key's identity from the node's identity by putting a stable intermediate layer between them.*

**Automatic vs manual rebalancing:** never make rebalancing fully automatic. A node that's merely *slow* can be mistaken for a dead node; the system rebalances away from it; the extra load makes the *next* node slow; it cascades. Rebalancing should be **suggested by the system, approved by a human.**

---

### 11. Scatter-gather, and why your p99 is the slowest partition

A **scatter-gather** query is one you can't route to a single partition, so you send it to *all* of them and merge the results.

```
                    ┌──▶ P1  ──▶ 8 ms  ┐
   Query ──▶ Router ├──▶ P2  ──▶ 11 ms ├──▶ merge ──▶ respond
   (no partition    ├──▶ P3  ──▶ 9 ms  │
    key available)  ├──▶ P4  ──▶ 7 ms  │
                    └──▶ P5  ──▶ 240ms ┘  ◀── GC pause. You wait 240 ms.
                                            Total latency = MAX, not average.
```

```javascript
async function scatterGather(shards, queryFn, { timeoutMs = 100 } = {}) {
  const withTimeout = (p) => Promise.race([
    p,
    new Promise((_, rej) => setTimeout(() => rej(new Error('shard timeout')), timeoutMs)),
  ]);

  // allSettled, not all: one slow/dead shard must not fail the whole query.
  const settled = await Promise.allSettled(shards.map(s => withTimeout(queryFn(s))));
  const ok = settled.filter(r => r.status === 'fulfilled').map(r => r.value);

  // Degrade honestly: tell the caller the answer is partial rather than lying.
  return { rows: ok.flat(), complete: ok.length === shards.length, reached: ok.length };
}
```

**The maths that should scare you.** Suppose each partition independently has a 1% chance of being slow (>100 ms). With 1 partition, 99% of queries are fast. With 100 partitions, the chance that *all* are fast is `0.99^100 ≈ 36.6%` — so **63% of your scatter-gather queries are slow**, even though every single machine is behaving 99% well. Fanning out converts a rare per-node event into a common per-query event. This is **tail latency amplification** — see **[06 — Latency, Throughput and Performance](./06-latency-throughput-performance.md)**.

**Mitigations:** hedged requests (after p95, re-send to a replica and take whichever answers first), per-shard timeouts with partial results, and — best of all — **design your partition key so your hot queries never need to scatter at all.**

---

## Visual / Diagram description

### Diagram 1: The four strategies, side by side

```
  ┌─────────────────────────── SAME 12 KEYS ──────────────────────────┐
  │  a1  a2  b1  c1  m4  m9  s1  s2  s3  s7  t1  z9                   │
  └───────────────────────────────────────────────────────────────────┘

  RANGE (by first letter)              HASH (hash % 3)
  ┌───────┐┌───────┐┌───────┐          ┌───────┐┌───────┐┌───────┐
  │ a–f   ││ g–r   ││ s–z   │          │  h0   ││  h1   ││  h2   │
  │a1 a2  ││ m4 m9 ││s1 s2  │          │a1 m9  ││a2 c1  ││b1 m4  │
  │b1 c1  ││       ││s3 s7  │          │s2 t1  ││s1 s7  ││s3 z9  │
  │       ││       ││t1 z9  │          │       ││       ││       │
  └───────┘└───────┘└───────┘          └───────┘└───────┘└───────┘
    4        2        6  ◀ SKEWED        4        4        4  ◀ EVEN
    ✓ range scans cheap                  ✗ range scans = scatter-gather

  DIRECTORY                            GEOGRAPHIC
  ┌──────────────┐                     ┌───────┐┌───────┐┌───────┐
  │ a1→N1  s1→N3 │  explicit map       │  US   ││  EU   ││ APAC  │
  │ a2→N1  s2→N3 │  (s7 is a whale:    │a1 a2  ││m4 m9  ││ z9    │
  │ b1→N2  s7→N4 │   own node!)        │b1 c1  ││s1 s2  ││       │
  └──────┬───────┘                     │s3 s7  ││t1     ││       │
    ┌────▼───┬────────┬────────┐       └───────┘└───────┘└───────┘
    │  N1    │  N2    │ N3  N4 │        ✓ low latency, GDPR-safe
    └────────┴────────┴────────┘        ✗ never balanced; travellers hurt
```

### Diagram 2: The composite-key request path (the one to memorise)

```
  Client: "last 20 messages in conversation c_88"
     │
     │  1. Extract the PARTITION KEY from the query.
     │     (If you can't, you're about to scatter-gather. Redesign.)
     ▼
  ┌────────────────────────────────────────────┐
  │  Router (stateless — just a hash function) │
  │     partition = hash("c_88") % 1024 = 391  │
  └───────────────────┬────────────────────────┘
                      │  2. partition → node (small cached map, ~1024 entries)
                      │     391 → node-C
                      ▼
  ┌──────────────────────────────────────────────────────────┐
  │  node-C                                                   │
  │  ┌────────── partition 391 ─────────────────────────────┐│
  │  │ pk=c_88 │ sk=2026-07-12T09:01 │ "hey"                ││
  │  │ pk=c_88 │ sk=2026-07-12T09:04 │ "you around?"        ││  sorted
  │  │ pk=c_88 │ sk=2026-07-12T17:58 │ "see you"   ◀ newest ││  by sk
  │  └──────────────────────────────────────────────────────┘│
  │  3. Seek to end of pk=c_88, read 20 rows backwards.       │
  └──────────────────────────────────────────────────────────┘
                      │
                      ▼   ONE node. ONE seek. ~2 ms.
                   Response
```

The whole art of partitioning is making your top 3 queries look like *this* diagram, and not like the scatter-gather diagram in §11.

---

## Real world examples

### 1. Discord — moving to Cassandra/ScyllaDB with a composite key

Discord stores billions of messages. Their published design partitions messages by a composite key of **`(channel_id, bucket)` as the partition key** and **`message_id` as the clustering key**, where `bucket` is a fixed window of time (roughly 10 days).

Why every part of that key exists:
- `channel_id` alone would be a valid partition key — every read is "messages in this channel" — **but** a huge, years-old channel would grow into an unbounded partition, which Cassandra hates (large partitions cause compaction pain and slow reads).
- Adding `bucket` **bounds the partition size**. A channel's messages are split into 10-day partitions, so no single partition grows forever.
- `message_id` as the clustering key (their IDs are Snowflake-style, so they sort by time) makes "give me the last N messages" a single sequential read.

This is §7 and §8 in one design: hash for spread, bucket for bounded size, sort for range scans.

### 2. DynamoDB — partition key + sort key, and adaptive splitting

DynamoDB makes the composite model the *only* model: every item has a partition key (hashed to choose a physical partition) and optionally a sort key (which orders items inside it). A `Query` **must** specify the partition key with `=`; if you don't have it, your only option is `Scan`, which reads the whole table. The API is designed to make bad partitioning *visibly* expensive.

DynamoDB also does dynamic splitting (§10): a partition splits when it exceeds ~10 GB or its throughput ceiling, and "adaptive capacity" gives extra throughput to hot partitions. But adaptive capacity cannot rescue a single hot *key* — AWS's own guidance is to salt (§9) if one key is hot.

### 3. Elasticsearch — fixed logical partitions, immutable by design

An Elasticsearch index is split into a fixed `number_of_shards`, chosen at index creation and **immutable** thereafter. A document routes to a shard via `hash(routing_value) % number_of_shards`, where the routing value defaults to the document `_id`. Shards are then assigned to nodes, and the *shard→node* assignment moves freely as you add or remove nodes.

That is exactly Approach A from §10 — and the reason `number_of_shards` can't be changed is now obvious: changing N would change `hash % N` for every document in the index. If you need to change it, you must reindex into a new index. Elasticsearch also lets you set a custom routing value (e.g. `?routing=customer_id`), which turns a scatter-gather query across all shards into a single-shard query — a direct application of §8's "put the partition key in your hot query."

---

## Trade-offs

| Strategy | Distribution | Range queries | Rebalancing cost | Extra hop? | Best for |
|---|---|---|---|---|---|
| **Vertical** | n/a | unaffected | trivial | no | Fat blob columns; separating a noisy feature. Do this first. |
| **Range** | Poor (hotspots) | **Excellent** | Cheap (move a range) | no | Time-series with active splitting; sorted key spaces |
| **Hash** | **Excellent** | Terrible (scatter) | Brutal with `% N` | no | Point lookups by a high-cardinality key |
| **Directory** | Fully controllable | Depends | **Trivial (move one tenant)** | **yes (~1–5 ms)** | Multi-tenant B2B with whale customers |
| **Geographic** | Poor (uneven regions) | Local: good. Global: terrible | Hard (data residency) | no | Data-residency law; global latency |
| **Composite (hash+sort)** | **Excellent** | **Excellent within a partition** | Depends on the hash scheme | no | **Almost everything** — this is the default answer |

| Decision | You gain | You give up |
|---|---|---|
| Partition at all | Scale, write throughput, blast-radius isolation | Cross-partition JOINs, cross-partition transactions, global secondary indexes get expensive, global unique constraints, operational simplicity |
| Choose hash over range | Even load, no hotspots | Ordered scans; every "between X and Y" becomes fan-out |
| Choose range over hash | Cheap ordered scans | You now own the hotspot problem and must actively split hot ranges |
| Choose a directory | Move any tenant anywhere; dedicate shards to whales | An extra hop on every request; a component that must never go down |
| More partitions (1024 vs 8) | Smooth rebalancing forever | Per-partition memory/file-handle overhead; wider fan-out on scatter queries |
| Salt a hot key | Write hotspot disappears | Every read of that key becomes an N-way fan-out and merge |

**Rule of thumb:** Do vertical partitioning first — it's a week of work and buys a year. When you finally must go horizontal, **default to a composite key: hash on the entity your hottest query already filters by, sort on time.** Reach for a directory only when you have whale tenants you need to isolate. And if a partition key is ever mutable, it is not a partition key.

---

## Common interview questions on this topic

### Q1: "You're designing a chat app. What's your partition key, and why?"
**Hint:** `conversation_id` as the **partition key**, `timestamp` (or a time-ordered message ID) as the **sort key**. Justify it against all three criteria out loud: cardinality is high (millions of conversations); access is roughly even (no single conversation dominates — and if one did, you'd salt it); and crucially, **the hottest query — "load this conversation's recent messages" — already contains `conversation_id`**, so it's a single-partition, single-seek read. Then pre-empt the follow-up: a huge, years-old channel grows an unbounded partition, so bound it with a time bucket in the partition key: `((conversation_id, month), timestamp)`.

### Q2: "Why not just partition by `timestamp`? Time-series data is naturally ordered."
**Hint:** Because time only moves forward, so **100% of writes land on the newest partition** while every other node idles. You bought N machines and are using one. The fix is to put a high-cardinality field *first* in the partition key (`(host_id, hour_bucket)`) and use time only as the **sort** key. Range partitioning by time is fine for *cold, immutable, historical* data you only read — it is fatal for the write path.

### Q3: "You have `shard = hash(user_id) % 8` and you need to add a 9th shard. What happens?"
**Hint:** About **8/9 ≈ 89% of all keys change shard** — a full data migration, live, with every cache cold. Two correct fixes: (a) **consistent hashing** ([74](./74-consistent-hashing.md)) — adding the 9th node moves only ~1/9 of keys; (b) **fixed logical partitions** — hash into 1024 stable logical partitions and rebalance the *partition→node* map instead. Mention that (b) is what Elasticsearch and Riak do and that it's easier to reason about. Bonus point: say out loud that `% N` where N is the node count is a bug, not a design.

### Q4: "One customer is 30% of your load and keeps taking down the shard they share. Fix it."
**Hint:** Three tools, and name the trade-off of each. **Cache** in front (fixes read hotspots, does nothing for writes, and is the cheapest thing to try). **Salt** the key (`cust#0..15`) — fixes write hotspots, but every read of that customer becomes a 16-way fan-out plus merge. **Dedicate a shard** via a directory/lookup layer — cleanest for a known whale, and it's why B2B SaaS uses directory partitioning. Emphasise you'd salt *only* the identified hot keys, not everything.

### Q5: "What's the difference between partitioning and replication? Do you need both?"
**Hint:** **Partitioning splits *different* data across nodes (for scale). Replication copies the *same* data to multiple nodes (for availability and read throughput).** They're orthogonal and every serious system does both: each partition is itself replicated 3×. Partitioning without replication means one dead node = permanent data loss for that slice. See [63 — Database Replication](./63-database-replication.md).

---

## Practice exercise

### Design the partitioning for a food-delivery app

You're the architect for a food-delivery service. Scale: **20M users, 500k restaurants, 5M orders/day** (peaking at ~400 orders/sec at 8pm), retained for 3 years.

Three entities need partitioning:
- `orders` — top queries: (1) "my last 20 orders" (customer app), (2) "today's orders for restaurant X" (restaurant tablet, polled every 5s), (3) "all orders in city Y in the last hour" (ops dashboard).
- `restaurants` — top query: "restaurants near me, open now."
- `driver_locations` — 50k drivers each writing a GPS ping **every 4 seconds**.

**Produce, on one page:**

1. For **each** of the three entities: a **partition key** and a **sort key**, written as `((partition_key), sort_key)`. Justify each against the three criteria (cardinality, even access, present in the hot query).
2. Work out the **write rate for `driver_locations`** (show the arithmetic: 50k drivers ÷ 4 s = ? writes/sec). Then state how many partitions you need if one node handles 10k writes/sec, and what your partition key must satisfy to actually spread that load.
3. Note that **query (1) and query (2) on `orders` want different partition keys.** They conflict — you cannot satisfy both with one table. Write down your resolution and its cost. (Hint: the answer is not "compromise." Consider a second table, or a global secondary index, and be honest about the write amplification and the eventual-consistency window it introduces.)
4. Identify **one hotspot** that will definitely occur in this system (there's an obvious one on Friday nights) and pick a fix from §9, naming what it costs you.
5. State which query is a **scatter-gather** and estimate what it does to your p99 if you have 64 partitions and each has a 1% chance of a >100 ms hiccup.

**Deliverable:** a one-page doc with three `((pk), sk)` declarations, the arithmetic for #2 and #5, and three sentences on the conflict in #3.

---

## Quick reference cheat sheet

- **Partition** = one slice of your data on one machine. **Sharding** = partitioning applied to a database.
- **Three reasons to partition:** data won't fit, writes won't fit, blast-radius isolation. Name all three in an interview.
- **Vertical partitioning first** — split fat blob columns off, split a noisy feature onto its own DB. A week of work, buys a year.
- **Horizontal partitioning** = split rows. Four strategies: **range, hash, directory, geographic**.
- **Range** → great range scans, **fatal hotspots** (partition by timestamp = 100% of writes on one node).
- **Hash** → perfect distribution, **kills range queries**, and `% N` rehashes ~everything when N changes.
- **Directory** → maximum flexibility (move any tenant, dedicate a shard to a whale), but it's an extra hop, a bottleneck, and a SPOF — replicate and cache it, always.
- **Geographic** → latency + GDPR/data-residency. Breaks on cross-region queries and travelling users.
- **Composite key = the real answer:** **partition key** (hashed → chooses the NODE) + **sort key** (orders rows WITHIN the node). Even spread *and* cheap range scans.
- **Partition key criteria:** high cardinality + even *access* (not just even data) + **present in your hottest query**. All three, or you scatter-gather forever.
- **Never partition on a mutable field** (`order_status`, `city`) — the row would have to physically move.
- **Hotspot fixes:** cache the hot key (reads only) → salt it `key#0..N` (fixes writes, costs an N-way read fan-out) → dedicate a shard (best for known whales).
- **Rebalancing:** use **fixed logical partitions** (1024 up front, move partitions not keys — Riak/Elasticsearch) or **dynamic splitting** (HBase/DynamoDB) or **consistent hashing**. Never `hash % nodeCount`.
- **Scatter-gather latency = the SLOWEST partition, not the average.** At 64 shards with a 1% slow rate, ~47% of queries are slow. Design the key so your hot queries never scatter.
- **Partitioning ≠ replication.** Partitioning splits different data (scale); replication copies the same data (availability). You need both.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [74 — Consistent Hashing](./74-consistent-hashing.md) — the algorithm that fixes hash partitioning's rebalancing disaster |
| **Next** | [86 — Data Replication Strategies](./86-data-replication-strategies.md) — the other half of the story: partitioning splits data, replication copies it |
| **Related** | [64 — Database Sharding](./64-database-sharding.md) — partitioning applied specifically to databases, with the operational mechanics of running shards |
| **Related** | [61 — Databases: SQL vs NoSQL](./61-databases-sql-vs-nosql.md) — why NoSQL stores force you to declare a partition key up front, and SQL lets you pretend you don't have one |
| **Related** | [63 — Database Replication](./63-database-replication.md) — each partition is itself replicated 3×; here's how |
| **Related** | [102 — HLD: Design Uber](./102-hld-uber.md) — geographic partitioning and driver-location write volume, in a real design |
