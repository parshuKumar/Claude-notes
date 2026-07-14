# 70 — Rate Limiting — Algorithms and Implementation
## Category: HLD Components

---

## What is this?

Rate limiting is a **cap on how many requests a client may make in a given time window**. "100 requests per minute per user." Go over, and the extra requests are refused instead of served.

Think of a nightclub with a bouncer and a counter clicker. The club holds 200 people. He clicks up on entry, down on exit, and at 200 he simply stops letting people in — not because he dislikes them, but because a club with 500 people inside is a disaster for everyone already inside. Rate limiting is that bouncer, standing in front of your API.

---

## Why does it matter?

Without a rate limiter, **any single client can consume all of your capacity**. That is not a theoretical worry — it is the normal failure mode of a public API. Four concrete reasons:

**1. Abuse and DoS protection.** A scraper or credential-stuffing bot hammers `POST /login` 5,000 times a second. Your login service falls over and *legitimate* users can't sign in. The attack didn't have to be sophisticated — just fast.

**2. Cost control.** You expose `POST /reports/generate`, which runs a 4-second query and costs real money in compute. A well-meaning customer writes a `for` loop over 100,000 IDs and fires them all. Your cloud bill for that afternoon is four figures. A rate limit turns "unbounded cost" into "bounded cost."

**3. Fairness between tenants.** In a multi-tenant SaaS, Tenant A has a runaway cron job. Without limits, A eats 90% of your database connections and Tenants B through Z see timeouts. The limiter is what makes one tenant's bad day *their* bad day — the classic "noisy neighbour" problem.

**4. Protecting fragile downstream services.** Your API is fast, but it calls a legacy billing system that handles at most 50 requests/second. Let 500 rps through and you don't just fail — you take down billing for the whole company. A limiter is a **backpressure** valve: refusing load at the edge is far cheaper than a cascading failure deep inside.

**Interview angle:** it appears as a *component* ("how would you protect this API?") in nearly every HLD, and as a *whole question* ("design a distributed rate limiter"). The distributed part is where most candidates fall down.

**Work angle:** you will configure one — in Nginx, an API gateway, Cloudflare, or hand-rolled in Express with Redis — within your first year on any backend team.

---

## The core idea — explained simply

### The Toll Booth Analogy

A toll booth on a bridge. The bridge safely carries 1,000 cars an hour. More than that and traffic seizes and *nobody* crosses. The operator has five ways to control flow — and each one is literally one of the five algorithms.

**A — the hourly clipboard.** He keeps a tally sheet, tears off a fresh one every hour on the hour. Once 1,000 cars cross, the gate stays down until the clock ticks over. Simple. But drivers *know* this: 1,000 pour across at 10:59, then 1,000 more at 11:00. Two thousand cars in two minutes. → **Fixed window counter.**

**B — the photograph log.** He photographs every car with a timestamp. On each arrival he discards photos older than 60 minutes and counts what's left. Perfectly accurate — never 1,000+ in *any* rolling hour. But his filing cabinet now holds a million photographs, and he searches it on every single car. → **Sliding window log.**

**C — the blended estimate.** He keeps only *two* numbers: last hour's total and this hour's running total. Fifteen minutes in he reasons: "this hour is 25% done, so 75% of last hour still overlaps my rolling window." He blends them. An estimate — but cheap, and it kills the 10:59/11:00 stampede. → **Sliding window counter.**

**D — the coin bucket.** A machine drops a coin into a bucket every 3.6 seconds; the bucket holds at most 1,000. Each car pays a coin; empty bucket, gate down. A car arriving after a quiet night finds a *full* bucket, so 1,000 cars cross immediately — a burst — but long-term they can never exceed one car per 3.6 seconds. → **Token bucket.**

**E — the metered gate.** The gate opens exactly once every 3.6 seconds, no matter what. Faster arrivals wait in a holding lane; when the lane is full, they're turned away. Traffic *leaving* the booth is perfectly, boringly smooth. → **Leaky bucket.**

| Analogy | Algorithm | Optimises for |
|---|---|---|
| Hourly clipboard | Fixed window counter | Simplicity, minimum memory |
| Photograph log | Sliding window log | Perfect accuracy |
| Blended estimate | Sliding window counter | Accuracy-per-byte — the practical compromise |
| Coin bucket | Token bucket | Allowing controlled bursts |
| Metered gate | Leaky bucket | Perfectly smooth output |

The whole topic is: **which of these five, keyed on what, enforced where — and how do you make it work when there are ten toll booths and one bridge?**

---

## Key concepts inside this topic

### 1. Fixed window counter — the simplest, and its fatal flaw

Chop time into fixed clock windows (per minute, say). Keep one integer per client per window. Increment on each request; reject above the limit; reset to zero at the boundary.

```
Limit: 5 req/min
 11:00:00 ───────── window A ────────── 11:01:00 ───────── window B ─────────
 counter: 0→1→2→3→4→5  [rejects...]              counter: 0→1→2 ...
                        reset ───────────────────▶
```

```javascript
// Fixed window counter. Memory: ONE integer per client. That's the whole appeal.
class FixedWindowLimiter {
  constructor({ limit, windowMs }) {
    this.limit = limit;
    this.windowMs = windowMs;
    this.counters = new Map(); // key -> { windowStart, count }
  }

  allow(key, now = Date.now()) {
    // Floor `now` to the window boundary, so every request in the same
    // minute computes the SAME windowStart and shares one counter.
    const windowStart = Math.floor(now / this.windowMs) * this.windowMs;
    const resetAt = windowStart + this.windowMs;
    const entry = this.counters.get(key);

    if (!entry || entry.windowStart !== windowStart) {
      // New window: the old count is thrown away. This wholesale forgetting
      // is EXACTLY what makes the boundary burst below possible.
      this.counters.set(key, { windowStart, count: 1 });
      return { allowed: true, remaining: this.limit - 1, resetAt };
    }
    if (entry.count >= this.limit) return { allowed: false, remaining: 0, resetAt };

    entry.count += 1;
    return { allowed: true, remaining: this.limit - entry.count, resetAt };
  }
}
```

