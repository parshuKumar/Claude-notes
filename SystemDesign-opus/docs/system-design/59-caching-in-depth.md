# 59 — Caching — Strategies, Layers, and Pitfalls
## Category: HLD Components

---

## What is this?

A **cache** is a small, fast store that keeps copies of data you're likely to ask for again, so you don't have to do the slow, expensive work of fetching it a second time.

Think of your kitchen counter. The supermarket has *everything*, but it's 20 minutes away. So you keep the milk, coffee, and bread on the counter and in the fridge — the things you reach for every single morning. The counter is tiny compared to the supermarket, but it holds the 5% of items you use 95% of the time. That's a cache: **small, fast, close, and holding the popular stuff.**

In system design, caching is the single highest-leverage move you have. A cache in front of your database can turn a 50-millisecond query into a 1-millisecond lookup and cut database load by 95% — with about 20 lines of code.

---

## Why does it matter?

**If you don't cache:**
- Every single request hits your database. Your database becomes the bottleneck at a few thousand queries per second, and vertically scaling it (bigger machine) gets expensive fast.
- Your p99 latency is whatever your slowest query is. Users feel it.
- A traffic spike (a celebrity posts, a sale starts) takes your database down, and the database going down takes the *whole* system down.

**If you cache badly:**
- Users see stale data — the price changed but the old one is still showing.
- A cache node restarts, all keys vanish at once, and 100% of traffic slams the database simultaneously. Your "optimization" just became your outage.
- Two users get different answers depending on which app server they hit.

**The interview angle:** Caching comes up in *every* HLD interview. If the interviewer says "how do you scale reads?", the answer starts with caching. And the follow-up is always one of: "what's your eviction policy?", "how do you invalidate?", "what happens when the cache is cold?" Knowing the failure modes (stampede, penetration, avalanche) is what separates a mid-level answer from a senior one.

**The real-work angle:** Adding Redis in front of a hot query is one of the most common performance fixes you will ever ship. Knowing *which* strategy to use — and which one silently loses data — matters.

---

## The core idea — explained simply

### The Library Reference Desk Analogy

You work at a university library reference desk. The stacks (where the books live) are three floors down. Fetching a book takes 5 minutes round trip.

Day 1, you fetch every book a student asks for. You're exhausted, and the queue is out the door.

Day 2, you notice something: **out of 50,000 books in the stacks, students ask for the same 30 books over and over.** It's exam week. Everyone wants the same intro textbooks.

So you put a shelf behind the desk and keep those 30 books on it. Now:
- Student asks for "Intro to Algorithms" → it's on your shelf → **5 seconds** (a *cache hit*)
- Student asks for an obscure 1974 monograph → not on your shelf → you go downstairs, fetch it, hand it over, **and put it on your shelf on the way back** in case someone asks again → 5 minutes (a *cache miss*)

Your shelf holds 30 books. When it's full and you need to add a new one, you have to **evict** something — take a book off the shelf and send it back down. Which one? The one nobody's touched in the longest time, probably. That's **LRU** (Least Recently Used).

There's one more problem. Overnight, the professor **updates** the recommended edition of a textbook. Your shelf still has the old edition. A student takes it and studies the wrong chapters. That is **stale data**, and deciding when to throw books off your shelf because the source changed is **cache invalidation** — famously one of the two hard problems in computer science.

### Mapping the analogy

| Library | System design |
|---------|---------------|
| The stacks, 3 floors down | The database (slow, complete, authoritative) |
| The shelf behind the desk | The cache (fast, small, a copy) |
| Book is on the shelf | **Cache hit** |
| Book isn't — you go downstairs | **Cache miss** |
| Putting it on the shelf on the way back | **Cache-aside / lazy loading** |
| The shelf is full, remove something | **Eviction** |
| "Remove the least-touched book" | **LRU eviction policy** |
| A book auto-returns after 7 days | **TTL** (time to live) |
| The professor updated the edition | **Invalidation** — the source of truth changed |
| Exam week: 200 students want the same book you just don't have yet | **Cache stampede** |

### Why caching works at all: the two localities

Caching only helps because real-world access patterns are **not uniform**. Two properties make it work:

1. **Temporal locality** — if something was requested recently, it will likely be requested again soon. (A viral tweet gets 90% of its reads in the first hour.)
2. **Spatial locality** — if you requested one item, you'll likely request items *near* it. (You read post #500, you'll probably scroll to #501.)

Combine these and you get the **80/20 rule** (Pareto): roughly 20% of your data serves 80% of your requests. On the internet it's usually far more extreme — for a social feed it's more like **1% of content serving 90% of reads**. That skew is the entire economic basis of caching: you can hold 1% of the data in RAM and serve 90% of the traffic from it.

### The latency win, in real numbers

These are the numbers to memorize. (Recall from [12 — Numbers Every Engineer Should Know](./12-numbers-every-engineer-should-know.md).)

| Where the data lives | Typical latency | Relative to L1 cache |
|---|---|---|
| CPU L1 cache | 1 ns | 1× |
| **Local process memory (a JS `Map`)** | **~100 ns** | 100× |
| RAM main memory read | 100 ns | 100× |
| **Redis / Memcached, same datacenter** | **0.5 – 1 ms** | ~10,000× |
| SSD random read | 100 µs (0.1 ms) | 100,000× |
| **Database query (indexed, with network + parse + plan)** | **10 – 50 ms** | ~50,000,000× |
| Database query (unindexed table scan) | 200 ms – 5 s | worse |
| Cross-continent network round trip | 150 – 250 ms | worse |

Read that middle block again:

```
Database query   : 50 ms      = 50,000,000 ns
Redis lookup     :  1 ms      =  1,000,000 ns    →   50x faster
Local in-memory  :  0.0001 ms =        100 ns    → 500,000x faster than the DB
```

**Local memory is 10,000× faster than Redis, and Redis is 50× faster than the DB.** That ordering drives every decision in this doc. But local memory doesn't scale (each server has its own copy → inconsistency), and Redis does. That's the trade-off you keep paying.

---

## Key concepts inside this topic

### 1. The cache layers — top to bottom

There isn't *one* cache. There are six, stacked, and a request falls through them like a coin through a coin sorter. The earlier you catch it, the cheaper it is.

| # | Layer | Lives where | Typical latency | Controlled by |
|---|---|---|---|---|
| 1 | **Browser cache** | User's laptop/phone disk | ~0 ms | `Cache-Control` headers |
| 2 | **CDN / edge cache** | PoP 20 km from the user | 10–30 ms | `Cache-Control`, `s-maxage` |
| 3 | **Reverse-proxy cache** | Nginx/Varnish in your DC | 1–5 ms | proxy config |
| 4 | **App in-process cache** | RAM of the Node process | ~100 ns | your code (a `Map`) |
| 5 | **Distributed cache** | Redis/Memcached cluster | 0.5–1 ms | your code |
| 6 | **DB buffer pool / query cache** | Inside Postgres/MySQL | 0.1–1 ms | the DB itself |
| — | **Disk** | The actual table | 10–50 ms | nobody's happy |

Key insight: **each layer only sees the misses of the layer above it.** If the browser cache has a 50% hit rate and the CDN has a 90% hit rate on what's left, your origin only sees 5% of the original traffic. Hit rates *multiply*.

### 2. Cache-hit ratio math — why even a mediocre cache is a huge win

The formula:

```
hit ratio = hits / (hits + misses)

effective latency = (hit_ratio × cache_latency) + (miss_ratio × (cache_latency + db_latency))
```

Note the miss path pays *both* — you check the cache, miss, *then* go to the DB.

**Worked example.** Your API does 10,000 requests/sec. A DB read is 50 ms. Redis is 1 ms. Your DB tops out at 2,000 queries/sec before it falls over.

**No cache:**
```
DB load          = 10,000 qps   → DB is at 500% of capacity → system is down
avg latency      = 50 ms
```

**With a merely OK 80% hit ratio:**
```
DB load          = 10,000 × 0.20 = 2,000 qps   → exactly at capacity. Survives.
avg latency      = (0.80 × 1 ms) + (0.20 × 51 ms)
                 = 0.8 + 10.2
                 = 11 ms                        → 4.5x faster
```

**With a good 95% hit ratio:**
```
DB load          = 10,000 × 0.05 = 500 qps     → DB is at 25% capacity. Comfortable.
avg latency      = (0.95 × 1) + (0.05 × 51)
                 = 0.95 + 2.55
                 = 3.5 ms                       → 14x faster
```

**With 99%:**
```
DB load          = 100 qps
avg latency      = (0.99 × 1) + (0.01 × 51) = 1.5 ms
```

Two lessons hide in those numbers:

1. **The load reduction is more dramatic than the latency reduction.** Going from 80% → 95% only takes latency from 11 ms → 3.5 ms (3×), but it takes DB load from 2,000 → 500 qps (4× headroom). Caching is usually a *capacity* play first and a *latency* play second.
2. **Every 9 you add to the hit ratio divides DB load by 10.** 90% → 99% → 99.9% cuts DB load 10× each time. This is why teams obsess over the last few percent.

### 3. Reading strategies

#### Cache-aside (lazy loading) — the default, and the one you'll use 90% of the time

The application talks to *both* the cache and the DB. The cache doesn't know the DB exists.

```
  read(key):
    1. value = cache.get(key)
    2. if value exists  → return it                       (HIT)
    3. else             → value = db.query(key)           (MISS)
                          cache.set(key, value, ttl)
                          return value
```

```javascript
// cache-aside: the canonical implementation
import { createClient } from 'redis';

const redis = createClient({ url: 'redis://localhost:6379' });
await redis.connect();

const TTL_SECONDS = 300; // 5 minutes

async function getUser(userId) {
  const key = `user:${userId}`;

  // 1. Try the cache first.
  const cached = await redis.get(key);
  if (cached !== null) {
    return JSON.parse(cached);          // HIT — ~1ms, done.
  }

  // 2. MISS. Go to the source of truth.
  const user = await db.query('SELECT * FROM users WHERE id = $1', [userId]);
  if (!user) return null;

  // 3. Populate the cache so the NEXT reader gets a hit.
  //    EX sets a TTL so an entry can never be stale forever.
  await redis.set(key, JSON.stringify(user), { EX: TTL_SECONDS });

  return user;
}
```

**Why it's the default:** it's resilient. If Redis dies, every read becomes a miss and falls through to the DB — slow, but *correct*. The system degrades instead of breaking. It also only caches what's actually asked for, so you never waste memory on cold data.

**The race condition hiding in it.** Look at steps 2 and 3 again. They are not atomic. Two requests can interleave:

```
 T=0ms   Request A: cache.get("user:7")  → MISS
 T=1ms   Request B: cache.get("user:7")  → MISS       (A hasn't written yet)
 T=2ms   Request A: db.query(...)        → {name: "Alice"}
 T=3ms   Writer:    UPDATE users SET name='Alicia' WHERE id=7
 T=4ms   Writer:    cache.del("user:7")                (invalidate — but it's empty!)
 T=5ms   Request B: db.query(...)        → {name: "Alicia"}   (B reads the NEW value)
 T=6ms   Request B: cache.set("user:7", "Alicia")
 T=7ms   Request A: cache.set("user:7", "Alice")   ← A's STALE read overwrites the fresh one
```

