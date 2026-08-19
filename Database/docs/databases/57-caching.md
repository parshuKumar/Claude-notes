# 57 — Caching
## Phase: Denormalisation & Scale

---

## ELI5 — The Simple Analogy

You keep asking a colleague across the office for the current price list. So you write it on a sticky note on your monitor.

Now you never walk over. Brilliant — until the price changes and your sticky note doesn't. Three things follow, and they are the entire topic:

1. ★ **Somebody has to remove the sticky note.** Either you write a date on it and throw it away after an hour (*a TTL*), or the colleague comes over and rips it off when the price changes (*invalidation*). The first is easy and always slightly wrong. The second is correct and **very hard to get right**, because they have to remember every sticky note in the building.

2. ★ **When the note goes missing, everybody walks over at once.** If forty people had that note and it expires at the same moment, forty people arrive at your colleague's desk simultaneously. She was fine serving one person a minute; she is not fine serving forty at once. **The cache didn't just stop helping — it created a spike that the system never faced before it existed.**

3. ★ **You now have two sources of truth and no way to tell which is right.** The colleague's list is authoritative. Your note is a guess. If they disagree, nothing in the building tells you.

Denormalisation (Topics 53–55) put a copy *inside* the database, where the engine could help. **Caching puts the copy outside, where nothing can.**

---

## Where this fits in the big picture

```
   53–55 denormalisation — copies INSIDE the database
   56 materialised views — async copies the ENGINE maintains
                          │
                          ▼
        ┌──────────────────────────────────────────────┐
        │ 57 CACHING ← YOU ARE HERE                    │
        │ ★ copies OUTSIDE, where nothing enforces      │
        └────────────────────┬─────────────────────────┘
                             ▼
              58 read replicas · 61 hot rows
              68 CAP & consistency models
```

Each step in this phase moves the copy further from the source of truth and buys more speed for less safety. **Caching is the far end of that spectrum** — the fastest reads available, and the only mechanism where the database cannot help you at all.

---

## What is this?

Storing a computed or fetched result somewhere faster, and serving subsequent requests from there.

**The layers, and where each actually helps:**

```
 ★ THERE ARE SIX, AND PEOPLE USUALLY ONLY THINK ABOUT ONE.

 ① ★ THE BUFFER POOL (shared_buffers)          — Topic 07
    already caching your data. ★ Free. Correct. Invisible.
    ⇒ ★ CHECK THIS FIRST. A 99.4% hit ratio means your "slow"
      query is not I/O-bound and Redis will not help.
 ② THE OS PAGE CACHE
    a second layer under the buffer pool
 ③ ★ PREPARED STATEMENTS / PLAN CACHE
    saves parse+plan time, not I/O
 ④ ★ APPLICATION-LOCAL (in-process LRU)
    ★ nanoseconds. No network. But: per-instance, so N instances
      means N copies and N invalidation problems.
 ⑤ ★ A SHARED CACHE (Redis / Memcached)
    ~0.2 ms over the network. Shared across instances.
    ⇒ ★ THE ONE PEOPLE MEAN BY "CACHING", AND THE ONE WITH THE
      MOST FAILURE MODES.
 ⑥ ★ CDN / HTTP CACHE
    the cheapest cache is the request you never receive.
    ⇒ ★ CONSISTENTLY UNDERUSED. An `ETag` + `Cache-Control` on a
      product page removes more load than any Redis layer.
```

**The two invalidation strategies, and there really are only two:**

| | TTL (expiry) | Explicit invalidation |
|---|---|---|
| Correctness | ★ always up to TTL stale | ★ correct *if* every write path remembers |
| Complexity | trivial | ★ high, and it grows with the codebase |
| Failure mode | stale reads | ★ **missed invalidation = permanently stale** |
| ⇒ | the default | only where staleness is unacceptable |

---

## Why does it matter for a backend developer?

```
 ★ BECAUSE A CACHE IS THE ONLY OPTIMISATION THAT CAN MAKE YOUR
   SYSTEM STRICTLY WORSE THAN NOT HAVING IT.

 An index is never worse than no index (for reads).
 A matview is never wrong, only old.
 ★ A CACHE CAN:
   • serve permanently wrong data (a missed invalidation)
   • ★ CREATE A LOAD SPIKE THE DATABASE NEVER FACED BEFORE
     (a stampede)
   • ★ TAKE DOWN THE SYSTEM WHEN IT FAILS, because the database
     was scaled for the cached load, not the real load
   • hide a missing index for two years, until the cache is
     unavailable for 90 seconds and everything collapses

 ⇒ ★ AND THE THIRD ONE IS THE ONE THAT ENDS CAREERS:
   a cache that has been absorbing 97% of reads for a year has
   silently become a LOAD-BEARING COMPONENT. The database can no
   longer serve the traffic. A cache restart is now an outage.
```

---

## The physical reality

### Check the buffer pool before adding a cache

```
 ★ POSTGRESQL IS ALREADY A CACHE. Most "we need Redis" arrives
   without anyone checking whether the data is already in memory.

   SELECT
     sum(heap_blks_hit)  AS hits,
     sum(heap_blks_read) AS reads,
     round(100.0*sum(heap_blks_hit)/
           nullif(sum(heap_blks_hit)+sum(heap_blks_read),0), 2) AS hit_pct
   FROM pg_statio_user_tables;

 ⇒ hit_pct > 99% ⇒ ★ your data is already in RAM. The query is
   slow for a CPU reason (a bad plan, a big sort, an aggregate),
   and a network round trip to Redis will make it SLOWER, not
   faster, unless it eliminates the computation entirely.

 ★ THE LATENCY LADDER — memorise the order of magnitude:
   buffer pool hit         ★ ~0.0001 ms   (a memcpy)
   in-process LRU          ★ ~0.0002 ms
   PostgreSQL, indexed,
     same host             ★ ~0.15 ms     (mostly the round trip)
   Redis, same DC          ★ ~0.2 ms      (mostly the round trip)
   PostgreSQL, disk read   ~0.5–5 ms
   PostgreSQL, big aggregate ~100–10,000 ms

 ⇒ ★ REDIS IS NOT FASTER THAN AN INDEXED POSTGRESQL LOOKUP.
   BOTH ARE DOMINATED BY THE NETWORK ROUND TRIP.
   ⇒ ★ A CACHE ONLY WINS WHEN IT ELIMINATES *COMPUTATION*, NOT
     WHEN IT ELIMINATES *STORAGE ACCESS*.
```

### The three stampede modes