**The flaw, with a real timeline.** Limit = 100 req/min.

| Wall clock | Window | Sent | Counter after | Result |
|---|---|---|---|---|
| 11:00:59.000 | 11:00–11:01 | 100 | 100 | all 100 allowed |
| 11:01:00.000 | 11:01–11:02 | 100 | 100 | all 100 allowed |

In the **one second from 11:00:59 to 11:01:00 the client got 200 requests through** — twice the limit you thought you were enforcing. Your backend, sized for 100/min, just took a 200-request spike. Generally: fixed windows permit up to **2× the limit** across any boundary. That is the single fact interviewers listen for.

**Use it when** the limit is a coarse safety net (10,000/day) and a 2× overshoot is survivable. **Don't** when the thing you're protecting actually breaks at 2×.

---

### 2. Sliding window log — perfectly accurate, unaffordably fat

Store the **timestamp of every request**. On each new one: drop timestamps older than `now - windowMs`, count what remains, and if under the limit, append.

```
Limit 5 / 60s.   now = 11:01:30, window = [11:00:30, 11:01:30]

log: [11:00:05] [11:00:20] │ [11:00:40] [11:01:02] [11:01:25] │ + new
      ─ evicted (too old) ─┘ ──── inside the window: 3 ───────┘
      3 < 5  →  ALLOW, append 11:01:30
```

```javascript
// Sliding window log. Exact — never allows limit+1 in ANY rolling window.
class SlidingWindowLogLimiter {
  constructor({ limit, windowMs }) {
    this.limit = limit;
    this.windowMs = windowMs;
    this.logs = new Map(); // key -> ascending array of timestamps
  }

  allow(key, now = Date.now()) {
    const cutoff = now - this.windowMs;
    const log = this.logs.get(key) ?? [];

    // Timestamps are appended in order, so everything expired is a PREFIX:
    // find the first survivor and slice once, rather than shifting per item.
    const firstAlive = log.findIndex((t) => t > cutoff);
    const alive = firstAlive === -1 ? [] : log.slice(firstAlive);
    this.logs.set(key, alive);

    if (alive.length >= this.limit) {
      // Retry-After = when the OLDEST survivor falls out of the window.
      return { allowed: false, remaining: 0, retryAfterMs: alive[0] + this.windowMs - now };
    }

    alive.push(now);
    return { allowed: true, remaining: this.limit - alive.length, retryAfterMs: 0 };
  }
}
```

**The flaw: O(n) memory per client**, where n is the limit. A user with a 5,000/hour limit who *uses* it stores 5,000 timestamps — 40 KB at 8 bytes each, for **one** user. A million active users: **40 GB of RAM just for timestamps.** In Redis you'd model each user as a sorted set (`ZREMRANGEBYSCORE` + `ZCARD` + `ZADD`) and the overhead is worse, since every member carries its own bookkeeping. You also pay CPU proportional to the expired entries on every request.

**Use it when** the limit is small (≤ ~100) or the client population is small (internal services, a few partners) and a single extra request costs real money.

---

### 3. Sliding window counter — the practical compromise

Keep **two counters**: the previous window's total and the current window's total. Estimate the rolling count by weighting the previous window by how much of it still overlaps.

```
                weight = 1 − (elapsedInCurrent / windowMs)
  ┌──── previous window ────┐┌──── current window ────┐
  │       count = 80        ││      count = 30         │
  └─────────────────────────┘└────────┬────────────────┘
             ◀── rolling window ──▶   │
                                     now (25% into current)
```

**The formula:**

```
estimate = previousCount × (1 − elapsedFraction) + currentCount
```

**Worked arithmetic.** Limit = 100/min. It is 11:01:15 — **15 s into a 60 s window → elapsedFraction = 0.25**. Previous window saw **80**; current window so far, **30**.

```
estimate = 80 × (1 − 0.25) + 30
         = 80 × 0.75 + 30
         = 60 + 30 = 90
90 < 100  →  ALLOW   (current counter becomes 31)
```

Now watch it kill the boundary burst. Same client blasts its full **100** in the previous window, then tries 100 more at 11:01:00 (`elapsedFraction ≈ 0`):

```
estimate = 100 × (1 − 0.0) + 0 = 100   →  100 >= 100  →  REJECT
```

The fixed-window doubling is gone: the previous window's spend still counts against you and only *decays* away as the current window advances.

```javascript
// Sliding window counter: 2 integers per client, near-exact behaviour.
class SlidingWindowCounterLimiter {
  constructor({ limit, windowMs }) {
    this.limit = limit;
    this.windowMs = windowMs;
    this.state = new Map(); // key -> { windowStart, current, previous }
  }

  allow(key, now = Date.now()) {
    const windowStart = Math.floor(now / this.windowMs) * this.windowMs;
    let s = this.state.get(key);

    if (!s) {
      s = { windowStart, current: 0, previous: 0 };
    } else if (s.windowStart !== windowStart) {
      // Roll forward. If we skipped MORE than one window, the "previous"
      // window is stale too — zero it, or we'd charge the client for
      // traffic that is long out of the rolling window.
      const skipped = (windowStart - s.windowStart) / this.windowMs;
      s = { windowStart, previous: skipped === 1 ? s.current : 0, current: 0 };
    }

    const elapsedFraction = (now - windowStart) / this.windowMs; // 0.0 → 1.0
    const estimate = s.previous * (1 - elapsedFraction) + s.current;
    this.state.set(key, s);

    if (estimate >= this.limit) {
      return { allowed: false, remaining: 0, resetAt: windowStart + this.windowMs };
    }
    s.current += 1;
    return { allowed: true, remaining: Math.max(0, Math.floor(this.limit - estimate - 1)) };
  }
}
```