Request A, which read the DB *before* the write, wrote its stale value into the cache **after** the invalidation. The cache now serves `Alice` until the TTL expires — potentially for 5 minutes. This is a real, reported-in-production bug.

**Fixes:** (a) always set a TTL so staleness is bounded; (b) use single-flight so only one request loads a key at a time (see §8); (c) for critical data, use a versioned key or `SET ... XX`/Lua compare-and-set so a writer's delete wins.

#### Read-through — the cache is a proxy for the DB

The application only ever talks to the cache. The cache itself knows how to fetch from the DB on a miss.

```
  app → cache.get(key)
             │ (miss)
             ▼
        cache calls loader() → db → stores → returns
```

Redis doesn't do this natively; you get it from a client library (or a wrapper you write — the one in §8 is effectively a read-through). Java folks get it from Caffeine/Ehcache; DynamoDB Accelerator (DAX) is a true read-through cache.

**Trade-off:** cleaner application code (one call site, no cache logic sprinkled around), but the caching logic is now infrastructure you can't easily bypass or reason about, and a cache outage becomes a *hard* outage rather than a slow one.

### 4. Writing strategies

Reading is the easy half. The hard question is: when data changes, what happens to the cache?

#### Write-through — write to cache AND DB, synchronously, before acking

```javascript
async function updateUser(userId, patch) {
  const user = await db.update('users', userId, patch);       // 1. DB first (source of truth)
  await redis.set(`user:${userId}`, JSON.stringify(user), { EX: 300 }); // 2. then cache
  return user;                                                // 3. only now ack the client
}
```

The cache is always fresh. But every write now pays *both* latencies, and you're caching data that may never be read again (write amplification). Order matters: **write the DB first.** If you write the cache first and the DB write fails, your cache is now lying about durable state.

#### Write-back (write-behind) — write to cache, ack, flush to DB later

```javascript
async function recordView(postId) {
  // Ack in ~1ms. The DB write happens on a batched flush.
  await redis.incr(`views:${postId}`);
  await redis.sAdd('dirty:views', String(postId)); // remember what to flush
}

// A background worker flushes every 10 seconds, batching thousands of increments
// into a handful of SQL statements.
setInterval(async () => {
  const ids = await redis.sPop('dirty:views', 500);
  for (const id of ids) {
    const count = await redis.get(`views:${id}`);
    await db.query('UPDATE posts SET views = $1 WHERE id = $2', [count, id]);
  }
}, 10_000);
```

Blisteringly fast writes and massive write coalescing — 10,000 view increments become 1 DB update. **But if the cache node dies before the flush, that data is gone forever.** Only use it for data you can afford to lose (view counters, analytics, "last seen" timestamps). Never for money.

#### Write-around — write to the DB only; let the cache find out on the next read

```javascript
async function importBatch(rows) {
  await db.bulkInsert('events', rows);   // DB only.
  // No cache write. If someone reads it later, cache-aside will populate it.
}
```

Perfect for write-heavy data that's rarely read back (logs, event ingestion, bulk imports). It stops you from flooding the cache with cold data and evicting the hot stuff. Downside: the first read after a write is always a miss.

#### The comparison table

| Strategy | Write latency | Read-after-write consistency | Data-loss risk on cache crash | Cache pollution | Use when |
|---|---|---|---|---|---|
| **Write-through** | Slow (DB + cache) | Strong | None (DB is written first) | High — caches unread data | Data is read soon and often after writing; you want freshness |
| **Write-back** | Very fast (cache only) | Strong (from cache) | **HIGH — unflushed writes are lost** | Low | Counters, metrics, high-frequency writes you can lose |
| **Write-around** | Fast (DB only) | Weak — next read is a miss | None | None | Write-heavy, read-rarely data (logs, imports) |
| **Invalidate-on-write** (`db.write()` then `cache.del()`) | Fast (DB + one DEL) | Strong-ish (next read re-fetches) | None | None | **The most common real-world default.** Pair it with cache-aside. |

That last row is what most production systems actually do: **cache-aside for reads + delete-on-write for invalidation.** Delete, don't update — updating races (see §3); deleting is idempotent and always safe.

### 5. Eviction policies

Your cache is finite. When it's full and a new key arrives, something must go.

| Policy | How it picks a victim | Wins when | Loses when |
|---|---|---|---|
| **LRU** (Least Recently Used) | Evict the key untouched for the longest | General purpose — temporal locality is real | A big sequential scan pushes out all your hot keys |
| **LFU** (Least Frequently Used) | Evict the key with the fewest accesses | Stable popularity; scan-heavy workloads | A once-popular key that's now dead stays forever (unless the counter decays) |
| **FIFO** | Evict the oldest inserted, regardless of use | Almost never — it's just simple | Your hottest key gets evicted purely for being old |
| **Random** | Pick a victim at random | Surprisingly OK; O(1) and no metadata cost | No guarantees; occasionally evicts the hot key |
| **TTL** | Evict on expiry, not on pressure | Data with a natural freshness window | Not really an eviction policy — it's a *correctness* mechanism |

**The classic LFU-vs-LRU exam question: the scan.**

You have a 1,000-key cache. 100 keys are genuinely hot (read constantly). Then an analytics job runs a full table scan, touching 1,000,000 rows once each.

