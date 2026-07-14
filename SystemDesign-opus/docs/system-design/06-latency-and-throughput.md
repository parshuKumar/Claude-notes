# 06 — Latency vs Throughput — The Two Performance Kings

## Category: HLD Fundamentals

---

## What is this?

**Latency** is how long **one** thing takes. **Throughput** is how many things you can finish **per second**. That's it — latency is a duration, throughput is a rate.

Think of a coffee shop. Latency is "how long *you* waited for your latte" (90 seconds). Throughput is "how many lattes the shop serves per hour" (200/hour). These are different numbers measuring different things, and — this is the part that trips everyone up — **making one better often makes the other worse.**

---

## Why does it matter?

Every non-functional requirement you'll ever write is secretly about one of these two. "The page must load fast" = latency. "The system must handle Black Friday" = throughput. If you can't tell them apart, you'll optimize the wrong thing and wonder why users are still complaining.

**What breaks without this knowledge:**

- You report "our average response time is 200ms, we're fine" — while 1 in 100 users waits 8 seconds and churns. (Averages lie. We'll prove it.)
- You add batching to boost throughput and accidentally triple your p99 latency, and the mobile app starts timing out.
- You size your server fleet by dividing traffic by average latency and end up with a fleet that collapses at peak.

**Interview angle:** The moment you say "we need low latency," a good interviewer asks *"at what percentile?"* Answering "p99 under 200ms" instead of "fast" instantly marks you as someone who has run a real system. Every capacity-estimation question (topic 05) also secretly requires throughput math.

**Real-work angle:** This is the vocabulary of on-call. Dashboards, SLOs, alerts, load tests — all of them speak in latency percentiles and requests-per-second. Amazon found that every 100ms of extra latency cost them roughly 1% in sales; Google found that an extra 500ms of search latency dropped traffic ~20%. Latency is money.

---

## The core idea — explained simply

### The Highway Analogy

Picture a highway between two cities, 120 km apart.

- **Latency** = how long **your car** takes to get from city A to city B. At 120 km/h, that's **1 hour**.
- **Throughput** = how many **cars per hour** arrive in city B. If the highway has 4 lanes and cars are spaced safely, maybe **8,000 cars/hour**.

Now watch what happens when you try to improve them:

**"Let's improve throughput — add 4 more lanes!"** Now 16,000 cars/hour arrive. But *your* drive still takes exactly 1 hour. **Throughput doubled, latency unchanged.**

**"Let's improve latency — raise the speed limit to 240 km/h!"** Now your drive takes 30 minutes. But cars need much bigger safety gaps at that speed, so fewer cars fit per lane. **Latency halved, throughput possibly dropped.**

**"Let's cram more cars in — reduce the gap between them!"** More cars per hour... until someone brakes, and now you have a traffic jam. Everyone's trip takes 3 hours. **You pushed throughput past the limit and latency exploded.**

That last one is the most important lesson in this entire document: **as utilization approaches 100%, latency goes to infinity.** A highway at 60% capacity flows. A highway at 98% capacity is a parking lot. Servers behave exactly the same way — a queue forms, and every request now waits behind the queue.

### Mapping the analogy back

| Highway | System | Notes |
|---|---|---|
| Your car's travel time | **Latency** of one request | Measured in ms |
| Cars arriving per hour | **Throughput** (QPS/RPS) | Measured in requests/sec |
| Number of lanes | **Concurrency** — servers, threads, connections | More lanes = more parallel work |
| Speed limit | Per-request processing speed | Faster CPU, better algorithm, cache hit |
| Cars currently on the road | **Requests in flight** (queue + in-service) | This is `L` in Little's Law |
| Traffic jam at 98% capacity | **Queueing collapse** | The reason you never run servers at 95% CPU |
| The one car with a flat tyre | **Tail latency** (p99) | Rare, slow, and it ruins someone's day |
| A bus carrying 50 people | **Batching** | Great throughput, terrible latency for the first person who boarded |

**The one-sentence version:** *Latency is what a single user feels. Throughput is what your bill looks like.*

---

## Key concepts inside this topic

### 1. Precise definitions (and the units)

```
Latency   = time elapsed for ONE unit of work
          = t_response_received - t_request_sent
          = measured in milliseconds (ms) or microseconds (µs)

Throughput = units of work completed PER UNIT TIME
          = requests / second   → QPS (queries per sec) or RPS (requests per sec)
          = also: MB/s for data pipelines, messages/sec for queues
```

Two sub-terms you'll hear: **response time** is latency *plus* queueing *plus* network transit — in an interview, "latency" means response time as the client experiences it, so always measure at the client, not inside your handler. **Bandwidth** is the theoretical *maximum* throughput of a link; throughput is what you actually achieve.

A very common mistake: *"Latency is 100ms, so throughput must be 1 / 0.100 = 10 requests/sec."* **Wrong** — that's only true if you handle one request at a time. Real servers handle many concurrently. With 100 concurrent workers each taking 100ms, throughput is 1,000 QPS while latency stays 100ms. The bridge between them is Little's Law (concept 4).

### 2. Why optimizing one can hurt the other

| Technique | Effect on throughput | Effect on latency | Why |
|---|---|---|---|
| **Batching** (group 100 writes into 1 DB call) | ▲ big improvement | ▼ worse | The first item in the batch waits for the other 99 |
| **Compression** (gzip responses) | ▲ (less bandwidth used) | ▼ or ▲ | Costs CPU time, but sends fewer bytes — a wash on fast networks, a win on slow ones |
| **Bigger connection pool** | ▲ up to a point | ▼ past that point | More concurrency → more queueing at the DB |
| **Caching** | ▲ | ▲ | The rare win-win — that's why caching is everywhere |
| **Adding a queue (async processing)** | ▲ | ▼ for that request | The work now happens later; the *user-facing* latency drops but end-to-end latency rises |
| **Running servers at 95% CPU** | ▲ (cheaper!) | ▼▼▼ catastrophic | Queueing theory punishes you (see concept 4) |

The batching example in Node, made concrete:

```js
// LOW LATENCY, LOW THROUGHPUT — one round trip per event.
// Each caller waits ~2ms (one DB round trip). Ceiling: ~500 writes/sec per connection.
async function writeEvent(db, event) {
  await db.insert('events', event); // 2ms
}

// HIGH THROUGHPUT, HIGHER LATENCY — accumulate for 50ms, then one bulk insert.
// Each caller now waits up to 50ms + 3ms. But we do 1000s of writes per DB call.
class BatchWriter {
  constructor(db, { maxSize = 500, maxWaitMs = 50 } = {}) {
    Object.assign(this, { db, maxSize, maxWaitMs });
    this.buffer = [];   // pending events + their promise resolvers
    this.timer = null;
  }

  write(event) {
    return new Promise((resolve, reject) => {
      this.buffer.push({ event, resolve, reject });
      // Flush early if the batch is full — bounds worst-case memory.
      if (this.buffer.length >= this.maxSize) return this.flush();
      // Otherwise start the clock. This timer IS the latency you are trading away.
      if (!this.timer) this.timer = setTimeout(() => this.flush(), this.maxWaitMs);
    });
  }

  async flush() {
    clearTimeout(this.timer);
    this.timer = null;
    const batch = this.buffer;
    this.buffer = [];
    if (batch.length === 0) return;
    try {
      await this.db.insertMany('events', batch.map(b => b.event)); // ~3ms for 500 rows
      batch.forEach(b => b.resolve());
    } catch (err) {
      batch.forEach(b => b.reject(err)); // all-or-nothing: a real system would retry
    }
  }
}
```

You did not make the database faster. You **moved latency from the server's problem into the caller's wait**, and bought a 20x throughput increase with it. That is a trade, not a free lunch. For analytics events: obviously worth it. For a "confirm payment" call: absolutely not.

### 3. Percentiles — and why averages lie

A **percentile** is a ranking. **p99 = 250ms** means: "sort every request by how long it took; 99% of them finished in 250ms or less."

- **p50** (the median) — the typical user
- **p95** — the unlucky 1-in-20
- **p99** — the unlucky 1-in-100
- **p999** (p99.9) — the unlucky 1-in-1000

**Worked example.** 10 requests came in, with these latencies in ms:

```
10, 10, 12, 11, 10, 13, 10, 12, 11, 3901
```

- **Average** = (10+10+12+11+10+13+10+12+11+3901) / 10 = 4000/10 = **400ms**
- **Median (p50)** = sort → `10,10,10,10,11,11,12,12,13,3901` → middle ≈ **11ms**
- **p90** = the 9th value = **13ms**
- **Max (p100)** = **3901ms**

Look at what the average told you: *400ms*. That number describes **literally nobody**. Nine users had a snappy 10ms experience. One user waited 4 seconds and probably closed the tab. The average is a number that no single request experienced — it's a blend of two totally different populations.

This is why **you never alert on averages.** A single 30-second garbage-collection pause can drag the average up while nothing is really wrong, and — far worse — a slow, growing tail can hide *completely* inside a healthy-looking average.

**The scale reality check:** at 1,000 QPS, "p99 = 2 seconds" is not a rare edge case. It is **10 requests every single second** taking 2 seconds. That's 864,000 miserable requests a day.

**Why the tail exists at all:** garbage collection pauses, TCP retransmits, a cold cache, a slow disk seek, a noisy neighbour VM, a lock held too long, a request that happens to hit an unusually large row. The tail is where all your rare bugs live. That's why senior engineers stare at p99 and ignore the mean.

### 4. Little's Law — the one formula worth memorizing

```
        L = λ × W

  L (Length) = average number of requests IN THE SYSTEM (in flight)
  λ (lambda) = average arrival rate = throughput  (requests / second)
  W (Wait)   = average time a request spends in the system = latency (seconds)
```

That's it. It's true for any stable system — no assumptions about the distribution, no statistics degree required. It's basically the highway: *cars on the road = cars per hour × hours spent driving.*

**Use 1 — how many workers do I need?** You need 2,000 QPS and each request takes 100ms (0.1s), so `L = λ × W = 2000 × 0.1 = 200 requests in flight`. You need **200 concurrent slots** — 200 threads, or 200 DB pool connections, or (in Node) enough event-loop headroom for 200 in-flight promises. If your pool has 50, you have just capped yourself at `λ = 50 / 0.1 = 500 QPS`, and everything above that queues.

**Use 2 — flip it to find the ceiling you've imposed.** Your Node service has a DB pool of 10 connections and each query takes 20ms, so `λ_max = L / W = 10 / 0.02 = 500 queries/sec`. Traffic hits 800 QPS. The extra 300 QPS doesn't vanish — it **queues**. Queue depth grows without bound, latency climbs, and you start timing out. The fix is a bigger pool (more L), a faster query (smaller W), or backpressure/rate limiting (smaller λ). There is no fourth option.

**Use 3 — the utilization trap.** Queueing theory adds a second, brutal result. For a simple queue, the wait time scales roughly as:

```
Wait ≈ ServiceTime × ( ρ / (1 - ρ) )        where ρ = utilization (0 to 1)
```

| Utilization ρ | Multiplier ρ/(1−ρ) | If service time = 10ms, wait ≈ |
|---|---|---|
| 50% | 1.0 | 10 ms |
| 80% | 4.0 | 40 ms |
| 90% | 9.0 | 90 ms |
| 95% | 19.0 | 190 ms |
| 99% | 99.0 | **990 ms** |

Going from 90% to 99% CPU "saves" you 9% of your servers and multiplies your queueing latency by **11x**. This is why SREs keep servers at 50-70% utilization and why "but the CPU is only at 85%!" is not the reassurance people think it is.

### 5. Tail latency amplification — the microservice killer

Here's the counterintuitive one that interviewers love.

Your gateway serves one page by calling **10 backend services in parallel** and waiting for all of them. Each service has an excellent p99 of 100ms and a p50 of 10ms.

What is the gateway's latency? Your instinct says "about 100ms, worst case." **Wrong.**

The gateway is only as fast as its **slowest** call. The chance that *all 10* calls avoid the slow 1% tail is:

```
P(all 10 are fast) = 0.99 ^ 10 = 0.904  →  90.4%

P(at least one call is a slow one) = 1 - 0.904 = 0.096 → 9.6%
```

**Nearly 10% of your page loads now hit at least one 100ms straggler.** In other words, the p99 of a single service has become roughly the **p90 of the aggregate**. Push to 100 fanned-out calls (a Twitter timeline, a search shard scatter-gather) and it gets ugly:

```
P(at least one slow) = 1 - 0.99^100 = 1 - 0.366 = 63.4%
```

**63% of requests hit a straggler.** The p99 of one service is now roughly the **p50 of the whole page.** This is **tail latency amplification**, and it's why "each service is fast" does not imply "the system is fast."

```
Fan-out of N services, each with p99 = 100ms:

  N=1   ──▶ ~1%   of requests are slow
  N=10  ──▶ ~10%  of requests are slow   (p99 → p90)
  N=100 ──▶ ~63%  of requests are slow   (p99 → p50)
```

**How real systems fight it:**

- **Hedged requests** — after waiting for p95, fire a duplicate request to a second replica and take whichever answers first. Costs ~5% extra load, can slash p99 dramatically. (Google's "The Tail at Scale" paper describes exactly this. Never hedge a non-idempotent call — see topic 85.)
- **Reduce the fan-out** — fewer, chunkier calls beat many tiny ones.
- **Aggressive per-call timeouts + fallbacks** — return a degraded page rather than waiting on the straggler.
- **Fix the source of the tail** — GC tuning, connection reuse, cache warming.

```js
// Hedged request: race the primary against a backup fired after a short delay.
async function hedgedFetch(callFn, { hedgeAfterMs = 50 } = {}) {
  const ctrls = [new AbortController(), new AbortController()];
  const primary = callFn(ctrls[0].signal);
  const backup = new Promise((res, rej) => {
    const t = setTimeout(() => callFn(ctrls[1].signal).then(res, rej), hedgeAfterMs);
    primary.finally(() => clearTimeout(t)); // primary won — don't leak the timer
  });
  const winner = await Promise.race([primary, backup]);
  ctrls.forEach(c => c.abort()); // cancel the loser, free the socket
  return winner;
}
```

### 6. Real latency numbers to memorize

These are the numbers interviewers expect you to pull out of thin air (order of magnitude is what matters, not the exact digits):

| Operation | Time | Intuition |
|---|---|---|
| L1 cache reference | 0.5 ns | — |
| L2 cache reference | 7 ns | 14x L1 |
| Mutex lock/unlock | 25 ns | — |
| Main memory (RAM) reference | 100 ns | 200x L1 |
| Read 1 MB sequentially from RAM | 100 µs (0.1 ms) | ~10 GB/s |
| SSD random read | 150 µs (0.15 ms) | ~1000x slower than RAM |
| Read 1 MB sequentially from SSD | 1 ms | ~1 GB/s |
| Round trip within the same datacenter | **0.5 ms** | ← memorize this |
| Disk seek (spinning) | 10 ms | why we love SSDs |
| Read 1 MB sequentially from spinning disk | 20 ms | — |
| Round trip California → Netherlands | **150 ms** | ← memorize this |
| **Typical DB query (indexed, warm)** | **1–10 ms** | — |
| **Typical Redis GET (same DC)** | **~1 ms** (mostly network) | — |
| **Human perceives as "instant"** | **< 100 ms** | ← the target |
| **Human notices a delay** | **> 300 ms** | — |
| **Human loses their train of thought** | **> 1000 ms** | — |

**The two derived rules:** (1) **RAM is ~100,000x faster than a cross-continent network call** — the entire justification for caching and CDNs. (2) **You cannot beat the speed of light** — ~150ms round trip to the far side of the planet is physics, not engineering, and the only fix is *moving the data closer to the user*, which is exactly what a CDN (topic 60) does.

### 7. Measuring both in Node.js

Don't guess. Measure. Here's a dependency-free latency + throughput recorder you can paste into an Express app.

```js
import { performance } from 'node:perf_hooks';

/**
 * Keeps every sample for the window, then sorts to get exact percentiles.
 * Fine for < ~1M samples/window. Beyond that, use a bounded reservoir
 * sample or an HDR histogram — sorting a huge array is itself a latency spike.
 */
class LatencyRecorder {
  constructor() { this.samples = []; }

  record(ms) { this.samples.push(ms); }

  percentile(p) {
    if (this.samples.length === 0) return 0;
    const sorted = [...this.samples].sort((a, b) => a - b);
    // Nearest-rank method: the smallest value at or above the p-th position.
    const idx = Math.ceil((p / 100) * sorted.length) - 1;
    return sorted[Math.min(Math.max(idx, 0), sorted.length - 1)];
  }

  summary(windowSeconds) {
    const n = this.samples.length;
    const mean = n ? this.samples.reduce((a, b) => a + b, 0) / n : 0;
    const pct = (p) => +this.percentile(p).toFixed(1);
    return {
      count: n,
      throughputQps: +(n / windowSeconds).toFixed(1),
      mean: +mean.toFixed(1),   // included ONLY to show how it lies
      p50: pct(50), p95: pct(95), p99: pct(99), p999: pct(99.9), max: pct(100),
    };
  }

  reset() { this.samples = []; }
}

const recorder = new LatencyRecorder();

// Express middleware. Time the FULL response, not just your handler —
// serialization and the socket write are latency the user actually feels.
export function latencyMiddleware(req, res, next) {
  const start = performance.now(); // monotonic; Date.now() can jump backwards on NTP sync
  res.on('finish', () => recorder.record(performance.now() - start));
  next();
}

// Roll up and report once every 10 seconds.
const WINDOW = 10;
setInterval(() => {
  const s = recorder.summary(WINDOW);
  if (s.count === 0) return;
  console.log(JSON.stringify({ ts: new Date().toISOString(), ...s }));
  recorder.reset();
}, WINDOW * 1000);
```

Sample output — notice how `mean` (31ms) makes everything look fine while `p99` is 12x worse:

```json
{"ts":"2026-07-12T10:00:10Z","count":14320,"throughputQps":1432,
 "mean":31.2,"p50":18.0,"p95":96.4,"p99":384.1,"p999":1205.0,"max":3902.0}
```

**One Node-specific trap:** Node is single-threaded for *your* JavaScript. If any handler does CPU-heavy synchronous work (`JSON.parse` of a 10MB body, a sync crypto call, a big loop), it **blocks the event loop** and every other in-flight request's latency inflates. That's a tail-latency generator hiding in plain sight. Watch it directly:

```js
import { monitorEventLoopDelay } from 'node:perf_hooks';

const loop = monitorEventLoopDelay({ resolution: 10 });
loop.enable();

setInterval(() => {
  // Nanoseconds → ms. Healthy: p99 under ~10ms. If this p99 is 200ms, NO request
  // can possibly be faster than 200ms, no matter what else you optimize.
  console.log('event loop delay p99 (ms):', (loop.percentile(99) / 1e6).toFixed(1));
  loop.reset();
}, 10_000);
```

---

## Visual / Diagram description

### Diagram 1: Latency vs Throughput on one request path

```
   CLIENT                                                          SERVER
     │                                                               │
     │──── request sent ────────────────────────────────────────────▶│
     │            ▲                                                  │
     │            │  network transit (~0.5ms same DC, ~150ms global) │
     │            │                                                  ├── waits in accept queue
     │            │                                                  │      (this is ρ/(1-ρ) pain)
     │            │                                                  ├── DB query      (5 ms)
     │            │                                                  ├── cache lookup  (1 ms)
     │            │                                                  ├── serialize     (1 ms)
     │◀─── response received ───────────────────────────────────────│
     │                                                               │
     └────────────────── LATENCY = the whole span ───────────────────┘
             (one request; measured at the CLIENT, in ms)

   THROUGHPUT = how many of these arrows finish per second, across ALL clients
   ───────────────────────────────────────────────────────────────────────────
      t=0s   ████████████████████████████████████   1,432 completed  → 1432 QPS
      t=1s   ███████████████████████████████████    1,410 completed  → 1410 QPS
```

The key thing to see: **latency is a horizontal measurement (across time for one request); throughput is a vertical measurement (a count of requests in one second).** They are perpendicular. Changing one does not automatically change the other.

### Diagram 2: The latency/throughput curve — where systems die

```
  Latency
   (ms)
    │
1000┤                                                    ╱  ← "the knee"
    │                                                  ╱     latency explodes
 800┤                                                ╱       (queue builds faster
    │                                              ╱          than it drains)
 600┤                                            ╱
 400┤                                        ╱
 200┤                              ╱────────
    │            ╱────────────────
  50┤────────────  flat: server is idle enough to serve you immediately
    └────┬──────────┬──────────┬──────────┬──────────┬────────▶ Throughput (QPS)
        200        400        600        800       1000
                                          ▲
                              SAFE ◀──────┼──────▶ DANGER
                                    ~70% utilization
                                    (run here — headroom for spikes)
```

**Read this diagram like an SRE:** the flat part on the left is where you want to live. The "knee" is the point where arrival rate meets service rate; past it, the queue grows without bound and latency runs to infinity. **Capacity planning means finding the knee via load testing, then running at ~60-70% of it.** The gap is not waste — it is the buffer that absorbs traffic spikes, deploys, and a dead node.

### Diagram 3: Tail latency amplification

```
                      ┌──────────────────┐
      user request ──▶│   API Gateway    │  waits for ALL 10 → as slow as the SLOWEST
                      └────────┬─────────┘
        ┌──────┬──────┬──────┬─┴────┬──────┬──────┬──────┬──────┬──────┐
        ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼      ▼
      ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐  ┌───┐
      │S1 │  │S2 │  │S3 │  │S4 │  │S5 │  │S6 │  │S7 │  │S8 │  │S9 │  │S10│
      └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘  └───┘
       10ms   12ms   9ms   11ms  ┌────────┐  10ms  13ms   9ms   10ms
                                 │ 100ms  │  ← 1-in-100 chance per service...
                                 │ (p99!) │     ...but with 10 services,
                                 └────────┘     it happens ~10% of the time.

      P(page is slow) = 1 - (0.99)^10 = 9.6%     ← the p99 of ONE became the p90 of ALL
      P(page is slow) = 1 - (0.99)^100 = 63%     ← at fan-out 100, it becomes the p50
```

---

## Real world examples

### 1. Amazon — latency is revenue

Amazon has publicly discussed experiments showing that an extra **100ms of page latency cost about 1% in sales**. That single number reshaped their architecture: they aggressively decomposed the page into independently rendered components, so a slow recommendations service degrades (renders a placeholder or is omitted) rather than delaying the whole page. Their internal services carry hard latency budgets and a slow dependency gets *cut off* rather than waited on — a direct defence against tail latency amplification.

### 2. Google — hedged requests and "The Tail at Scale"

Google's search fans a query out to hundreds or thousands of index shards and must wait for the results. With that fan-out, per-shard p99 stragglers would dominate every query. Their published approach (the "The Tail at Scale" paper by Dean and Barroso) includes **hedged requests** (send the same request to a second replica after a short delay and use whichever returns first) and **tied requests** (replicas tell each other when they've started, so the duplicate gets cancelled). They report these techniques cutting p99 latency dramatically for a few percent of extra load.

### 3. Netflix — throughput at the edge, latency at the CDN

Netflix's streaming problem is a **throughput** problem (tens of terabits per second of video bytes) with a hard **latency** requirement only at the *start* of playback and during quality switches. Their answer is Open Connect: physical caching appliances placed inside ISP networks. This doesn't make their servers faster — it removes the long-haul network round trip entirely, moving the bytes physically closer to the user. It's the clearest possible demonstration of the rule *"you cannot beat the speed of light; you can only shorten the distance."*

---

## Trade-offs

| Optimize for LATENCY | Pros | Cons |
|---|---|---|
| Small/no batching | Every request answered immediately | More round trips, higher per-request overhead, lower ceiling |
| Low utilization (50-70%) | Short queues, stable p99 | You pay for idle capacity |
| Replicate data close to users (CDN, multi-region) | Kills the speed-of-light tax | Expensive; introduces consistency problems (topic 08) |
| Hedged requests | Big p99 win | Extra load; duplicate side effects — never hedge a non-idempotent call |

| Optimize for THROUGHPUT | Pros | Cons |
|---|---|---|
| Batching / bulk writes | 10-100x fewer round trips, huge cost savings | Every item waits for the batch window |
| High utilization (90%+) | Fewest servers, lowest bill | Queueing collapse; one traffic spike takes you down |
| Async processing (queue the work) | Perceived latency drops, absorbs spikes | End-to-end latency grows; you now need idempotency (topic 85) + a DLQ |
| Compression | Less bandwidth, more requests per link | CPU cost — can *raise* latency on already-fast links |

| Metric | Use it when | Never use it for |
|---|---|---|
| **Average / mean** | Rough capacity math (Little's Law wants means) | Alerting, SLOs, user-experience claims |
| **p50** | "What does a typical user feel?" | Anything about your worst users |
| **p95 / p99** | SLOs, alerting, capacity headroom | Cost estimation |
| **p999 / max** | Debugging rare bugs, GC pauses, huge payloads | Alerting (too noisy — you'd page on one bad request) |

**Rule of thumb:** Set your SLO on **p99**, alert on **p99**, run your fleet at **~60-70% utilization**, and treat **any use of "average latency" as a bug in the conversation.** Optimize latency first for anything a human is waiting on; optimize throughput for anything a machine is waiting on.

---

## Common interview questions on this topic

### Q1: "What's the difference between latency and throughput? Can you have low latency and low throughput at the same time?"
**Hint:** Latency = time for one request; throughput = requests per second. And yes, absolutely — a single-threaded server that answers in 1ms but only handles one request at a time has *excellent* latency (1ms) and *terrible* throughput (1,000 QPS max). Conversely a batch pipeline can push 1M records/sec with a 10-minute latency. They're independent axes; use Little's Law (`L = λW`) to connect them.

### Q2: "Your service has an average latency of 50ms. Is that good?"
**Hint:** The right answer is a question back: *"What's the p99?"* An average of 50ms is compatible with a p99 of 5 seconds. Averages blend two populations (fast and slow) into a number nobody experienced. Show the 10-sample example: nine 10ms requests plus one 3901ms request gives an average of 400ms — a number that describes nobody. Always ask for percentiles.

### Q3: "Each of your 10 microservices has a p99 of 100ms. What's the p99 of a page that calls all 10 in parallel?"
**Hint:** Much worse than 100ms. The page is as slow as its slowest call. `P(no straggler) = 0.99^10 = 90.4%`, so ~9.6% of pages hit at least one 100ms call — the single-service p99 became roughly the aggregate's **p90**. This is tail latency amplification. Mitigations: reduce fan-out, hedged/tied requests, per-call timeouts with fallbacks, and fixing the tail's root cause (GC, cold caches).

### Q4: "We need to handle 5,000 QPS and each request takes 200ms. How much concurrency do we need?"
**Hint:** Little's Law. `L = λ × W = 5000 × 0.2 = 1,000 requests in flight.` So you need 1,000 concurrent slots. Then add the operational layer: don't run at 100% — target ~65% utilization, so provision for ~7,700 QPS of capacity. And check the downstream: if your DB pool is 100 connections and each query is 20ms, that DB can only do `100/0.02 = 5,000 QPS` — you're exactly at the cliff.

### Q5: "Your p99 latency spiked but CPU, memory, and traffic all look normal. Where do you look?"
**Hint:** The tail is where rare things live. Check: (a) **event-loop delay** in Node — one sync CPU-heavy handler blocks everyone; (b) **GC pauses**; (c) **connection pool exhaustion** — requests are queueing for a DB connection, so it looks idle while everyone waits; (d) a **downstream dependency's** tail (amplification); (e) **cold cache / cache stampede** after a deploy or eviction; (f) a **data skew** — one shard or one very large row. Note that pool exhaustion and queueing produce exactly this signature: high latency, low CPU.

---

## Practice exercise

### The Percentile Detective

Build a tiny Node script — no libraries — that proves to yourself that averages lie.

**Step 1 — Generate a realistic latency distribution.** Produce 10,000 fake latency samples where:
- 95% of samples are drawn from a "fast path": a random value between 5ms and 30ms
- 4% are drawn from a "slow path" (cache miss → DB): 80ms to 200ms
- 1% are drawn from a "disaster path" (GC pause + retry): 1000ms to 4000ms

**Step 2 — Compute and print:** `mean`, `p50`, `p90`, `p95`, `p99`, `p999`, `max`.

**Step 3 — Answer these in a comment at the bottom of your file:**
1. How far off is the mean from the p50? Which one would you put on a status page, and why?
2. Your SLO is "p99 < 300ms." Do you pass? What single change to the distribution would fix it — and would improving the *fast path* help at all?
3. If this service is called 20 times in parallel to build one page, what fraction of pages hit at least one sample from the disaster path? (Compute `1 - 0.99^20`.) Now argue for or against reducing the fan-out.

**Step 4 (stretch) — Little's Law.** At 2,000 QPS with this distribution, use `L = λ × W` (with W = the *mean*, since Little's Law works on means) to compute how many requests are in flight on average. Then compute it again using the p99 as W, and explain in two sentences why that second number isn't what Little's Law is for — but why it still tells you something useful about your worst moments.

**Produce:** one `.js` file that runs with `node percentiles.js` and prints the table, plus your written answers.

---

## Quick reference cheat sheet

- **Latency** = time for ONE request (ms). **Throughput** = requests per SECOND (QPS). Different units, different axes.
- **They are not inverses.** `throughput ≠ 1/latency` unless concurrency is 1. Concurrency is the bridge.
- **Little's Law: `L = λ × W`** — requests in flight = throughput × latency. Use it to size thread pools, DB connection pools, and worker counts.
- **The utilization trap:** wait time ≈ `service × ρ/(1−ρ)`. At 99% utilization you queue **99x** the service time. Run at **60-70%**.
- **Averages lie.** Nine 10ms requests + one 4000ms request = "400ms average" — a number no request experienced. Never alert on the mean.
- **Percentiles:** p50 = typical user; p95/p99 = the users who churn; p999 = where bugs hide. **Set SLOs on p99.** At 1,000 QPS a 2-second p99 means 10 furious users *every second* — the tail is not an edge case at scale.
- **Tail latency amplification:** fan out to N services and `P(slow) = 1 − 0.99^N`. At N=10 the p99 becomes the p90; at N=100 it becomes the p50. Fight it with hedged requests, smaller fan-out, hard timeouts + fallbacks.
- **Numbers to memorize:** RAM read 100ns · SSD random read 150µs · same-DC round trip **0.5ms** · cross-continent round trip **150ms** · indexed DB query 1-10ms · human "instant" threshold **100ms**.
- **RAM is ~100,000x faster than a global network hop** — this one fact justifies every cache and every CDN. And **you cannot beat the speed of light**: the only cure for geographic latency is moving data closer.
- **Batching buys throughput with latency.** Great for analytics events, fatal for a payment confirmation.
- **In Node, watch `monitorEventLoopDelay`.** One synchronous CPU-heavy handler sets a floor under *everyone's* latency.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [05 — Capacity Estimation](./05-capacity-estimation.md) — where the QPS numbers you feed into Little's Law come from |
| **Next** | [07 — Availability and Reliability](./07-availability-and-reliability.md) — the other half of non-functional requirements: not "how fast" but "how often does it work at all" |
| **Related** | [12 — Numbers Every Engineer Should Know](./12-numbers-every-engineer-should-know.md) — the full latency table, expanded and drilled |
| **Related** | [59 — Caching in Depth](./59-caching-in-depth.md) — the rare optimization that improves latency AND throughput at the same time |
| **Related** | [80 — Monitoring and Observability](./80-monitoring-and-observability.md) — how to actually collect p99s in production with Prometheus histograms |