**The flaw: it's an approximation.** It assumes the previous window's traffic was spread *evenly*. If those 80 requests all landed in the last two seconds of that window, the true rolling count is higher than the estimate and a few extra slip through. Cloudflare measured this against exact logs on live traffic and found the error to be a fraction of a percent — which is why it's the standard choice.

**Use it when** you're at scale and can tolerate ~1% slop. This is the default recommendation for most APIs.

---

### 4. Token bucket — bursts allowed, average enforced

A bucket holds up to `capacity` tokens, refilled at `refillPerSec`. Each request removes one. No token → reject.

```
      refill: 10 tokens/sec
             │
             ▼
       ┌───────────┐
       │ ● ● ● ● ● │  capacity = 50   ← an idle client sits here: FULL bucket
       │ ● ● ● ● ● │
       └─────┬─────┘
             │ 1 token per request
             ▼
   request ──▶ token available? ──yes──▶ ALLOW
                     │
                     no ──▶ REJECT (429)
```

**The key property: controlled bursts.** A client idle for a minute has a **full bucket** and may fire 50 requests instantly. Once drained, it is throttled to the refill rate — 10 req/sec, forever. So the bucket enforces a **long-run average of 10 rps with a burst allowance of 50**. That is exactly what real clients need: a mobile app waking up and firing 8 calls to render one screen shouldn't be punished, but nor should it sustain 8 rps for an hour.

**Worked timeline.** `capacity = 10`, `refill = 2/sec`. Client idle, bucket full.

| Time | Event | Tokens before | After | Result |
|---|---|---|---|---|
| t=0.0s | burst of 10 | 10 | 0 | all 10 ALLOWED — the burst |
| t=0.1s | 1 request | 0.2 | — | REJECTED (0.2 < 1) |
| t=1.0s | 1 request | 2.0 | 1.0 | ALLOWED |
| t=1.2s | 1 request | 1.4 | 0.4 | ALLOWED |
| t=1.3s | 1 request | 0.6 | — | REJECTED |
| t=6.0s | 1 request | 10 (capped) | 9 | ALLOWED |

Look at t=6.0s: 4.7 idle seconds × 2 tokens/sec = 9.4 tokens earned, but the bucket **caps at 10**. Unused allowance does not accumulate forever — that cap is what bounds the maximum burst.

**Lazy refill — the implementation trick.** Do *not* run a `setInterval` per bucket; a million users would mean a million timers. Store `lastRefillAt` and compute tokens earned from elapsed time, on demand, at request time. Zero timers, exact arithmetic.

```javascript
// Token bucket with LAZY refill: no timers, tokens derived from elapsed time.
class TokenBucketLimiter {
  /** capacity = max burst size;  refillPerSec = sustained long-run rate */
  constructor({ capacity, refillPerSec }) {
    this.capacity = capacity;
    this.refillPerSec = refillPerSec;
    this.buckets = new Map(); // key -> { tokens, lastRefillAt }
  }

  #refill(bucket, now) {
    const elapsedSec = (now - bucket.lastRefillAt) / 1000;
    if (elapsedSec <= 0) return;
    // Tokens stay FRACTIONAL on purpose — integer maths would silently
    // round away the allowance of any client slower than 1 req/sec.
    bucket.tokens = Math.min(this.capacity, bucket.tokens + elapsedSec * this.refillPerSec);
    bucket.lastRefillAt = now;
  }

  allow(key, cost = 1, now = Date.now()) {
    let bucket = this.buckets.get(key);
    if (!bucket) {
      // New clients start FULL: never punish someone for showing up.
      bucket = { tokens: this.capacity, lastRefillAt: now };
      this.buckets.set(key, bucket);
    }
    this.#refill(bucket, now);

    if (bucket.tokens < cost) {
      // Exactly how long until enough tokens exist. This is your Retry-After.
      const retryAfterMs = Math.ceil(((cost - bucket.tokens) / this.refillPerSec) * 1000);
      return { allowed: false, remaining: 0, retryAfterMs };
    }
    bucket.tokens -= cost;
    return { allowed: true, remaining: Math.floor(bucket.tokens), retryAfterMs: 0 };
  }
}

// `cost` WEIGHTS endpoints: a cheap GET costs 1, an expensive report costs 25.
// Same bucket — but now the limit tracks real load, not request count.
```

**The flaw:** a client can legitimately dump `capacity` requests on you in one millisecond. If your downstream truly cannot take a burst, this is the wrong tool — you want leaky bucket. And two knobs instead of one means teams routinely set `capacity` far too high, then act surprised by spikes.

**Who uses it:** the workhorse. AWS API Gateway exposes exactly these two knobs (a *burst* limit and a *rate* limit); Stripe describes its limiter in token-bucket terms. Most cloud APIs behave this way.

---

### 5. Leaky bucket — perfectly smooth output

Requests are appended to a **FIFO queue** (the bucket). A worker drains it at a **strictly constant rate**. If the queue is full on arrival, the request overflows and is rejected.

```
  bursty arrivals            fixed-size queue        constant drain
  ▶ ▶▶▶  ▶    ▶▶▶▶           ┌───────────┐
  ──────────────────────────▶│ □ □ □ □ □ │────▶ ▶   ▶   ▶   ▶   ▶
                             │ (max 20)  │      one every 200ms
                             └─────┬─────┘
                                   │ full? ──▶ REJECT (overflow)
```