- **Under LRU:** every scanned row becomes "most recently used" and pushes a hot key out. By the end of the scan, **all 100 hot keys are gone.** Your hit ratio drops to ~0 and the DB gets hammered for the next few minutes while the cache re-warms. This is *cache pollution*, and LRU is defenceless against it.
- **Under LFU:** each scanned row is accessed exactly *once*, so its frequency count is 1. The hot keys have counts in the thousands. The scanned rows evict *each other* and never touch the hot set. **The hit ratio barely moves.**

That's the whole argument. **LFU is scan-resistant; LRU is not.** Redis's `allkeys-lfu` uses an approximated, *decaying* LFU counter — the decay is what stops yesterday's viral post from squatting in memory forever.

Redis's eviction settings, worth memorizing:

```
maxmemory 4gb
maxmemory-policy allkeys-lru   # evict any key, LRU     ← common default for a pure cache
                 allkeys-lfu   # evict any key, LFU     ← better for scan-heavy read patterns
                 volatile-lru  # only evict keys with a TTL set
                 noeviction    # reject writes when full ← use when Redis is a datastore, not a cache
```

If you use Redis as a cache and leave it on `noeviction`, a full instance will start throwing `OOM command not allowed` errors on writes. This surprises people at 3am. Set `allkeys-lru`.

### 6. The hard problems (this is what interviewers actually probe)

#### Cache stampede / thundering herd

A single very hot key expires. In the microseconds before anyone repopulates it, **10,000 concurrent requests all miss simultaneously and all query the database for the same row.** The DB, which was doing 500 qps, is suddenly asked for 10,000 identical queries. It falls over. And because it's down, nobody can repopulate the cache, so the herd keeps coming.

```
                      key "homepage:feed" expires at T=0
                                   │
   ┌────┬────┬────┬────┬───────────┴───────────┬────┬────┬────┬────┐
   ▼    ▼    ▼    ▼    ▼                       ▼    ▼    ▼    ▼    ▼
 req  req  req  req  req    ... 10,000 ...    req  req  req  req  req
   │    │    │    │    │                       │    │    │    │    │
   └────┴────┴────┴────┴──────────┬────────────┴────┴────┴────┴────┘
                                  ▼
                        ┌───────────────────┐
                        │    DATABASE       │   ← 10,000 identical queries
                        │   💀 (dies)       │      for ONE row
                        └───────────────────┘
```

**Three fixes, in increasing order of sophistication:**

1. **Single-flight / mutex lock.** Only the first request that misses is allowed to hit the DB. Everyone else waits for its result. In a single Node process this is an in-memory promise map (§8). Across many processes, it's a Redis lock: `SET lock:key <uuid> NX EX 5`. The winner loads; the losers sleep 20 ms and retry the cache.
2. **Probabilistic early expiry (XFetch).** Instead of everyone expiring at the same instant, each reader independently rolls the dice as the TTL approaches and, with rising probability, refreshes *early* — while the value is still valid and being served. One unlucky request does the work; nobody ever sees a miss.
   ```javascript
   // XFetch: refresh early with probability that grows as expiry nears.
   // delta = how long the last recompute took. beta = aggressiveness (1.0 is a fine default).
   function shouldRefreshEarly(deltaMs, expiryEpochMs, beta = 1.0) {
     const now = Date.now();
     return now - deltaMs * beta * Math.log(Math.random()) >= expiryEpochMs;
   }
   ```
3. **Stale-while-revalidate.** Never delete the value on expiry. Mark it stale, **serve the stale value immediately**, and kick off a background refresh. Users never wait; the DB sees exactly one query. This is the best answer for anything where 30 seconds of staleness is acceptable — which is most things.

#### Cache penetration

Requests for keys that **don't exist in the database at all**. `GET /user/999999999`. The cache misses (nothing to cache), the DB misses (no row), nothing gets stored — so *every single one of those requests hits your DB*. An attacker can bypass your cache entirely just by requesting random IDs.

**Fixes:**
- **Negative caching.** Cache the "not found" result too, with a short TTL: `redis.set(key, 'NULL', { EX: 60 })`. Now the second request for a missing key is a hit. Short TTL, because the row might get created.
- **Bloom filter.** A probabilistic set that answers "is this ID *definitely not* in the DB?" in O(1) using ~10 bits per key. It can produce false positives ("maybe present") but **never false negatives** — so if the filter says "not present", you can reject the request without touching the DB. 100M user IDs fit in ~120 MB. This is what you use when negative caching would blow up your memory (the key space is huge and mostly empty).
- **Input validation.** Half of penetration attacks die if you reject IDs that don't match your ID format.

#### Cache avalanche

You warm your cache at deploy time and set every key to `EX 3600`. One hour later, **every key expires within the same millisecond.** 100% of traffic goes to the DB at once. Same effect as a stampede but across your whole key space. (The other flavour: a Redis node reboots and its entire keyspace vanishes.)

**Fixes:**
- **Jitter the TTLs.** Never use a constant. Randomize ±10–20%:
  ```javascript
  const jitteredTtl = (baseSeconds, jitterPct = 0.2) => {
    const spread = baseSeconds * jitterPct;
    return Math.floor(baseSeconds - spread + Math.random() * 2 * spread);
  };
  // jitteredTtl(3600) → some value in [2880, 4320]. Expiries now spread over 24 minutes.
  ```
- **Cache warming**, plus a rate limiter / circuit breaker in front of the DB so that even a total cache loss can't push more than N qps at it.
- **Redis replicas + persistence** so a node restart doesn't mean an empty keyspace.

#### Hot keys