```
 ★ ① THUNDERING HERD (a single hot key expires)
    one popular key, 4,000 rps, TTL 60 s.
    at the TTL boundary: ★ 4,000 concurrent requests all miss and
    all run the same 400 ms query.
    ⇒ ★ 4,000 × 400 ms of database work in one instant, for ONE
      value.
    ⇒ and while they run, the next 4,000 arrive.
    ⇒ ★ MEASURED: p99 180 ms → 14,000 ms, every 60 seconds.

 ★ ② CACHE STAMPEDE / SYNCHRONISED EXPIRY
    10,000 keys all populated at deploy time with TTL 3600.
    ⇒ ★ they ALL expire in the same second, one hour later.
    ⇒ the database receives an hour's worth of misses at once.
    ⇒ ★ FIX: JITTER THE TTL. ttl = base × (0.75 + random()×0.5).

 ★ ③ CACHE PENETRATION (misses that can never be cached)
    requests for keys that DON'T EXIST — a scraper enumerating
    product IDs, or an attacker.
    ⇒ every one is a miss, every one hits the database, and
      ★ nothing is ever cached because there's nothing to cache.
    ⇒ ★ FIX: CACHE THE NEGATIVE RESULT (a short TTL null marker),
      or a Bloom filter of existing keys.

 ⇒ ★ THE FIX FOR ① IS THE IMPORTANT ONE AND HAS A NAME:
   SINGLE-FLIGHT (request coalescing). One request per key
   recomputes; the rest wait for it.
   ⇒ 4,000 database queries become ★ 1.
```

### The consistency problem — why explicit invalidation is hard

```
 ★ THE RACE THAT CORRUPTS A CACHE PERMANENTLY:

  t1  request A: cache MISS on key `product:7`
  t2  request A: SELECT … → price = 49900
  t3  request B: UPDATE products SET price = 59900 WHERE id = 7
  t4  request B: DELETE cache key `product:7`     ← invalidate
  t5  request A: SET cache `product:7` = 49900    ← ★ WRITES STALE
  ⇒ ★ THE CACHE NOW HOLDS 49900 FOREVER (until the TTL, if any).
    The invalidation happened BEFORE the stale write landed.

 ★ THIS IS NOT RARE. It happens whenever a read and a write
   interleave, which at 4,000 rps is constantly.

 ⇒ THE THREE MITIGATIONS, WEAKEST TO STRONGEST:
   ① ★ ALWAYS SET A TTL, EVEN WITH EXPLICIT INVALIDATION.
      ⇒ turns "permanently wrong" into "wrong for at most N
        seconds". ★ This alone removes most of the danger.
   ② ★ DELETE, DON'T UPDATE, ON WRITE.
      writing the new value into the cache from the write path
      has the same race PLUS a second one (two writers racing).
      ⇒ deleting is idempotent and always safe.
   ③ ★ VERSIONED KEYS — the strongest, and it removes the race
      entirely:
        key = `product:7:v${version}`
      ⇒ a write bumps `version` in the same transaction
      ⇒ ★ a stale writer writes to an OLD KEY that nobody reads
      ⇒ no deletion needed; old keys expire naturally
      ⇒ ★ costs: one extra read to get the version, or embed it
        in the parent object you already fetched.

 ★ AND THE ONE THAT ELIMINATES THE PROBLEM:
   ④ DON'T CACHE MUTABLE DATA. Cache things that are immutable
      by construction — a rendered invoice PDF, a completed order,
      yesterday's report. ★ Immutable data needs no invalidation.
```

### What happens when the cache goes away

```
 ★ THE FAILURE MODE PEOPLE DISCOVER IN PRODUCTION.

 STEADY STATE:
   4,200 rps to the API
   97% cache hit rate
   ⇒ ★ 126 rps reaching PostgreSQL
   ⇒ the database looks bored. Someone downsizes it.

 REDIS RESTARTS (or fails over, or is flushed):
   ⇒ hit rate 97% → 0%
   ⇒ ★ 4,200 rps reaching PostgreSQL — 33× the load it has
     handled for a year
   ⇒ connection pool exhausted in ★ under 2 seconds
   ⇒ every request queues, times out, and RETRIES
   ⇒ ★ retries multiply the load further
   ⇒ ★ TOTAL OUTAGE, and it does not self-recover, because the
     cache cannot repopulate while the database is saturated.

 ⇒ ★ THE THREE DEFENCES, AND YOU NEED ALL THREE:
   ① ★ SINGLE-FLIGHT — one recompute per key, not 4,000
   ② ★ A CONCURRENCY LIMIT / CIRCUIT BREAKER on the database path
      ⇒ shed load deliberately rather than collapsing
   ③ ★ TEST IT. Flush the cache in staging under production-shaped
      load and measure what actually happens.
      ⇒ ★ IF YOU HAVE NEVER DONE THIS, YOU DO NOT KNOW WHETHER
        YOUR CACHE IS LOAD-BEARING.
```

---

## How it works — step by step

### The patterns, and which to use

```
 ★ ① CACHE-ASIDE (lazy loading) — the default, use this
    read:  v = cache.get(k); if miss { v = db(); cache.set(k,v,ttl) }
    write: db.update(); cache.del(k)
    ✓ simple · only caches what's asked for · survives cache loss
    ✗ the first request after a miss is slow
    ✗ ★ the stale-write race above (mitigate with TTL + versioning)

 ② READ-THROUGH — the cache library does the fetch
    ⇒ same semantics as cache-aside, less application code

 ③ ★ WRITE-THROUGH — write to cache and DB together
    ✓ the cache is never stale
    ✗ ★ every write pays cache latency
    ✗ ★ caches data nobody reads
    ✗ ★ the two writes are not atomic — a dual write (Topic 52)

 ✗ ④ WRITE-BEHIND — write to cache, flush to DB later
    ★ THE CACHE BECOMES THE SOURCE OF TRUTH.
    ⇒ ★ DATA LOSS ON CACHE FAILURE. Almost never acceptable for
      anything you would put in a database.

 ⇒ ★ USE CACHE-ASIDE. The others solve problems most systems
   don't have and introduce ones they do.
```

### Cache-aside, done properly

```js
const DEFAULT_TTL = 300;

// ★ jitter — prevents synchronised expiry (stampede mode ②)
const jitter = (ttl) => Math.floor(ttl * (0.75 + Math.random() * 0.5));

// ★ single-flight — one recompute per key per process (mode ①)
const inflight = new Map();

async function cached(key, ttl, loader) {
  const hit = await redis.get(key);
  if (hit !== null) {
    metrics.increment('cache.hit', { key: keyClass(key) });
    return hit === NULL_MARKER ? null : JSON.parse(hit);
  }
  metrics.increment('cache.miss', { key: keyClass(key) });

  // ★ if another request in THIS process is already loading, wait
  if (inflight.has(key)) return inflight.get(key);

  const p = (async () => {
    try {
      // ★ cross-process single-flight: a short lock so only ONE
      //   instance recomputes. The rest briefly serve stale or wait.
      const gotLock = await redis.set(`lock:${key}`, '1', 'NX', 'EX', 10);
      if (!gotLock) {
        await sleep(50);
        const retry = await redis.get(key);
        if (retry !== null)
          return retry === NULL_MARKER ? null : JSON.parse(retry);
        // fall through and load anyway — better slow than wrong
      }

      const value = await loader();

      // ★ cache the NEGATIVE result too (mode ③: penetration)
      if (value == null) {
        await redis.set(key, NULL_MARKER, 'EX', jitter(30));
        return null;
      }
      await redis.set(key, JSON.stringify(value), 'EX', jitter(ttl));
      return value;
    } finally {
      inflight.delete(key);
      redis.del(`lock:${key}`).catch(() => {});
    }
  })();

  inflight.set(key, p);
  return p;
}
```

