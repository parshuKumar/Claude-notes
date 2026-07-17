# 137 — Performance Optimization Patterns
## Category: Advanced

---

## What is this?

**Performance optimization** is the craft of making a system faster — but more importantly, it's the *discipline* of knowing **what** to make faster and **when to stop**. It's a catalogue of well-worn techniques (caching, batching, indexing, precomputation, async processing) paired with a strict methodology: **measure first, guess never.**

Think of it like a doctor treating a patient who says "I feel slow." A bad doctor prescribes ten random pills and hopes one works. A good doctor runs tests, finds the *one* blocked artery, fixes exactly that, and re-tests. Optimization without measurement is prescribing pills in the dark — you add complexity, cost, and bugs, and usually the patient is no faster.

This document is the good doctor's playbook: how to find the real bottleneck, the toolbox of fixes, and the discipline to apply them.

---

## Why does it matter?

**What breaks if you skip this:** You waste weeks. The classic failure is spending three days hand-optimizing a JSON serializer that accounts for 2% of request time, while a single missing database index — a 5-minute fix — is eating 80% of it. You shipped complexity and gained nothing. Worse, over-optimized code is harder to read, harder to change, and more likely to hide bugs. Every optimization you add is a permanent tax on everyone who touches that code later.

**Interview angle:** Interviewers love watching *how* you optimize. The junior answer is "I'd add a cache." The senior answer is "First I'd profile to find where the time actually goes, look at p99 not the average, then attack the biggest contributor." Saying "measure first" and naming the trade-off of each fix instantly marks you as someone who has done this for real. They'll also probe: "You added a cache — what's the cost?" If you can't name the trade-off (stale data, invalidation complexity, memory), you don't really understand the tool.

**Real-work angle:** In production, performance is money and user trust. Recall from [06 — Latency and Throughput](./06-latency-and-throughput.md) that a 100ms slowdown measurably drops conversion. And recall from [136 — Cost Optimization](./136-cost-optimization.md) that a faster system is usually a *cheaper* system — fewer servers, smaller bills. But the same lesson from 136 applies in reverse: past a point, optimization has diminishing returns and rising cost. Knowing when to stop is as valuable as knowing how to start.

---

## The core idea — explained simply

### The "Fix the Traffic Jam" Analogy

Imagine your commute takes 60 minutes and you want it shorter. There are five segments:

| Segment | Time | % of trip |
|---|---|---|
| Walk to car | 2 min | 3% |
| Suburban streets | 8 min | 13% |
| **Highway (bumper-to-bumper)** | **42 min** | **70%** |
| City streets | 6 min | 10% |
| Park and walk | 2 min | 3% |

Now suppose you buy running shoes to walk to your car twice as fast. You saved **1 minute** on a 60-minute trip. You optimized the wrong segment. Even if you made walking *instant*, the ceiling on your improvement was 3%.

The highway is the bottleneck. It's 70% of the trip. If you find a route that halves it (42 → 21 min), you just cut 21 minutes off your commute — a 35% improvement from one change. **That** is where effort belongs.

This is **Amdahl's Law** in plain English: the maximum you can gain from optimizing one part is capped by how big that part is. Optimize something that's 5% of the time, and even reducing it to zero gives you at most a 5% speedup. So the first job — always — is to find the highway, not polish the shoes.

Mapping the analogy back to systems:

| Commute idea | System idea |
|---|---|
| Total trip time | End-to-end request latency (p99) |
| Timing each segment | Profiling / distributed tracing |
| The 42-min highway | The bottleneck — the dominant cost |
| Buying running shoes | Optimizing a 3% component (wasted effort) |
| Amdahl's cap of 3% | Max gain is bounded by the part's share |
| Deciding 40 min is "good enough" | Hitting your NFR target and stopping |

The whole discipline reduces to: **find the highway, fix the highway, re-measure, stop when good enough.**

---

## Key concepts inside this topic

### 1. The Golden Rule: measure, don't guess

> "Premature optimization is the root of all evil." — Donald Knuth (and the second half people forget: *"...yet we should not pass up our opportunities in that critical 3%."*)