One key is *so* popular that the single Redis shard holding it is saturated — e.g. `celebrity:profile:1` during a live event, or a global feature-flag key read on every request. Sharding doesn't help: hashing puts one key on exactly one node.

**Fixes:**
- **Local (L1) cache in front of Redis.** Even a 1-second in-process TTL turns 50,000 Redis reads/sec into 1 per server per second. This is the big one.
- **Key splitting / replication.** Store `hotkey:0` … `hotkey:9` (10 copies with identical values) across shards and have each client read a random suffix. Ten shards now share the load.
- **Client-side request coalescing** (single-flight again).

#### Cache invalidation — the famously hard one

> "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton

Hard because the cache holds a *copy*, and the moment the original changes, the copy is a lie. You have four options and they are all imperfect:

| Approach | How | Freshness | Complexity | Failure mode |
|---|---|---|---|---|
| **TTL only** | Let entries expire | Eventually consistent — stale for up to the TTL | Trivial | Users see old data for TTL seconds. Tune the TTL to your tolerance. |
| **Explicit invalidation** | On write: `db.update()` then `cache.del(key)` | Near-immediate | Medium | You must find *every* key affected by a write. Miss one derived key (e.g. `feed:user:5` contains post 9) and it's stale forever. |
| **Versioned keys** | Key includes a version: `user:7:v12`. On write, bump the version. | Immediate and **race-free** | Medium | Old versions linger in memory until evicted (waste, but harmless). Needs a version counter. |
| **Write-through** | Update the cache in the same operation as the DB | Immediate | Medium | Only works for keys you can compute at write time. Doesn't help with derived/aggregated keys. |

**In practice:** use **TTL as your safety net** (so nothing can ever be stale forever, even if you have an invalidation bug), plus **explicit delete-on-write** for the keys you can identify, plus **versioned keys** for anything where a race would be catastrophic. The TTL is the seatbelt — always wear it.

### 7. Redis vs Memcached

| | **Redis** | **Memcached** |
|---|---|---|
| **Data types** | Strings, hashes, lists, sets, sorted sets, bitmaps, HyperLogLog, streams, geo | Strings (opaque blobs) only |
| **Persistence** | Yes — RDB snapshots + AOF log. Survives restart. | No. Restart = empty. |
| **Replication / HA** | Yes — replicas, Redis Sentinel, Redis Cluster | No (client-side sharding only) |
| **Threading** | Single-threaded core (multi-threaded I/O in 6+) | Truly multi-threaded |
| **Multi-core scaling** | Run multiple instances / cluster | Scales up on one box naturally |
| **Eviction policies** | 8 (LRU, LFU, TTL, random, volatile variants) | LRU only |
| **Extras** | Pub/sub, Lua scripting, transactions, `SETNX` locks, sorted-set leaderboards, rate limiting | Nothing. It caches. |
| **Memory efficiency** | Slightly more overhead per key | Slightly leaner for plain string blobs |
| **Ops complexity** | Higher | Very low |

**When to pick Memcached:** you want a pure, dumb, huge, multi-threaded string cache and nothing else, and you value simplicity. (Facebook famously runs a heavily modified Memcached at enormous scale.)

**When to pick Redis:** basically every other time. The extra data structures (a sorted set is a leaderboard; a set is a rate limiter; `SETNX` is a distributed lock) mean Redis solves five problems, not one. **Default to Redis.** In an interview, say "Redis, because I also want it for rate limiting and the leaderboard, and I don't want a second system."

### 8. Node.js: a cache-aside wrapper with single-flight de-duplication

This is the code worth actually knowing. It fixes the stampede *within a process* with an in-memory promise map, and *across processes* it dramatically cuts the herd because each server only sends one query.

```javascript
// cache.js — production-shaped cache-aside with single-flight, jitter,
//            negative caching, and fail-open behaviour.
import { createClient } from 'redis';

const NEGATIVE = '__NULL__';   // sentinel so we can cache "row does not exist"

export class Cache {
  #redis;
  #inflight = new Map();       // key -> Promise. THIS is the single-flight de-duplicator.

  constructor(redisUrl) {
    this.#redis = createClient({ url: redisUrl });
    this.#redis.on('error', (e) => console.error('[cache] redis error', e.message));
  }

  async connect() { await this.#redis.connect(); }

  // Jitter every TTL so a batch of keys written together can't expire together (avalanche).
  #ttlWithJitter(seconds, pct = 0.2) {
    const spread = seconds * pct;
    return Math.max(1, Math.floor(seconds - spread + Math.random() * 2 * spread));
  }

  /**
   * Read-through with single-flight.
   *   key      : cache key
   *   loader   : async () => value   — how to fetch from the source of truth on a miss
   *   ttl      : base seconds; jitter is applied automatically
   */
  async get(key, loader, ttl = 300) {
    // 1. Fast path: is it in Redis?
    //    Fail OPEN — if Redis is down we still serve traffic (slowly) from the DB.
    let raw = null;
    try {
      raw = await this.#redis.get(key);
    } catch (err) {
      console.warn(`[cache] read failed for ${key}, falling through to DB`);
      return loader();
    }

    if (raw !== null) {
      return raw === NEGATIVE ? null : JSON.parse(raw);   // HIT (possibly a cached miss)
    }

    // 2. MISS. Is another request in this process ALREADY loading this key?
    //    If so, don't start a second DB query — just await the one in flight.
    //    10,000 concurrent misses collapse into exactly ONE database query.
    if (this.#inflight.has(key)) {
      return this.#inflight.get(key);
    }

    // 3. We are the elected loader. Register our promise BEFORE awaiting,
    //    so anyone who arrives mid-flight finds it. (No await between has() and set().)
    const promise = this.#load(key, loader, ttl)
      .finally(() => this.#inflight.delete(key));   // always clean up, even on throw

    this.#inflight.set(key, promise);
    return promise;
  }

  async #load(key, loader, ttl) {
    const value = await loader();

    try {
      if (value === null || value === undefined) {
        // NEGATIVE CACHING: remember that this key doesn't exist, but only briefly —
        // the row may be created a second from now. Kills cache penetration.
        await this.#redis.set(key, NEGATIVE, { EX: this.#ttlWithJitter(60) });
      } else {
        await this.#redis.set(key, JSON.stringify(value), { EX: this.#ttlWithJitter(ttl) });
      }
    } catch {
      // A failed cache WRITE must never fail the user's request. Swallow it.
    }
    return value;
  }

  /** Delete-on-write. Idempotent and race-free — unlike SET-on-write. */
  async invalidate(...keys) {
    try { await this.#redis.del(keys); } catch { /* TTL is the backstop */ }
  }
}
```