### Versioned keys — invalidation without the race

```sql
-- the version lives in the database, bumped in the same transaction
ALTER TABLE products ADD COLUMN cache_version integer NOT NULL DEFAULT 1;

CREATE OR REPLACE FUNCTION bump_cache_version() RETURNS trigger AS $$
BEGIN
  NEW.cache_version := OLD.cache_version + 1;
  RETURN NEW;
END $$ LANGUAGE plpgsql;

CREATE TRIGGER trg_products_cache_version
  BEFORE UPDATE ON products
  FOR EACH ROW EXECUTE FUNCTION bump_cache_version();
-- ★ a trigger, so no write path can forget (Topic 53)
```

```js
// ★ the version is part of the key. A stale writer writes to a key
//   nobody will ever read.
async function getProduct(id) {
  const { rows: [v] } = await pool.query(
    'SELECT cache_version FROM products WHERE id = $1', [id]);
  if (!v) return null;

  return cached(`product:${id}:v${v.cache_version}`, 3600, async () => {
    const { rows } = await pool.query(
      `SELECT p.*, s.name AS seller_name FROM products p
         JOIN sellers s ON s.id = p.seller_id WHERE p.id = $1`, [id]);
    return rows[0] ?? null;
  });
}
// ★ NOTE: this still costs one DB round trip for the version.
//   Worth it only when the cached value is EXPENSIVE to compute.
//   For a single-row fetch it is pointless — just query the row.
```

### The HTTP cache — the layer people skip

```js
// ★ the cheapest cache is the request you never receive.
app.get('/api/products/:id', async (req, res) => {
  const product = await getProduct(req.params.id);
  if (!product) return res.status(404).end();

  const etag = `W/"${product.id}-${product.cache_version}"`;
  res.set('ETag', etag);
  res.set('Cache-Control', 'public, max-age=60, stale-while-revalidate=300');
  // ★ stale-while-revalidate: the CDN serves the stale copy
  //   INSTANTLY while refreshing in the background.
  //   ⇒ this is single-flight, implemented by the CDN, for free.

  if (req.get('If-None-Match') === etag) return res.status(304).end();
  res.json(product);
});
```

---

## Concept breakdown

```
★ SIX LAYERS — CHECK THEM IN THIS ORDER
├── ① ★ shared_buffers      already caching. ★ CHECK hit_pct FIRST.
├── ② OS page cache
├── ③ prepared statements   saves plan time, not I/O
├── ④ in-process LRU        ★ nanoseconds, but N instances = N copies
├── ⑤ Redis/Memcached       ~0.2 ms — ★ the same as a PG round trip
└── ⑥ ★ CDN / HTTP          ★ the cheapest cache is the request you
                              never receive. Consistently underused.

★ THE CENTRAL FACT
   Redis is NOT faster than an indexed PostgreSQL lookup — both are
   dominated by the network round trip.
   ⇒ ★ A CACHE WINS BY ELIMINATING COMPUTATION, NOT STORAGE ACCESS.

★ THE THREE STAMPEDE MODES
├── ① THUNDERING HERD    one hot key expires → 4,000 concurrent
│                        recomputes  ⇒ ★ SINGLE-FLIGHT
├── ② SYNCHRONISED EXPIRY  10k keys expire together
│                        ⇒ ★ JITTER: ttl × (0.75 + rand×0.5)
└── ③ PENETRATION       requests for keys that don't exist
                        ⇒ ★ CACHE THE NEGATIVE, short TTL

★ THE STALE-WRITE RACE — permanent corruption
   miss → read → (another txn writes + invalidates) → set stale
   ⇒ ① ★ ALWAYS SET A TTL, even with explicit invalidation
     ② ★ DELETE on write, never UPDATE
     ③ ★ VERSIONED KEYS — removes the race entirely
     ④ ★ or cache only IMMUTABLE data — no invalidation needed

★ THE LOAD-BEARING CACHE — the career-ending failure
   97% hit rate ⇒ the DB sees 3% of traffic ⇒ someone downsizes it
   ⇒ cache restart ⇒ ★ 33× load instantly ⇒ pool exhausted in 2 s
   ⇒ retries multiply it ⇒ ★ cannot self-recover
   ⇒ DEFENCES: ★ single-flight · ★ a concurrency limit/breaker ·
     ★ TEST IT by flushing under load in staging

PATTERNS
├── ★ CACHE-ASIDE   the default. Use this.
├── read-through    same semantics, less code
├── write-through   ★ every write pays cache latency; a dual write
└── ✗ write-behind  ★ the cache becomes the source of truth ⇒ data loss

★ WHAT NOT TO CACHE
   anything a user reads back immediately after writing it
   anything where staleness is a correctness bug
   ★ anything a missing index would have fixed
```

---

## Diagrams

**Diagram 1 — big picture: the latency ladder, to scale**

```
  buffer pool hit        ▏                          ★ 0.0001 ms
  in-process LRU         ▏                          ★ 0.0002 ms
  PG indexed lookup      ███                         ★ 0.15 ms
  Redis GET (same DC)    ████                        ★ 0.20 ms
  PG disk read           ████████████                ~0.5 ms
  PG aggregate (40M)     ████████████████████████… ★ 8,400 ms
                                                     (off the chart)

 ★ READ THIS CAREFULLY:
   Redis is SLOWER than an indexed PostgreSQL lookup on the same
   host. Both are ~1 network round trip. The Redis "speed" people
   quote is against a SLOW query, not against a fast one.

 ⇒ ★ THE ONLY GAP WORTH CACHING IS THE LAST ONE:
   expensive COMPUTATION. If your query is 0.15 ms, a cache adds
   latency, adds a failure mode, and adds an invalidation problem,
   for nothing.
```

**Diagram 2 — data flow: the thundering herd, and single-flight**