```javascript
// Leaky bucket: a queue that drains at a fixed rate. Output is SMOOTH.
class LeakyBucketLimiter {
  constructor({ capacity, leakPerSec }) {
    this.capacity = capacity;
    this.intervalMs = 1000 / leakPerSec; // 5/sec -> one every 200ms
    this.queues = new Map(); // key -> { queue: [], timer }
  }

  submit(key, job) {
    let b = this.queues.get(key);
    if (!b) this.queues.set(key, (b = { queue: [], timer: null }));

    // Bucket overflow. NOT "your request is slow" — it is DROPPED.
    if (b.queue.length >= this.capacity) {
      return Promise.reject(new Error('RATE_LIMITED: bucket full'));
    }

    return new Promise((resolve, reject) => {
      b.queue.push({ job, resolve, reject });
      if (!b.timer) this.#drain(b);
    });
  }

  #drain(b) {
    // The interval IS the rate limit. Downstream sees a perfectly even
    // stream no matter how spiky the arrivals were.
    b.timer = setInterval(async () => {
      const next = b.queue.shift();
      if (!next) { clearInterval(b.timer); b.timer = null; return; }
      try { next.resolve(await next.job()); } catch (err) { next.reject(err); }
    }, this.intervalMs);
  }
}
```

**Token vs leaky — the contrast that gets asked.** Same input: 10 requests in one millisecond, limit 5/sec.

```
TOKEN BUCKET (capacity 10)         LEAKY BUCKET (leak 5/sec)
in : ||||||||||                    in : ||||||||||
out: ||||||||||  ← burst passes    out: |  |  |  |  |  |  |  |  |  |
     (then throttled to 5/sec)          ← smoothed: one every 200ms
```

| | Token bucket | Leaky bucket |
|---|---|---|
| Bursts | **Allowed** up to capacity | **Flattened** — never exit faster than the leak rate |
| Excess | Rejected immediately | **Queued**, rejected only on overflow |
| Latency | Unchanged for accepted requests | Added — queued requests wait |
| Output shape | Spiky | Perfectly constant |
| Use when | Clients burst naturally and you want responsiveness | Downstream is fragile and needs an *even* feed |

Different tools. Token bucket protects *you* from too much total load while staying friendly. Leaky bucket protects *someone else* from ever seeing a spike. If your API calls a bank that permits exactly 50 rps, you want a **leaky bucket in front of the bank and a token bucket in front of your users**.

---

### 6. The distributed problem — where good answers become great ones

Everything above assumed one process with one `Map`. Now deploy 10 API servers behind a load balancer, each with its own in-memory limiter.

```
                     ┌── server 1  (local count: 100) ──┐
  client ──▶ LB ────▶│── server 2  (local count: 100) ──│──▶ your DB
   (round robin)     │        ...                        │
                     └── server 10 (local count: 100) ──┘

  Configured limit: 100 req/min.    ACTUAL limit: 1,000 req/min.
```

Your "100 req/min" is silently **1,000 req/min**. Worse, add an autoscaler and the real limit now *changes with your instance count* — a nasty bug, because the limiter appears to work in staging (one instance) and quietly fails in production (twenty).

**The fix: a shared counter in Redis.** But *how* you touch Redis matters enormously.

**The naive version, which races:**

```javascript
// ❌ BROKEN. Read-modify-write across a network is not atomic.
async function allowNaive(redis, key, limit, ttlSec) {
  const count = Number(await redis.get(key)) || 0;   // (1) read
  if (count >= limit) return false;                  // (2) decide ← in APP memory
  await redis.set(key, count + 1, 'EX', ttlSec);     // (3) write
  return true;
}
```

The race, on a limit of 100:

```
 t   Server A                    Server B                  Redis value
──────────────────────────────────────────────────────────────────────
 1   GET key → 99                                          99
 2                               GET key → 99              99
 3   99 < 100 → allow                                      99
 4                               99 < 100 → allow          99
 5   SET key 100                                           100
 6                               SET key 100               100  ← !!
```

Both servers read 99, both decided "there's room for one more," both wrote 100. **Two requests got through on the last slot: 101 requests served on a limit of 100.** With dozens of servers this leaks far more than one request — a check-then-act gap is a lost update, and the wider your fleet the wider the leak.

**Fix 1 — atomic `INCR`.** Redis is single-threaded; `INCR` is atomic and *returns the new value*. Decide from the return value, never from a prior `GET`.

```javascript
// ✅ Fixed window, distributed, atomic. Pipelined into ONE round trip.
async function allowFixedWindow(redis, userId, limit, windowSec) {
  const windowStart = Math.floor(Date.now() / 1000 / windowSec) * windowSec;
  const key = `rl:${userId}:${windowStart}`;

  const res = await redis.multi()
    .incr(key)                   // atomic; returns the POST-increment value
    .expire(key, windowSec + 1)  // self-cleaning: old windows evaporate
    .exec();

  const count = res[0][1];
  return { allowed: count <= limit, remaining: Math.max(0, limit - count) };
}
```

Subtlety: this increments *before* checking, so rejected requests are counted too. Usually fine (it makes abuse visible), but the counter can climb well past the limit.

**Fix 2 — a Lua script.** `INCR` only serves the *fixed window*. Token bucket must read tokens, compute a refill, compare, and write back — a multi-step check-and-act. Redis runs a Lua script **atomically, as one indivisible unit**, so nothing can interleave. This is the real production answer.