Using it:

```javascript
const cache = new Cache('redis://localhost:6379');
await cache.connect();

// READ — cache-aside, stampede-proof, penetration-proof.
async function getUser(id) {
  return cache.get(
    `user:${id}`,
    () => db.query('SELECT * FROM users WHERE id = $1', [id]),
    300 // 5 min base TTL (± jitter)
  );
}

// WRITE — DB first (source of truth), then invalidate. Never SET.
async function renameUser(id, name) {
  await db.query('UPDATE users SET name = $2 WHERE id = $1', [id, name]);
  await cache.invalidate(`user:${id}`, `user:${id}:profile`);
}

// Prove the single-flight works: 1000 concurrent cold reads → exactly 1 DB query.
await Promise.all(Array.from({ length: 1000 }, () => getUser(42)));
```

---

## Visual / Diagram description

### Diagram 1: The full cache stack, and where a request dies

```
    USER
     │  GET /api/products/42
     ▼
┌─────────────────┐
│ 1. BROWSER CACHE│  Cache-Control: max-age=60         hit → 0 ms, no network at all
│    (disk/memory)│
└────────┬────────┘  miss
         ▼
┌─────────────────┐
│ 2. CDN EDGE PoP │  Cloudflare/Akamai, 20 km away     hit → ~20 ms
│    (Mumbai)     │
└────────┬────────┘  miss  ─────────────────────┐
         ▼                                       │  (long-haul network, 100-200 ms)
┌─────────────────┐                              │
│ 3. REVERSE PROXY│  Nginx / Varnish             │     hit → ~2 ms
│    (in your DC) │                              │
└────────┬────────┘  miss                        │
         ▼                                       │
┌─────────────────────────────────────┐          │
│         APP SERVER (Node.js)        │◀─────────┘
│                                     │
│   ┌───────────────────────────┐     │
│   │ 4. L1 IN-PROCESS CACHE    │     │  a JS Map / LRU        hit → ~100 ns  ★fastest
│   │    (this process's RAM)   │     │
│   └────────────┬──────────────┘     │
└────────────────┼────────────────────┘
                 │ miss
                 ▼
      ┌──────────────────────┐
      │ 5. DISTRIBUTED CACHE │  Redis / Memcached cluster      hit → ~1 ms
      │    (shared by ALL    │  ── shared, so all servers
      │     app servers)     │     agree on the value
      └──────────┬───────────┘
                 │ miss
                 ▼
      ┌──────────────────────┐
      │ 6. DB BUFFER POOL    │  Postgres shared_buffers        hit → ~0.5 ms
      │    (inside the DB)   │  (pages already in the DB's RAM)
      └──────────┬───────────┘
                 │ miss
                 ▼
      ┌──────────────────────┐
      │      DISK / SSD      │  the actual table               ~10-50 ms  ★slowest
      └──────────────────────┘
```

**What this shows:** the request falls through layers, getting slower at each step. Layers 1–3 are configured with **HTTP headers**; layers 4–5 are configured with **your code**; layer 6 is the database's own business. Each layer only ever sees the *misses* of the layer above it — which is why a 50% browser hit rate followed by a 90% CDN hit rate leaves your origin with just 5% of the traffic.

