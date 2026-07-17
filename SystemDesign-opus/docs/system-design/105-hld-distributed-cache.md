# 105 — Design a Distributed Cache (Redis)
## Category: HLD Case Study

---

## What is this?

A distributed cache is a **giant, extremely fast, shared scratchpad** that lives in the RAM of many machines at once. Your application asks it "do you already have the answer for key `X`?" — if yes, you get it back in a fraction of a millisecond and you never bother the slow database. If no, you compute the answer the hard way, hand a copy to the cache, and the next person gets it for free.

That's Redis. That's Memcached. That's the thing sitting between almost every serious web application and its database.

Think of it as the **short-term memory of your whole system.** A database is long-term memory: durable, organized, slow to reach, the source of truth. A cache is what you keep "on the tip of your tongue" — the handful of facts you've used recently, held somewhere you can grab them instantly, and which you're allowed to *forget* at any moment without disaster.

That last clause is the entire personality of this system, so say it out loud in the interview: **a cache is allowed to lose data.** It is not the source of truth. If a cache node dies and takes 64 GB of cached values with it, nothing is *lost* — the real data still sits safely in the database, and the cache simply repopulates itself on the next few requests. This single permission — "you may forget" — relaxes the design enormously compared to designing a database, where losing a committed write is a catastrophe. Everything hard about databases (durability, strict consistency, never losing an acknowledged write) we are allowed to *trade away* for the one thing we care about: **speed.**

---

## Why does it matter?

This is one of the highest-signal system design interview questions, because it forces you to build the very component that every *other* case study casually says "and then we add a cache" about. In the URL shortener, the chat system, the news feed — the cache is a magic box. Here you have to open the box.

The interviewer gets to watch you handle three genuinely hard, genuinely famous problems in one sitting:

- **Eviction** — when RAM fills up, which key do you throw away? (LRU with O(1) operations is a top-5 interview coding question, and it lives right here.)
- **Cache invalidation** — Phil Karlton's famous line, "there are only two hard things in Computer Science: cache invalidation and naming things." When the database changes, your cached copy is now a *lie*. How do you stop serving lies?
- **The thundering herd** — a single hot key expires and ten thousand requests stampede your database in the same millisecond. How do you not fall over?

**At work**, this matters even more than it looks. Almost every latency win in a large system, and a huge fraction of the cost savings, come from caching. A cache with a 95% hit rate means your database sees **1/20th** of the traffic — that is the difference between one database and forty. But that same fact hides a landmine: teams start treating the cache as "just an optimization," and then one day the cache fails, the full unfiltered load lands on a database that was quietly sized for 5% of it, and the whole site goes down. Understanding a distributed cache deeply is understanding both the biggest lever and one of the sharpest edges in production systems.

---

## The core idea — explained simply

### The Restaurant Kitchen Analogy

Imagine a busy restaurant kitchen.

- The **walk-in freezer** in the back is your **database.** It holds everything, it never runs out, and it's the truth about what food exists. But walking to the back, opening the heavy door, and finding the right box takes time — maybe 20 seconds each trip. On a busy night you cannot make that trip for every single order.
- So next to the cooking line there's a small **counter of prepped ingredients** — the chopped onions, the mise en place, the sauces used constantly. That's your **cache.** It holds only a tiny fraction of everything in the freezer, but it holds *the right* tiny fraction: the stuff you actually reach for. Grabbing from the counter takes half a second, not twenty.
- The counter is **small.** When you prep a new ingredient and there's no room, you have to clear something off to make space. Which one do you toss? Probably the thing you haven't touched in the longest time — the wilted garnish nobody's ordered all night. That decision is **eviction (LRU).**
- Some things **spoil.** The cut herbs are only good for a couple hours. After that, even if there's room on the counter, you can't trust them. That's **TTL / expiration.**
- Here's the dangerous part: suppose the freezer gets a **new, better recipe** for the tomato sauce, but the old sauce is still sitting prepped on your counter. You'll happily serve the *old* sauce to customers, fast and confidently — and *wrong.* That's a **stale cache**, and fixing it is **cache invalidation**, the genuinely hard problem.
- And one more: it's a Saturday, the counter runs out of the one sauce every dish needs, and **all twelve cooks sprint to the freezer at the exact same moment,** jamming the door and stopping the whole kitchen. That's the **thundering herd.**

One kitchen's counter can only hold so much. When your menu is enormous — terabytes of hot data — no single counter is big enough, so you build **many kitchens** and agree on a rule for which kitchen preps which ingredient. That rule is **consistent hashing**, and it's how one cache becomes a *distributed* cache.

Here's the mapping:

| Restaurant kitchen | Distributed cache |
|---|---|
| Walk-in freezer (the truth, slow) | The database / source of truth |
| Prep counter (small, instant) | The cache (RAM) |
| Grabbing from the counter | `get(key)` — a cache hit |
| Walking to the freezer | A cache miss → hit the database |
| Clearing the oldest item to make room | Eviction (LRU / LFU) |
| Herbs that spoil after 2 hours | TTL expiration |
| Serving old sauce after the recipe changed | Stale read → cache invalidation |
| All cooks rushing the freezer at once | Thundering herd / cache stampede |
| Many kitchens, each owning some ingredients | Partitioning via consistent hashing |
| A backup cook who shadows the head cook | Primary/replica replication |

