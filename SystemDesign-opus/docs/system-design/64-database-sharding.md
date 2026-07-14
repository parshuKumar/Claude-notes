# 64 — Database Sharding — Horizontal Partitioning
## Category: HLD Components

---

## What is this?

**Sharding** means splitting one logical database into many physical databases, and putting a *different subset of the rows* in each one. Users A–M live on machine 1, users N–Z live on machine 2. Together they hold all your data; separately, no single machine holds all of it.

Think of a **library that outgrew its building**. You can't just buy taller shelves forever — the floor will collapse. So you open a second building, and split the books: A–M in the north branch, N–Z in the south branch. Every book still exists exactly once. But now, "give me every book by an author whose surname starts with a vowel" means visiting *both* buildings and merging the results. And the card catalogue at the entrance has to tell every visitor which building to walk to. That routing problem, and that merging problem, are what sharding costs you.

---

## Why does it matter?

Say this in an interview and then say it again at work: **sharding is the LAST resort.**

Before you shard, you exhaust — in this order:

1. **Indexing** ([62](./62-database-indexing.md)) — a missing index is a 100× win, for free.
2. **Caching** ([59](./59-caching-in-depth.md)) — Redis absorbs 90% of reads for a few hundred dollars a month.
3. **Read replicas** ([63](./63-database-replication.md)) — scales *reads* linearly, with no data-model change.
4. **Vertical scaling** ([56](./56-horizontal-vs-vertical-scaling.md)) — a modern single box reaches 128 vCPU / 4 TB RAM / 60 TB NVMe. That is *enormous*. Stack Overflow famously ran on a handful of big SQL Servers for years.
5. **Vertical partitioning** — move one hot table or feature to its own database. Often enough on its own.

Only when **one machine can no longer hold the data, or absorb the write throughput**, do you shard. Because sharding permanently takes away: cross-shard `JOIN`s; cross-shard transactions (ACID across two machines); `AUTO_INCREMENT` primary keys; globally unique secondary indexes; `SELECT COUNT(*) FROM users` being a one-liner; and **your ability to change your mind** — the shard key is the most irreversible decision you will make.

**Interview angle:** "We're at 50 TB and 200k writes/sec" is the setup that *requires* sharding. Jumping to it at "10k users" is the fastest way to look junior — interviewers deliberately test whether you reach for it too early.

**Work angle:** Resharding a live 20 TB production database is a multi-quarter project. Picking the wrong shard key in week 2 costs you a year in year 3.

---

## The core idea — explained simply

### The Post Office Sorting Analogy

One post office serves an entire country. Every letter arrives at one building, gets sorted by one team, and goes out. It works — until the volume grows and the building physically cannot hold the mail.

So you open **10 regional sorting centres** and make **one rule: the postcode decides which centre handles the letter.** Postcode 1xxxxx → Centre 1, 2xxxxx → Centre 2, and so on.

Everything good follows from that one rule: each centre handles 1/10th of the volume (**the load is split**) and stores 1/10th of the mail (**the data is split**), and a truck arriving anywhere can compute the destination from the postcode alone — no lookup, no central bottleneck (**routing is cheap**).

And everything painful also follows from that same rule:
- **"How many letters are in the country right now?"** — ask all 10 centres and add up. (**Scatter-gather.**)
- **"Find the letter from Aunt Meera"** — you don't know her postcode, so ask all 10. (**No global secondary index.**)
- **A festival in region 3** floods Centre 3 while the rest idle. (**Hot shard.**)
- **Opening an 11th centre** means re-drawing every boundary and physically moving mail. (**Resharding.**)
- **Moving a letter from Centre 2 to Centre 7 atomically** — both centres must agree, and there's no single manager anymore. (**Distributed transaction.**)

| Analogy piece | Technical concept |
|---|---|
| Regional sorting centre | **Shard** (a physical database instance) |
| The postcode | **Shard key** / partition key |
| Rule mapping postcode → centre | **Sharding strategy** (range / hash / directory / geo) |
| The dispatcher who reads the postcode | **Shard router** (in the app, a proxy, or the DB itself) |
| Asking all 10 centres and merging | **Scatter-gather query** |
| Festival flooding Centre 3 | **Hotspot / celebrity problem** |
| Opening an 11th centre | **Resharding / rebalancing** |
| Two centres must both agree | **Distributed transaction (2PC / SAGA)** |