```
 ✗ WITHOUT SINGLE-FLIGHT
   t=59.999  cache: product:7 = {…}   4,000 rps served from cache
   ─────────────────── TTL EXPIRES ───────────────────
   t=60.000  ┌─ req 1    MISS ──► DB ──► 400 ms query ─┐
             ├─ req 2    MISS ──► DB ──► 400 ms query ─┤
             ├─ req 3    MISS ──► DB ──► 400 ms query ─┤ ★ 4,000
             │  …                                       │  IDENTICAL
             └─ req 4000 MISS ──► DB ──► 400 ms query ─┘  QUERIES
   t=60.001  ★ 4,000 more requests arrive; the DB is saturated
             ★ pool exhausted, everything queues
             ★ p99 180 ms → 14,000 ms
   t=60.400  the first response finally lands and populates
             ⇒ ★ AND THIS REPEATS EVERY 60 SECONDS.

 ✓ WITH SINGLE-FLIGHT + JITTER
   t=60.000  ┌─ req 1    MISS ──► ★ acquires lock ──► DB (400 ms)
             ├─ req 2    MISS ──► lock held ──► ★ waits 50 ms, retries
             ├─ req 3    MISS ──► lock held ──► waits
             │  …                                (or serves stale)
             └─ req 4000 MISS ──► lock held ──► waits
   t=60.400  req 1 populates the cache; all 3,999 read it
             ⇒ ★ 1 DATABASE QUERY INSTEAD OF 4,000
             ⇒ p99 stays at 180 ms
   ★ AND jittered TTLs mean the 10,000 OTHER keys don't all expire
     in this same second.
```

**Diagram 3 — before/after: the stale-write race and versioned keys**

```
 ✗ DELETE-ON-WRITE — a race that corrupts permanently
 ┌───────────────────────────────────────────────────────────────┐
 │  REQ A (read)                    REQ B (write)                │
 │  ─────────────────               ─────────────────            │
 │  t1  GET product:7 → MISS                                     │
 │  t2  SELECT … → 49900                                         │
 │                                  t3  UPDATE price = 59900     │
 │                                  t4  DEL product:7            │
 │  t5  SET product:7 = ★ 49900                                  │
 │                                                                │
 │  ⇒ ★ THE CACHE HOLDS 49900. THE DATABASE HOLDS 59900.         │
 │    The invalidation ran BEFORE the stale write landed.         │
 │    ⇒ wrong until the TTL — and if there's no TTL, ★ FOREVER.  │
 └───────────────────────────────────────────────────────────────┘

 ✓ VERSIONED KEYS — the race cannot corrupt anything
 ┌───────────────────────────────────────────────────────────────┐
 │  REQ A (read)                    REQ B (write)                │
 │  ─────────────────               ─────────────────            │
 │  t1  version = 4                                              │
 │  t2  GET product:7:v4 → MISS                                  │
 │  t3  SELECT … → 49900                                         │
 │                                  t4  UPDATE price = 59900     │
 │                                      ★ trigger: version → 5   │
 │  t5  SET product:7:★v4 = 49900                                │
 │                                                                │
 │  ⇒ ★ A WROTE TO KEY v4. EVERY SUBSEQUENT READER LOOKS UP v5.  │
 │    The stale value is unreachable and expires on its own.      │
 │  ⇒ ★ NO DELETION NEEDED. NO RACE POSSIBLE.                    │
 │  ✗ COST: one extra read for the version — so this is only     │
 │    worth it when the cached value is EXPENSIVE to compute.     │
 └───────────────────────────────────────────────────────────────┘
```

---

## Example 1 — basic

**Check whether you need a cache at all.**
```sql
SELECT sum(heap_blks_hit) AS hits, sum(heap_blks_read) AS reads,
       round(100.0*sum(heap_blks_hit)/
             nullif(sum(heap_blks_hit)+sum(heap_blks_read),0), 2) AS hit_pct
  FROM pg_statio_user_tables;
```
```
   hits    | reads  | hit_pct
-----------+--------+---------
 884201188 | 412008 | ★ 99.95
   ★ the data is already in RAM. A cache will not save I/O.
     It can only save COMPUTATION.
```

**Measure the two things a cache could replace.**
```sql
-- ① a cheap indexed lookup
EXPLAIN (ANALYZE, BUFFERS)
SELECT * FROM products WHERE id = 8842;
```
```
 Index Scan using products_pkey on products  (actual time=0.018..0.019 rows=1)
   Buffers: shared hit=4
 Execution Time: ★ 0.041 ms
```
```
 ★ CACHING THIS IS A NET LOSS. A Redis GET is ~0.2 ms. You would
   make it 5× SLOWER and add an invalidation problem.
```
```sql
-- ② an expensive aggregate
EXPLAIN (ANALYZE, BUFFERS)
SELECT category_id, count(*), avg(price_minor)::bigint
  FROM products WHERE status='active' GROUP BY category_id;
```
```
 HashAggregate  (actual time=1884.2..1884.9 rows=412)
   ->  Seq Scan on products  (rows=4,102,884)
 Execution Time: ★ 1,884.9 ms
```
```
 ★ THIS IS WORTH CACHING — 1,884 ms → 0.2 ms. A 9,400× win,
   because it eliminates COMPUTATION.
```

**Demonstrate the thundering herd.**
```bash
# a 400 ms query behind a cache with TTL 10, no single-flight
redis-cli SET slowkey '{"v":1}' EX 10
sleep 10
# 200 concurrent requests at the moment of expiry
seq 200 | xargs -P 200 -I{} curl -s -o /dev/null -w '%{time_total}\n' \
  http://localhost:3000/api/slow | sort -n | tail -3
```
```
 ★ 12.884
 ★ 13.104
 ★ 14.221        — p99 14 s, from a 400 ms query
```
```bash
psql -c "SELECT count(*) FROM pg_stat_activity WHERE state='active'"
```
```
 count
-------
  ★ 187        — 187 identical queries running at once
```

**With single-flight.**
```bash
seq 200 | xargs -P 200 -I{} curl -s -o /dev/null -w '%{time_total}\n' \
  http://localhost:3000/api/slow-singleflight | sort -n | tail -3
```
```
 0.462
 0.478
 ★ 0.501
```
```bash
psql -c "SELECT count(*) FROM pg_stat_activity WHERE state='active'"
```
```
 count
-------
   ★ 1        — one query served 200 requests
```

**Demonstrate synchronised expiry, then jitter.**
```js
// ✗ without jitter — all 10,000 keys expire in the same second
for (const id of ids) await redis.set(`p:${id}`, v, 'EX', 3600);

// ✓ with jitter
const jitter = (ttl) => Math.floor(ttl * (0.75 + Math.random() * 0.5));
for (const id of ids) await redis.set(`p:${id}`, v, 'EX', jitter(3600));
```
```bash
# measure the spread
redis-cli --scan --pattern 'p:*' | head -1000 | \
  xargs -I{} redis-cli TTL {} | sort -n | awk 'NR==1{min=$1} END{print min, $1}'
```
```
 ★ without jitter: 3599 3600      — a 1-second window
 ★ with jitter:    2701 4498      — a 30-minute spread
```