```lua
-- token_bucket.lua — atomic check-and-consume
-- KEYS[1] = bucket key   ARGV = capacity, refillPerSec, nowMs, cost
local capacity     = tonumber(ARGV[1])
local refillPerSec = tonumber(ARGV[2])
local now          = tonumber(ARGV[3])
local cost         = tonumber(ARGV[4])

local b      = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens = tonumber(b[1])
local lastTs = tonumber(b[2])

if tokens == nil then          -- unseen client: start with a FULL bucket
  tokens = capacity
  lastTs = now
end

-- Lazy refill from elapsed time — same maths as the JS version.
local elapsed = math.max(0, now - lastTs) / 1000
tokens = math.min(capacity, tokens + elapsed * refillPerSec)

local allowed = 0
if tokens >= cost then
  tokens  = tokens - cost
  allowed = 1
end

redis.call('HMSET', KEYS[1], 'tokens', tokens, 'ts', now)
-- TTL = time to refill empty→full. Idle buckets clean themselves up, and
-- re-creating a forgotten bucket as full is exactly the right behaviour.
redis.call('EXPIRE', KEYS[1], math.ceil(capacity / refillPerSec) + 1)

return { allowed, math.floor(tokens) }
```

```javascript
import { readFileSync } from 'node:fs';
import Redis from 'ioredis';

const redis = new Redis();
// defineCommand ships the script ONCE (EVALSHA), not on every call.
redis.defineCommand('tokenBucket', {
  numberOfKeys: 1,
  lua: readFileSync('./token_bucket.lua', 'utf8'),
});

export async function allow(key, { capacity = 50, refillPerSec = 10, cost = 1 } = {}) {
  const [allowed, remaining] = await redis.tokenBucket(
    `rl:tb:${key}`, capacity, refillPerSec, Date.now(), cost,
  );
  return { allowed: allowed === 1, remaining };
}
```

**The costs you must name in an interview:**

- **A Redis hop on every request.** Same-AZ Redis is ~0.5–1 ms. On a 20 ms endpoint that's a ~5% latency tax, and it sits on the critical path of *literally every request*. Acceptable for most APIs.
- **Redis is now a hard dependency.** If it's down, do you **fail open** (allow everything — you keep serving, but the limiter is gone) or **fail closed** (reject everything — safe, but you've caused your own outage)? For a public API guarding against abuse: usually **fail open**, and alert loudly. For a limiter protecting a downstream that will *melt* at 2× load: **fail closed**. Say this out loud.
- **Redis itself becomes the bottleneck** past roughly 100k+ rps. The standard compromise: **local buckets + approximate sync.** Each server holds a local token bucket carrying its *share* of the global budget, and asynchronously (every 100–500 ms) reports usage to Redis and pulls back a refreshed share. Redis leaves the request path; you over-allow slightly during each sync interval. That's the price, and it's the design behind systems like Google's Doorman.

---

### 7. What to key on

The key decides *who* is being limited. Get this wrong and the algorithm doesn't matter.

| Key | Good for | The danger |
|---|---|---|
| **User ID** | Authenticated traffic. The best default — it's the real unit of fairness. | Useless before login. Attackers just create accounts (so also limit signups). |
| **API key / tenant ID** | B2B APIs. Lets you sell tiers: free 100/min, pro 10,000/min. | A leaked key is abused as a legitimate one — pair with per-IP limits. |
| **IP address** | Anonymous traffic — the *only* option pre-auth (login, signup, password reset). | **Shared IPs.** An entire office, university, or country behind CGNAT is *one IP*: limit it and you ban them all. Meanwhile IPv6 and cheap proxies give an attacker millions of IPs. |
| **Endpoint** | Weighting expensive routes: `POST /reports` at 5/min, `GET /me` at 600/min. | More config surface; combinatorial explosion if overdone. |

**Layer them.** A realistic production config composes several limiters, all of which must pass:

```javascript
// Every layer must allow. Report the STRICTEST rejection so Retry-After is honest.
const layers = [
  { name: 'global-ip', key: (r) => `ip:${r.ip}`,               capacity: 1000, refillPerSec: 1000 / 60 },
  { name: 'per-user',  key: (r) => `u:${r.user?.id ?? r.ip}`,  capacity: 300,  refillPerSec: 300 / 60 },
  { name: 'expensive', key: (r) => `u:${r.user?.id}:${r.path}`, capacity: 5,   refillPerSec: 5 / 60,
    appliesTo: (r) => r.path.startsWith('/reports') },
];

export async function checkAll(req) {
  for (const layer of layers) {
    if (layer.appliesTo && !layer.appliesTo(req)) continue;
    const res = await allow(layer.key(req), layer);
    if (!res.allowed) return { ...res, blockedBy: layer.name };
  }
  return { allowed: true };
}
```

One extra warning on IP: behind a load balancer or CDN, `req.socket.remoteAddress` is the *proxy's* IP — every request looks like one client. Read the client IP from `X-Forwarded-For` **and** configure your trust boundary (`app.set('trust proxy', 1)` in Express), because a header the client can forge is a limiter the client can bypass.

---

### 8. The response — how to reject politely

Rejecting isn't enough; a good limiter **tells the client how to behave**.

- **`429 Too Many Requests`** — the correct code. Not 403 ("never allowed"), not 503.
- **`Retry-After: 30`** — seconds until it's worth trying again. Your algorithm already knows this exactly (token bucket: `deficit / refillRate`).
- **`X-RateLimit-Limit: 100`** / **`X-RateLimit-Remaining: 0`** / **`X-RateLimit-Reset: 1720780800`** — the ceiling, what's left, and when it refreshes (unix seconds).

Send the `X-RateLimit-*` headers on **successful** responses too — that's the entire point. A client watching `Remaining` fall from 100 to 12 can slow itself down *before* it gets a 429. Only sending them on rejections teaches the client nothing until it has already failed.