Human intuition about where time goes is **reliably wrong**. The slow part is almost never where you think. So before touching anything, you *measure* — three complementary tools (you met the first in [80 — Monitoring and Observability](./80-monitoring-and-observability.md), which this recalls):

- **Profiling** — sample where CPU time or wall-clock time is spent *inside one process*. In Node: `node --prof app.js` then `node --prof-process`, or `console.time()` for quick spot checks, or a flame graph via `clinic flame`.
- **Distributed tracing** — follow a single request as it hops across services (API → cache → DB → third-party API), timing each span. This is how you find *which service* is the highway when the work spans many machines.
- **Metrics** — aggregate numbers over time (requests/sec, latency histograms, DB query counts). This tells you *what changed* and *when*.

A cheap, honest profiler you can drop into any handler:

```js
// A tiny span timer. Wrap each phase of a request and print the breakdown.
async function timed(label, fn, timings) {
  const start = process.hrtime.bigint();
  try {
    return await fn();
  } finally {
    const ms = Number(process.hrtime.bigint() - start) / 1e6;
    timings[label] = (timings[label] || 0) + ms;
  }
}

async function handleRequest(req) {
  const timings = {};
  const user = await timed('db.user', () => loadUser(req.userId), timings);
  const feed = await timed('db.feed', () => loadFeed(user), timings);
  const html = await timed('render', () => render(feed), timings);
  console.log(timings); // { 'db.feed': 812, 'db.user': 14, render: 22 }
  return html;                       //          ^^^ there's your highway
}
```

The output screams the answer: `db.feed` is 812ms of an ~850ms request. Cache or fix *that*. Ignore `render`.

**Percentiles, not averages.** Recall from [06 — Latency and Throughput](./06-latency-and-throughput.md): averages lie. If 99 requests take 10ms and one takes 5,000ms, the average is 60ms — sounds fine — but 1% of users had an awful time. Always optimize against **p95 / p99**, because tail latency is what users actually feel and what compounds when requests fan out.

---

### 2. Caching — the highest-leverage optimization

Recall from [59 — Caching in Depth](./59-caching-in-depth.md): the fastest work is work you never do. Caching stores the result of an expensive read or computation so the next request gets it for free. It is almost always the single biggest lever, because most systems read the same data far more often than it changes.

Cache at **every layer**, cheapest and closest first:

- **In-process memoization** — a local `Map`. Nanoseconds, but per-instance and lost on restart.
- **Redis / shared cache** — one source of truth for the whole fleet. Sub-millisecond, survives restarts.
- **HTTP / CDN at the edge** — recall [60 — CDN](./60-cdn.md); the response is served from a city near the user and never touches your servers at all.

```js
// In-process memoization with TTL — trade a little memory + staleness for huge speed.
const cache = new Map(); // key -> { value, expires }

async function getConfig(key, ttlMs = 60_000) {
  const hit = cache.get(key);
  if (hit && hit.expires > Date.now()) return hit.value; // free
  const value = await expensiveDbLookup(key);            // paid, once per TTL
  cache.set(key, { value, expires: Date.now() + ttlMs });
  return value;
}
```

**The trade-off, named:** memory-for-speed, plus **staleness**. A cached value can be out of date until it expires or you invalidate it. Cache invalidation is famously one of the hard problems — never add a cache without deciding how it gets updated or expired.

---

### 2b. Database query optimization

The database is the bottleneck in the vast majority of web systems. Four fixes, in order of leverage:

**a) Indexing.** Recall from [62 — Database Indexing](./62-database-indexing.md): an index turns a full-table scan (O(n)) into a lookup (O(log n)). A query filtering on an unindexed column reads every row. Adding the right index is often a 100x win for a 5-minute change — the single highest ratio of gain-to-effort in this whole document.

**b) Kill N+1 queries.** This is the most common performance bug in ORM-heavy code. You load a list, then fire one more query *per item* to load a relation:

```js
// N+1 problem: 1 query for posts + N queries for authors = 101 round trips for 100 posts.
const posts = await db.query('SELECT * FROM posts LIMIT 100');
for (const post of posts) {
  post.author = await db.query('SELECT * FROM users WHERE id = $1', [post.authorId]);
} // 101 network round trips. Each round trip is ~1ms of pure waiting.
```

The fix: **batch** the child loads into ONE query, then stitch in memory:

```js
// Fixed: 2 queries total, regardless of list size.
const posts = await db.query('SELECT * FROM posts LIMIT 100');
const ids = [...new Set(posts.map(p => p.authorId))];
const authors = await db.query('SELECT * FROM users WHERE id = ANY($1)', [ids]);
const byId = new Map(authors.map(a => [a.id, a]));
for (const post of posts) post.author = byId.get(post.authorId); // in-memory, free
```

101 round trips → 2. At 1ms each, that's ~101ms → ~2ms.

**c) `SELECT` only the columns you need.** `SELECT *` on a table with a large `body` or `blob` column ships megabytes you throw away. Ask for the five fields you'll render.

**d) `EXPLAIN` your queries.** Every SQL database has `EXPLAIN ANALYZE`, which shows the actual query plan — "Seq Scan" (bad, full scan) vs "Index Scan" (good). Read it before assuming; it tells you if your index is even being used.

**e) Denormalize read-heavy paths.** Recall [63 — Database Replication and Sharding](./59-caching-in-depth.md) territory: sometimes the right fix is to *store data pre-joined* so reads skip the join entirely. Covered more in concept 6.

---

### 3. Connection pooling — reuse expensive connections

Recall from [65 — Database Connection Pooling](./65-database-connection-pooling.md): opening a fresh DB connection costs a TCP handshake plus TLS plus auth — often 20-50ms. Doing that *per request* is madness. A pool keeps a set of warm connections open and hands them out:

```js
const { Pool } = require('pg');
const pool = new Pool({ max: 20 }); // 20 warm connections reused across all requests

async function getUser(id) {
  // No handshake — a ready connection is checked out and returned to the pool after.
  const { rows } = await pool.query('SELECT id, name FROM users WHERE id = $1', [id]);
  return rows[0];
}
```

**Trade-off:** you hold open connections even when idle (a little DB-side memory), in exchange for removing tens of milliseconds from every single query. Almost always worth it.

---

### 4. Batching — trade a little latency for much more throughput

Recall from [06 — Latency and Throughput](./06-latency-and-throughput.md) that latency and throughput are different axes. Batching sacrifices a tiny bit of the first to massively boost the second: instead of doing 100 tiny operations with 100 round trips, you wait a few milliseconds, collect them, and do **one** operation.