The rest of this doc works through that analogy with real numbers, real code, and the two topologies Redis actually ships.

---

## Key concepts inside this topic

We execute the **5-step HLD framework** visibly and in order. Do it this way in every interview:

```
  STEP 1  Requirements      →  what are we even building? (5 min)
  STEP 2  Estimation        →  how big is the working set? (5 min)
  STEP 3  Single-node       →  the hash map, eviction, expiration (10 min)
  STEP 4  Distributed       →  partitioning + consistent hashing (10 min)
  STEP 5  Replication +      →  availability, invalidation, herds (15 min)
          deep dives
```

The shape here is a little different from a normal case study: because a cache is a piece of *infrastructure*, the framework bends toward "solve the single node perfectly, then multiply it." So we nail the single-node design first, then go distributed.

---

### 1. Requirements (Step 1)

**Clarifying questions to ask out loud.** "Is this a read-through cache in front of a database, or a standalone key-value store? Do we need persistence, or is pure in-memory fine? What's the eviction policy — is bounded staleness acceptable? Single data center or multi-region?" These questions decide the whole shape.

**Functional requirements — the API is tiny, and that's the point:**

- `get(key)` → returns the value, or a miss.
- `set(key, value, ttl)` → stores the value, optionally with a time-to-live.
- `delete(key)` → removes a key (this is the workhorse of cache invalidation).
- **Eviction** when memory is full — the cache must never OOM; it must instead throw out old data to make room.
- **Expiry (TTL)** — a key set with a TTL must disappear (become a miss) after that time.

**Non-functional requirements — this is where the real design pressure comes from:**

- **Extremely low latency — sub-millisecond.** This is *the whole point.* A cache exists only to be faster than the database. If a cache `get` isn't dramatically faster than the query it replaces, the cache is worse than useless — it's an extra network hop for nothing. Sub-millisecond p99 is the bar. Everything in the single-node design (in-memory hash map, no disk on the hot path, single-threaded event loop to avoid locks) serves this one number.
- **High throughput** — 100,000+ operations per second *per node.* Redis routinely does this; a small cluster does millions.
- **Scale beyond one machine's RAM** — the hot data is bigger than any single box, so we must shard across many nodes.
- **High availability** — if a node dies, the system keeps serving (from a replica), and the loss is a small, tolerable dip in hit rate, not an outage.

**The crucial framing — state it explicitly, because it changes everything:** a cache is **allowed to lose data.** It is not the source of truth. This one relaxation is why we can:

- use **asynchronous** replication (and accept a small data-loss window),
- **evict** data on purpose,
- skip strong consistency and disk durability on the hot path,
- treat a whole node dying as "the hit rate dropped for a minute," not "we lost customer orders."

Designing a database, none of those are allowed. Designing a cache, all of them are on the table. Notice how much lighter the problem gets the moment you're permitted to forget.

---

### 2. Capacity estimation (Step 2)

The number that drives everything is the **working-set size** — not the total data, the *hot* data. Recall the **80/20 rule** from [59 — Caching in Depth](./59-caching-in-depth.md): a small fraction of keys serve the vast majority of requests. You cache *that* fraction, not the whole database.

**Worked example.** Suppose the database holds **10 TB** of data, but the hot 20% — the part that's actually requested — is about **2 TB.** That 2 TB is what has to live in RAM.

```
  Working set to cache        ≈  2 TB
  RAM per node (usable)       ≈  64 GB   (leave headroom; Redis overhead + fragmentation)
  Nodes needed for data       =  2 TB / 64 GB  =  2000 / 64  ≈  32 nodes

  Add replication (1 replica per primary, RF=2):
  Total nodes                 =  32 primaries + 32 replicas   =  64 nodes
```

So the working-set size *directly* gives you the node count. That's the single most important number in the whole design — write it on the board.

**Throughput sizing.** Say the system does **1,000,000 cache ops/sec** at peak.

```
  Peak ops/sec                ≈  1,000,000
  Per-node capacity           ≈  100,000 ops/sec   (comfortable for Redis)
  Nodes needed for throughput =  1,000,000 / 100,000  =  10 nodes
```

Notice throughput needs only ~10 nodes but *data* needs ~32 — so here we're **memory-bound, not CPU-bound.** That's typical for a cache and worth saying: you buy nodes to hold the working set, and you get the throughput for free. If instead you were throughput-bound (small dataset, insane QPS), you'd add nodes and/or read replicas for CPU.

**A note on hit rate.** If this cache achieves a 95% hit rate, the database behind it sees only 5% of 1,000,000 = **50,000 ops/sec** instead of a million. That 20x reduction is the entire economic justification for the 64 nodes.

---

### 3. Single-node design (Step 3)

Get one node perfect, then multiply. A single cache node is, at its heart, an **in-memory hash map**: key → value, giving **O(1)** average `get` and `set`. No disk, no B-tree, no query planner — just a hash table in RAM. That's where the sub-millisecond latency comes from.

But a hash map alone isn't a cache. Two hard problems appear the moment memory is finite and data can go stale.

#### 3a. Eviction — what do you throw out when RAM is full?

When the node hits its memory limit, a `set` must first make room. The choice of *what* to evict is the eviction policy (recall [59](./59-caching-in-depth.md)):