Note that layers 4 and 5 form the classic **L1/L2 pair**: L1 is per-server (blazing fast, but each server has its own possibly-divergent copy — that's the cost); L2 is shared (slower, but consistent across servers). Putting a 1-second L1 in front of Redis is the standard fix for hot keys.

### Diagram 2: Stampede — before and after single-flight

```
 WITHOUT SINGLE-FLIGHT                    WITH SINGLE-FLIGHT
 (key expires, 5 requests miss)           (key expires, 5 requests miss)

  R1 ──miss──┐                             R1 ──miss──┬─▶ [elected loader] ──▶ DB
  R2 ──miss──┤                             R2 ──miss──┤        │
  R3 ──miss──┼──▶ DB  (5 identical         R3 ──miss──┼── wait │  (1 query)
  R4 ──miss──┤        queries!)            R4 ──miss──┤   on   │
  R5 ──miss──┘                             R5 ──miss──┴─ promise│
                                                              ▼
      ┌──────────┐                                    ┌──────────┐
      │    DB    │ 5x load                            │    DB    │ 1x load
      └──────────┘                                    └──────────┘
                                                              │
                                              all 5 resolve from the SAME promise
```

At 10,000 concurrent requests instead of 5, the left side is an outage and the right side is a non-event. That `#inflight` Map is the whole difference.

---

## Real world examples

### 1. Facebook — Memcached at absurd scale

Facebook runs one of the largest Memcached deployments on earth (their "look-aside" — i.e. cache-aside — tier sits in front of MySQL). Two of their documented engineering problems map directly onto this doc:

- **The stampede.** Their published solution is a *lease*: on a miss, Memcached hands exactly one client a lease token authorizing it to fetch from the DB and set the key. Other clients that miss are told "wait, someone's fetching" and briefly retry. That is single-flight, implemented in the cache server itself.
- **Invalidation across regions.** They invalidate caches by piggybacking delete commands on the MySQL replication stream — when a write commits and replicates to another region, that region's caches get the delete. This makes invalidation a property of the data pipeline rather than something application code has to remember.

### 2. Twitter/X — the timeline is a cache

A celebrity's tweet cannot be fanned out to 100 million follower timelines on write in real time. The read path is dominated by cached, materialized timelines held in Redis. The hot-key problem is explicit and structural: a small number of accounts have follower counts orders of magnitude above the median, so they are handled on a *different code path* (merged in at read time rather than fanned out on write). This is the "hot key" problem from §6 elevated to an architectural decision.

### 3. Stack Overflow — aggressive multi-layer caching on modest hardware

Stack Overflow has publicly described serving enormous traffic from a famously small number of servers, and caching is central to how: a local in-process cache per web server (L1), a shared Redis tier (L2), and heavy use of HTTP caching for anonymous users — the vast majority of their traffic is logged-out readers hitting pages that can be cached wholesale. The lesson worth stealing: **segment your traffic.** Anonymous users can be served from a shared cache; logged-in users need personalization and can't. Cache the 90% and let the 10% hit the DB.

---

## Trade-offs

| Benefit of caching | The price you pay |
|---|---|
| 10–50× lower read latency | **Staleness** — the cache can serve data that is no longer true |
| 90–99% less database load | **A new failure mode** — cache down means a thundering herd at the DB |
| Cheaper: RAM per QPS is far cheaper than DB per QPS | **Complexity** — invalidation logic, a whole extra system to run and monitor |
| Absorbs traffic spikes | **Cold-start problem** — a fresh/restarted cache offers zero protection when you need it most |
| Lets you scale reads horizontally | **Consistency bugs** that are nondeterministic and miserable to debug |

| Decision | Choose A | Choose B |
|---|---|---|
| Local in-memory vs Redis | **Local**: nanoseconds, zero network, no extra infra — but each server diverges and you can't invalidate across the fleet | **Redis**: one shared truth, invalidate once, survives deploys — but 1 ms and an extra system |
| Short TTL vs long TTL | **Short**: fresher, but lower hit ratio → more DB load | **Long**: higher hit ratio → less DB load, but staler data |
| Cache-aside vs write-through | **Cache-aside**: resilient, only caches what's used, but first read is always a miss | **Write-through**: always fresh, but slows writes and caches data nobody reads |
| Delete-on-write vs update-on-write | **Delete**: idempotent, race-free, simple — next read repopulates | **Update**: avoids one miss, but races with concurrent readers and can persist a stale value |

**Rule of thumb:** Start with **cache-aside + a jittered TTL + delete-on-write**, and reach for anything fancier only when a specific problem forces you to. Cache the *read-heavy, change-rarely, expensive-to-compute* things first — that's where the 80/20 lives. And whatever else you do, **always set a TTL**: it is the backstop that guarantees no invalidation bug can make your data wrong *forever*.

---

## Common interview questions on this topic

### Q1: "You add Redis in front of your database. Walk me through what happens when a very popular key expires."
**Hint:** This is the stampede question. Say the word *thundering herd*. Thousands of concurrent requests miss simultaneously and all query the DB for the same row, which can take the DB down — and since the DB is down, nothing can repopulate the cache, so it doesn't self-heal. Give three fixes ranked: (1) **single-flight/mutex** — one loader, the rest wait; (2) **probabilistic early expiry** — refresh before expiry so a miss never happens; (3) **stale-while-revalidate** — serve the stale value and refresh in the background. Mention that (3) is usually the best UX if slight staleness is acceptable.

### Q2: "How do you keep the cache consistent with the database?"
**Hint:** Say honestly: you don't get strong consistency cheaply, you get *bounded staleness*. Layer three things: **TTL** as the safety net (nothing stale forever), **explicit delete-on-write** for keys you can identify (delete, never update — updating races), and **versioned keys** (`user:7:v12`) for anything where a race is unacceptable. Then name the failure case they're fishing for: derived/aggregate keys (a user's name change should invalidate `user:7` *and* every cached feed that embeds their name) — that's why explicit invalidation is hard, and why a short TTL on derived data is often the pragmatic answer.

### Q3: "An attacker hammers your API with `GET /user/<random-uuid>`. What happens?"
**Hint:** **Cache penetration.** Every request misses the cache (nothing to cache), misses the DB (no row), and nothing gets stored — so 100% of that traffic reaches your database. The cache provides zero protection. Fixes: **negative caching** (cache the null with a short TTL, e.g. 60s), and a **Bloom filter** of valid IDs in front — it never gives false negatives, so a "definitely not present" answer lets you reject the request without touching the DB, at ~10 bits per key. Add input validation and rate limiting as defence in depth.

### Q4: "LRU or LFU?"
**Hint:** Default LRU — it's cheap and temporal locality is real. Switch to LFU when your workload has **scans**: a batch job or crawler touching a million rows once each will, under LRU, evict your entire hot set (each scanned row becomes "most recently used"). Under LFU those rows have a frequency of 1 and evict *each other*, leaving the hot keys untouched. **LFU is scan-resistant.** Caveat: naive LFU lets a formerly-popular key squat forever, so you want a *decaying* counter — which is exactly what Redis's `allkeys-lfu` implements.

### Q5: "Your cache hit ratio is 70%. Is that good?"
**Hint:** Refuse the yes/no and do the math. 70% means the DB still sees **30%** of traffic — at 10k rps that's 3,000 qps, which will kill most single Postgres instances. Getting to 95% would drop it to 500 qps. Then ask *why* it's 70%: (a) TTL too short → lengthen it; (b) cache too small → keys are being evicted before re-read, check the eviction rate metric; (c) genuinely low locality → caching may be the wrong tool; (d) a scan is polluting the cache → switch to LFU. The right instinct is to look at *eviction rate* alongside hit rate — high evictions means you're memory-bound, not locality-bound.

---

## Practice exercise

### Design and size the cache for a product-detail page

You're building an e-commerce product page. Given:

- 50 million products in Postgres.
- 20,000 page views/second at peak.
- Rendering one product page requires 4 DB queries (product row, seller info, 20 reviews, inventory count), totaling ~60 ms.
- Access is heavily skewed: **1% of products get 90% of the views.**
- Average serialized product payload: **4 KB**.
- Inventory count changes every few seconds. Reviews change a few times a day. Product name/description changes maybe once a month.

**Produce:**

1. **The estimation.** How much RAM do you need to cache the hot 1% of products? (Show the arithmetic: 50M × 1% × 4 KB = ?) Now: what if you wanted to cache 10%? Does that still fit on one Redis node?
2. **The hit ratio and the DB load.** If you cache the hot 1%, and access follows the 90/10 skew, what's your expected hit ratio? At 20,000 rps, how many qps does the DB now see? Compare with the uncached number.
3. **The TTL table.** Draw a table with a row for each of the four data types (product info, seller, reviews, inventory) and pick a **different TTL for each**, with a one-line justification. Hint: inventory does *not* belong in the same cache entry as the product description. Why not?
4. **Handle the failure modes.** Write down, for this specific system, your concrete answer to each of: (a) a flash sale makes one product 1000× hotter than any other — what breaks and what do you do? (b) You deploy and warm 500k keys at once with a 1-hour TTL — what happens in 60 minutes, and how do you prevent it? (c) A bot requests 1M random product IDs that don't exist — what happens and what do you add?
5. **Write the code.** Extend the `Cache` class from §8 with a `getMany(keys, loader)` that does a Redis `MGET`, works out which keys missed, and calls the loader **once** with the full list of missing keys (not once per key). This is how you avoid N+1 cache lookups on a listing page.

Aim for 30–40 minutes. Write the numbers out longhand — the arithmetic *is* the exercise.

---

## Quick reference cheat sheet

- **Why caching works:** temporal + spatial locality → the 80/20 rule. Roughly 20% of data serves 80% of requests; on the web it's often 1% serving 90%.
- **The numbers:** local memory ~100 ns · Redis ~1 ms · DB query ~50 ms. Redis is **50× faster than the DB**; local memory is **10,000× faster than Redis**.
- **The layers, top to bottom:** browser → CDN → reverse proxy → app in-process (L1) → distributed cache (L2, Redis) → DB buffer pool → disk. Each layer only sees the misses of the one above.
- **Cache-aside (lazy loading) is the default:** check cache → miss → read DB → populate cache → return. Resilient: if the cache dies, reads still work (just slowly).
- **Effective latency** = `(hit% × cache_ms) + (miss% × (cache_ms + db_ms))`. A miss pays *both*.
- **Every extra 9 in the hit ratio divides DB load by 10.** 90% → 99% → 99.9%. Caching is a capacity play first, a latency play second.
- **Write strategies:** write-through (fresh, slow writes) · write-back (fast, **can lose data**) · write-around (write-heavy read-rarely) · **delete-on-write + cache-aside is the real-world default**.
- **Delete, don't update, on write.** Updating races with concurrent readers and can leave a stale value; deleting is idempotent.
- **Eviction:** LRU is the default. **LFU is scan-resistant** — use it when batch jobs would otherwise flush your hot set. Set `maxmemory-policy allkeys-lru`, never leave it on `noeviction` for a cache.
- **Stampede (thundering herd):** hot key expires → 10k simultaneous misses → DB dies. Fix with **single-flight**, probabilistic early expiry, or stale-while-revalidate.
- **Penetration:** requests for keys that don't exist bypass the cache entirely. Fix with **negative caching** (short TTL) and a **Bloom filter**.
- **Avalanche:** mass simultaneous expiry. Fix by **jittering every TTL** (±20%). Never use a constant TTL across a batch of keys.
- **Hot key:** one key saturates one shard. Fix with an **L1 local cache** (even 1s TTL) or key-splitting across shards.
- **Always set a TTL.** It's the seatbelt: it bounds the damage from every invalidation bug you haven't found yet.
- **Redis vs Memcached:** default to **Redis** — you get data structures, persistence, replication, pub/sub, and locks for free. Memcached only if you need a dumb multi-threaded string cache.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [58 — API Gateway](./58-api-gateway.md) — the front door, and a place caching often lives |
| **Next** | [60 — CDN](./60-cdn.md) — caching pushed all the way to the network edge, next to the user |
| **Related** | [12 — Numbers Every Engineer Should Know](./12-numbers-every-engineer-should-know.md) — the latency table that justifies every cache you'll ever add |
| **Related** | [09 — CAP Theorem](./09-cap-theorem.md) — a cache is a replica, so cache staleness is just eventual consistency wearing a hat |