**The single sentence to remember:** *sharding turns one hard problem (a machine that's too small) into five hard problems (routing, joins, transactions, IDs, and rebalancing).* Take that deal only when you must.

---

## Key concepts inside this topic

### 1. Vertical partitioning vs horizontal sharding — different things

Beginners mix these up constantly. They split *different axes*.

**Vertical partitioning** splits by **column or by table** — by *feature*. All rows stay together; you move *fields* out.

```
BEFORE — one wide table, one database
┌──────────────────────────────────────────────────────────────┐
│ users:  id | email | pw_hash | name | bio | avatar_blob |prefs│
└──────────────────────────────────────────────────────────────┘
   ▲ every read of `email` drags the 2 MB avatar_blob's page around

AFTER — VERTICAL partitioning (split COLUMNS; different schema per machine)
┌──────────────────────┐ ┌────────────────────┐ ┌──────────────────┐
│ users_auth   (DB A)  │ │ users_profile (DB B)│ │ users_media (S3) │
│ id | email | pw_hash │ │ id | name | bio     │ │ id → avatar URL  │
│ HOT, tiny, high QPS  │ │ WARM                │ │ COLD, huge       │
└──────────────────────┘ └────────────────────┘ └──────────────────┘

AFTER — HORIZONTAL sharding (split ROWS; IDENTICAL schema per machine)
┌──────────────────────┐ ┌──────────────────────┐ ┌──────────────────────┐
│ SHARD 0              │ │ SHARD 1              │ │ SHARD 2              │
│ users  (id 1..999)   │ │ users  (1000..1999)  │ │ users  (2000..2999)  │
│ orders for those ids │ │ orders for those ids │ │ orders for those ids │
└──────────────────────┘ └──────────────────────┘ └──────────────────────┘
```

Vertical partitioning is what "split the monolith DB per microservice" really is. It gives you a **bounded, constant-factor** win — maybe 3–5 databases, one per feature. It does **not** scale with your user count: when `users_auth` alone is 8 TB, it has nothing left to give.

Horizontal sharding scales **linearly and indefinitely** — 10 shards, 100, 1000. That is why it is both the last resort *and* the only real answer at extreme scale.

| | Vertical partitioning | Horizontal sharding |
|---|---|---|
| Splits | Columns / tables | Rows |
| Schema on each machine | **Different** | **Identical** |
| Scales with | Number of features (bounded) | Number of rows (unbounded) |
| Cross-machine JOIN | Lost (between partitions) | Lost (between shards) |
| Complexity | Moderate | High |
| Do it | Early and often | Only when forced |

### 2. Range-based sharding

**Rule:** shard = the range the key falls into.

```
users.signup_date < 2023-01-01           → SHARD 0
2023-01-01 ≤ signup_date < 2024-01-01    → SHARD 1
2024-01-01 ≤ signup_date < 2025-01-01    → SHARD 2
signup_date ≥ 2025-01-01                 → SHARD 3
```

**Worked example — and why this exact one is a trap.** You shard `users` by `signup_date`. On launch day it looks perfect: four shards, roughly even data. Then look at the *write* pattern:

```
Every NEW user who signs up today has signup_date = today
   → every signup INSERT, every welcome-email update, every onboarding write → SHARD 3
   → and new users are your MOST ACTIVE users, so their reads hit SHARD 3 too

RESULT:  Shard 0: 2% CPU  |  Shard 1: 3%  |  Shard 2: 5%  |  Shard 3: 97% 🔥
```

You bought four machines and you are still bottlenecked on one. This is the **hot shard** problem, and **monotonically increasing shard keys (timestamps, auto-increment IDs) cause it by construction.** HBase and Bigtable users hit exactly this and call it "region hotspotting."

**What range sharding is genuinely good at:** range queries stay on one shard. `SELECT * FROM users WHERE signup_date BETWEEN '2024-03-01' AND '2024-03-31'` is answered by **one machine** — fast, no fan-out.

**Use range sharding when** you need range scans and your key is *not* monotonically increasing on the write path — or for time-series data where you genuinely *want* recent data co-located, and you accept the hot write shard (mitigating it by giving the current window many sub-shards).

### 3. Hash-based sharding

**Rule:** `shard = hash(key) % N`.

```
shard = crc32("user_8842") % 4 = 2   → SHARD 2
shard = crc32("user_8843") % 4 = 0   → SHARD 0
```

**Worked example.** Shard `users` by `hash(user_id) % 4`. A good hash function scatters *adjacent* IDs to *different* shards, so a burst of new signups spreads evenly. Load is flat across all four machines. This is the **default choice** and the one to name first in an interview.

**But you lose range queries.** `WHERE signup_date BETWEEN x AND y` no longer maps to one shard — the matching users are hashed all over the place. Every such query becomes **scatter-gather**: send it to all 4 shards, wait for the slowest, merge in the app. Your latency becomes the **max** of 4 shards, not the average — with 100 shards and a p99 of 50 ms each, your p99 is dominated by the *tail* (the "tail at scale" problem).

**And the killer: `% N` means adding a shard rehashes almost everything.**

```
hash(k) = 8844:   8844 % 4 = 0 → SHARD 0
                  8844 % 5 = 4 → SHARD 4   ✗ MOVED

Going from N to N+1 relocates roughly  1 - 1/(N+1)  of ALL keys.
4 → 5 shards moves ~80% of your data. On 20 TB that is 16 TB of live migration.
```

**The fix is [74 — Consistent Hashing](./74-consistent-hashing.md)**, which puts shards on a hash *ring* so adding a shard only moves ~1/N of the keys (~20% for 4→5) instead of ~80%. Say "consistent hashing" the moment an interviewer asks "what happens when you add a shard?" — that's the whole point of topic 74.

**A cheaper industrial trick — virtual shards / logical buckets.** Hash into a large *fixed* number of logical buckets (say 1024), then map buckets → physical machines in a small table. Adding a machine moves *buckets*; the hash function never changes, so **you never rehash**. This is how Vitess, Citus, and Elasticsearch do it.

```
hash(key) % 1024 = bucket 731  →  bucket_map[731] = "shard-db-03"
Adding a 5th machine? Reassign ~205 of the 1024 buckets to it. Nothing rehashes.
```

### 4. Directory / lookup-based sharding

**Rule:** a **lookup service** stores an explicit `key → shard` mapping. No formula at all — `user_1→S0`, `user_2→S3`, `taylor→S7` (her own dedicated shard).

**Worked example.** You're Slack, sharding by **workspace**, and workspaces are wildly unequal: 5 people vs 200,000 at IBM. A hash would drop IBM randomly onto some shard and melt it. A directory lets you say *"IBM's workspace lives on shard 12, which has no other tenants."*

**Pros:** maximum flexibility. Rebalance a single key. Isolate a whale. Migrate one tenant with zero effect on anyone else.

**Cons — and this is the whole exam question:** the directory is **a single point of failure and a bottleneck.** Every single query must consult it first.

**Mitigations (say all four):**
- **Cache it aggressively** in every app server's memory — the mapping changes rarely (minutes/hours), so a 60-second TTL is fine.
- **Replicate it** — it's tiny; run it as a 3–5 node replicated store (etcd, ZooKeeper, a small replicated Postgres).
- **Version it** — bump a version on change; a client holding a stale map gets rejected and refetches (a fencing token, same idea as [84 — Distributed Locking](./84-distributed-locking.md)).
- **Keep it coarse** — map *buckets* → shards (1024 entries), not *keys* → shards (1 billion entries). The whole directory becomes 20 KB and fits in every process's RAM.

### 5. Geo-based sharding

**Rule:** shard = the user's region. EU users → `eu-west-1`; US users → `us-east-1`; India → `ap-south-1`. Two motives, one technical and one legal:

- **Latency.** An Indian user's writes go 20 ms to Mumbai instead of 250 ms to Virginia.
- **Data residency / compliance.** GDPR, India's DPDP Act, and China's PIPL can *require* that citizen data physically stays in-country. Here geo-sharding isn't an optimization — it's the only legal architecture.

**The catch:** a user who relocates must be migrated between shards; any cross-region query is scatter-gather across oceans; and you've made every shard hot/cold by timezone — the EU shard idles at 3am while the US shard peaks.

### 6. Choosing the shard key — the most important, most irreversible decision

Everything else in this doc is recoverable. **This isn't.** Changing the shard key means rewriting every row into a new layout while serving live traffic.

**The four criteria, in priority order:**

1. **High cardinality.** Enough distinct values to spread across all shards, now and at 100× scale. `country` (195 values) is bad. `user_id` (billions) is good. `is_premium` (2 values) is a catastrophe.
2. **Even distribution.** Values must be roughly uniformly hot. `user_id` is even. `signup_date` is not (all writes to today's shard). `company_id` is not (IBM vs a 3-person startup).
3. **Query alignment — the one people forget.** Your *most common query* must be answerable from **one shard**. If you shard `orders` by `order_id` but 95% of queries are `WHERE user_id = ?`, then 95% of your queries are scatter-gather and sharding bought you *nothing*. **Shard by what you filter on.**
4. **Transaction alignment.** Things that must change atomically together must land on the *same* shard. Shard `orders` and `order_items` both by `user_id` and an order plus its items live together — a normal local transaction still works.

**The worked decision, for an e-commerce app:**

| Candidate key | Cardinality | Even? | Aligns with `WHERE user_id=?` (95% of queries)? | Verdict |
|---|---|---|---|---|
| `order_id` | Huge | Yes | **No** — scatter-gather every time | ✗ |
| `product_id` | High | **No** — one viral product melts a shard | No | ✗ |
| `country` | 195 | **No** — US is 40% of traffic | No | ✗ |
| `created_at` | High | **No** — all writes hit today's shard | No | ✗ |
| **`user_id`** | Huge | Yes | **Yes** — "my orders" hits one shard | ✓ **Winner** |

A **composite key** makes it even better: shard by `hash(user_id)`, and *within* the shard cluster by `(user_id, created_at DESC)` — so "my recent orders" is one shard, one index, one range scan.

**The interview line:** *"I'd shard by user_id, because our dominant access pattern is 'everything for one user', so that query stays single-shard. The cost is that an admin query like 'all orders yesterday across all users' becomes scatter-gather — I'd serve those from a separate analytics store, not from the OLTP shards."* That last sentence is what separates a good answer from a great one.

### 7. What breaks after sharding (the real price list)

**(a) Cross-shard JOINs — gone.**

```sql
-- Before sharding: trivially works. After sharding, ONLY works if `users` and
-- `orders` are sharded by the SAME key, so the rows sit on the same machine.
SELECT u.name, o.total FROM users u JOIN orders o ON o.user_id = u.id
WHERE o.created_at > NOW() - INTERVAL '1 day';
```
If they're sharded differently, the database simply *cannot* do it — the rows live on different machines. Your options:

- **Co-locate** (shard both by `user_id`) — the correct answer, and the reason criterion #4 exists.
- **Application-side join** — fetch orders, collect the `user_id`s, batch-fetch users, stitch in JS. Two round trips instead of one.
- **Denormalize** — store `user_name` *on the order row*. Duplicated data you must keep in sync. At scale, this is what everyone actually does.
- **Reference tables** — replicate small, rarely-changing tables (`countries`, `products`) to *every* shard so joins against them stay local.

**(b) Cross-shard transactions — need 2PC or SAGA.** Transfer ₹500 from Alice (shard 2) to Bob (shard 7): there is no single database that can `BEGIN…COMMIT` across both. Either **2PC** (a coordinator asks both shards to PREPARE, then COMMIT — correct, but it *blocks* if the coordinator dies mid-flight, and it throttles throughput) or **SAGA** (a chain of local transactions plus **compensating** transactions to undo — debit Alice; if crediting Bob fails, run "refund Alice"). SAGA trades atomicity for availability: you get eventual consistency and a visible intermediate state. Full treatment in [77 — Distributed Transactions](./77-distributed-transactions.md). **Best strategy: design so cross-shard transactions are rare — co-locate whatever must be atomic.**

**(c) Global unique IDs — `AUTO_INCREMENT` is dead.**

Each shard has its own counter, so shard 0 and shard 1 both mint `id = 1`. Three real answers:

| Approach | How | Pros | Cons |
|---|---|---|---|
| **UUID v4** | 128 random bits | Zero coordination, generate anywhere, offline-safe | 16 bytes (2× a bigint); **random ⇒ terrible B-tree locality** — every insert lands on a random page, destroying index cache hit rate and bloating the index. Not sortable. |
| **UUID v7 / ULID** | 48-bit timestamp + randomness | Time-sortable ⇒ good insert locality, still coordination-free | Still 16 bytes; leaks creation time |
| **Snowflake ID** (Twitter) | 64-bit: `41-bit ms timestamp \| 10-bit machine id \| 12-bit sequence` | 8 bytes, **k-sorted by time**, no coordination at generation time, ~4M IDs/sec/machine | Needs a unique machine ID per generator (from ZooKeeper/etcd/config); vulnerable to clock skew / NTP going backwards |
| **Ticket server** | One tiny DB whose only job is `AUTO_INCREMENT` (Flickr's approach; batch out 1000 IDs at a time) | Small, sequential, dense integers | A central dependency; needs its own HA. Batching hides the latency. |

**Snowflake is the standard answer.** Here it is, actually implemented:

```javascript
// snowflake.js — 64-bit, time-sortable, coordination-free IDs.
// Layout:  [ 41 bits: ms since epoch ][ 10 bits: machine id ][ 12 bits: sequence ]
// BigInt, because a JS Number only holds 53 bits exactly.
const EPOCH = 1735689600000n; // 2025-01-01 — a custom epoch buys ~69 years from 41 bits

export class SnowflakeGenerator {
  constructor(machineId) {
    if (machineId < 0 || machineId > 1023) throw new Error('machineId must be 0..1023');
    this.machineId = BigInt(machineId); // assigned by etcd/ZooKeeper at boot — MUST be unique
    this.sequence = 0n;
    this.lastMs = -1n;
  }

  nextId() {
    let now = BigInt(Date.now());

    // NTP corrected the clock backwards. We must NEVER mint a duplicate ID,
    // so refuse to generate until wall-clock catches up.
    if (now < this.lastMs) throw new Error('Clock moved backwards; refusing to generate ID');

    if (now === this.lastMs) {
      this.sequence = (this.sequence + 1n) & 0xfffn; // 4096 IDs per ms per machine
      if (this.sequence === 0n) {
        while (now <= this.lastMs) now = BigInt(Date.now()); // exhausted — spin to next ms
      }
    } else {
      this.sequence = 0n;
    }
    this.lastMs = now;

    return ((now - EPOCH) << 22n) | (this.machineId << 12n) | this.sequence;
  }
}
```

**(d) Global secondary indexes — gone.** You shard by `user_id`. Now run `SELECT * FROM users WHERE email = 'x@y.com'`. Email isn't the shard key, so **no shard knows where that row lives** — you must ask all of them. Fixes: a **secondary lookup table** sharded by `email` (`email → user_id`; two hops, but both single-shard — a hand-built global secondary index); a **search index** (push a copy into Elasticsearch, [79](./79-search-systems.md)); or **local secondary indexes** (an index *within* each shard — only helps once you already know the shard).

**(e) Scatter-gather.** Any query not keyed by the shard key fans out to every shard, and **latency becomes the MAX over shards, not the average** — with 16 shards each at a 40 ms p99, your p99 is the p99 of the *worst of 16 samples*, which is far worse than 40 ms. Also: `ORDER BY … LIMIT 20` across N shards means fetching **20 rows from every shard** and merging in the app — you cannot push the limit down, and `OFFSET 10000` is catastrophic. Use **cursor/keyset pagination** ([69](./69-api-design-rest-graphql-grpc.md)), never offset.

**(f) Rebalancing / resharding.** Adding shard #5 means moving data while serving live traffic. The safe playbook — never a big-bang cutover — is **DUAL-WRITE** (write to both old and new shard for affected keys) → **BACKFILL** (copy history in throttled background batches) → **VERIFY** (compare row counts and checksums) → **FLIP READS** (one key range at a time) → **CLEAN UP** (stop dual-writing, drop the old copy). And use **virtual buckets** (concept 3) so this only ever moves buckets — the hash function never changes.

### 8. The celebrity / hotspot problem

Hash sharding gives you even distribution **of keys**, not **of load**. One key can be catastrophically hot.

```
Shard by user_id. A celebrity with 400M followers is still ONE key.
    Shard 3:            2,000 QPS   ← the celebrity lives here 🔥
    Shards 0,1,2,4..15:    60 QPS each
You bought 16 machines and one of them is dying.
```

The same shape appears as: one viral product on Black Friday, one enormous Slack workspace, one hot Kafka partition key, one trending hashtag.

**Fixes, in the order you'd try them:**

1. **Cache the celebrity.** 99% of the load on a hot key is *reads of the same value*. Redis with a 5-second TTL and the shard sees 1 QPS instead of 2,000. **This solves most real hotspots and costs nothing — say it first.**
2. **Key salting / sub-sharding.** Split one logical key into K physical keys: `taylor#0 … taylor#9`. Writes pick a random suffix; reads scatter across all 10 and merge. One hot key becomes 10 warm ones. Cost: reads of that key are now a 10-way scatter-gather — so salt **only keys you've detected as hot**, never everything.
3. **Dedicated shard.** Give the whale its own machine — exactly what directory-based sharding buys you.
4. **A different code path.** Instagram/Twitter genuinely do this for feeds: normal users get **fan-out-on-write** (push the post into every follower's feed); celebrities get **fan-out-on-read** (don't push to 400M feeds — merge their posts in at read time). A different *algorithm* for hot keys is often the real production answer.

### 9. The shard router in Node.js

This is the code an interviewer wants to see: given a key, which connection do I use?

```javascript
// shard-router.js
import pg from 'pg';
import crypto from 'node:crypto';

const VIRTUAL_BUCKETS = 1024; // fixed forever. We remap BUCKETS, never rehash keys.

export class ShardRouter {
  /**
   * @param shards      [{ name, connectionString }]
   * @param bucketMap   Int array of length 1024: bucket -> index into `shards`.
   *                    Lives in etcd/Redis in production, cached here, refreshed on a TTL.
   * @param overrides   Map of key -> shard name. Directory-style escape hatch for whales.
   */
  constructor({ shards, bucketMap, overrides = new Map() }) {
    this.pools = shards.map((s) => ({
      name: s.name,
      pool: new pg.Pool({ connectionString: s.connectionString, max: 20 }),
    }));
    this.bucketMap = bucketMap;
    this.overrides = overrides;
  }

  /** Stable across processes and restarts — never use JS's built-in hashing here. */
  bucketFor(shardKey) {
    const digest = crypto.createHash('md5').update(String(shardKey)).digest();
    return digest.readUInt32BE(0) % VIRTUAL_BUCKETS;
  }

  poolFor(shardKey) {
    // 1. Directory override wins — this is how we isolate a celebrity/whale tenant.
    const pinned = this.overrides.get(String(shardKey));
    if (pinned) return this.pools.find((p) => p.name === pinned).pool;

    // 2. Otherwise: key -> bucket -> shard. Adding a machine only remaps buckets.
    return this.pools[this.bucketMap[this.bucketFor(shardKey)]].pool;
  }

  /** FAST path — single shard. 95%+ of your traffic must look like this. */
  async query(shardKey, sql, params) {
    return this.poolFor(shardKey).query(sql, params);
  }

  /** SLOW path — scatter-gather. Latency = the SLOWEST shard, not the average.
   *  Every call here is a smell: the query isn't aligned to the shard key. */
  async queryAll(sql, params, { mergeSort, limit } = {}) {
    const results = await Promise.all(this.pools.map((p) => p.pool.query(sql, params)));
    let rows = results.flatMap((r) => r.rows);
    if (mergeSort) rows.sort(mergeSort);   // ORDER BY must now happen in the APP
    return limit ? rows.slice(0, limit) : rows;
  }
}
```

```javascript
const router = new ShardRouter({
  shards: [0, 1, 2, 3].map((i) => ({ name: `shard-${i}`, connectionString: process.env[`SHARD_${i}`] })),
  bucketMap: loadBucketMapFromEtcd(),                  // 1024 ints, cached, 60s TTL
  overrides: new Map([['workspace_ibm', 'shard-3']]),  // whale gets a dedicated box
});
const idGen = new SnowflakeGenerator(Number(process.env.MACHINE_ID));

// FAST path: single-shard. Order + items are co-located by user_id, so a plain
// LOCAL transaction still works — no 2PC needed.
async function createOrder(userId, items, total) {
  const orderId = idGen.nextId();          // NOT auto-increment — that's dead after sharding
  const client = await router.poolFor(userId).connect();
  try {
    await client.query('BEGIN');
    await client.query(`INSERT INTO orders (id, user_id, total) VALUES ($1,$2,$3)`,
      [orderId.toString(), userId, total]);
    for (const it of items) {
      // user_id is carried on the ITEM row on purpose — it's what forces items onto
      // the same shard as their order. Denormalization in service of co-location.
      await client.query(
        `INSERT INTO order_items (id, order_id, user_id, sku, qty) VALUES ($1,$2,$3,$4,$5)`,
        [idGen.nextId().toString(), orderId.toString(), userId, it.sku, it.qty]);
    }
    await client.query('COMMIT');
    return orderId;
  } catch (e) {
    await client.query('ROLLBACK');
    throw e;
  } finally {
    client.release();
  }
}

// FAST: the dominant query. One shard. This is WHY we chose user_id as the shard key.
await router.query(userId,
  `SELECT * FROM orders WHERE user_id = $1 ORDER BY created_at DESC LIMIT 20`, [userId]);

// SLOW: not keyed by user_id → hits every shard. This belongs in an analytics warehouse.
await router.queryAll(
  `SELECT * FROM orders WHERE created_at > NOW() - INTERVAL '1 day' ORDER BY created_at DESC LIMIT 20`,
  [], { mergeSort: (a, b) => b.created_at - a.created_at, limit: 20 });
```

---

## Visual / Diagram description

### Diagram 1 — The sharded architecture, end to end

```
                     ┌────────────────────────────┐
                     │      Node.js API tier      │
                     │  1. shardKey = userId      │ ← from the JWT
                     │  2. bucket   = h(key)%1024 │
                     │  3. shard    = bucketMap[b]│
                     └─────────────┬──────────────┘
        ┌──────────────────┐       │       ┌──────────────────┐
        │ DIRECTORY /      │       │       │ ID GENERATOR     │
        │ BUCKET MAP       │◀──────┼──────▶│ Snowflake        │
        │ etcd, 20 KB,     │  SHARD ROUTER │ machineId = 7    │
        │ 3 replicas,      │  (in-process) │ (auto-increment  │
        │ cached 60s       │       │       │  is dead)        │
        └──────────────────┘       │       └──────────────────┘
              ┌──────────┬─────────┴────┬──────────────┐
              ▼          ▼              ▼              ▼
       ┌───────────┐┌───────────┐┌───────────┐┌───────────┐
       │  SHARD 0  ││  SHARD 1  ││  SHARD 2  ││  SHARD 3  │
       │ buckets   ││ buckets   ││ buckets   ││ buckets   │
       │ 0..255    ││ 256..511  ││ 512..767  ││ 768..1023 │
       │           ││           ││           ││ + ibm_ws  │◀ directory override
       │ users     ││ users     ││ users     ││ users     │   (the whale)
       │ orders    ││ orders    ││ orders    ││ orders    │ ← IDENTICAL schema
       │ order_items││order_items││order_items││order_items│   on every shard
       └─────┬─────┘└─────┬─────┘└─────┬─────┘└─────┬─────┘
          leader+      leader+      leader+      leader+
        2 replicas   2 replicas   2 replicas   2 replicas  ← topic 63: each shard
                                                             is ITSELF replicated
```

**What it shows.** The router is *in your application process* — it is not a database feature. Three things feed it: the **bucket map** (a tiny replicated directory, so adding a machine never rehashes), the **shard key** (from the request), and the **ID generator** (because auto-increment is gone). Every shard has the *identical schema* and is *itself* a replicated leader/follower group — **sharding and replication compose**: sharding splits the write load, replication protects each split and scales its reads.

### Diagram 2 — The four strategies, side by side

```
 RANGE                         HASH                       DIRECTORY
 ─────────────────────────     ──────────────────────     ──────────────────────
 key: signup_date              key: hash(user_id)%4       key: looked up
 ┌────┬────┬────┬────┐         ┌────┬────┬────┬────┐      ┌──────────────────┐
 │ S0 │ S1 │ S2 │ S3 │         │ S0 │ S1 │ S2 │ S3 │      │ u1→S0   u2→S3    │
 │2022│2023│2024│2025│         │ 25%│ 25%│ 25%│ 25%│      │ u3→S0   ibm→S7   │
 │ 2% │ 3% │ 5% │97% │         │ 25%│ 25%│ 25%│ 25%│      └────────┬─────────┘
 └────┴────┴────┴────┘         └────┴────┴────┴────┘        every query hits it
   ▲ ALL new writes hit S3       ▲ EVEN load                       │ first
                                                                   ▼
 ✓ range query = 1 shard       ✓ even distribution         ┌────┬────┬────┬────┐
 ✗ HOT SHARD on time keys      ✗ range query = scatter     │ S0 │ S1 │ .. │ S7 │
                               ✗ N→N+1 rehashes ~80%       └────┴────┴────┴────┘
                                 → CONSISTENT HASHING (74)  ✓ move ANY key anytime
                                   or virtual buckets       ✓ isolate a whale
                                                            ✗ directory = SPOF
                                                              → cache + replicate

 THE HOTSPOT — happens under EVERY strategy:
 ┌───────────────────────────────────────────────────────────────────┐
 │ hash(taylor_swift) → always the SAME shard → 2,000 QPS on one box │
 │  FIX 1: cache the key (Redis, 5s TTL)      ← always try this first│
 │  FIX 2: salt it → taylor#0 … taylor#9      ← 1 hot key → 10 warm  │
 │  FIX 3: dedicated shard (via the directory)                       │
 │  FIX 4: a different code path (fan-out-on-READ for celebrities)   │
 └───────────────────────────────────────────────────────────────────┘
```

---

## Real world examples

### 1. Instagram — sharding Postgres by user, with logical shards

Instagram's engineering blog describes sharding PostgreSQL with the **shard ID embedded in the primary key itself** — a 64-bit Snowflake variant laid out as `[41 bits: ms timestamp][13 bits: logical shard id][10 bits: per-shard sequence]`. Two things worth stealing: the ID is **time-sortable** (good B-tree insert locality, and feeds sort naturally), and **you can compute the shard from the ID alone** — no directory lookup to route a request that already has an object ID. They also ran **thousands of logical shards mapped onto far fewer physical machines** — exactly the virtual-bucket trick — so growing the cluster meant moving logical shards, never rehashing.

### 2. Vitess / YouTube — sharding MySQL without rewriting the app

YouTube's MySQL outgrew one machine, and rather than teach every application to shard, they built **Vitess** (now a CNCF project, and the engine behind Slack's and Square's MySQL). Vitess is a **proxy that speaks the MySQL protocol**: your app issues normal SQL, and Vitess parses it, finds the shard key (its "VIndex"), routes to the right shard, and where it must, scatter-gathers and merges results itself. It also automates resharding with exactly the dual-write → backfill → verify → flip playbook. The lesson: at big enough scale, sharding gets pushed *below* the application, into a routing layer.

### 3. Discord — sharding by channel, and the hot-partition problem

Discord's public posts describe storing messages partitioned by `(channel_id, bucket)` — the shard key is the **channel**, because the dominant query is "give me the recent messages in this channel." That is criterion #3 (query alignment) applied perfectly. They also hit a textbook **hotspot**: a handful of enormous, extremely active channels created hot partitions that dominated the cluster, addressed with time-bucketing of the partition key and heavy caching in front.

### 4. Slack — the whale tenant problem

Slack shards by **workspace/team**, the natural query boundary (nearly every query is scoped to one workspace). But workspace sizes span five orders of magnitude — a 4-person startup and a 200,000-person enterprise. A pure hash would randomly drop a whale onto a shared machine. This is precisely where **directory-based mapping** earns its keep: a whale can be assigned its own dedicated infrastructure without changing anything for anyone else.

---

## Trade-offs

| Strategy | Even load | Range queries | Adding a shard | Flexibility | Best for |
|---|---|---|---|---|---|
| **Range** | ✗ (hot shard on time-ordered keys) | ✓ single shard | Split a range | Low | Time-series, natural bounded ranges |
| **Hash** | ✓ | ✗ scatter-gather | ✗ rehashes ~80% with `% N` — use consistent hashing / virtual buckets | Low | The default. Even, key-value access |
| **Directory** | ✓ (you control it) | ✗ | ✓ move any key | **Highest** | Multi-tenant with unequal tenants; whale isolation |
| **Geo** | ✗ (timezone peaks) | Within a region | Add a region | Medium | Latency + data-residency law |

| Decision | You gain | You give up |
|---|---|---|
| Shard at all | Unbounded write + storage scaling | JOINs, ACID across shards, auto-increment, global indexes, cheap `COUNT(*)`, operational simplicity |
| Denormalize to co-locate | Single-shard transactions and joins | Duplicated data you must keep in sync |
| More, smaller shards | More headroom, smaller blast radius each | More connections, worse scatter-gather tail latency, more machines to operate |
| Directory-based routing | Move any key at any time; isolate whales | A lookup on the critical path — must be cached and replicated |
| UUID v4 primary keys | Zero coordination | Random inserts destroy B-tree locality; 2× the bytes |
| Snowflake IDs | 8 bytes, time-sortable, coordination-free | Needs a unique machine ID; breaks if the clock rewinds |

**Rule of thumb:** **Don't shard.** Add an index, add a cache, add read replicas, buy a bigger box, and split the DB by feature — in that order. If you have genuinely exhausted all of those and one machine still can't hold the data or take the write load, then: shard **horizontally**, **hash the shard key into ~1024 virtual buckets** (so you never rehash), pick the key that makes your **single most common query hit exactly one shard**, generate **Snowflake IDs**, **co-locate anything that must be transactional**, keep a **directory override** for whales, **cache your celebrities**, and push every scatter-gather/analytics query out to a separate store. And write the shard key on a whiteboard, because you are stuck with it.

---

## Common interview questions on this topic

### Q1: "Our database is slow. Should we shard?"

**Hint:** Almost certainly not — and the interviewer is testing whether you jump. Walk the ladder out loud: *profile the slow queries → add the missing index → add a cache → add read replicas → vertically scale (a single box does 128 vCPU / 4 TB RAM / 60 TB NVMe) → vertically partition by feature.* Only if the working set exceeds one machine or **writes** (which replicas cannot help) exceed one machine do you shard. Then name the price: no cross-shard joins, no cross-shard ACID, no auto-increment, no global secondary indexes.

### Q2: "You shard `orders` by `order_id`. What's wrong with that?"

**Hint:** The dominant query is `WHERE user_id = ?` ("show me my orders"), and `order_id` doesn't align with it — so *every* user-facing read becomes a scatter-gather over all N shards. Shard by **`user_id`** so a user's entire history is on one shard; that also co-locates `orders` with `order_items`, so placing an order stays a single **local** transaction instead of needing 2PC. Rule: **shard by what you filter on.**

### Q3: "You're going from 4 shards to 5. What happens?"

**Hint:** With naive `hash(key) % N`, roughly `1 − 1/(N+1)` ≈ **80%** of all keys move — on 20 TB that's 16 TB of live migration. Two fixes: **consistent hashing** ([74](./74-consistent-hashing.md)), where a hash ring means only ~1/N of keys move; or **virtual buckets** — hash into a fixed 1024 buckets and keep a small `bucket → shard` map, so adding a machine reassigns ~205 buckets and the hash function never changes. Then describe the safe migration: **dual-write → backfill → verify → flip reads → drop the old copy.**

### Q4: "Your shard 3 is at 95% CPU and the others are at 20%. What do you do?"

**Hint:** First **diagnose the shape**: is it (a) a *hot key* (one celebrity/whale), (b) a *hot range* (you sharded on a timestamp so all new writes land on the newest shard), or (c) *uneven data* (one giant tenant)? For a hot key: **cache it** (Redis, short TTL — this alone usually fixes it), then **salt** it into `key#0..key#9`, then give it a **dedicated shard** via a directory override, and consider a **different code path** entirely (fan-out-on-read for celebrities). For a hot range: your shard key is time-ordered — that's a design bug, move to a hashed key.

### Q5: "After sharding, how do you generate primary keys?"

**Hint:** `AUTO_INCREMENT` is dead — every shard would mint `id=1`. Options: **UUID v4** (coordination-free, but 16 random bytes destroy B-tree insert locality and bloat the index), **UUID v7/ULID** (time-prefixed, so locality is restored), **Snowflake** (64-bit: `timestamp | machineId | sequence` — 8 bytes, time-sortable, no coordination at generation; needs a unique machine ID and breaks if the clock rewinds), or a **ticket server** (a dedicated tiny DB handing out ID blocks — Flickr's approach). Say **Snowflake** as the default, and mention that Instagram embeds the **shard ID inside the primary key** so you can route from the ID alone.

---

## Practice exercise

### Shard "Chatly" — pick the key, then break it

Chatly is a Slack clone: 500,000 workspaces, 40M users, **8 billion messages**, 30 TB of message data, 120,000 message writes/sec at peak. One PostgreSQL box cannot do this. The dominant queries: **Q_A (90% of traffic)** the last 50 messages in a channel; **Q_B (8%)** all channels a user belongs to; **Q_C (2%)** full-text search across a workspace; **Q_D (rare, admin)** total message count platform-wide.

**Produce these six things (~40 min):**

1. **Show the arithmetic** for why one machine is impossible: 30 TB ÷ (the largest single-instance storage you'd actually trust) = minimum shards? 120,000 writes/sec ÷ (a realistic per-shard write ceiling, say 8,000/sec) = minimum shards? Take the max. Show every step.
2. **Choose the shard key.** Consider `message_id`, `user_id`, `channel_id`, `workspace_id`. Build the 4-criteria table from concept 6 and defend your winner. There is a real tension between Q_A and Q_B — name it, and say which query you optimize for and which you sacrifice.
3. **Say what breaks.** For each of Q_A–Q_D: single-shard, or scatter-gather? For every scatter-gather, propose where it should *actually* be served from (hint: Q_C and Q_D do not belong on your OLTP shards).
4. **Draw the diagram** — router, bucket map/directory, N shards, and the replicas behind each shard. Box characters, whiteboard-ready.
5. **Handle the whale.** A 200,000-person enterprise workspace generates 15% of all traffic by itself. Which hotspot fixes do you apply, in what order, and what changes in your routing code?
6. **Write the resharding plan** for 8 shards → 12, in five numbered steps, naming the specific safety check you'd run before flipping reads.

---

## Quick reference cheat sheet

- **Sharding is the LAST resort.** Index → cache → read replicas → vertical scale → vertical partition → *then* shard. Reaching for it early is the classic junior tell.
- **Vertical partitioning splits COLUMNS/tables** (by feature; bounded gain). **Horizontal sharding splits ROWS** (unbounded gain). Never confuse them.
- **Replication scales reads. Sharding scales WRITES and storage.** Different problems — and you almost always end up doing both (each shard is itself replicated).
- **The shard key is the most important and most irreversible decision.** Four criteria: high cardinality, even distribution, **aligns with your #1 query**, and co-locates whatever must be transactional.
- **Shard by what you filter on.** If 95% of queries are `WHERE user_id=?`, shard by `user_id` — otherwise 95% of queries become scatter-gather and sharding bought you nothing.
- **Range sharding:** great range queries, but a time-ordered key ⇒ **all new writes hit one shard** (hot shard by construction).
- **Hash sharding:** even load, but range queries scatter, and `hash % N` **rehashes ~80% of keys when you add a shard** → use **consistent hashing** ([74](./74-consistent-hashing.md)) or ~**1024 virtual buckets**.
- **Directory sharding:** total flexibility (move any key, isolate any whale), but the directory is a bottleneck/SPOF → cache, replicate, version, and keep it coarse (buckets, not keys). **Geo sharding:** for latency *and* for data-residency law — sometimes the only legal option.
- **What dies after sharding:** cross-shard JOINs, cross-shard ACID (→ **2PC or SAGA**, [77](./77-distributed-transactions.md)), `AUTO_INCREMENT`, global secondary indexes, cheap `COUNT(*)`, `OFFSET` pagination.
- **Global IDs: Snowflake** (`ts | machineId | seq` — 8 bytes, time-sortable, coordination-free) is the default. UUIDv4 needs no coordination but its randomness wrecks B-tree insert locality. Ticket servers work but centralize.
- **Denormalize to co-locate.** Carrying `user_id` on `order_items` is what keeps an order + its items on one shard — and keeps `createOrder` a single local transaction instead of a 2PC.
- **Scatter-gather latency = the SLOWEST shard, not the average**, and you can't push `LIMIT`/`OFFSET` down. Use keyset pagination; push analytics to a warehouse.
- **Hotspots: cache the celebrity first** (a 5-second Redis TTL fixes most of them), then salt the key (`taylor#0..9`), then give it a dedicated shard, then use a different code path (fan-out-on-read).
- **Resharding is never a big bang:** dual-write → backfill → verify → flip reads → drop old.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [63 — Database Replication](./63-database-replication.md) — replication scales reads; when *writes* are the wall, you shard. Each shard is itself replicated. |
| **Next** | [65 — Database Connection Pooling](./65-database-connection-pooling.md) — N shards × M app servers × a pool each: connection count explodes after sharding |
| **Related** | [74 — Consistent Hashing](./74-consistent-hashing.md) — the fix for "adding a shard rehashes everything" |
| **Related** | [75 — Data Partitioning Strategies](./75-data-partitioning.md) — the same range/hash/directory/geo ideas generalized beyond databases |
| **Related** | [77 — Distributed Transactions](./77-distributed-transactions.md) — 2PC and SAGA, for the cross-shard writes you couldn't avoid |