**Demonstrate cache penetration.**
```js
// ✗ nulls are never cached ⇒ every request for a missing id hits the DB
const v = await redis.get(key);
if (v) return JSON.parse(v);
const row = await db(...);            // ★ null → nothing cached
if (row) await redis.set(key, JSON.stringify(row), 'EX', 300);
return row;

// ✓ cache the negative
if (v === NULL_MARKER) return null;
if (!row) { await redis.set(key, NULL_MARKER, 'EX', 30); return null; }
```
```bash
# a scraper enumerating non-existent ids
seq 900000 901000 | xargs -P 20 -I{} curl -s -o /dev/null \
  http://localhost:3000/api/products/{}
psql -c "SELECT calls FROM pg_stat_statements WHERE query LIKE '%products WHERE id%'"
```
```
 ★ without negative caching: 1001 DB queries
 ★ with negative caching:      1 DB query per id, then cached 30 s
```

**Demonstrate the stale-write race.**
```js
// two concurrent operations, deliberately interleaved
await Promise.all([
  (async () => {                                    // reader
    await redis.del('product:7');
    const row = await db.query('SELECT price_minor FROM products WHERE id=7');
    await sleep(100);                               // ★ the window
    await redis.set('product:7', JSON.stringify(row.rows[0]));
  })(),
  (async () => {                                    // writer
    await sleep(20);
    await db.query('UPDATE products SET price_minor=59900 WHERE id=7');
    await redis.del('product:7');                   // invalidate
  })(),
]);

console.log(await redis.get('product:7'));
console.log((await db.query('SELECT price_minor FROM products WHERE id=7')).rows[0]);
```
```
 cache: {"price_minor":49900}        ★ STALE
 db:    {"price_minor":59900}
 ⇒ ★ and with no TTL, permanently so.
```

**And versioned keys fixing it.**
```sql
ALTER TABLE products ADD COLUMN cache_version integer NOT NULL DEFAULT 1;
CREATE TRIGGER trg_cache_version BEFORE UPDATE ON products
  FOR EACH ROW EXECUTE FUNCTION bump_cache_version();
```
```js
// same interleaving, versioned keys
// reader writes  product:7:v4
// writer bumps   version → 5
// next reader looks up product:7:v5 ⇒ MISS ⇒ fresh read
// ★ the stale v4 entry is unreachable and expires on its own.
```

**Prove the cache is load-bearing.**
```bash
# steady state
curl -s localhost:3000/metrics | grep cache_hit_ratio
#  cache_hit_ratio 0.971
psql -c "SELECT count(*) FROM pg_stat_activity WHERE state='active'"
#  count: 4

# ★ the test everyone should run and almost nobody does
redis-cli FLUSHALL
sleep 2
psql -c "SELECT count(*) FROM pg_stat_activity WHERE state='active'"
```
```
 count
-------
  ★ 200        — the pool maximum, instantly
```
```bash
curl -s -o /dev/null -w '%{http_code} %{time_total}\n' localhost:3000/api/products
```
```
 ★ 504 30.001        — total outage from a cache flush
```

---

## Example 2 — production scenario

**The situation.** A news and content platform. Article pages, category listings, and a "trending" sidebar. Redis has been in front of PostgreSQL for two years.

```
 STEADY STATE
   API traffic          ★ 41,000 rps
   cache hit rate       ★ 98.4%
   PostgreSQL           ★ 656 rps, 11% CPU
   p99                  ★ 42 ms

 THEN, AT 14:22 ON A TUESDAY:
   a Redis cluster node fails over. 90 seconds of degraded cache.
   ⇒ ★ 41,000 rps hits PostgreSQL
   ⇒ pool exhausted in ★ 1.8 seconds
   ⇒ p99 42 ms → ★ 30,000 ms (gateway timeout)
   ⇒ ★ TOTAL OUTAGE: 41 MINUTES
   ⇒ ★ the outage OUTLASTED the Redis problem by 39.5 minutes,
     because the cache could not repopulate while the database
     was saturated, and client retries kept the load at 3× normal.
```

**Step 1 — the post-mortem finding nobody expected.**

```sql
-- what were those 41,000 rps actually asking for?
SELECT calls, mean_exec_time::numeric(10,3) AS mean_ms,
       substring(query from 1 for 70) AS q
  FROM pg_stat_statements ORDER BY calls DESC LIMIT 5;
```
```
  calls   | mean_ms |                          q
----------+---------+------------------------------------------------------
 18402118 |   ★ 0.038 | SELECT * FROM articles WHERE slug = $1
  8842119 |   ★ 0.041 | SELECT * FROM authors WHERE id = $1
  4102884 |  1884.220 | SELECT category_id, count(*) … GROUP BY category_id
  2104882 |   ★ 0.019 | SELECT * FROM categories WHERE id = $1
```

```
 ★ 68% OF CACHED READS WERE INDEXED SINGLE-ROW LOOKUPS AT 0.04 ms.
   Caching them saved nothing — a Redis GET is 0.2 ms, so the
   cache was making those requests ★ 5× SLOWER while adding an
   invalidation problem and a failure mode.
 ⇒ ★ THE CACHE EXISTED BECAUSE SOMEONE ADDED IT, NOT BECAUSE
   ANYTHING WAS MEASURED.
```

**Step 2 — what actually needed caching.**

```sql
SELECT calls, mean_exec_time::numeric(10,2) AS mean_ms,
       (calls*mean_exec_time/1000/3600)::numeric(10,1) AS hours,
       substring(query from 1 for 60) AS q
  FROM pg_stat_statements ORDER BY calls*mean_exec_time DESC LIMIT 4;
```
```
  calls  | mean_ms | hours  |                      q
---------+---------+--------+---------------------------------------------
 4102884 | 1884.22 | ★ 2147 | SELECT category_id, count(*) … GROUP BY …
  188402 | 4102.88 |  ★ 215 | SELECT … trending … window function over …
 8842119 |    0.04 |    0.1 | SELECT * FROM authors WHERE id = $1
18402118 |    0.04 |    0.2 | SELECT * FROM articles WHERE slug = $1
```
```
 ★ TWO QUERIES ACCOUNT FOR 99.98% OF THE DATABASE TIME.
   The other 27 million calls account for 0.02%.
 ⇒ ★ THE CACHE SHOULD HAVE HELD TWO THINGS, NOT EVERYTHING.
```

**Step 3 — the redesign, in five layers.**