This is the **DataLoader pattern** (from Facebook's GraphQL stack). A batcher collects keys requested within the same tick, then issues one query:

```js
// Collects individual .load(id) calls in a 5ms window, then fires ONE batched query.
class Batcher {
  constructor(batchFn, windowMs = 5) {
    this.batchFn = batchFn;      // (keys[]) => Promise<values[]>
    this.windowMs = windowMs;
    this.queue = [];             // { key, resolve, reject }
    this.timer = null;
  }
  load(key) {
    return new Promise((resolve, reject) => {
      this.queue.push({ key, resolve, reject });
      if (!this.timer) this.timer = setTimeout(() => this.flush(), this.windowMs);
    });
  }
  async flush() {
    const batch = this.queue; this.queue = []; this.timer = null;
    try {
      const keys = batch.map(b => b.key);
      const values = await this.batchFn(keys);        // ONE query for all keys
      batch.forEach((b, i) => b.resolve(values[i]));
    } catch (err) {
      batch.forEach(b => b.reject(err));
    }
  }
}

// Usage: 100 independent .load() calls collapse into a single SELECT ... WHERE id = ANY(...)
const userLoader = new Batcher(async (ids) => {
  const { rows } = await pool.query('SELECT * FROM users WHERE id = ANY($1)', [ids]);
  const byId = new Map(rows.map(r => [r.id, r]));
  return ids.map(id => byId.get(id));
});
```

The same idea applies to **batch DB inserts** (`INSERT ... VALUES (...),(...),(...)` beats 1,000 single inserts) and **batch API calls** (send 500 events in one request, not 500 requests). **Trade-off:** each individual caller waits up to `windowMs` longer, but total system throughput can jump 10-50x.

---

### 5. Pagination, async processing, precomputation, compression

Four more tools, grouped because each is short:

**Pagination / limiting.** Never load an unbounded result set — `SELECT * FROM events` on a billion-row table will OOM your process and time out. Always cap it. Prefer **cursor pagination** over `OFFSET` (which gets slower the deeper you page):

```js
// Cursor pagination: pass the last id seen. O(1)-ish regardless of depth.
async function page(afterId = 0, limit = 50) {
  return pool.query(
    'SELECT * FROM events WHERE id > $1 ORDER BY id LIMIT $2', [afterId, limit]
  );
}
```

**Asynchronous / non-blocking processing.** If work is slow but the user doesn't need the result *now* (send email, transcode video, generate a report), get it off the request path onto a queue — recall [67 — Message Queues](./59-caching-in-depth.md). The request returns in 20ms; a worker does the 5-second job later. And in Node specifically, recall [52 — Node.js Event Loop](./62-database-indexing.md): **never block the event loop.** A CPU-heavy loop (image resize, crypto, big JSON parse) freezes *every* concurrent request. Offload it to a worker thread:

```js
const { Worker } = require('worker_threads');
// CPU-bound work runs on a separate thread so the event loop keeps serving other requests.
function hashInWorker(data) {
  return new Promise((resolve, reject) => {
    const w = new Worker('./hash-worker.js', { workerData: data });
    w.once('message', resolve);
    w.once('error', reject);
  });
}
```

**Precomputation / materialized views.** Compute expensive results *ahead of time* so reads are trivial. Instagram doesn't compute your feed when you open the app — it precomputes ("fan-out on write") and stores it, so the read is a cheap lookup. A materialized view is a stored, pre-run query result. **Trade-off:** you pay storage plus extra write work, in exchange for near-instant reads. Great when reads vastly outnumber writes.

**Compression.** Gzip or Brotli your HTTP responses and compress stored blobs. A 200KB JSON response gzips to ~20KB — 10x less bandwidth, faster over the wire. **Trade-off:** CPU-for-bandwidth. Usually a clear win because bandwidth and its latency cost dwarf a few milliseconds of CPU.

---

### 6. Lazy vs eager loading, denormalization, CDN/edge, payload & concurrency

The remaining tools, each a lever with an explicit trade-off:

**Lazy loading vs eager loading / prefetching.** Opposite tools for opposite access patterns. *Lazy* defers work until it's actually needed (don't load the comments until the user scrolls to them) — great when most users never need it. *Eager / prefetch* does the work early or predictively (preload the next page while the user reads this one) — great when you're confident they'll need it. Guessing wrong wastes work either way, so match the tool to the real access pattern.

**Denormalization & read replicas.** Recall [63 — Replication] territory. Denormalization stores redundant, pre-joined data so reads skip joins; read replicas copy the DB so read traffic spreads across many machines. **Trade-off:** both cost you on the write side — more data to keep consistent, and replicas are *eventually* consistent (a read replica can be a few ms stale). You trade write complexity and consistency for read speed.

**CDN & edge.** Recall [60 — CDN](./59-caching-in-depth.md): push content and even computation physically closer to users. A user in Tokyo hitting an edge node 5ms away instead of your Virginia origin 150ms away feels a 30x latency drop for static assets. **Trade-off:** cache invalidation across edge nodes, and cost.