| Policy | Evict… | Good when | Weakness |
|---|---|---|---|
| **LRU** (Least Recently Used) | the key untouched longest | recency predicts reuse (usually true) | one big scan can flush hot keys |
| **LFU** (Least Frequently Used) | the key used fewest times | popularity is stable over time | slow to adapt; a once-hot key lingers |
| **FIFO** | the oldest inserted key | simple; rarely the best choice | ignores actual usage |
| **Random** | a random key | dead cheap, no bookkeeping | can evict a hot key by bad luck |
| **TTL-based** | the key nearest expiry | data has natural freshness windows | needs TTLs set |

**LRU is the default answer, and its O(1) implementation is the classic interview question.** The trick is combining two data structures:

- A **hash map** `key → node` gives **O(1) lookup**.
- A **doubly-linked list** orders nodes by recency: most-recently-used at the head, least-recently-used at the tail. Moving a node to the head is O(1) (you have direct pointers), and evicting is just "remove the tail" — also O(1).

On every `get` or `set`, you move the touched node to the head. When you need room, you drop the tail. Both operations O(1). The full class-level build is in [124 — LLD: LRU Cache](./124-lld-lru-cache.md); the working code is in the Visual section below.

**The great real-world detail: Redis uses *approximate* LRU.** Maintaining a *perfect* global LRU order means every single access touches that shared linked list — which costs memory (two pointers per key) and serializes access. At Redis's scale that overhead is unacceptable. So Redis cheats intelligently: on eviction it **samples a handful of random keys** (5 by default, tunable via `maxmemory-samples`) and evicts the least-recently-used *among the sample.* It's not the globally-oldest key, but it's almost always close — and it costs almost nothing. Mention this in the interview; it shows you know the difference between the textbook algorithm and what production actually does, and *why* the tradeoff (a little accuracy for a lot of memory and speed) is the right one for a cache.

#### 3b. Expiration — how do you expire millions of keys with TTLs?

You set a key with `ttl = 3600`. An hour later it must be gone. The naive idea — one timer per key — is a disaster: millions of keys means millions of timers, enormous memory and scheduler overhead. You can't do it.

Redis uses **two strategies together**, and the interview gold is explaining why you need *both:*

**Lazy (passive) expiration** — check on access. When someone does `get(key)`, you look at the key's expiry timestamp. If it's in the past, you **delete it right then and return a miss.** Cost: zero background work; you only pay when someone actually touches the key.

**The problem with lazy alone:** a key that is set with a TTL and then *never read again* will sit in memory *forever*, expired but never cleaned up, silently leaking RAM. If 30% of your keys are write-once-never-read, lazy expiration alone slowly fills the cache with corpses.

**Active (background) expiration** — a background job runs periodically (Redis: ~10 times/sec), **samples a batch of random keys** with TTLs, and deletes the expired ones. If it finds that a large fraction of the sample was expired, it immediately samples again (the assumption being lots more are expired), and repeats until the expired fraction drops below a threshold. This is probabilistic garbage collection — it never guarantees an expired key is gone *the instant* it expires, but it guarantees expired keys don't accumulate unbounded.

**Why both:** lazy handles the hot keys correctly and for free (you never serve an expired hot key), while active reclaims the memory of cold keys that lazy would never touch. Together they bound both correctness (no stale hits) and memory (no leak). Neither alone is enough. Say exactly that.

---

### 4. Going distributed — partitioning (Step 4)

One machine's RAM tops out (in our estimate we needed ~2 TB but a node holds 64 GB). So we **shard the keyspace across N nodes.** The question is: given a key, which node owns it?

**Naive hashing (`hash(key) % N`) is a trap.** It works until N changes. Add one node — go from `% 8` to `% 9` — and *almost every key* now maps to a different node. That's a mass remap: nearly the entire cache misses at once, and the whole stampede lands on the database. Unacceptable.

**Consistent hashing** (recall [74 — Consistent Hashing](./74-consistent-hashing.md)) fixes this. Hash both nodes and keys onto a ring (0 … 2³²). A key is owned by the first node clockwise from its position. Now adding or removing a node only moves the keys in **one arc** of the ring — roughly `1/N` of keys — instead of all of them. Virtual nodes (many points per physical node) smooth out the distribution so no single node gets a lopsided share.

#### The three topologies — who does the routing?

| Topology | Who decides the node? | Pros | Cons |
|---|---|---|---|
| **Client-side** | The client library hashes the key and connects directly to the right node | fewest hops, no middleman | every client must know the full cluster map; hard to update all clients |
| **Proxy-based** | A proxy (e.g. Twemproxy) sits in front; clients talk to it, it routes | clients stay dumb; central place to manage topology | extra network hop; proxy is a scaling point |
| **Cluster (server-coordinated)** | Nodes gossip among themselves; any node can redirect a client to the right one | no separate proxy; nodes self-manage membership | more complex node software; clients need to follow redirects |

#### Redis Cluster's actual approach — 16,384 fixed hash slots

Here's a detail worth knowing precisely, because it differs from "pure consistent hashing" in a way interviewers love to probe. Redis Cluster does **not** hash keys directly onto a ring of nodes. Instead:

1. Every key is hashed into one of **16,384 fixed slots**: `slot = CRC16(key) mod 16384`.
2. The **16,384 slots are assigned to nodes** — e.g. with 3 nodes, node A owns slots 0–5460, B owns 5461–10922, C owns 10923–16383.
3. To look up a key: hash to a slot, look up which node owns that slot, go there.

**Why fixed slots make resharding cleaner than pure consistent hashing** — be honest about the difference:

- With **pure consistent hashing**, node positions and key positions are all on one continuous ring. When you rebalance, you reason about *arcs* and hash positions, and moving load means recomputing ring positions. It works, but the unit of movement is fuzzy.
- With **fixed slots**, the unit of movement is a **slot** — a clean, countable bucket. Resharding is just "reassign slots 5461–6000 from node B to node D and migrate the keys in those slots." Slots are an **explicit indirection layer** between keys and nodes: keys→slots is fixed forever (a key's slot never changes), and only slots→nodes moves. That makes rebalancing a discrete, trackable, resumable operation ("we've moved 400 of 540 slots"), and it makes the cluster map tiny and easy to gossip (just 16,384 slot→node entries). It's essentially consistent hashing with a fixed, finite set of virtual buckets baked in — trading the ring's mathematical elegance for operational clarity. For an operator managing failovers and migrations at 3 a.m., the discrete version is far easier to reason about.

(One consequence: keys you want on the same node — e.g. to run a multi-key operation — must hash to the same slot. Redis supports **hash tags**: `{user123}:profile` and `{user123}:settings` both hash only on `user123`, forcing them into the same slot and node.)

---

### 5. Replication and availability (Step 5)

Sharding gives us capacity, but each shard is now a **single point of failure** — lose that node and you lose its whole slice of the cache. Fix: **primary/replica per shard** (recall [86 — Data Replication Strategies](./86-data-replication-strategies.md)).

Each shard has one **primary** (handles writes) and one or more **replicas** (copies of the primary's data). This buys two things:

- **Read scaling** — reads can fan out to replicas, multiplying read throughput for hot shards.
- **Availability** — if the primary dies, a replica is **promoted** to primary and serving continues.

**Replication is asynchronous, and that's fine here.** The primary acknowledges a write to the client immediately, *then* streams it to replicas in the background. This keeps writes fast (sub-millisecond) but creates a **data-loss window**: if the primary crashes after acking a write but before the replica received it, that write is gone. For a *database* that's a serious problem. For a *cache* — which is allowed to lose data — it's a perfectly acceptable trade: the lost value simply repopulates from the source of truth on the next miss. This is the "allowed to forget" permission from Step 1 paying off directly.

**Failover — detecting death and coordinating a new primary.** Two mechanisms in the Redis world:

- **Redis Sentinel** (for non-clustered setups): a small fleet of Sentinel processes continuously ping primaries and replicas. When *enough* Sentinels agree a primary is down (a **quorum** — this avoids one flaky Sentinel triggering a needless failover), they **elect** one Sentinel to run the failover, which promotes the best replica and repoints clients.
- **Redis Cluster gossip** (for clustered setups): nodes gossip health among themselves. When a majority of primaries agree a primary is unreachable, the failed primary's replicas hold an **election** and one is promoted, taking over its slots.

Both are **leader election** under the hood (recall the election topic): a distributed set of processes must agree on *one* new primary despite failures, without split-brain (two nodes both thinking they're primary). The quorum/majority requirement is exactly what prevents split-brain.

---

### 6. The consistency problem — cache invalidation

This is the famously hard part (recall [59](./59-caching-in-depth.md)). The moment the underlying database changes, any cached copy of that data is **stale** — a confident, fast lie. The strategies differ in *when* and *how* the cache learns about the change.

| Strategy | On a write, you… | Freshness | Cost |
|---|---|---|---|
| **Write-through** | write to cache *and* DB synchronously, together | cache always fresh | slower writes; caches data nobody may read |
| **Write-around** | write only to DB; cache learns on next read-miss | avoids caching cold data | first read after a write is a miss |
| **Cache-aside + TTL** | write to DB; let the cached copy expire on its own | bounded staleness (≤ TTL) | serves stale data up to the TTL |
| **Explicit invalidation** (delete-on-write) | write to DB, then **delete** the key from cache | fresh on next read | needs write path to know the key; a race exists |

**Cache-aside is the most common pattern**, and its most-recommended variant is **delete-on-write**: on a write, update the DB and *delete* (don't update) the cache entry, so the next reader repopulates it fresh from the DB. Why delete rather than update? Because updating the cache from the write path re-introduces a stale-value race and caches data that may never be read.

**The famous cache-aside race condition.** Even delete-on-write has a subtle interleaving:

```
  Reader R gets a MISS, reads value v1 from DB          (but is slow, pauses here)
  Writer W updates DB to v2, then DELETEs the cache key
  Reader R now resumes and writes v1 into the cache      ← stale! v1 overwrites nothing, sits there
  → cache now holds v1 forever (until TTL), DB holds v2
```

The read repopulated a *stale* value into the cache *right after* the write deleted the fresh state. Mitigations:

- **A short TTL** puts a hard ceiling on how long the lie survives — the pragmatic favorite.
- **Set-if-not-modified / versioning** — attach a version to the value and only let a write into the cache if it's newer.
- **Single-flight / locking** on repopulation so the read-modify-write is serialized (see deep dive).
- **Delayed double-delete** — the writer deletes the key, waits a moment, and deletes again to catch a racing repopulation.

**The pragmatic truth: often you just use a short TTL and accept bounded staleness.** For a huge class of data (a user's follower count, a product's rating), being a few seconds or minutes stale is completely fine, and a short TTL is *far* simpler and more robust than trying to make invalidation perfect. Knowing *when* "good enough" is genuinely good enough — rather than reaching for a distributed-consensus cannon to swat a staleness fly — is a senior-engineer signal. State the trade explicitly: perfect freshness costs complexity and write latency; bounded staleness costs a slightly-old value. Pick per-use-case.

---

### 7. Deep dives

#### (a) The thundering herd / cache stampede

A single **hot key expires.** In the microsecond after it vanishes, 10,000 concurrent requests all `get` it, all **miss**, and all stampede the database *simultaneously* to recompute the same value. The database, sized for the *cached* load, buckles. This is the single most dangerous cache failure mode (recall [59](./59-caching-in-depth.md)).

Three fixes:

- **Per-key lock / single-flight** — only *one* request is allowed to recompute the value; the other 9,999 **wait** for that one to finish and then read its result. This collapses 10,000 DB hits into exactly 1. (Working `singleFlight` code below.) This is the most common, most robust fix.
- **Probabilistic early recomputation** — instead of everyone waiting for the exact expiry moment, each reader, as the TTL *approaches*, has a small random chance of recomputing *early* and refreshing the key. The hotter the key (more readers), the more likely *someone* refreshes it before it ever expires — so it never actually expires under load. Elegant because there's no lock and no synchronized cliff.
- **Stale-while-revalidate** — when a key is expired, serve the *stale* value immediately to keep latency flat, and kick off a **single** background refresh. Readers never block and the DB sees one recompute. Perfect when slightly-stale is acceptable (which, per the previous section, is often).

#### (b) Hot keys — one key so popular it overloads its shard

Consistent hashing spreads *keys* evenly, but it can't spread a *single* key: a celebrity's profile or a viral post hashes to exactly one slot on one node, and that node melts under the concentrated read load while its neighbors idle. Fixes:

- **Replicate the hot key across shards** — deliberately store copies under keys like `hotkey#1`, `hotkey#2`, `hotkey#3` on different nodes, and have readers pick one at random, spreading the read load. (You give up write-atomicity across the copies, which is fine for read-mostly hot data.)
- **Client-side local cache** — cache the handful of hottest keys in the *application's own* memory for a few seconds, in front of the distributed cache. A viral key then gets served from local RAM and never even reaches the cache node. (Beware: now you have another layer to invalidate, so keep the local TTL tiny.)

#### (c) Persistence — why would a *cache* want durability at all?

Redis offers two persistence options:

- **RDB snapshots** — periodically fork and dump the whole dataset to a compact binary file. Fast to load, small on disk, but you lose everything since the last snapshot on a crash.
- **AOF (Append-Only File)** — log every write command to a file; replay it on restart. More durable (down to per-second or per-write fsync), but larger files and slower load.

But wait — a cache is allowed to lose data, so why persist at all? **The killer reason is fast warm restart.** When a cache node reboots, it comes back **cold** — empty. Every request now misses, and the *entire* load slams the database until the cache slowly refills. On a big node that "refill" stampede can take down the DB. If instead the node reloads its dataset from an RDB/AOF file on startup, it comes back **warm** in seconds and the DB never feels the reboot. So persistence in a cache isn't about durability of the data — it's about **not causing a database stampede every time a node restarts.**

#### (d) Data structures beyond strings — why Redis is "more than a cache"

Memcached stores opaque strings; Redis stores **typed data structures** operated on *server-side*:

- **Lists** — queues, timelines (`LPUSH`/`RPOP`).
- **Sets** — unique membership, "who liked this" (`SADD`/`SISMEMBER`).
- **Sorted sets (ZSET)** — leaderboards and rate limiters, ordered by score (`ZADD`/`ZRANGE`) — arguably Redis's killer feature.
- **Hashes** — store an object's fields without serializing the whole thing.
- **Bitmaps / HyperLogLog** — space-efficient counting (unique visitors in a few KB).

Because the *operations run on the server*, you can do `INCR counter` atomically or `ZADD` to a leaderboard without a read-modify-write round trip. That turns Redis from a passive cache into an active data structure server — the reason it's used for rate limiting, leaderboards, queues, session stores, and pub/sub, not just caching.

---

### 8. Bottlenecks, failure modes, and how to scale further

- **Cold cache after a mass restart** — a whole cluster reboot (deploy gone wrong, power event) brings back empty nodes, every request misses, and the DB gets the full load at once. Defenses: persistence for warm restart (deep dive c), **gradual warming** (replay recent keys before taking traffic), and rolling restarts (never restart all nodes at once).
- **Memory fragmentation** — over time, allocating and freeing values of varying sizes leaves RAM checkerboarded with unusable gaps, so the node reports "full" while holding less real data than its limit. Mitigations: a good allocator (jemalloc), matching value sizes where possible, and Redis's active-defrag. Watch the fragmentation ratio.
- **The big one — what happens when the CACHE itself goes down.** Here's the dangerous illusion: everyone calls the cache "just an optimization." But at scale, the database has been quietly sized to handle only the ~5% of traffic that misses. When the cache dies, **100%** of traffic — 20x what the DB was built for — lands on it at once, and the database falls over. The cache was never "just an optimization"; it became **load-bearing** the moment you sized the database assuming it. Defenses:
  - **Request coalescing** at the app tier — collapse duplicate concurrent DB reads into one (single-flight again), so even a total cache outage doesn't multiply into 10,000 identical queries.
  - **DB-side protection** — a hard connection-pool cap and a load shedder so the database *sheds* excess load and survives degraded rather than crashing completely (a slow site beats a dead site).
  - **Circuit breakers** so the app fails fast to a fallback instead of piling onto a dying DB.
  - **Multi-replica caches** so a *single* node death is a small hit-rate dip, not a full outage.
  - Crucially: **load-test with the cache disabled.** If your database can't survive the cache being gone, you don't have a resilient system — you have a time bomb with a great hit rate. Say this in the interview; it's the mark of someone who's been paged at 3 a.m.

---

## Visual / Diagram description

**The overall architecture** — app tier in front of a sharded, replicated cache, with the database as the source of truth behind it:

```
                        ┌──────────────────────────────┐
     Clients ─────────► │        Application Tier       │
                        │  (cache-aside logic + local   │
                        │   hot-key cache + singleFlight)│
                        └───────────────┬───────────────┘
                                        │  get / set / delete
                     ┌──────────────────┼──────────────────┐
                     ▼                  ▼                  ▼
            ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
            │   SHARD A   │    │   SHARD B   │    │   SHARD C   │
            │  slots      │    │  slots      │    │  slots      │
            │  0–5460     │    │  5461–10922 │    │  10923–16383│
            │             │    │             │    │             │
            │  Primary    │    │  Primary    │    │  Primary    │
            │    │async   │    │    │async   │    │    │async   │
            │    ▼        │    │    ▼        │    │    ▼        │
            │  Replica    │    │  Replica    │    │  Replica    │
            └─────────────┘    └─────────────┘    └─────────────┘
                     │  gossip / Sentinel: detect death, elect new primary
                     └──────────────────┬──────────────────┘
                                        │  MISS → read/repopulate
                                        ▼
                            ┌───────────────────────┐
                            │   DATABASE (truth)    │
                            │  sized for ~5% load — │
                            │  MUST survive cache=0 │
                            └───────────────────────┘
```

**The slot/ring diagram** — Redis Cluster's 16,384 fixed slots assigned to nodes (compare with a pure consistent-hashing ring):

```
   key ──CRC16(key) mod 16384──►  slot 7342
                                     │  (slot→node lookup)
                                     ▼
        16,384 SLOTS laid across 3 nodes:

        0 ───────── 5460 ───────── 10922 ──────── 16383
        │◄─ Node A ─►│◄── Node B ──►│◄─── Node C ──►│
              slot 7342 lives in Node B's range ────┘

   Resharding = move a RANGE of slots (a discrete, countable unit):
        "migrate slots 5461–6000 : Node B ──► Node D"
        keys→slots NEVER changes; only slots→nodes moves.

   Pure consistent-hashing ring (for contrast):
                    0/2³²
                 ┌────●────┐        ● = node position (virtual nodes omitted)
              N3 ●         ● N1      key k lands at hash(k),
                 │  →k→    │         owned by first node CLOCKWISE
                 └────●────┘         add a node → only ONE arc's keys move
                     N2
```

**The LRU structure** — hash map for O(1) lookup, doubly-linked list for O(1) recency ordering:

```
   MAP:  { "a": ◄─┐   "b": ◄─┐   "c": ◄─┐ }
                  │          │          │
   LIST (head=MRU ... tail=LRU):
        HEAD ⇄ [ c ] ⇄ [ b ] ⇄ [ a ] ⇄ TAIL
                 ▲                  ▲
          just touched         evict from here
          (moved to head)      when full
```

---

## Real world examples

### Redis
The reference implementation of everything in this doc. Single-threaded event loop (no lock contention on the hot path → predictable sub-ms latency), typed data structures, approximate-LRU eviction, lazy+active expiration, RDB/AOF persistence, Sentinel and Cluster for HA, and 16,384 hash slots. When someone says "add a cache," they usually mean "add Redis."

### Memcached
The older, deliberately-simpler cousin. Multi-threaded, strings-only, no persistence, no replication built in — a pure, fast, in-memory hash map with a slab allocator (which sidesteps fragmentation by design). Client-side sharding. Facebook famously ran one of the largest Memcached deployments on earth and wrote the paper ("Scaling Memcache at Facebook") on the exact herd/consistency problems above.

### Amazon ElastiCache / Google Memorystore
Managed Redis/Memcached. You get the same design, but the cloud handles failover, replication, and node replacement. This is what most teams actually deploy — the design knowledge here is what lets you *configure and reason about* it correctly (eviction policy, TTLs, cluster mode on/off, replica count).

---

## Trade-offs

- **Speed vs. durability.** The whole system trades durability for latency: in-memory, async replication, allowed to lose data. Right for a cache; wrong for a database. The permission to forget is the design's foundation.
- **Freshness vs. simplicity (invalidation).** Write-through gives freshness at the cost of write latency and caching cold data; short-TTL cache-aside gives simplicity at the cost of bounded staleness. Most systems rightly choose bounded staleness.
- **Perfect LRU vs. cheap approximate LRU.** A true global LRU costs memory and serializes access; sampled approximate LRU is nearly as good for a tiny fraction of the cost. Redis chose approximate, and it's the right call.
- **Fixed slots vs. pure consistent hashing.** Fixed slots trade the ring's mathematical elegance for operational clarity — discrete, countable, resumable resharding. Easier to run in production.
- **Client-side vs. proxy vs. cluster topology.** Client-side = fewest hops but fat clients; proxy = dumb clients but an extra hop; cluster = self-managing but complex node software. Redis Cluster chose server-coordination.
- **The load-bearing illusion.** Treating the cache as "just an optimization" buys development simplicity but hides a systemic risk: the DB gets sized for the cached load and can't survive the cache's absence. The honest trade is to *know* the cache is load-bearing and protect the DB accordingly.

---

## Common interview questions on this topic

### Q1: "How do you implement an LRU cache with O(1) get and set?"
Hash map for O(1) lookup + doubly-linked list for O(1) recency ordering. On access, move the node to the head; on eviction, drop the tail. Then note Redis uses *approximate* (sampled) LRU because a perfect global order costs too much memory. (Full build: [124](./124-lld-lru-cache.md).)

### Q2: "How do you expire millions of keys with TTLs?"
Not one timer per key. Use **lazy** expiration (delete on access if expired) plus **active** expiration (a background job samples random keys and purges expired ones). You need both: lazy alone leaks memory for keys that are never read again.

### Q3: "How do you shard the cache, and what happens when you add a node?"
Consistent hashing so adding/removing a node only remaps ~1/N of keys, not all of them (naive `mod N` remaps almost everything → mass miss → DB stampede). Redis Cluster uses 16,384 fixed slots on top, so resharding moves discrete, countable slot ranges.

### Q4: "The DB changed and the cache is now stale. How do you handle it?"
Name the strategies: write-through, write-around, cache-aside with TTL, explicit delete-on-write. Explain the cache-aside race (a slow read repopulates a stale value right after a write deletes it) and mitigations (short TTL, versioning, single-flight, delayed double-delete). Note that a short TTL with bounded staleness is often the pragmatic best answer.

### Q5: "A hot key expires and 10,000 requests miss at once. What happens and how do you fix it?"
That's the thundering herd — all 10,000 stampede the DB. Fixes: single-flight/per-key lock (one recomputes, rest wait), probabilistic early recomputation, or stale-while-revalidate.

### Q6: "What happens to your system when the cache goes down?"
The DB gets 100% of traffic — often 20x what it was sized for — and may fall over. The cache is load-bearing, not "just an optimization." Protect with request coalescing, DB load shedding/connection caps, circuit breakers, replicas, and load-testing with the cache disabled.

### Q7: "Why would a cache need persistence if it's allowed to lose data?"
Fast warm restart. A rebooted node comes back cold, every request misses, and the DB gets stampeded. RDB/AOF lets the node reload its dataset in seconds and come back warm, sparing the DB.

---

## Practice exercise

### Design a distributed session cache for a global web app — and defend three decisions

**Scenario.** 50 million active users; every request carries a session token that must be validated against a session object (user id, permissions, expiry). Reads: ~500,000/sec globally. Sessions expire after 30 minutes of inactivity. Losing a session is *tolerable* — the user just re-logs-in.

**Deliverable 1 — capacity.** Each session object is ~2 KB. Size the working set and the node count at 64 GB/node with one replica each. Is this memory-bound or throughput-bound?

**Deliverable 2 — expiration.** Sessions have a sliding 30-minute TTL (every request resets it). Which expiration strategy(ies) do you use, and how does the sliding window change your `get` implementation?

**Deliverable 3 — defend three decisions in writing (3–4 sentences each):**
1. **Eviction policy** — LRU, LFU, or TTL-based for sessions? Justify against the access pattern (a session is read constantly right up until the user leaves, then never again).
2. **Consistency** — a user's permissions were just revoked but their session is cached. How stale are you willing to be, and what does that cost? Would you invalidate explicitly or ride the TTL?
3. **Failure** — the entire cache region goes down. What happens to 50M logged-in users, and what do you put in place *now* so it's a degraded experience, not an outage?

**The point:** at least one of your choices should differ from this doc's defaults, because sessions are not a read-through DB cache — they're closer to a primary store of soft state. If your design is identical to the generic one, you were pattern-matching, not reading the requirements.

---

## Quick reference cheat sheet

- **The one-liner:** a distributed cache is a fast, shared, in-memory key-value store that is **allowed to lose data** — which is exactly why it can be so fast.
- **The framework:** Requirements → Estimation → Single-node (map + eviction + expiration) → Distributed (consistent hashing) → Replication + deep dives.
- **Sizing:** cache the *working set* (80/20 rule), not the whole DB. `nodes = working_set / RAM_per_node`, then ×2 for replicas. Usually **memory-bound**, so throughput comes free.
- **Single node = an in-memory hash map** → O(1) get/set → sub-millisecond latency. That latency *is the product*.
- **Eviction:** LRU (map + doubly-linked list = O(1)) is the default; also LFU, FIFO, random, TTL. Redis uses **approximate/sampled LRU** because perfect LRU costs too much memory.
- **Expiration:** **lazy** (delete on access) **+ active** (background samples random keys). Both — lazy alone leaks memory for never-read keys.
- **Partitioning:** consistent hashing (not `mod N`, which mass-remaps). Redis Cluster = **16,384 fixed slots**, so resharding moves discrete slot ranges — cleaner than a pure ring.
- **Replication:** primary/replica per shard, **async** (data-loss window is fine for a cache). Failover via **Sentinel quorum** or **cluster gossip** — leader election with a majority to avoid split-brain.
- **Invalidation** (the hard part): write-through / write-around / cache-aside+TTL / delete-on-write. Watch the cache-aside race (stale repopulate after delete). **A short TTL with bounded staleness is usually the pragmatic winner.**
- **Thundering herd:** hot key expires → 10k DB misses. Fix with **single-flight**, probabilistic early recompute, or stale-while-revalidate.
- **Hot key:** one key melts one shard → replicate it across shards or add a tiny client-side local cache.
- **Persistence (RDB/AOF):** not for durability — for **warm restart** so a rebooted node doesn't cold-start-stampede the DB.
- **The danger:** the cache is **load-bearing**, not "just an optimization." When it dies the DB gets 20x load. Protect it and **load-test with the cache off.**

---

## Appendix — key code

**O(1) LRU cache (hash map + doubly-linked list), Node.js:**

```javascript
class DListNode {
  constructor(key, value) {
    this.key = key;
    this.value = value;
    this.prev = null;
    this.next = null;
  }
}

class LRUCache {
  constructor(capacity) {
    this.capacity = capacity;
    this.map = new Map();              // key -> DListNode  (O(1) lookup)
    // Sentinel head/tail so we never null-check at the edges.
    this.head = new DListNode(null, null); // MRU side
    this.tail = new DListNode(null, null); // LRU side
    this.head.next = this.tail;
    this.tail.prev = this.head;
  }

  _remove(node) {                      // unlink a node — O(1)
    node.prev.next = node.next;
    node.next.prev = node.prev;
  }

  _addToFront(node) {                  // insert right after head — O(1)
    node.next = this.head.next;
    node.prev = this.head;
    this.head.next.prev = node;
    this.head.next = node;
  }

  get(key) {
    const node = this.map.get(key);
    if (!node) return undefined;       // miss
    this._remove(node);                // touch → move to front (most recent)
    this._addToFront(node);
    return node.value;
  }

  set(key, value) {
    let node = this.map.get(key);
    if (node) {                        // update existing + promote
      node.value = value;
      this._remove(node);
      this._addToFront(node);
      return;
    }
    if (this.map.size >= this.capacity) {   // full → evict LRU (the tail)
      const lru = this.tail.prev;
      this._remove(lru);
      this.map.delete(lru.key);
    }
    node = new DListNode(key, value);
    this._addToFront(node);
    this.map.set(key, node);
  }
}
// Every operation is O(1): the Map gives O(1) lookup,
// the doubly-linked list gives O(1) move-to-front and evict-from-tail.
```

**Single-flight wrapper — collapses a thundering herd into one recompute:**

```javascript
// Wraps a "load from DB" function so that if N callers ask for the same
// key at the same time, only ONE actually hits the DB; the rest await it.
class SingleFlight {
  constructor() {
    this.inFlight = new Map();         // key -> Promise
  }

  async do(key, loader) {
    const pending = this.inFlight.get(key);
    if (pending) return pending;       // someone's already loading → wait on it

    const promise = (async () => {
      try {
        return await loader();         // the ONE real DB call
      } finally {
        this.inFlight.delete(key);     // clear so future misses can reload
      }
    })();

    this.inFlight.set(key, promise);
    return promise;
  }
}

// Cache-aside read protected against the thundering herd:
const flight = new SingleFlight();

async function cachedGet(cache, db, key, ttl) {
  const hit = await cache.get(key);
  if (hit !== undefined && hit !== null) return hit;   // fast path

  // MISS: only one caller per key recomputes; the herd awaits the same promise.
  return flight.do(key, async () => {
    // Double-check the cache — another request may have filled it while we queued.
    const recheck = await cache.get(key);
    if (recheck !== undefined && recheck !== null) return recheck;

    const value = await db.load(key);          // the single DB hit
    await cache.set(key, value, ttl);
    return value;
  });
}
```

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Foundation** | [59 — Caching in Depth](./59-caching-in-depth.md) — the 80/20 working set, eviction policies, TTLs, cache-aside, and the stampede fixes this whole design is built on; read it first |
| **Related** | [74 — Consistent Hashing](./74-consistent-hashing.md) — how keys are partitioned across nodes without a mass remap when the cluster changes; the basis for the 16,384-slot scheme |
| **Related** | [86 — Data Replication Strategies](./86-data-replication-strategies.md) — primary/replica, async replication and its data-loss window, and how failover promotes a replica |
| **Related** | [96 — Design a Key-Value Store](./96-hld-key-value-store.md) — the durable, source-of-truth sibling of this system; contrast "allowed to lose data" here with "must never lose data" there |
| **Deep dive** | [124 — LLD: LRU Cache](./124-lld-lru-cache.md) — the full class-level, test-driven build of the map + doubly-linked-list LRU sketched here |