```
 ★ LAYER 1 — HTTP/CDN. The cheapest cache is the request you never
   receive. Article pages are read-mostly and public.
```
```js
app.get('/api/articles/:slug', async (req, res) => {
  const article = await getArticle(req.params.slug);   // ★ direct DB, 0.04 ms
  if (!article) return res.status(404).set('Cache-Control',
    'public, max-age=30').end();                       // ★ negative caching, in HTTP

  const etag = `W/"${article.id}-${article.updated_at.getTime()}"`;
  res.set('ETag', etag);
  res.set('Cache-Control',
    'public, max-age=60, stale-while-revalidate=600');
  // ★ stale-while-revalidate IS single-flight, implemented by the CDN:
  //   the stale copy is served instantly while ONE background request
  //   refreshes it.
  if (req.get('If-None-Match') === etag) return res.status(304).end();
  res.json(article);
});
```
```
 ★ MEASURED: 41,000 rps → ★ 2,100 rps reaching the origin at all.
   94.9% absorbed by the CDN, at zero cost and with no invalidation
   problem (the ETag is derived from updated_at).
```

```
 ★ LAYER 2 — NO CACHE for the 0.04 ms lookups. Removed entirely.
```
```
 ★ MEASURED IMPACT OF *REMOVING* THE CACHE FOR THESE:
   p50 latency 0.24 ms → ★ 0.09 ms   (2.7× FASTER)
   invalidation code deleted: ★ 340 lines
   ⇒ ★ THE CACHE WAS A PESSIMISATION.
```

```
 ★ LAYER 3 — a MATERIALISED VIEW for the category aggregate
   (Topic 56), not a cache. It tolerates 5 minutes of staleness,
   it's the same for every user, and the engine keeps it correct.
```
```sql
CREATE MATERIALIZED VIEW mv_category_counts AS
SELECT category_id, count(*) AS n, avg(price_minor)::bigint AS avg_minor,
       now() AS refreshed_at
  FROM articles WHERE status = 'published' GROUP BY category_id;
CREATE UNIQUE INDEX ON mv_category_counts (category_id);
-- refresh CONCURRENTLY every 5 min, advisory-locked (Topic 56)
```
```
 ★ 1,884 ms → 0.06 ms, ★ with no invalidation problem and no
   stampede possible — it is always populated.
```

```
 ★ LAYER 4 — REDIS, for the ONE thing that genuinely needs it:
   "trending", which is expensive (4.1 s), personalised by region,
   and changes continuously.
```
```js
async function getTrending(region) {
  return cached(`trending:${region}`, 120, async () => {
    const { rows } = await pool.query(TRENDING_SQL, [region]);
    return rows;
  });
}
// ★ with: jittered TTL, single-flight (in-process + a Redis lock),
//   negative caching, and a metric per key class.
```

```
 ★ LAYER 5 — THE DEFENCES. All three.
```
```js
// ① a concurrency limiter on the database path — shed load rather
//    than collapse
const dbSemaphore = new Semaphore(60);      // ★ < pool size

async function loadTrending(region) {
  const release = await dbSemaphore.tryAcquire(50 /* ms */);
  if (!release) {
    metrics.increment('db.shed');
    // ★ serve the LAST KNOWN GOOD value rather than an error
    const stale = await redis.get(`trending:${region}:lkg`);
    if (stale) return JSON.parse(stale);
    throw new AppError('BUSY');
  }
  try {
    const rows = await pool.query(TRENDING_SQL, [region]);
    // ★ a long-TTL "last known good" copy, separate from the hot key
    await redis.set(`trending:${region}:lkg`, JSON.stringify(rows.rows),
                    'EX', 86400);
    return rows.rows;
  } finally { release(); }
}
```
```js
// ② a circuit breaker, so a saturated database isn't hammered
const breaker = new CircuitBreaker(loadTrending, {
  timeout: 3000, errorThresholdPercentage: 50, resetTimeout: 10000,
});
breaker.fallback((region) => redis.get(`trending:${region}:lkg`)
  .then(v => v ? JSON.parse(v) : []));
```
```bash
# ③ ★ TEST IT. In staging, at production-shaped load.
k6 run --vus 400 --duration 5m load.js &
sleep 120
redis-cli -h staging-redis FLUSHALL          # ★ the actual test
```
```
 ★ RESULT, WITH THE DEFENCES:
   PostgreSQL active connections   4 → ★ 61 (the semaphore limit)
   p99                             42 ms → ★ 780 ms
   error rate                      0% → ★ 0.4% (shed, served stale)
   ★ recovery to steady state      ★ 14 seconds
 ⇒ compare to the real incident: total outage, 41 minutes.
```

**Step 4 — what the metrics now show.**

```js
// ★ per key CLASS, not per key — otherwise cardinality explodes
metrics.increment('cache.hit',  { class: 'trending' });
metrics.increment('cache.miss', { class: 'trending' });
metrics.increment('cache.singleflight_waited', { class: 'trending' });
metrics.increment('db.shed');
metrics.gauge('cache.hit_ratio', hits / (hits + misses));
```
```sql
-- ★ and the alert that matters most: what happens if the cache dies?
--   Track the ratio and compute the implied database load.
--   alert if:  traffic_rps × (1 - hit_ratio_if_cache_died)
--                  > db_capacity_rps × 0.7
```

**Step 5 — results.**

| | Before | After |
|---|---|---|
| Origin traffic | 41,000 rps | **2,100 rps** (CDN absorbs 94.9%) |
| Redis keys | ~4.2M | **~180** (regions only) |
| Cached single-row lookups | 27M/day | **0** — removed |
| p50 on those lookups | 0.24 ms | **0.09 ms** (2.7× **faster**) |
| Invalidation code | 340 lines | **0** for those paths |
| Category aggregate | Redis, 1,884 ms on miss | matview, **0.06 ms** |
| Cache-flush blast radius | **41-min outage** | **780 ms p99, 14 s recovery** |
| Load-shedding | none | semaphore + breaker + last-known-good |

```
 ★ FOUR LESSONS:
 ① ★ 68% OF THE CACHE MADE THINGS SLOWER. Redis (0.2 ms) is not
   faster than an indexed PostgreSQL lookup (0.04 ms). Caching a
   fast query adds latency, a failure mode, and an invalidation
   problem, for nothing.
 ② ★ TWO QUERIES WERE 99.98% OF THE DATABASE TIME. The cache
   should have held two things.
 ③ ★ THE CDN ABSORBED 94.9% AT ZERO COST, using `ETag` and
   `stale-while-revalidate` — which is single-flight implemented
   by someone else, for free. It was the layer nobody had touched.
 ④ ★ THE OUTAGE LASTED 41 MINUTES FOR A 90-SECOND REDIS PROBLEM,
   because nothing shed load and the cache couldn't repopulate.
   ⇒ ★ IF YOU HAVE NEVER FLUSHED YOUR CACHE UNDER LOAD IN
     STAGING, YOU DO NOT KNOW WHAT YOUR SYSTEM DOES.
```

---

## Common mistakes

**1. Caching a query that's already fast.**
- *Symptom:* p50 gets *worse*; you now maintain invalidation for nothing.
- *Engine-level why:* a Redis GET (~0.2 ms) is a network round trip, the same as the PostgreSQL query you replaced.
- *Fix:* measure. Cache only what eliminates *computation*.