```javascript
// Express middleware. Note: headers on BOTH the 200 and the 429 path.
export function rateLimit({ capacity = 100, refillPerSec = 100 / 60 } = {}) {
  return async function rateLimitMiddleware(req, res, next) {
    const key = req.user?.id ? `u:${req.user.id}` : `ip:${req.ip}`;

    let result;
    try {
      result = await allow(key, { capacity, refillPerSec });
    } catch (err) {
      // FAIL OPEN: Redis is down. Serving unlimited traffic beats serving none.
      // Flip to a 503 if the thing you protect cannot survive an overload.
      req.log?.error({ err }, 'rate limiter unavailable — failing open');
      return next();
    }

    res.set('X-RateLimit-Limit', String(capacity));
    res.set('X-RateLimit-Remaining', String(result.remaining));
    res.set('X-RateLimit-Reset', String(Math.ceil(Date.now() / 1000) + 60));

    if (!result.allowed) {
      const retryAfter = Math.max(1, Math.ceil((result.retryAfterMs ?? 1000) / 1000));
      res.set('Retry-After', String(retryAfter));
      return res.status(429).json({
        error: 'rate_limit_exceeded',
        message: `Too many requests. Retry in ${retryAfter}s.`,
        retry_after: retryAfter,
      });
    }
    next();
  };
}
```

One more rule for clients: **retry with exponential backoff and jitter.** If 500 rejected clients all obey `Retry-After: 30` precisely, they all return at the same instant and you get a thundering herd. Jitter (`retryAfter + random(0, retryAfter / 2)`) spreads them out.

---

### 9. Where to enforce it, and soft vs hard limits

```
 client ──▶ ┌─────────┐ ──▶ ┌──────────────┐ ──▶ ┌────────────┐ ──▶ ┌──────┐
            │  CDN /  │     │ API Gateway  │     │  Service   │     │  DB  │
            │  Edge   │     │              │     │            │     └──────┘
            └─────────┘     └──────────────┘     └────────────┘
             cheapest,       central, knows       richest context,
             dumbest         auth + tenant        most expensive
```

| Where | Pros | Cons |
|---|---|---|
| **CDN / edge** (Cloudflare, CloudFront) | Blocks abuse before it touches your infra — zero origin cost. Globally distributed, absorbs volumetric attacks. | Knows only IP/headers, not business logic ("this tenant is on the pro plan"). Config lives outside your codebase. |
| **API gateway** (Kong, Envoy, AWS API GW) | The sweet spot. It already terminates auth, so it knows the user and tenant. One place to configure; no service reimplements it. | An extra hop; a shared choke point. |
| **In the service** | Full business context — price a request by its actual cost, per-endpoint, per-feature. | Every service must implement it. Load already reached your app. Easy to get inconsistent across teams. |

**Interview answer: all three, in layers.** Volumetric/IP limits at the edge, per-user/per-tenant at the gateway, expensive-endpoint limits in the service that owns the endpoint.

**Soft vs hard limits.** A hard limit **rejects** (429). A soft limit **degrades** instead:

- **Hard:** "You're over 100/min. 429." Correct for abuse, unauthenticated traffic, anything that must be *stopped*.
- **Soft:** serve from a stale cache; return a cheaper, lower-fidelity result (fewer search results, no personalisation); deliberately add 200 ms of delay (**throttling**) so the client slows without failing; or downgrade priority so the request runs only when there's spare capacity.

Soft limits are far kinder to paying customers who briefly exceed their plan — a 429 should not be a paying user's first experience of a limit. A common production ladder: **soft-throttle at 80% of limit → warn (headers + email) at 100% → hard-reject at 150%.**

---

## Visual / Diagram description

### Diagram 1: A distributed rate limiter, end to end

```
   ┌────────┐   ┌───────────────┐
   │ Client │──▶│  CDN / Edge   │ ── layer 1: crude per-IP volumetric limit
   └────────┘   └───────┬───────┘             (blocks floods before they cost you)
                        │ survivors
                        ▼
                ┌───────────────┐
                │ Load Balancer │
                └───────┬───────┘
            ┌───────────┼───────────┐
            ▼           ▼           ▼
      ┌─────────┐ ┌─────────┐ ┌─────────┐
      │ API GW  │ │ API GW  │ │ API GW  │ ── layer 2: per-user / per-tenant
      │ node 1  │ │ node 2  │ │ node 3  │    Each node is STATELESS. State must
      └────┬────┘ └────┬────┘ └────┬────┘    be SHARED, or the limit ×3.
           └───────────┼───────────┘
                       ▼
            ┌──────────────────────┐
            │   REDIS (shared)     │ ◀── EVAL token_bucket.lua  (ATOMIC)
            │  rl:tb:user_42       │     read + refill + compare + write
            │    tokens: 37.4      │     as ONE indivisible step
            │    ts: 1720780812345 │     ~0.7 ms same-AZ round trip
            └──────────┬───────────┘
                       │ allowed?
            ┌──────────┴──────────┐
           yes                    no
            ▼                     ▼
   ┌──────────────┐      ┌──────────────────┐
   │ Service      │      │ 429              │
   │ (layer 3:    │      │ Retry-After: 12  │
   │  per-endpoint│      │ X-RateLimit-*    │
   │  cost limits)│      └──────────────────┘
   └──────────────┘
```

**What it shows.** Three layers, each with more context than the last, and the crucial detail: the gateway nodes hold **no counters of their own**. The counter lives in Redis, and the check-and-decrement happens *inside* Redis as one atomic Lua call — the only thing standing between you and the read-modify-write race. Redraw this on a whiteboard and you have most of "design a distributed rate limiter" already answered.

### Diagram 2: The fixed-window boundary burst