**Payload / serialization optimization.** Smaller responses travel faster. Return fewer fields (don't ship 40 columns when the client renders 4), collapse multiple round trips into one, and for high-volume internal traffic consider a compact binary format like **Protocol Buffers** instead of JSON (smaller and faster to parse). **Trade-off:** binary formats are less human-readable and need a schema.

**Concurrency & parallelism.** If several pieces of work are independent, don't await them one at a time — run them together:

```js
// Sequential: 300ms (100+100+100). The awaits stack up.
const user = await getUser(id);
const posts = await getPosts(id);
const stats = await getStats(id);

// Parallel: ~100ms. Independent work runs concurrently; total = the slowest one.
const [user, posts, stats] = await Promise.all([
  getUser(id), getPosts(id), getStats(id),
]);
```

`Promise.all` is the single easiest latency win in Node when your slow calls don't depend on each other. Pair it with the connection/worker pools from concepts 3 and 5 so the parallel calls actually have resources to run on. Recall [52 — Event Loop] again: parallel *I/O* is free on Node's single thread; parallel *CPU* needs worker threads.

---

### 7. The methodology — the loop that ties it together

The tools are useless without the discipline. Every optimization effort follows the same loop:

1. **Set a target** — from the NFRs. "p99 under 200ms." Without a target you can't know when you're done, and you'll over-optimize forever.
2. **Measure the baseline** — what is it *now*? p99 = 850ms.
3. **Find the biggest bottleneck** — profile/trace. Amdahl says: attack the largest share.
4. **Fix that one thing** — apply the right tool from the catalogue.
5. **Measure again** — did it help? By how much? (Sometimes a "fix" makes it worse — you only know because you measured.)
6. **Repeat** — the bottleneck moves; find the new biggest one.
7. **STOP when you hit the target.** This is the step everyone skips. Recall [136 — Cost Optimization](./136-cost-optimization.md): past your target, every further millisecond costs disproportionately more effort, complexity, and money for value nobody asked for. 190ms against a 200ms target is *done*. Ship it.

**And name every trade-off.** There is no free optimization. Each one costs you something — usually **memory-for-speed** (caching, precomputation), **complexity-for-speed** (batching, denormalization), or **consistency-for-speed** (replicas, caching). If you can't name what a change costs, you don't understand it well enough to ship it.

---

## Visual / Diagram description

**The optimize-measure loop** — the core methodology as a cycle:

```
        ┌───────────────────────────────────────────┐
        │                                           ▼
  ┌───────────┐   ┌──────────┐   ┌──────────┐   ┌──────────┐
  │ SET TARGET│──▶│ MEASURE  │──▶│  FIND    │──▶│  FIX     │
  │ (from NFR)│   │ BASELINE │   │ BOTTLENECK│  │ BIGGEST 1│
  └───────────┘   └──────────┘   └──────────┘   └────┬─────┘
                       ▲                              │
                       │        ┌──────────┐          │
                       └────────│ MEASURE  │◀─────────┘
                        (repeat)│  AGAIN   │
                                └────┬─────┘
                                     │ target met?
                                     ▼
                                ┌──────────┐
                                │  STOP.   │  ◀── the step everyone skips
                                │  SHIP.   │
                                └──────────┘
```

**The layers where you can optimize** — a request travels through many tiers, and *each* is an optimization surface:

```
  CLIENT          CDN / EDGE        LOAD BALANCER      APP SERVER        CACHE           DATABASE
 ┌───────┐       ┌───────────┐      ┌───────────┐     ┌───────────┐    ┌───────┐        ┌─────────┐
 │lazy   │──────▶│ static    │─────▶│ spread    │────▶│ async     │───▶│ Redis │───────▶│ indexes │
 │load,  │ HTTP  │ assets,   │      │ load,     │     │ queue,    │ hit│ memo  │  miss  │ pooling │
 │prefetch│cache │ geo-close │      │ keep-alive│     │Promise.all│◀───│ TTL   │◀───────│ no N+1  │
 │compress│      │ compress  │      │           │     │ workers   │    └───────┘        │ EXPLAIN │
 └───────┘       └───────────┘      └───────────┘     └───────────┘   fastest,        │ replicas│
   fewer                                                               closest         └─────────┘
   round trips                                                         cheapest         slowest,
                                                                                        farthest
```

**Reading the diagrams.** The first shows the discipline: you never jump straight to "fix." You always set a target, measure, and — critically — loop back to measure again and *stop* when the target is met. The second shows *where* the tools live. The golden rule: optimize **leftward and cheaply first** — a request served from the CDN never touches your database, and a request served from cache never runs a query. The further right (and down) a request has to travel, the slower and more expensive it gets, so the best optimizations stop it as early as possible.

---

## Real world examples

### Facebook / Meta — DataLoader and precomputed feeds

Facebook created the **DataLoader** pattern (the batcher in concept 4) precisely to kill N+1 queries in their GraphQL layer, where a single page might request thousands of related objects. They also famously **precompute** news feeds rather than assembling them on read — representative of the "fan-out on write" strategy where the expensive work happens once at write time so billions of reads stay cheap. Both are textbook: batching for throughput, precomputation for read latency.

### Netflix — CDN and edge everywhere

Netflix serves the overwhelming majority of its bytes from **Open Connect**, its own CDN, with appliances placed inside ISPs — content sits physically close to viewers so playback starts fast and never round-trips to a central origin. This is CDN/edge optimization at planet scale: the cheapest, closest layer serves the traffic. (Representative of publicly described architecture.)

### Shopify / Rails-scale commerce — caching layers and query discipline

High-traffic commerce platforms lean heavily on **multi-layer caching** (in-process, Redis, CDN) plus relentless database discipline — indexing hot queries, eliminating N+1s, and using read replicas so read-heavy storefront traffic doesn't compete with checkout writes. This is the everyday, unglamorous 90% of real performance work: caching plus query optimization, applied consistently and measured. (Representative of commonly described patterns.)

---

## Trade-offs

Every technique buys speed with a specific currency. Know which one you're spending.

| Technique | What you gain | What it costs |
|---|---|---|
| Caching | Massive read speedup | Memory + staleness + invalidation complexity |
| Indexing | 10-100x faster filtered reads | Slower writes, more storage |
| Batching / DataLoader | 10-50x throughput | A little latency per call; more code |
| Connection pooling | -20-50ms per query | Idle open connections |
| Precomputation / materialized views | Near-instant reads | Storage + write-time cost + staleness |
| Compression | 5-10x less bandwidth | CPU on both ends |
| Denormalization / replicas | Faster reads, spread load | Write complexity + eventual consistency |
| CDN / edge | 10-30x latency for static content | Invalidation across edges + cost |
| Worker threads / async | Unblocked event loop | Complexity, serialization overhead |
| `Promise.all` parallelism | Latency = slowest call, not sum | Resource pressure; harder error handling |

**The trap to avoid:**

| Good discipline | Bad discipline |
|---|---|
| Profile, then fix the biggest thing | Guess and optimize a hunch |
| Optimize against p99 | Optimize against the average |
| Stop at the NFR target | Chase the last millisecond forever |
| Name each trade-off before shipping | Add complexity blind to its cost |

**The sweet spot:** measure to find the *one* biggest bottleneck, apply the *cheapest* tool that fixes it (usually caching or an index), re-measure, and **stop the moment you hit your target**. The best-performing systems aren't the most cleverly optimized — they're the ones where someone measured, fixed the obvious highway, and had the discipline to walk away.

---

## Common interview questions on this topic

### Q1: "This endpoint is slow. Walk me through how you'd speed it up."

**Hint:** Do NOT jump to "add a cache." Say: first I'd *measure* — add tracing/profiling to see where the time actually goes, looking at p99 not the average. Find the dominant cost (usually a slow or N+1 query). Fix that biggest one, re-measure, repeat, and stop when I hit the latency target. Naming "measure first" is the whole point of the question.

### Q2: "What's the N+1 query problem and how do you fix it?"

**Hint:** Loading a list (1 query) then querying once per item for a relation (N queries) = N+1 round trips. Fix by batching: collect all the child keys and issue ONE query with `WHERE id = ANY($ids)`, then stitch results in memory with a `Map`. The DataLoader pattern automates this.

### Q3: "You added a cache. What are the downsides?"

**Hint:** Staleness (data can be out of date until it expires/invalidates), invalidation complexity (one of the genuinely hard problems), memory cost, and a cold-cache thundering-herd risk on restart. A cache is memory-and-staleness traded for speed — never add one without a plan for how it gets updated.

### Q4: "If optimizing X gives at most a 4% speedup, why might you skip it?"

**Hint:** Amdahl's Law — the max gain from optimizing one part is capped by that part's share of total time. If X is 4% of the time, even reducing it to zero yields ≤4%. That effort is better spent on whatever dominates the profile. Effort should follow the biggest bottleneck, not the most interesting code.

### Q5: "When do you stop optimizing?"

**Hint:** When you hit the target set by the NFRs. Beyond that, returns diminish sharply while complexity, cost, and bug-risk rise (recall cost optimization). 190ms against a 200ms target is done — shipping simple-and-fast-enough beats clever-and-fragile.

---

## Practice exercise

### Optimize a deliberately slow endpoint

Build a tiny Node/Express endpoint `GET /users/:id/dashboard` that is *intentionally* slow, then optimize it step by step, recording the latency at each stage.

**Setup (make it slow on purpose):**
1. It loads a user, then loops over their 50 posts and queries the author + comment count **per post** (N+1).
2. It calls `getUser`, `getPosts`, and `getStats` **sequentially** with `await`.
3. It runs a synchronous CPU-heavy loop (e.g. hash something 1M times) on the request thread.
4. It returns the full row of every object (`SELECT *`) with no compression.

**Your task — apply one fix at a time and log the p99 after each:**
- Add `console.time`/`hrtime` timing around each phase and identify the biggest cost first.
- Kill the N+1 with a single batched query + `Map`.
- Parallelize the three independent loads with `Promise.all`.
- Move the CPU-heavy work to a `worker_threads` worker (or a queue).
- Trim the payload to needed fields and enable gzip.
- Add an in-process TTL cache for the whole dashboard.

**What to produce:** a short table showing latency after each step (e.g. 850ms → 210ms → 120ms → 95ms → 60ms → 8ms cached) and a one-line note naming the trade-off you accepted at each step. The lesson lands when you *see* the number drop — and when you notice the last few optimizations gave the least return.

---

## Quick reference cheat sheet

- **Golden rule:** MEASURE, don't guess. The slow part is never where you think.
- **Profile / trace / metrics** before touching code — find the real bottleneck first.
- **Amdahl's Law:** max gain is capped by the part's share of total time. Fix the biggest thing.
- **Percentiles, not averages:** optimize p95/p99 — averages hide the tail users feel.
- **Caching** is the highest-leverage lever: in-process → Redis → CDN. Cost: memory + staleness.
- **Indexing** is the best gain-to-effort ratio in databases: O(n) scan → O(log n) lookup.
- **N+1 → batch:** collect keys, one `WHERE id = ANY(...)` query, stitch with a `Map`.
- **Connection pooling:** reuse warm connections, save 20-50ms of handshake per query.
- **Batching (DataLoader):** trade a little latency for 10-50x throughput.
- **Async/queue + worker threads:** never block the Node event loop; offload slow/CPU work.
- **Precompute / materialized views:** trade storage + write cost for near-instant reads.
- **`Promise.all`:** parallelize independent work — latency becomes the slowest call, not the sum.
- **Every optimization is a trade-off** — name it: memory-, complexity-, or consistency-for-speed.
- **STOP at the target.** Over-optimization has diminishing returns and rising cost.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [136 — Cost Optimization](./136-cost-optimization.md) | The mirror discipline — a faster system is usually cheaper, and both teach *when to stop*. |
| **Next** | [59 — Caching in Depth](./59-caching-in-depth.md) | The single highest-leverage optimization; go deep on the tool you'll reach for most. |
| **Related** | [62 — Database Indexing](./62-database-indexing.md) | The best gain-to-effort fix in the catalogue — turns table scans into lookups. |
| **Related** | [65 — Database Connection Pooling](./65-database-connection-pooling.md) | Reusing expensive connections; a foundational per-query optimization. |
| **Related** | [06 — Latency and Throughput](./06-latency-and-throughput.md) | The two axes you're optimizing, and why percentiles (not averages) are what matter. |