**2. Not checking `shared_buffers` hit ratio first.**
- *Symptom:* adding Redis "to reduce disk I/O" when the hit ratio is 99.95%.
- *Fix:* `pg_statio_user_tables`. Above 99%, there is no I/O to save.

**3. No single-flight.**
- *Symptom:* p99 spikes to seconds at every TTL boundary; hundreds of identical queries in `pg_stat_activity`.
- *Fix:* in-process coalescing plus a short cross-process lock.

**4. No TTL jitter.**
- *Symptom:* a load spike exactly one TTL after every deploy.
- *Fix:* `ttl × (0.75 + random() × 0.5)`.

**5. Not caching negative results.**
- *Symptom:* a scraper enumerating IDs sends every request to the database.
- *Fix:* a short-TTL null marker, or a Bloom filter.

**6. Explicit invalidation with no TTL.**
- *Symptom:* one missed invalidation means permanently wrong data.
- *Fix:* always set a TTL, even when you invalidate explicitly. It converts "forever" into "at most N seconds".

**7. Writing the new value into the cache on write.**
- *Symptom:* two writers race and the loser's value persists.
- *Fix:* `DEL`, never `SET`, from a write path. Deleting is idempotent.

**8. A cache that has silently become load-bearing.**
- *Symptom:* a 90-second cache outage becomes a 41-minute application outage.
- *Fix:* single-flight, a concurrency limit, a circuit breaker with last-known-good, and **a periodic flush test under load**.

**9. Write-behind caching.**
- *Symptom:* data loss when the cache fails.
- *Fix:* don't. The database is the source of truth.

**10. Caching per-user data at high cardinality.**
- *Symptom:* millions of keys, a low hit rate, and Redis memory pressure.
- *Fix:* cache the shared, expensive parts; compose per-user views from them.

**11. Skipping the HTTP layer.**
- *Symptom:* an elaborate Redis tier in front of content that a `Cache-Control` header would have removed from your infrastructure entirely.
- *Fix:* `ETag` + `Cache-Control` + `stale-while-revalidate` first.

**12. Per-key metric labels.**
- *Symptom:* metric cardinality explosion.
- *Fix:* label by key *class*, never by key.

---

## Hands-on proof

**PROVE IT #1–#8 — Example 1** (the buffer-pool hit ratio, a 0.041 ms lookup that must not be cached, a 1,884 ms aggregate that should be, the thundering herd at 187 concurrent identical queries, single-flight reducing it to 1, jitter spreading expiry from 1 second to 30 minutes, penetration and negative caching, the stale-write race producing a permanently wrong value, and the cache-flush test producing a 504).

**PROVE IT #9 — versioned keys survive the race.**
```js
// run the same interleaving as the stale-write proof, but with
// version-suffixed keys. Assert that no reader ever observes 49900.
const before = (await db.query('SELECT cache_version FROM products WHERE id=7')).rows[0];
// … race …
const after  = (await db.query('SELECT cache_version FROM products WHERE id=7')).rows[0];
console.log(await redis.get(`product:7:v${before.cache_version}`));  // stale, unreachable
console.log(await redis.get(`product:7:v${after.cache_version}`));   // ★ null ⇒ fresh read
```

**PROVE IT #10 — `stale-while-revalidate` is free single-flight.**
```bash
curl -sD- -o /dev/null http://cdn.example.com/api/articles/x | grep -i 'age\|cache'
# Age: 84
# Cache-Control: public, max-age=60, stale-while-revalidate=600
# ★ age > max-age ⇒ the CDN served a STALE copy instantly and is
#   refreshing in the background with ONE origin request.
```

**PROVE IT #11 — measure the implied load if the cache died.**
```js
const impliedDbRps = trafficRps * 1.0;          // ★ hit ratio → 0
const dbCapacityRps = 1200;                     // measured with pgbench
if (impliedDbRps > dbCapacityRps * 0.7)
  alert('★ the cache is LOAD-BEARING: a flush would exceed DB capacity');
```

**PROVE IT #12 — the flush test.**
```bash
k6 run --vus 400 --duration 5m load.js &
sleep 120 && redis-cli -h staging FLUSHALL
# ★ watch: pg_stat_activity active count, p99, error rate, time to recover.
# ★ If you have never run this, you do not know your blast radius.
```

---

## The design decision framework

```
★★★ A CACHE IS THE ONLY OPTIMISATION THAT CAN MAKE THINGS WORSE. ★★★

 ① MEASURE FIRST — THE THREE NUMBERS
    ★ buffer pool hit ratio (pg_statio_user_tables)
      > 99% ⇒ there is no I/O to save
    ★ the query's actual latency
      < 1 ms ⇒ ★ DO NOT CACHE IT. Redis is 0.2 ms; you will make
      it slower and add a failure mode.
    ★ calls × mean_time from pg_stat_statements
      ⇒ ★ usually 2–3 queries are 99% of the time. Cache those.

 ② TRY THE CHEAPER LAYERS FIRST — IN THIS ORDER
    ① ★ AN INDEX. More "we need a cache" problems are indexes.
    ② ★ HTTP/CDN — ETag + Cache-Control + stale-while-revalidate.
       ★ The cheapest cache is the request you never receive, and
       stale-while-revalidate is single-flight for free.
    ③ ★ A MATERIALISED VIEW (Topic 56) for shared aggregates —
       no invalidation problem, cannot stampede, cannot be wrong.
    ④ in-process LRU for small, shared, hot values
    ⑤ Redis — ★ last, and only for expensive computation

 ③ IF YOU CACHE, THE MANDATORY FIVE
    ✓ ★ CACHE-ASIDE (not write-through, never write-behind)
    ✓ ★ A TTL, ALWAYS — even with explicit invalidation.
        It converts "permanently wrong" into "wrong for ≤ N s".
    ✓ ★ JITTER: ttl × (0.75 + random() × 0.5)
    ✓ ★ SINGLE-FLIGHT: in-process map + a short Redis lock
    ✓ ★ NEGATIVE CACHING with a short TTL
    ⇒ missing any one of these produces a specific, known outage.

 ④ INVALIDATION — CHOOSE DELIBERATELY
    ★ TTL only          — simplest; accept the staleness window
    ★ DELETE on write   — never SET; deleting is idempotent
    ★ VERSIONED KEYS    — removes the race entirely; costs a
                          version read, so use it for EXPENSIVE
                          values only
    ★ IMMUTABLE VALUES  — the best answer. No invalidation needed.
                          Cache rendered documents, completed
                          orders, yesterday's report.

 ⑤ ★ ASSUME THE CACHE WILL BE EMPTY AT THE WORST MOMENT
    ✓ a concurrency limit on the DB path (shed, don't collapse)
    ✓ a circuit breaker with a LAST-KNOWN-GOOD long-TTL copy
    ✓ ★ FLUSH THE CACHE IN STAGING UNDER PRODUCTION LOAD,
        ON A SCHEDULE.
    ✓ ★ ALERT: traffic_rps × 1.0 > db_capacity × 0.7
        ⇒ "this cache is load-bearing"

 ⑥ WHAT NOT TO CACHE
    ✗ anything a user reads back immediately after writing
    ✗ anything where staleness is a correctness bug
    ✗ ★ anything an index would have fixed
    ✗ ★ anything already under 1 ms
    ✗ high-cardinality per-user data with a low hit rate

 ⑦ METRICS — label by key CLASS, never by key
    hit / miss / hit_ratio · single-flight waits · shed count ·
    ★ implied-DB-load-if-cache-died
```