```
 limit = 100/min
           window 11:00–11:01          window 11:01–11:02
 ┌──────────────────────────────┐┌──────────────────────────────┐
 │                       ██████ ││ ██████                       │
 │                       100 req││ 100 req                      │
 └───────────────────────┬──────┘└─┬────────────────────────────┘
                    11:00:59      11:01:00
                         └────┬────┘
                     1 second │ 200 requests
                              ▼
                    ██  2× the limit  ██
```

The counter reset is a cliff. Any traffic pattern that hugs the boundary from both sides doubles your effective limit. The sliding window counter fixes exactly this, by refusing to forget the previous window all at once.

---

## Real world examples

### 1. Stripe

Stripe describes its API rate limits in token-bucket terms: a bucket per API key that refills continuously, which is why a client can burst briefly and must then settle to a sustained rate. They return `429` and publicly advise clients to retry with **exponential backoff**. They also separate *request rate* limits from **concurrency** limits (how many of your requests may be in flight at once) — because a handful of very slow requests can hurt as much as a flood of fast ones. Almost nobody mentions concurrency limits in an interview; it's an easy way to stand out.

### 2. GitHub API

GitHub allows 5,000 requests/hour for authenticated requests keyed on the token, and a far lower limit for unauthenticated requests keyed on IP. Every response carries `x-ratelimit-limit`, `x-ratelimit-remaining` and `x-ratelimit-reset`. Their GraphQL API prices queries by **computed cost** rather than request count: a query fanning out to thousands of nodes costs many "points," a trivial one costs one. That is exactly the weighted-`cost` idea from the token-bucket code above, running in production.

### 3. Cloudflare

Cloudflare rate limits at the **edge**, across hundreds of PoPs, before traffic reaches an origin at all. Their published work on the **sliding window counter** is the standard reference for why the two-counter approximation is good enough: measured against exact sliding-window logs on live traffic, the error rate was tiny, while the memory cost is a rounding error next to storing per-request timestamps. It is the clearest real-world argument for the accuracy-vs-memory trade-off.

---

## Trade-offs

| Algorithm | Memory / client | Accuracy | Allows bursts? | Complexity |
|---|---|---|---|---|
| Fixed window | 1 integer | Poor — 2× at boundaries | Accidentally | Trivial |
| Sliding window log | O(limit) timestamps | Exact | No | Low |
| Sliding window counter | 2 integers | ~99% | No | Low |
| Token bucket | 2 numbers (tokens, ts) | Exact for its model | **Yes, by design** | Medium |
| Leaky bucket | O(queue size) | Exact | **No — flattens them** | Medium (needs a worker) |

| Decision | You gain | You give up |
|---|---|---|
| In-memory limiter | Zero latency, no dependency | Correctness across N servers — the limit silently becomes N× |
| Redis-backed limiter | One true global limit | ~1 ms per request; Redis is now on your critical path |
| Local cache + async sync | Redis off the hot path; scales past 100k rps | Exactness — you over-allow during each sync interval |
| Fail open on limiter outage | You keep serving traffic | The limiter is gone exactly when an attack might be causing the outage |
| Fail closed on limiter outage | The protected system stays safe | A limiter outage becomes a full API outage |
| Hard limit (429) | Absolute protection | Real users see errors |
| Soft limit (degrade/throttle) | Paying users stay happy | Load isn't actually shed — you must still cap somewhere |

**The sweet spot:** **token bucket in Redis via a Lua script, keyed on user ID (falling back to IP for unauthenticated routes), enforced at the API gateway, with per-endpoint cost weights for expensive routes.** Fail open, return proper `429` + `Retry-After` + `X-RateLimit-*` headers, and put a crude per-IP limit at the CDN in front of everything. That one sentence is a complete, credible answer to most rate-limiting questions.

**Rule of thumb:** if you can't say what happens when your rate limiter's datastore goes down, you haven't finished designing it.

---

## Common interview questions on this topic

### Q1: "Design a rate limiter for an API serving 1 million users at 10,000 requests/second."
**Hint:** Walk the ladder. Requirements → 100 req/min per user, distributed, low latency. Pick **token bucket** (real clients burst naturally). State the distributed problem explicitly — 10 servers × local counters = 10× the limit — then move state to Redis and make check-and-consume atomic with a **Lua script**. Do the memory maths out loud: 1M users × ~64 bytes/bucket ≈ **64 MB**, which fits comfortably in one Redis node, so you shard for *throughput*, not memory. Finish with 429 + `Retry-After` + headers, and say what happens when Redis dies (fail open, alert).

### Q2: "Why not just use a fixed window counter? It's the simplest."
**Hint:** Draw the 11:00:59 / 11:01:00 timeline and show 2× the limit passing in one second. Then ask what you're protecting: if it's a downstream that dies at 1.5× load, that 2× spike is an outage. Offer the sliding window counter as the near-free fix — still two integers, but the boundary burst disappears.

### Q3: "Token bucket vs leaky bucket — when would you pick each?"
**Hint:** Token bucket **allows** bursts up to capacity while bounding the long-run average; leaky bucket **flattens** them into a constant output stream. Pick token bucket for user-facing APIs — an app opening fires 8 calls at once and punishing that is bad UX. Pick leaky bucket when the thing downstream cannot tolerate a spike at all: a legacy mainframe, a partner API with a hard 50 rps cap, a payment processor. They compose — token bucket facing your users, leaky bucket facing the fragile dependency.

### Q4: "Your Redis counter is at 99 and two servers check it at the same instant. What happens?"
**Hint:** The race question — a trap for anyone who wrote `GET` → compare → `SET`. Both read 99, both decide there's room, both write 100: **101 requests served on a limit of 100.** The gap between check and act *is* the bug. Fix it with an operation that is atomic *and returns the new value*: `INCR` for fixed windows, a Lua script for anything needing read-compute-write (token bucket). Redis executes a script as one indivisible unit, so no interleaving is possible.