---

## Practice exercises

### Exercise 1 — easy (concept check)
Measure your buffer-pool hit ratio and the latency of (a) an indexed single-row lookup and (b) an aggregate over a million rows. Compute what a 0.2 ms Redis GET would do to each. State which one should be cached and why the other must not be.

### Exercise 2 — medium (apply it)
Build a cache-aside layer with a deliberately slow (400 ms) loader. Then, with 200 concurrent clients at the TTL boundary, measure: (a) p99 and concurrent database queries without single-flight; (b) the same with in-process coalescing; (c) the same with a cross-process Redis lock. Then add jitter and show the expiry spread before and after.

Finally, reproduce the stale-write race and fix it with versioned keys.

### Exercise 3 — hard (production simulation)
A content platform serves 41,000 rps with a 98.4% Redis hit rate. A 90-second Redis failover caused a **41-minute** total outage.

(a) Explain why the outage outlasted the Redis problem by 39.5 minutes. Name both mechanisms.
(b) From `pg_stat_statements`, 68% of cached reads were 0.04 ms indexed lookups. Explain why caching those made the system *slower*, with numbers.
(c) Two queries account for 99.98% of database time. What does that imply about how the cache should have been designed?
(d) Design the HTTP/CDN layer: `ETag` derivation, `Cache-Control`, and why `stale-while-revalidate` is single-flight for free. Estimate the origin-traffic reduction.
(e) The category aggregate was cached in Redis. Argue for a materialised view instead, naming three properties it has that a cache doesn't.
(f) Write the Redis layer for the one query that needs it, with all five mandatory elements.
(g) Design the load-shedding: the semaphore limit relative to pool size, the circuit breaker, and the last-known-good key. Explain why LKG uses a *separate* key.
(h) Write the staging flush test and state the four numbers you'd measure.
(i) Write the alert that detects "this cache has become load-bearing" before an incident does.
(j) Removing the cache from the single-row lookups made p50 2.7× *faster* and deleted 340 lines. Write the one-paragraph justification you'd put in the PR.

---

## Mental model checkpoint

1. Name the six caching layers in order. Which should you check first, and which is most underused?
2. Why is Redis not faster than an indexed PostgreSQL lookup? What does a cache actually save?
3. Name the three stampede modes and the fix for each.
4. Describe the stale-write race step by step. Why does a TTL help even with explicit invalidation?
5. Why `DEL` rather than `SET` from a write path?
6. How do versioned keys eliminate the race, and what do they cost?
7. What makes a cache "load-bearing", and what are the three defences?
8. Why did a 90-second cache outage become a 41-minute application outage?
9. When is a materialised view better than a cache for the same query? Name three properties.
10. What is `stale-while-revalidate` and why is it single-flight for free?
11. Name four categories of data that should never be cached.

---

## Quick reference card

**The latency ladder**
```
buffer pool hit   ★ 0.0001 ms │ PG indexed    ★ 0.15 ms
in-process LRU    ★ 0.0002 ms │ Redis GET     ★ 0.20 ms  ← ★ SLOWER
                              │ PG aggregate  ★ 8,400 ms ← ★ cache THIS
```
**★ A cache saves computation, not storage access.**

**The order to try things**
```
① ★ an index  ② ★ HTTP/CDN (ETag + stale-while-revalidate)
③ ★ a matview (56)  ④ in-process LRU  ⑤ Redis — last
```

**The five mandatory elements**
```js
① cache-aside (never write-behind)
② ★ a TTL always — even with explicit invalidation
③ ★ jitter:  ttl * (0.75 + Math.random() * 0.5)
④ ★ single-flight: in-process Map + SET NX EX lock
⑤ ★ negative caching: a NULL marker, short TTL
```

**Invalidation, weakest → strongest**

| | Race-safe | Cost |
|---|---|---|
| TTL only | ★ stale ≤ TTL | free |
| `DEL` on write | ★ no (see the race) | free |
| ★ versioned keys | ★ **yes** | a version read |
| ★ immutable values | ★ **n/a** | free |

**★ Load-bearing defences**
```js
new Semaphore(60)                 // < pool size — shed, don't collapse
CircuitBreaker(..., { fallback: lastKnownGood })
// ★ and: redis-cli FLUSHALL in staging under production load, on a schedule
// ★ alert: traffic_rps × 1.0 > db_capacity_rps × 0.7
```

**Never cache:** sub-millisecond queries · data read back immediately after writing · anything an index fixes · high-cardinality per-user data with a low hit rate.

---

## When would I use this at work?

1. **Before adding Redis to anything.** Two numbers decide it: the query's latency (under 1 ms ⇒ a cache makes it slower) and `calls × mean_time` (usually 2–3 queries are 99% of the load). In the example, 68% of the cache was a pessimisation nobody had measured.

2. **When p99 spikes periodically with no traffic change.** That sawtooth is a TTL boundary. Single-flight and jitter fix it, and both are a few lines.

3. **Reviewing any content-serving endpoint.** `ETag` + `Cache-Control` + `stale-while-revalidate` removed 94.9% of traffic from the origin at zero cost, with no invalidation problem and no stampede possible. It's the layer teams reach for last and should reach for first.

4. **Disaster planning.** Ask "what happens if the cache is empty?" and then *actually flush it in staging under load*. A cache absorbing 97% of reads has quietly become a load-bearing component, and the first time you find out should not be during an incident.

---

## Connected topics

**Understand before this:** 07 (the buffer pool — the cache you already have), 53–55 (denormalisation — copies inside the database), 56 (materialised views — the better answer for shared aggregates), 52 (idempotency — the dual-write problem a write-through cache recreates).

**This unlocks:**
- **58** — read replicas: scaling reads without a consistency cliff
- **61** — counters and hot rows: where caches and hot keys interact
- **65** — connection pooling: why the pool exhausts in 1.8 seconds
- **68** — CAP and consistency models: where a cache sits among the guarantees
- **Case study 10** — product catalogue and search, built on these layers