### Q5: "You rate limit by IP. What breaks?"
**Hint:** Shared IPs. A 3,000-person office, a university campus, or an entire mobile carrier behind CGNAT presents as **one IP** — limit it and you've banned thousands of innocent users. Meanwhile the attacker rents 10,000 IPs for pocket change, so IP limiting barely inconveniences the people you're aiming at. Use IP only where you have no choice (pre-auth: login, signup, password reset), set it generously, and layer a per-user or per-API-key limit as the real control. Also: behind a proxy you must read `X-Forwarded-For` with a configured trust boundary, or every request appears to come from your load balancer.

### Q6: "How do you limit an endpoint that's 50× more expensive than the others?"
**Hint:** Don't count requests — count **cost**. Give each request a weight and deduct that many tokens from the same bucket: `GET /me` costs 1, `POST /reports/generate` costs 50. One bucket, and now the limit tracks actual load rather than request count. GitHub's GraphQL API does exactly this with query "points."

---

## Practice exercise

### Build and Break a Rate Limiter

**~35 minutes. Produce: one Node file plus a short written answer.**

**Part A — implement (20 min).** In `limiter.js`, export **two** in-memory limiters sharing the interface `allow(key, now) -> { allowed, remaining }`:
1. `FixedWindowLimiter({ limit: 5, windowMs: 60_000 })`
2. `TokenBucketLimiter({ capacity: 5, refillPerSec: 5 / 60 })` — lazy refill, **no timers**.

**Part B — prove the flaw (10 min).** Write a test that does **not** use real time — pass `now` in explicitly. Feed both limiters this exact sequence and print the decision for each request:

```
5 requests at t = 11:00:59.000
5 requests at t = 11:01:00.000
```

Confirm the fixed window allows **all 10** (2× the limit in one second), while the token bucket allows roughly 5, since it refilled by only about `5/60` of a token in that second. **Write down the two numbers you observe.**

**Part C — go distributed (5 min, written).** In three or four sentences: you now run 8 servers. Explain (a) what your configured limit of 5/min *actually* becomes, (b) why moving the counter to Redis with `GET` then `SET` still leaks requests, and (c) the exact Redis primitive you'd use instead and why it closes the gap.

**Stretch:** add a `cost` parameter to the token bucket and show that one 5-cost request drains the same bucket as five 1-cost requests.

---

## Quick reference cheat sheet

- **Rate limiting** = capping requests per client per unit time. Four reasons: abuse/DoS, cost control, tenant fairness, protecting fragile downstreams.
- **Fixed window** — one integer per client. **Flaw: 2× the limit across a boundary** (100 at 11:00:59 + 100 at 11:01:00 = 200 in one second).
- **Sliding window log** — a timestamp per request. Perfectly exact. **Flaw: O(limit) memory per user** — unaffordable at scale.
- **Sliding window counter** — `prev × (1 − elapsedFraction) + current`. Two integers, ~99% accurate, kills the boundary burst. **The practical default.**
- **Token bucket** — capacity N, refill R/sec, one token per request. **Allows bursts, bounds the average.** What Stripe and AWS use. Always use **lazy refill** — derive tokens from elapsed time, never run a timer.
- **Leaky bucket** — a queue draining at a constant rate. **Flattens** bursts instead of allowing them. For fragile downstreams.
- **Token vs leaky:** token bucket lets a burst *through*; leaky bucket *smooths* it. Different tools for different needs.
- **The distributed trap** — N servers × local counters = **N × your limit**, and it worsens silently as you autoscale.
- **The race** — both servers `GET` 99, both decide "room for one," both `SET` 100 → 101 served. Never check-then-act across a network.
- **The fix** — atomic `INCR` + `EXPIRE` for fixed windows; a **Lua script** for token bucket (Redis runs it as one indivisible unit).
- **Key on** user ID (best) > API key (B2B) > IP (necessary pre-auth, but **CGNAT means one IP = a whole office**). Layer them.
- **Respond with** `429`, `Retry-After`, and `X-RateLimit-Limit/Remaining/Reset` — send those headers on **successes too**, so clients can self-throttle. Clients should back off with **jitter**.
- **Enforce at** CDN (cheap, dumb) → gateway (sweet spot: knows the user) → service (rich context, most expensive). Use all three.
- **Soft limit** = degrade/throttle/serve-stale. **Hard limit** = 429. Ladder them: throttle at 80%, warn at 100%, reject at 150%.
- **Always decide:** limiter datastore down → **fail open** (keep serving) or **fail closed** (stay safe)? Never leave it unanswered.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [69 — API Design: REST, GraphQL, gRPC](./69-api-design-rest-graphql-grpc.md) — the APIs you are about to put a limiter in front of |
| **Next** | [73 — Circuit Breaker](./73-circuit-breaker.md) — rate limiting protects you from your *clients*; the circuit breaker protects you from your *dependencies* |
| **Related** | [58 — API Gateway](./58-api-gateway.md) — the single best place to enforce a limit, because it already knows who the caller is |
| **Related** | [95 — HLD: Rate Limiter](./95-hld-rate-limiter.md) — the full distributed design question, end to end |
| **Related** | [123 — LLD: Rate Limiter](./123-lld-rate-limiter.md) — the class-level design: Strategy pattern over the five algorithms |
| **Related** | [59 — Caching in Depth](./59-caching-in-depth.md) — Redis is the shared counter store here; the same latency and failure trade-offs apply |
| **Related** | [81 — Security Fundamentals](./81-security-fundamentals.md) — rate limiting is your first line of defence against credential stuffing and brute force |
