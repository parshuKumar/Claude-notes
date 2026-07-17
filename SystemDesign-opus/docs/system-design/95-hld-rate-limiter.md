# 95 — Design a Rate Limiter
## Category: HLD Case Study

---

## What is this?

A **rate limiter** is the bouncer at the door of your API. It counts how many requests each client has made in the recent past, and when a client goes over its allowance, it turns them away with a `429 Too Many Requests` instead of letting them through.

You already know the *algorithms* from [70 — Rate Limiting](./70-rate-limiting.md) — fixed window, sliding window, token bucket, leaky bucket. This document is a different job. Here we design rate limiting as a **distributed system**: something that sits in front of a fleet of 20+ servers, keeps a shared count, answers in under a millisecond, and doesn't take your whole API down when it breaks.

The hard part isn't counting. Counting is easy. The hard part is counting **correctly, across many machines, without adding latency, without a single point of failure.**

---

## Why does it matter?

Without a rate limiter:

- **One buggy client loop can DoS you.** A mobile app with a retry bug in a `while` loop will happily send 10,000 requests/second from a single phone and saturate your database connection pool.
- **You cannot monetize tiers.** "Free: 100 req/hour, Pro: 10,000 req/hour" is a rate limiter. Without one, your pricing page is fiction.
- **Scrapers eat your infrastructure budget.** Someone will discover your public search endpoint and mine your entire catalog.
- **Expensive endpoints get abused.** An endpoint that costs you $0.02 in LLM tokens per call needs a limiter more than a static JSON endpoint does.

**Interview angle:** "Design a rate limiter" is one of the most common warm-up HLD questions because it is small enough to finish in 45 minutes but has one genuinely hard question buried in it — *how do 20 servers agree on a count?* Interviewers use that to separate people who memorized algorithms from people who understand distributed systems.

**Real-work angle:** Every API gateway you'll ever configure (Kong, Envoy, AWS API Gateway, Cloudflare) has a rate limiter in it, and the day it misfires and blocks your biggest customer, someone has to understand exactly how it works.

---

## The core idea — explained simply

### The Nightclub Bouncer Analogy

A nightclub has a capacity limit. There's a bouncer at the door with a clicker.

**One door, one bouncer.** Easy. He clicks up on entry, down on exit, and when the clicker hits 300 he says "sorry, we're full." The count is perfect because there's exactly one clicker and exactly one person holding it.

**Now the club gets popular and opens five doors.** Each door gets its own bouncer with his own clicker. Each bouncer lets 300 people in. The club now has **1,500 people inside** and the fire marshal is on the phone. Nobody did anything wrong — each bouncer enforced the limit *he could see*. The limit was never global.

There are only three ways to fix this:

1. **One shared clicker.** All five bouncers walk to a central podium and update the same counter. Correct — but now every entry involves a walk across the lobby (a network round trip), and if the podium catches fire, nobody gets in at all.
2. **Assign each guest to a fixed door.** "If your last name starts with A–E, you must use door 1." Now each bouncer's local clicker is accurate for his own guests. Works — until a door closes and you have to reassign everyone mid-night.
3. **Give each bouncer 60 of the 300 and let them shout updates at each other every few seconds.** Fast, no lobby walk — but for a few seconds the count is stale, and you might let 310 people in. Usually fine. Sometimes not.

That's the entire case study. Everything below is those three options with more precision.

| Analogy | Technical concept |
|---------|------------------|
| The club | Your API / backend fleet |
| The bouncer | The rate-limiter component |
| The clicker | The counter for one client key |
| Five doors, five clickers | 20 API servers with local in-memory counters |
| The central podium | Redis (centralized counter store) |
| Walking to the podium | The network round trip on every request |
| The podium catching fire | Redis is down — do you fail open or fail closed? |
| "Names A–E use door 1" | Sticky sessions / consistent hashing |
| Shouting updates every few seconds | Async gossip / eventually-consistent local counters |
| "Sorry, we're full" | HTTP 429 + `Retry-After` |

---

## Key concepts inside this topic

This is the [93 — HLD Approach Framework](./93-hld-approach-framework.md) executed end to end: requirements → estimation → architecture → deep dives → failure modes.

### 1. Requirements

**Clarifying questions to ask the interviewer first** (never skip this — it's half the score):

- Are we limiting **our own users** (authenticated API) or **anonymous internet traffic** (public endpoint)? This changes the key: user ID vs IP.
- Is this a **client-side library**, a **server-side middleware**, or a **standalone service**?
- Do we need **exact** limits, or is "roughly 100/min, maybe 105 under load" acceptable?
- Do rules change often? Do we need to change a limit **without a deploy**?
- Are we limiting requests only, or also **cost** (bytes, tokens, CPU-seconds)?

**Functional requirements:**

| # | Requirement |
|---|-------------|
| F1 | Limit requests per client according to configurable rules |
| F2 | Return `429 Too Many Requests` with correct headers when a limit is exceeded |
| F3 | Rules must be changeable **without a redeploy** (config store, not code) |
| F4 | Support multiple granularities: per **user**, per **IP**, per **API key**, per **endpoint** — and combinations |
| F5 | Support different limits per tier (free / pro / enterprise) |

**Non-functional requirements** — this is where the design actually lives:

**NFR1 — Very low latency. This is the dominant constraint.**
The limiter sits in the path of **every single request**. If it adds 50ms, you have not "added a rate limiter" — you have made your entire API 50ms slower, for every user, forever. A p99 of **< 5ms** is the target; **< 1ms** for a same-datacenter Redis call is achievable. If your design needs two network hops to decide, it is the wrong design.

**NFR2 — Highly available, and: fail open or fail closed?**
This is the question interviewers love. The limiter is now a dependency of every request. When Redis times out, you have two choices:

- **Fail open** — allow the request through. You lose protection for the duration of the outage; an attacker who happens to be attacking right then gets a free pass.
- **Fail closed** — reject the request. You are now **causing a total outage of your own API** because a *supporting* component is unhealthy.

**Argue for fail-open as the default for most APIs.** The rate limiter is a *protective* layer, not a *correctness* layer. A Redis blip is far more likely than a coordinated attack timed to that blip. Turning a 30-second Redis failover into a 30-second full API outage is a spectacularly bad trade. Say this out loud in the interview: *"the limiter must never be more dangerous than the thing it protects against."*

**Fail closed is right when the limit protects something you cannot un-spend:**
- **Money** — an endpoint that charges a payment provider or burns LLM tokens per call.
- **A hard downstream quota** — you have a contract allowing 1,000 calls/sec to a third party and they cut you off (or bill you) above it.
- **Security-critical counters** — login attempts. Unlimited password guessing during a Redis outage is a real breach vector, not a theoretical one.

The mature answer is **per-rule**: each rule declares `onFailure: "allow" | "deny"`. Default `allow`; `deny` on the handful of rules where over-admitting is worse than an outage.

**NFR3 — Accurate enough.** Not perfectly accurate. If the limit is 100/min and a burst lets through 103, nobody is harmed. Chasing exactness costs latency (see §5).

**NFR4 — Works across a distributed fleet.** N servers must enforce one shared limit. This is the whole problem.

**NFR5 — Low memory.** The counter state must be small enough to hold the entire active working set in RAM (see §2).

---

### 2. Capacity estimation

Show the arithmetic. Interviewers grade the *method*, not the number.

**Traffic:**

```
DAU                       = 1,000,000 users
Avg requests / user / day = 100
Total requests / day      = 1,000,000 × 100 = 100,000,000 = 100M

Seconds in a day          = 86,400
Average QPS               = 100,000,000 / 86,400
                          ≈ 1,157  →  round to ~1,200 QPS

Peak factor               = ~4x (traffic clusters in waking hours)
Peak QPS                  ≈ 1,200 × 4 ≈ 5,000 QPS
```

**So the limiter must handle ~5,000 checks/second at peak.**
Context: a single Redis node does **~100,000 simple ops/sec** comfortably (often more). We are asking for **5% of one Redis node.** Redis is not the bottleneck at this scale — good, but say it anyway, because it tells the interviewer you know when *not* to shard.

**Memory:**

With token bucket, each key stores exactly two numbers:

```
tokens          : float   → 8 bytes
last_refill_ms  : int64   → 8 bytes
                            ------
raw state                 = 16 bytes
```

But Redis has overhead. A realistic accounting per key:

```
Key string  "rl:user:9f3a21c4:search"        ≈  30 bytes
Redis hash + key entry + expiry overhead     ≈  70–100 bytes
Value payload (2 fields)                     ≈  16 bytes
                                                ---------
Practical total per key                      ≈  ~120 bytes  (be generous: call it 150)
```

Now the number of live keys. A user isn't subject to one limit — they're subject to several (a global account limit, plus per-endpoint limits on the endpoints they actually touch). Assume **5 active keys per active user**:

```
Active users in any window     = 1,000,000
Keys per user                  = 5
Total live keys                = 5,000,000

Memory = 5,000,000 × 150 bytes
       = 750,000,000 bytes
       ≈ 750 MB
```

**750 MB.** That fits in RAM on a single commodity Redis instance (a 4 GB node is roomy) with headroom to spare. And it *shrinks*, because every key carries a TTL — an idle user's keys evaporate a couple of minutes after their last request, so the real steady-state number is well under this ceiling.

**Conclusion to state out loud:** *"At 1M DAU the entire rate-limit working set is under a gigabyte and under 10% of one Redis node's throughput. We do not need a Redis cluster for capacity — we would only add one for availability, or later, for the hot-key problem."* That is a senior-sounding sentence and it is true.

**Network cost:** 5,000 QPS × ~200 bytes per Redis request/response round trip ≈ **1 MB/s**. Negligible.

---

### 3. Where to put it — the architectural decision

Four options. Draw all four, pick one, defend it.

**(a) Client-side**

```
┌────────────────────┐         ┌─────────────┐
│  Client            │         │             │
│  ┌──────────────┐  │         │   Server    │
│  │ rate limiter │──┼────────▶│             │
│  └──────────────┘  │         │             │
└────────────────────┘         └─────────────┘
        ▲
        └── attacker just deletes this
```

**Useless for enforcement.** The client is code you do not control. An attacker opens devtools, or curls your endpoint directly, and your limiter is gone. Never rely on it.

**But it is genuinely good for UX:** a client-side limiter stops a well-behaved app from spamming your server on a laggy connection, and it lets you disable the "Send" button and show "try again in 12s" instead of firing a request you know will 429. **Defense, never enforcement.**

**(b) At the API Gateway / edge**

```
                        ┌───────────────────────────┐
┌────────┐              │      API GATEWAY          │        ┌────────────┐
│ Client │─────────────▶│  ┌─────────────────────┐  │───────▶│  Service A │
└────────┘              │  │   RATE LIMITER      │  │        └────────────┘
                        │  │  (auth already done)│  │        ┌────────────┐
     ◀─── 429 ──────────│  └──────────┬──────────┘  │───────▶│  Service B │
                        └─────────────┼─────────────┘        └────────────┘
                                      │
                                 ┌────▼────┐
                                 │  Redis  │
                                 └─────────┘
```

**This is the usual answer.** Bad traffic is rejected **before it consumes a single application resource** — no service thread, no DB connection, no cache lookup. The gateway already terminates TLS, already authenticates the caller (so it already knows the user ID and tier), and is already a required hop. Marginal cost: one Redis call. See [58 — API Gateway](./58-api-gateway.md).

**(c) Middleware inside each service**

```
┌────────┐   ┌─────────┐   ┌──────────────────────────┐
│ Client │──▶│ Gateway │──▶│ Service A                │
└────────┘   └─────────┘   │ ┌──────────┐             │
                           │ │ limiter  │──▶ handler  │
                           │ └──────────┘             │
                           └──────────────────────────┘
                           ┌──────────────────────────┐
                        ──▶│ Service B                │
                           │ ┌──────────┐             │
                           │ │ limiter  │──▶ handler  │  ← same logic, copied
                           │ └──────────┘             │
                           └──────────────────────────┘
```

Fine-grained — the service knows its own expensive endpoints and its own business context ("this tenant is on a trial"). But the logic is **duplicated in every service**, in every language, and the request has already crossed the gateway and burned a connection before being rejected. Use this **in addition to** the gateway for a few endpoints that need business-aware limits, not instead of it.

**(d) A standalone rate-limiter microservice**

```
┌────────┐   ┌───────────┐   gRPC    ┌──────────────────┐
│ Client │──▶│  Gateway  │──────────▶│ RateLimiter Svc  │
└────────┘   └─────┬─────┘◀──────────│  (rules cached)  │
                   │      allow/deny └────────┬─────────┘
                   ▼                          │
             ┌──────────┐               ┌─────▼─────┐
             │ Services │               │   Redis   │
             └──────────┘               └───────────┘
```

Centralized rules and one place to deploy limiter logic (this is roughly Envoy's `ratelimit` service model). But you've put **an extra network hop in the hot path of every request**, and you've created a new tier to keep alive. Worth it when many heterogeneous gateways/services need identical rules, or when limiting requires state the gateway can't hold.

**Recommendation: (b) — a limiter library embedded in the API Gateway, backed by Redis, with (a) as a UX nicety and (c) for a handful of business-aware endpoints.**

Reasoning, in one breath: the gateway is already in the path so it adds **zero extra hops**; it already knows the identity so the key is free; it rejects abuse **before any application resource is touched**; and Redis gives us the one shared counter the distributed problem demands. Option (d) buys centralization at the cost of the single thing we said matters most — latency.

---

### 4. Algorithm choice

Recap of the five from [70 — Rate Limiting](./70-rate-limiting.md), evaluated **as a distributed system**, which is a different lens than the algorithms doc used:

| Algorithm | State per key | Memory | Burst handling | Boundary bug | Distributed cost |
|-----------|---------------|--------|----------------|--------------|------------------|
| **Fixed window** | 1 counter | Tiny | Poor | **Yes** — 2× the limit can pass across the boundary | 1 `INCR` — cheapest possible |
| **Sliding window log** | Timestamp of *every* request | **Bad** — O(n) per key | Perfect | No | Big sorted-set ops; memory blows up under attack |
| **Sliding window counter** | 2 counters | Tiny | Good | Smoothed away | 2 reads + 1 write |
| **Token bucket** | **2 numbers** (tokens, last refill) | Tiny | **Excellent — allows bursts up to capacity** | No | 1 Lua call |
| **Leaky bucket** | Queue or 2 numbers | Small | **Rejects** bursts — enforces a smooth output rate | No | 1 Lua call |

**Commit: token bucket.**

Justify it on two axes:

1. **It matches how real clients behave.** A legitimate user opens your dashboard and it fires 12 API calls in 400ms, then goes quiet for two minutes. That is a *burst*, and it is *fine*. Fixed window and leaky bucket punish it. Token bucket **allows bursts up to the bucket capacity while still enforcing the long-run average rate** — exactly the semantics you actually want. "Burst to 20, sustain 5/sec" is a sentence you can put in your API docs and a customer will understand it.
2. **It's cheap and it's stateless-friendly.** The entire state is **two numbers**: `tokens` and `last_refill_ms`. Refill is computed lazily from elapsed time on read — you never run a background timer to top up millions of buckets. Two numbers is exactly what makes the atomic Lua script in §5 short enough to be fast.

Use **leaky bucket instead** when the thing you're protecting cannot absorb bursts at all — e.g. you're relaying to a legacy SOAP system that falls over above 50 concurrent requests. Then a smooth output rate is the *point*, and rejecting bursts is a feature.

---

### 5. The distributed problem — the heart of this case study

**The setup.** The limit is "100 requests per minute per user." You have **20 API servers** behind a load balancer, each with an in-memory counter. The load balancer round-robins.

```
                     ┌── Server 1  (local count: 100) ──┐
                     ├── Server 2  (local count: 100) ──┤
User ──▶ LB ─────────┼── Server 3  (local count: 100) ──┼──▶  Actual traffic
   (round robin)     ├── ...                            │     admitted: 2,000/min
                     └── Server 20 (local count: 100) ──┘     Limit advertised: 100/min
```

**Each server correctly enforced 100. The user got 2,000.** Your limit is silently 20× wrong, and it gets *wronger* every time you autoscale. This is the bug the whole case study exists to fix.

#### Option A — Centralized Redis counter (the standard answer)

Move the counter out of process memory into a shared store all 20 servers can see.

**But the naive implementation races.** Here is the bug, in the code people actually write:

```javascript
// ❌ BROKEN — a read-modify-write race across processes.
async function isAllowedNaive(redis, key, limit) {
  const count = Number(await redis.get(key)) || 0;   // ── (1) READ
  if (count >= limit) return false;
  await redis.set(key, count + 1, 'EX', 60);          // ── (2) WRITE
  return true;
}
```

Two of the user's requests land on server 3 and server 11 **at the same millisecond**, with the true count at 99 and the limit at 100:

```
  time   Server 3                       Server 11
  ────   ─────────────────────────      ─────────────────────────
   t0    GET key → 99
   t1                                   GET key → 99      ← reads the SAME stale 99
   t2    99 < 100 → ALLOW
   t3                                   99 < 100 → ALLOW  ← both allowed!
   t4    SET key 100
   t5                                   SET key 100       ← the increment is LOST
```

Two requests admitted where one was allowed, **and** the counter is now 100 instead of 101. Under real concurrency (a client with 50 parallel connections) this doesn't leak one extra request — it leaks dozens. The gap between the GET and the SET is a **network round trip**, which is an eternity.

**The fix: make check-and-decrement a single atomic operation.** Redis is single-threaded, so a Lua script runs to completion with nothing interleaved. One round trip, no race.

```lua
-- token_bucket.lua
-- KEYS[1]   = the rate-limit key,  e.g. "rl:user:9f3a21c4:search"
-- ARGV[1]   = capacity        (max tokens the bucket holds — the burst size)
-- ARGV[2]   = refill_rate     (tokens added per second — the sustained rate)
-- ARGV[3]   = now_ms          (caller's clock; ONE clock for all callers, so no skew)
-- ARGV[4]   = cost            (tokens this request consumes — usually 1)
-- Returns: { allowed(1|0), tokens_remaining, retry_after_ms }

local capacity    = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local now_ms      = tonumber(ARGV[3])
local cost        = tonumber(ARGV[4])

local bucket      = redis.call('HMGET', KEYS[1], 'tokens', 'ts')
local tokens      = tonumber(bucket[1])
local last_ts     = tonumber(bucket[2])

-- First time we've seen this key: the bucket starts full.
if tokens == nil then
  tokens  = capacity
  last_ts = now_ms
end

-- Lazy refill: add whatever the bucket earned while we weren't looking.
-- No background job tops up millions of buckets — we compute it on read.
local elapsed_ms = math.max(0, now_ms - last_ts)
tokens = math.min(capacity, tokens + (elapsed_ms / 1000.0) * refill_rate)

local allowed        = 0
local retry_after_ms = 0

if tokens >= cost then
  tokens  = tokens - cost      -- CHECK AND DECREMENT, atomically, in one place
  allowed = 1
else
  -- How long until we've earned the tokens this request needs?
  retry_after_ms = math.ceil(((cost - tokens) / refill_rate) * 1000)
end

redis.call('HSET', KEYS[1], 'tokens', tokens, 'ts', now_ms)

-- Expire idle buckets so memory self-cleans. A bucket refills fully in
-- capacity/refill_rate seconds; after that its state is indistinguishable
-- from "new", so keeping it is pure waste.
local ttl_s = math.ceil(capacity / refill_rate) + 10
redis.call('EXPIRE', KEYS[1], ttl_s)

return { allowed, math.floor(tokens), retry_after_ms }
```

**Why this is the right shape:**
- **One network round trip.** Not two. Latency is the top NFR.
- **Atomic.** Redis runs the script to completion; no other command interleaves. The race is gone by construction.
- **`now_ms` comes from the caller, not from Redis.** Deliberate: it keeps the script deterministic (older Redis required this for replication), and all callers hitting the same Redis are already time-synced by NTP to within a few ms. If you're paranoid, use `redis.call('TIME')` on Redis 5+ with effect replication — one clock for everyone, zero skew.
- **TTL means memory self-cleans.** No cron job sweeping dead keys.

**Costs, stated honestly:** one network round trip on every request (~0.2–1ms same-DC — acceptable), and Redis becomes a **SPOF / bottleneck**. Mitigate with Redis Sentinel or Cluster for failover, replicas, and the fail-open fallback in §8. Recall from [105 — HLD Distributed Cache](./105-hld-distributed-cache.md) that the same replication/sharding machinery applies here.

#### Option B — Sticky sessions

Route every request from user X to the same server (consistent hashing on user ID at the load balancer). Then the **local** in-memory counter is the *only* counter for that user, and it's exact — with **zero network calls**. Fastest possible option.

**Why it's usually wrong:**
- **It breaks on rebalancing.** Add a server, lose a server, deploy — and keys remap. The user's counter is now on a machine with no history: their count silently resets to zero. An attacker who can trigger reconnections can farm free quota.
- **It undermines load balancing.** You can no longer route by least-connections or by health. One whale customer hashes onto one server and hotspots it.
- **Counters are lost on restart.** Every deploy wipes the state.

Right when: you're already sticky for another reason (in-memory WebSocket sessions), the limits are advisory, and resets on deploy don't matter.

#### Option C — Local counters + async sync (gossip / approximate limiting)

Each server keeps a **local** bucket and periodically (every 1–5 seconds) publishes its consumption to peers or to a central aggregator, then adjusts its local allowance.

- **Latency: effectively zero.** The check is a hashmap lookup — nanoseconds. Nothing leaves the process.
- **Correctness: approximate.** Between sync ticks, servers don't know what the others admitted. A client can briefly exceed the limit — bounded roughly by `(number of servers) × (per-server allowance drift)` over one sync interval.

**Be honest about the trade-off, because this is the senior answer:** at very high scale (Cloudflare-, Stripe-, Google-scale), **most companies accept approximate limiting.** If the limit is 1,000/min and a client occasionally gets 1,050, *literally nothing bad happens* — you set the limit as a safety threshold, not a legal contract. Paying a network round trip on 10 million requests/second to turn 1,050 into exactly 1,000 is a terrible trade. **Exactness is not worth the latency unless the limit protects money or a hard external quota.**

The pragmatic hybrid everyone actually ships: **local bucket as the first line, Redis as the second.** Check the local bucket first; only when it's exhausted (or every Nth request) do you consult Redis. Cuts Redis traffic by an order of magnitude while keeping the global count roughly honest.

#### Option D — Shard the keys across a Redis cluster

Keys hash to slots; slots live on different nodes. `rl:user:abc` lands on node 2, `rl:user:xyz` on node 5.

Buys **horizontal capacity and blast-radius reduction** (one node down = a fraction of keys affected, not all). Doesn't fix the hot-key problem (§8) — a single hot key still lives on a single node.

At our 5,000 QPS estimate we **do not need this for throughput.** Add it when you actually outgrow one node, or for availability. Don't shard preemptively; say so in the interview.

#### Recommendation

| Situation | Choose |
|-----------|--------|
| **Default — most APIs, up to ~50k QPS** | **Centralized Redis + atomic Lua**, fail-open, local fallback |
| Hyperscale (>100k QPS), limits are safety thresholds | Local buckets + async sync (approximate) |
| Limits gate money / a hard external quota | Centralized Redis, **fail closed**, exactness required |
| Already sticky for another reason, limits advisory | Sticky sessions + local counters |
| Outgrew one Redis node | Shard by key hash (Redis Cluster) |

---

### 6. The rules engine

Limits must change **without a redeploy** (F3). So rules are **data**, not code.

**Storage:** rules live in a config store (PostgreSQL, DynamoDB, or a config service like Consul/etcd). Each limiter instance **caches all rules in memory** — this is critical, because looking a rule up over the network on every request would blow the latency budget just as badly as a bad counter design.

**Cache invalidation** (two mechanisms, belt and braces):
1. **Poll** the config store every 10–30 seconds. Simple, self-healing, and a rule change propagating in 30s is *fine* for something a human edits.
2. **Pub/sub push** on the Redis channel `ratelimit:rules:changed` for near-instant propagation when someone needs to raise a limit *right now* during an incident.

**Rule schema:**

```javascript
// A rule is pure data — editable in an admin UI, no deploy required.
const rule = {
  id:        'rule_free_search',
  domain:    'search-api',       // which service/API this applies to
  keyType:   'user',             // 'user' | 'ip' | 'api_key' | 'endpoint' | 'global'
  match: {                       // when does this rule apply?
    endpoint: 'GET /v1/search',
    tier:     'free'
  },
  limit:     100,                // capacity — the burst size
  window:    '1m',               // together with limit → refillRate = 100/60 = 1.67 tokens/sec
  cost:      1,                  // some endpoints cost more than one token
  action:    'reject',           // 'reject' (429) | 'throttle' (queue+delay) | 'log_only' (shadow mode)
  onFailure: 'allow',            // Redis down → fail open for this rule
  priority:  10
};
```

**`action: 'log_only'` is the most underrated field here.** Ship every new rule in shadow mode first: evaluate it, emit a metric for what *would* have been blocked, block nothing. Look at the dashboard for a day. *Then* flip it to `reject`. This is how you avoid the incident in §9.

**Hierarchy — the most restrictive wins.** A single request usually matches several rules at once. A free-tier user calling `/v1/search` may be subject to:

```
  ┌─────────────────────────────────────────────────────┐
  │  Request: GET /v1/search   user=9f3a  tier=free      │
  └───────────────────────┬─────────────────────────────┘
                          │  matches 3 rules
        ┌─────────────────┼──────────────────┐
        ▼                 ▼                  ▼
  ┌───────────┐    ┌────────────┐     ┌────────────┐
  │ per-IP    │    │ per-user   │     │ per-user   │
  │ 1000/min  │    │ ACCOUNT    │     │ ENDPOINT   │
  │ (abuse)   │    │ 5000/hour  │     │ 100/min    │  ← most restrictive
  └───────────┘    └────────────┘     └────────────┘
        │                 │                  │
        └─────────────────┴──────────────────┘
                          ▼
              ALL must pass → request allowed
        The 429 reports the FIRST rule that rejected
```

**Every matching rule is evaluated; the request passes only if all of them pass.** That means one Lua call per matching rule — so pipeline them into a single round trip (`redis.pipeline()`), or better, pass all keys to one script. Don't turn "3 rules" into "3 network hops."

**Sharp edge:** if rule A allows and rule B rejects, you have already *spent a token on rule A*. Under strict semantics you'd refund it. In practice, almost nobody does — the drift is tiny and the complexity of a compensating transaction is not. Know that it's there; mention it if the interviewer asks; move on.

---

### 7. The response contract

When you reject, **tell the client exactly what happened and when to come back.** This is not politeness — it is load shedding. A client that knows to wait 12 seconds *waits 12 seconds*. A client that gets a bare `429` **retries immediately, in a loop**, and now you're serving 10× the rejected traffic you were before. **Bad error responses make an overload worse.**

```http
HTTP/1.1 429 Too Many Requests
Content-Type: application/json
Retry-After: 12
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1752345678

{
  "error": "rate_limit_exceeded",
  "message": "Rate limit of 100 requests per minute exceeded for this endpoint.",
  "retry_after_seconds": 12,
  "docs": "https://api.example.com/docs/rate-limits"
}
```

| Header | Meaning | Why it matters |
|--------|---------|----------------|
| `Retry-After: 12` | Wait 12 seconds (or an HTTP date) | The single most important one. Standard, machine-readable, and every decent HTTP client library honors it automatically. |
| `X-RateLimit-Limit: 100` | Your ceiling in this window | Lets the client self-tune concurrency |
| `X-RateLimit-Remaining: 0` | Tokens left | Send on **successful** responses too — a good client sees `Remaining: 3` and slows down *before* hitting the wall |
| `X-RateLimit-Reset: 1752...` | Unix timestamp when the bucket is full again | Lets a client schedule work |

**Send `X-RateLimit-*` on 200s, not just 429s.** That's what turns a client from reactive (hit the wall, back off) to proactive (see the wall coming, glide). Stripe, GitHub, and Twitter all do this, and it's why their clients are well-behaved.

**Add jitter guidance.** If 5,000 clients all get `Retry-After: 12`, they will all retry at exactly t+12 — a **thundering herd** that recreates the overload on a 12-second cycle. Document "retry after `Retry-After` **plus a random 0–20% jitter**", and implement it in your own SDKs.

---

### 8. Deep dives / edge cases

**a) Race conditions under concurrency from one user.**
Covered in §5 — the naive GET/SET loses increments. **The fix is atomicity, not locking.** Never reach for a distributed lock (Redlock) here: a lock costs *more* round trips than the thing it's protecting, and a lock is a liveness hazard in the hot path of every request. The single-round-trip Lua script *is* the mutual exclusion, for free, because Redis is single-threaded.

**b) Redis latency and timeouts — the local fallback limiter.**
Set an **aggressive client timeout: 5–10ms, not the driver's default of several seconds.** If Redis can't answer in 10ms it is unhealthy, and blocking a user request on it for 3 seconds is strictly worse than guessing.

On timeout: **fail open, but not blindly.** Fall back to a **per-process in-memory limiter** with a *generous* limit (say 3× the real one). Reasoning: with Redis down you can't do global counting, but a single process can still stop one client from sending 10,000 req/sec from one connection. You degrade from "exact global limit" to "loose local limit" — not to "no limit at all." Wrap the Redis call in a **circuit breaker** so that once it trips, you skip Redis entirely for ~5 seconds instead of paying a 10ms timeout on every request (which would add 10ms to your p50 — see NFR1).

**c) The hot-key problem.**
One enormous customer — a partner sending 30% of all your traffic — has **one key**, `rl:api_key:bigcorp`. That key hashes to **one Redis slot on one node**. Sharding doesn't help; every one of their requests hammers the same node, and that node is now your bottleneck while the other five sit idle.

Fixes, in order of preference:
1. **Key splitting (sub-bucketing).** Split the logical key into N physical keys — `rl:api_key:bigcorp:{0..9}` — each granted `limit/10`. Requests hash into a random or server-pinned shard. The load spreads across 10 slots. Cost: accuracy at the edges (a client unlucky enough to hash into an exhausted shard gets rejected while another shard has tokens). Usually a fine trade.
2. **Local-first counting** for known-hot keys: give each server a slice of the big customer's quota to spend locally, sync in the background. This is §5's Option C applied surgically to exactly the keys that need it.
3. **A dedicated Redis node** for the whale's keys. Crude, but sometimes the right call for a customer big enough to have an account manager.

**d) Per-tenant vs global limits.**
Two different jobs, and you need both:
- **Per-tenant** limits enforce fairness and monetize tiers. They stop *one* customer from hogging.
- A **global** limit protects *the system itself*. If your DB tips over at 20,000 QPS, you want a global ceiling around 15,000 QPS regardless of how many well-behaved tenants show up. Without it, 200 customers each perfectly within their 100/min limit can still collectively melt your database. **Per-tenant limits do not protect the system; only a global limit does.**

**e) Legitimate burst vs attack.**
Both look like "lots of requests, fast." Signals that separate them:

| Signal | Legitimate burst | Attack / abuse |
|--------|------------------|----------------|
| **Shape** | Spike, then quiet (page load, batch job at midnight) | Sustained flat line for hours |
| **Endpoints** | Varied — the natural mix of a real workflow | One endpoint, hammered (`/login`, `/search`) |
| **Response codes** | Mostly 200s | Lots of 401/404 (credential stuffing, enumeration) |
| **Client** | Real User-Agent, honors `Retry-After`, backs off | Ignores `Retry-After`, retries instantly, rotating IPs |
| **Payload** | Varied inputs | Identical or sequentially-enumerated inputs |

The design response is **layered, not binary**: let token bucket's *capacity* absorb the legitimate burst (that's what capacity is *for*), and enforce the sustained abuse case with the *refill rate*. Then add a **separate, slower "abuse" rule** on top — e.g. "if this IP 429s more than 100 times in 10 minutes, block it at the edge/WAF for an hour." Rate limiting handles the *shape* of traffic; ban lists and WAFs handle *intent*.

---

### 9. Failure modes and monitoring

| Failure mode | Blast radius | Mitigation |
|--------------|--------------|------------|
| **Redis down** | Every request loses its limit check | Fail open + local fallback limiter + circuit breaker (§8b) |
| **Redis slow (not down)** | **Worse than down** — every request pays the timeout, your p50 explodes | Aggressive 10ms timeout; circuit breaker so you stop *trying* |
| **Rule misconfigured** (limit set to 10 instead of 10,000) | Your biggest customer is fully down and it looks like *their* bug | Shadow mode (`log_only`) before enforcing; require review on rule changes; instant rollback |
| **Config store down** | Rules can't refresh | Serve the **last-known-good cached rules** from memory indefinitely. Never fail requests because you couldn't *read a config*. |
| **Clock skew across gateways** | Buckets refill slightly wrong | NTP everywhere; or take time from Redis (`TIME`) so there is exactly one clock |
| **Hot key** | One Redis node saturates | Key splitting (§8c) |
| **Thundering herd on retry** | Rejected clients all return at once | `Retry-After` + mandatory jitter in SDKs |

**What to alert on — and the one alert that's actually subtle:**

```
  429 rate spikes  ───┬──▶ Is it ONE key, or MANY keys?
                      │
      ONE key ────────┴──▶ ONE key, huge volume, one endpoint
                            → probably ABUSE. Investigate, consider a ban.
                      │
      MANY keys ──────┴──▶ MANY distinct legitimate customers,
                            starting right after a config change
                            → you BROKE something. ROLL BACK THE RULE. Now.
```

**A spike in 429s is not self-explanatory, and this is the trap.** It means either *"we are successfully repelling an attack"* (great, the system is working) or *"we just deployed a rule that is breaking every paying customer"* (a Sev-1). The metric looks **identical** on a naive dashboard, and teams have absolutely sat watching a rising 429 graph feeling *good* about it while their customers were down.

**So instrument to disambiguate.** Every 429 emits a structured event with `{ ruleId, keyType, keyHash, endpoint, tier, timestamp }`, which lets you ask the only question that matters: **is this concentrated in a few keys, or spread across many?**

- **Concentrated** (a handful of keys, huge volume, one endpoint) → abuse. The system is doing its job.
- **Distributed** (thousands of distinct keys, many of them long-standing paying customers, all starting within a minute of each other — and *especially* if that minute lines up with a config change) → **you did this.** Roll back the rule.

**The alerts to actually configure:**
- `rate(429) by ruleId` — **alert on any rule whose 429 rate jumps >10× in 5 minutes.** Ties the spike to the rule that caused it.
- **`count(distinct keys 429ing)`** — the disambiguating metric. A jump *here* means a config break, not an attack.
- `p99 of the limiter's own added latency` — alert above 10ms. The limiter must never be the slow thing.
- **`rate(fail_open_events)`** — the silent one. This means Redis is unhealthy and **you are currently unprotected**, but every request is succeeding, so nothing else looks wrong. **Page on this.** Nobody notices a rate limiter that has quietly stopped limiting until the bill arrives.
- `redis_memory_used` vs your 750 MB estimate — a big divergence means a key-cardinality bug (usually: someone put a raw URL with query params in the key, so every request makes a *new* key).

---

## Visual / Diagram description

### The full architecture

```
   ┌──────────┐   ┌──────────┐   ┌──────────┐
   │ Web App  │   │ Mobile   │   │ Partner  │
   │ (client- │   │ App      │   │ API user │
   │  side UX │   │          │   │          │
   │  limiter)│   │          │   │          │
   └────┬─────┘   └────┬─────┘   └────┬─────┘
        │              │              │
        └──────────────┼──────────────┘
                       │  HTTPS
                       ▼
              ┌─────────────────┐
              │  Load Balancer  │
              └────────┬────────┘
                       │
   ┌───────────────────▼───────────────────────────────────────┐
   │                  API GATEWAY  (×N instances)              │
   │                                                           │
   │   1. TLS terminate                                        │
   │   2. Authenticate  → we now know userId, apiKey, tier     │
   │   3. ┌───────────────────────────────────────────┐        │
   │      │        RATE LIMITER MIDDLEWARE            │        │
   │      │                                           │        │
   │      │  ┌─────────────────┐   ┌───────────────┐  │        │
   │      │  │  RULES CACHE    │   │ LOCAL FALLBACK│  │        │
   │      │  │  (in-memory,    │   │ LIMITER       │  │        │
   │      │  │   refreshed via │   │ (used only if │  │        │
   │      │  │   poll+pubsub)  │   │  Redis fails) │  │        │
   │      │  └────────┬────────┘   └───────────────┘  │        │
   │      │           │ match rules → build keys      │        │
   │      │           ▼                               │        │
   │      │   EVALSHA token_bucket.lua  (1 round trip)│        │
   │      └───────────┬───────────────────┬───────────┘        │
   │                  │                   │                    │
   │            allowed│             rejected│                  │
   │                  │                   │                    │
   └──────────────────┼───────────────────┼────────────────────┘
                      │                   │
                      │                   ▼
                      │            ┌──────────────────────┐
                      │            │ 429 Too Many Requests│
                      │            │ Retry-After: 12      │
                      │            │ X-RateLimit-*        │
                      │            └──────────────────────┘
                      │
        ┌─────────────┼──────────────┐
        ▼             ▼              ▼
   ┌─────────┐  ┌──────────┐   ┌──────────┐
   │ User    │  │ Search   │   │ Payments │
   │ Service │  │ Service  │   │ Service  │      ← never even sees rejected traffic
   └─────────┘  └──────────┘   └──────────┘

           ┌───────────────────────────────────────────┐
           │            SUPPORTING STORES              │
           │                                           │
           │   ┌──────────────┐    ┌───────────────┐   │
           │   │    REDIS     │    │  CONFIG STORE │   │
           │   │  (counters)  │    │  (rules, in   │   │
           │   │  + replica   │    │   Postgres)   │   │
           │   │  ~750 MB     │    │               │   │
           │   └──────▲───────┘    └───────┬───────┘   │
           │          │                    │           │
           │      EVALSHA            poll every 30s    │
           │   (from gateways)      + pubsub invalidate│
           └───────────────────────────────────────────┘
```

**What the diagram shows.** Rate limiting happens **inside the gateway, after auth** (so we know *who* the caller is) and **before routing** (so rejected traffic never touches a service). Redis holds the counters and is the *only* shared state. The rules cache means the config store is **not** in the hot path — a request never waits on Postgres. The local fallback limiter is the safety net for when Redis is unreachable. Trace one allowed request left-to-right, and one rejected request down the 429 branch — that's the whole system.

### The request path, in time

```
  t=0.0ms   Request arrives at gateway
  t=0.1ms   Auth resolves → userId=9f3a, tier=free
  t=0.1ms   Rules cache lookup (in-memory hashmap)   → 2 matching rules
  t=0.2ms   EVALSHA token_bucket.lua × 2, pipelined  ─┐
                                                       │  ~0.5ms Redis RTT
  t=0.7ms   Redis replies: [1, 42, 0], [1, 380, 0]   ─┘
  t=0.7ms   Both allowed → forward to Search Service
            Response gets X-RateLimit-Remaining: 42

  TOTAL LIMITER OVERHEAD: ~0.7ms   ← this is the number that matters (NFR1)
```

---

## The Node.js implementation

Express middleware for the gateway. This is the real thing: atomic Lua, rule matching, headers, circuit breaker, local fallback.

```javascript
// rate-limiter.js
import Redis from 'ioredis';
import { readFileSync } from 'node:fs';

const TOKEN_BUCKET_LUA = readFileSync('./token_bucket.lua', 'utf8');

/**
 * A tiny in-memory token bucket. Used ONLY as the fallback when Redis is
 * unreachable. It is per-process, so with 20 gateways the effective limit is
 * 20x — which is exactly why we give it a generous multiplier and treat it as
 * "loose protection", not "the limit". Loose protection >> no protection.
 */
class LocalBucket {
  constructor() { this.buckets = new Map(); }

  consume(key, capacity, refillRate, cost = 1) {
    const now = Date.now();
    let b = this.buckets.get(key);
    if (!b) { b = { tokens: capacity, ts: now }; this.buckets.set(key, b); }

    b.tokens = Math.min(capacity, b.tokens + ((now - b.ts) / 1000) * refillRate);
    b.ts = now;

    if (b.tokens >= cost) { b.tokens -= cost; return { allowed: true, remaining: Math.floor(b.tokens) }; }
    return { allowed: false, remaining: 0, retryAfterMs: Math.ceil(((cost - b.tokens) / refillRate) * 1000) };
  }
}

/**
 * Opens after `threshold` consecutive Redis failures. While open we skip Redis
 * ENTIRELY rather than paying the timeout on every request — a 10ms timeout on
 * every request would add 10ms to our p50, which violates our top NFR just as
 * badly as a slow limiter would.
 */
class CircuitBreaker {
  constructor({ threshold = 5, resetMs = 5000 } = {}) {
    this.threshold = threshold; this.resetMs = resetMs;
    this.failures = 0; this.openedAt = null;
  }
  get isOpen() {
    if (this.openedAt === null) return false;
    if (Date.now() - this.openedAt > this.resetMs) { this.openedAt = null; this.failures = 0; return false; }
    return true;
  }
  recordSuccess() { this.failures = 0; this.openedAt = null; }
  recordFailure() { if (++this.failures >= this.threshold) this.openedAt = Date.now(); }
}

export class RateLimiter {
  constructor({ redisUrl, ruleStore, metrics }) {
    this.redis = new Redis(redisUrl, {
      // Aggressive. If Redis can't answer in 10ms it is unhealthy, and making a
      // user wait 3s (the driver default) is strictly worse than guessing.
      commandTimeout: 10,
      maxRetriesPerRequest: 1,
      enableOfflineQueue: false, // fail fast instead of silently queueing forever
    });
    this.redis.defineCommand('tokenBucket', { numberOfKeys: 1, lua: TOKEN_BUCKET_LUA });

    this.ruleStore = ruleStore;
    this.metrics   = metrics;
    this.rules     = [];
    this.breaker   = new CircuitBreaker();
    this.local     = new LocalBucket();
  }

  /** Load rules into memory and keep them fresh. The config store must NEVER be
   *  in the hot path — a request may not wait on Postgres. */
  async start() {
    await this.refreshRules();
    this.timer = setInterval(() => this.refreshRules().catch(() => {}), 30_000);

    // Push-based invalidation for "raise this limit RIGHT NOW" during an incident.
    this.sub = this.redis.duplicate();
    await this.sub.subscribe('ratelimit:rules:changed');
    this.sub.on('message', () => this.refreshRules().catch(() => {}));
  }

  async refreshRules() {
    try {
      this.rules = await this.ruleStore.loadAll();
    } catch (err) {
      // Keep serving the last-known-good rules. Never fail requests because we
      // couldn't READ A CONFIG FILE.
      this.metrics?.increment('ratelimit.rules.refresh_failed');
    }
  }

  /** Which rules apply to this request? A request commonly matches several. */
  matchRules(ctx) {
    return this.rules.filter(r =>
      (!r.match.endpoint || r.match.endpoint === `${ctx.method} ${ctx.route}`) &&
      (!r.match.tier     || r.match.tier     === ctx.tier)
    );
  }

  buildKey(rule, ctx) {
    const idPart = {
      user:     ctx.userId,
      ip:       ctx.ip,
      api_key:  ctx.apiKey,
      endpoint: `${ctx.method}:${ctx.route}`,
      global:   'all',
    }[rule.keyType];
    // NOTE: the key is built from the ROUTE PATTERN (/v1/users/:id), never the raw
    // URL. Putting a raw URL here would mint a new Redis key per request and blow
    // your key cardinality — the classic memory-leak bug in a rate limiter.
    return `rl:${rule.id}:${idPart}`;
  }

  /** Evaluate every matching rule. ALL must pass. Most restrictive wins by construction. */
  async check(ctx) {
    const rules = this.matchRules(ctx);
    if (rules.length === 0) return { allowed: true, headers: {} };

    const windowSec  = parseWindow(rules[0].window);
    const decisions  = this.breaker.isOpen
      ? this.checkLocal(rules, ctx)
      : await this.checkRedis(rules, ctx);

    const rejected = decisions.find(d => !d.allowed);
    const tightest = decisions.reduce((a, b) => (a.remaining <= b.remaining ? a : b));

    const headers = {
      'X-RateLimit-Limit':     String(tightest.limit),
      'X-RateLimit-Remaining': String(tightest.remaining),
      // Sent on 200s too — a client seeing "3 left" can slow down BEFORE it hits
      // the wall. This is what turns a reactive client into a proactive one.
      'X-RateLimit-Reset':     String(Math.ceil(Date.now() / 1000) + windowSec),
    };

    if (rejected) {
      const retryAfter = Math.ceil(rejected.retryAfterMs / 1000) || 1;
      headers['Retry-After'] = String(retryAfter);
      this.metrics?.increment('ratelimit.rejected', {
        // These tags are what let you answer the ONLY question that matters during
        // a 429 spike: is this ONE key (abuse) or MANY keys (we broke a rule)?
        ruleId: rejected.ruleId, keyType: rejected.keyType, endpoint: ctx.route,
      });
      return { allowed: false, retryAfter, ruleId: rejected.ruleId, headers };
    }
    return { allowed: true, headers };
  }

  async checkRedis(rules, ctx) {
    const now = Date.now();
    try {
      // Pipeline: N rules, ONE network round trip. Never turn "3 rules" into "3 hops".
      const pipeline = this.redis.pipeline();
      for (const r of rules) {
        const refillRate = r.limit / parseWindow(r.window); // tokens per second
        pipeline.tokenBucket(this.buildKey(r, ctx), r.limit, refillRate, now, r.cost ?? 1);
      }
      const results = await pipeline.exec();
      this.breaker.recordSuccess();

      return results.map(([err, res], i) => {
        if (err) throw err;
        const [allowed, remaining, retryAfterMs] = res;
        return {
          allowed: allowed === 1, remaining, retryAfterMs,
          limit: rules[i].limit, ruleId: rules[i].id, keyType: rules[i].keyType,
        };
      });
    } catch (err) {
      this.breaker.recordFailure();
      // The silent failure. Every request still SUCCEEDS, so nothing looks broken —
      // but we are currently UNPROTECTED. Page on this metric.
      this.metrics?.increment('ratelimit.fail_open');

      // Per-rule policy: money/security rules fail CLOSED, everything else fails OPEN.
      const mustDeny = rules.find(r => r.onFailure === 'deny');
      if (mustDeny) {
        return [{ allowed: false, remaining: 0, retryAfterMs: 1000,
                  limit: mustDeny.limit, ruleId: mustDeny.id, keyType: mustDeny.keyType }];
      }
      return this.checkLocal(rules, ctx);
    }
  }

  /** Degrade to "loose per-process limit", NOT to "no limit at all". */
  checkLocal(rules, ctx) {
    const SLACK = 3; // generous: we can't count globally, so don't pretend we can
    return rules.map(r => {
      const cap  = r.limit * SLACK;
      const rate = cap / parseWindow(r.window);
      const res  = this.local.consume(this.buildKey(r, ctx), cap, rate, r.cost ?? 1);
      return { ...res, retryAfterMs: res.retryAfterMs ?? 0,
               limit: r.limit, ruleId: r.id, keyType: r.keyType };
    });
  }
}

function parseWindow(w) {
  const [, n, unit] = /^(\d+)([smhd])$/.exec(w);
  return Number(n) * { s: 1, m: 60, h: 3600, d: 86400 }[unit];
}

/** Express middleware. Mount it in the gateway AFTER auth, BEFORE routing. */
export function rateLimitMiddleware(limiter) {
  return async (req, res, next) => {
    const ctx = {
      userId: req.user?.id,
      apiKey: req.headers['x-api-key'],
      // Behind a proxy, req.ip is the PROXY's IP unless you set `trust proxy`.
      // Get this wrong and every user shares one key — a self-inflicted outage.
      ip:     req.ip,
      tier:   req.user?.tier ?? 'anonymous',
      method: req.method,
      route:  req.route?.path ?? req.path, // pattern, not raw URL — see buildKey()
    };

    const decision = await limiter.check(ctx);
    res.set(decision.headers);

    if (decision.allowed) return next();

    return res.status(429).json({
      error:   'rate_limit_exceeded',
      message: `Rate limit exceeded. Retry in ${decision.retryAfter}s.`,
      retry_after_seconds: decision.retryAfter,
      docs:    'https://api.example.com/docs/rate-limits',
    });
  };
}
```

---

## Real world examples

### 1. Stripe

Stripe publicly documents a layered approach: a **request-rate limiter** (requests per second per account) and a **concurrency limiter** (how many requests may be *in flight* at once) — because a small number of very slow requests can exhaust your capacity even at a modest request rate. They also distinguish **load shedders** from limiters: a limiter enforces a customer's quota; a load shedder protects the *system*, reserving capacity for critical traffic (charges) over less critical traffic (analytics reads) when things get hot. That's §8d's per-tenant-vs-global distinction, shipped.

### 2. GitHub

GitHub's REST API is the textbook example of the **response contract** in §7. Every response — success or failure — carries `x-ratelimit-limit`, `x-ratelimit-remaining`, `x-ratelimit-used`, and `x-ratelimit-reset`. Limits are **tier-based** (60/hour unauthenticated, 5,000/hour authenticated, higher for apps), and there's a separate, stricter secondary limit for expensive operations like content creation. Because the headers are so good, essentially every GitHub client library self-throttles — which is the whole point.

### 3. Cloudflare

Cloudflare runs rate limiting **at the edge**, across hundreds of PoPs — the extreme version of §5's Option C. Perfectly synchronizing a counter across globally distributed datacenters would mean cross-ocean round trips (100ms+), which would obliterate the latency budget for every request on the internet. So they don't. They use approximate, locally-counted, asynchronously-reconciled limiting — accepting a small overshoot in exchange for a limiter that costs microseconds. This is the "exactness isn't worth the latency" trade taken to its logical conclusion, by the people with the most reason to care.

---

## Trade-offs

**Where to put the limiter:**

| Placement | Pros | Cons |
|-----------|------|------|
| Client-side | Zero server cost; best UX (disable the button) | **Trivially bypassed. Never enforcement.** |
| **API Gateway (recommended)** | No extra hop; rejects before any app resource is used; identity already known | Gateway becomes a critical dependency |
| Service middleware | Business-aware, fine-grained | Duplicated in every service/language; traffic already crossed the gateway |
| Separate microservice | Centralized rules and logic | **Extra network hop in the hot path** — violates the top NFR |

**How to count across the fleet:**

| Approach | Latency added | Accuracy | Failure behavior |
|----------|--------------|----------|------------------|
| Local counters only | ~0 | **Broken** (N× the limit) | Fine, but wrong |
| **Centralized Redis + Lua** | ~0.5–1ms | Exact | Redis is a SPOF → fail open + fallback |
| Sticky sessions | ~0 | Exact *until rebalance* | Counters reset on deploy |
| Local + async sync | ~0 | Approximate (small overshoot) | Excellent |
| Redis Cluster (sharded) | ~0.5–1ms | Exact | Reduced blast radius; hot key still hurts |

**Fail open vs fail closed:**

| | Fail open | Fail closed |
|---|-----------|-------------|
| **When Redis dies** | Traffic flows, unprotected | Your API is down |
| **Risk** | Abuse during the outage window | **You cause the outage yourself** |
| **Right for** | Almost every API | Money, hard external quotas, login attempts |

**Rule of thumb:** **Put the limiter in the gateway. Use token bucket. Keep the count in Redis behind an atomic Lua script. Fail open with a local fallback — except where over-admitting costs real money, and then fail closed.** And if you ever get big enough that a Redis round trip per request is too expensive, that is the day you switch to approximate local counting — not before.

---

## Common interview questions on this topic

### Q1: "You have 20 servers each enforcing a 100 req/min limit locally. What's the actual limit a user experiences?"
**Hint:** 2,000 req/min — 20× what you advertised, and it gets worse every time you autoscale. This is *the* question of the whole problem. The fix is shared state (Redis) or accepted approximation (local + gossip), and the answer they want is that you *name the trade-off*, not that you pick one.

### Q2: "Two concurrent requests from the same user hit two different servers at the same instant. Walk me through what goes wrong with a naive GET-then-SET, and fix it."
**Hint:** Both `GET` the same stale count, both decide "allowed", both `SET` — one increment is lost and one extra request slipped through. Under 50 parallel connections it's not one extra, it's dozens. The fix is **atomicity, not locking**: one Lua script doing check-and-decrement in a single round trip. Redis is single-threaded, so the script *is* the mutex — for free. Reaching for Redlock here is the wrong answer and interviewers know it.

### Q3: "Your Redis is down. Do you allow all traffic or block all traffic?"
**Hint:** **Fail open, for most APIs** — a protective layer must never be more dangerous than the thing it protects against, and turning a 30-second Redis failover into a full API outage is a terrible trade. But degrade to a *local* fallback limiter rather than to nothing. Then name the exceptions: **payments, endpoints with a per-call cost, hard third-party quotas, and login attempts** all fail *closed*, because over-admitting there is unrecoverable. The best answer is per-rule: `onFailure: 'allow' | 'deny'`.

### Q4: "Which algorithm and why?"
**Hint:** **Token bucket.** Two reasons, both concrete: it **allows legitimate bursts up to the bucket capacity while still enforcing the sustained average rate** (real clients burst — a dashboard fires 12 calls on load and then goes quiet), and its state is **just two numbers** (tokens, last-refill timestamp), which keeps the atomic script short and the memory tiny. Choose leaky bucket instead only when the downstream literally cannot absorb a burst.

### Q5: "You see a 10× spike in 429s. What do you do?"
**Hint:** **Do not assume it's an attack.** Check whether the 429s are concentrated in a *few keys* (abuse — the system is working) or spread across *many distinct customers* (you broke something — a bad rule, roll it back). If it started right after a config change, it's you. This is exactly why every 429 must emit `{ruleId, keyType, endpoint}` and why "count of distinct keys being 429'd" is a first-class metric.

### Q6: "One customer sends 30% of your traffic. What breaks?"
**Hint:** The **hot key**. Their single counter key hashes to one Redis slot on one node, so sharding doesn't help — that one node saturates while the rest idle. Fix by **key splitting**: shard the logical key into `key:{0..9}`, each with `limit/10`, and spread the load across ten slots. You trade a little edge-case accuracy for a lot of headroom.

---

## Practice exercise

### Design and defend the limiter for a public LLM API

You are designing rate limiting for an API where **each call costs you real money** (LLM tokens). Free tier: 20 requests/minute. Pro: 1,000/minute. There are 30 gateway instances.

Produce, on paper:

1. **The estimation.** 200,000 DAU, 40 requests/user/day. Compute average QPS, peak QPS (assume 4×), and the Redis memory needed if each user has 3 active keys. Show every step.
2. **The architecture diagram.** Client → LB → Gateway (limiter) → Redis → services. Label where the rules cache lives and where the fallback limiter lives.
3. **The fail-open/fail-closed decision — and defend it.** This is the interesting part: the calls cost money, which argues for fail-closed. But a total outage costs revenue and trust, which argues for fail-open. **Pick one and write three sentences justifying it.** Then design the hybrid: which specific rules get `onFailure: 'deny'` and which get `'allow'`?
4. **The hot-key plan.** Your biggest Pro customer sends 25% of all traffic. Write the key-splitting scheme and state exactly what accuracy you give up.
5. **Two alerts**, with thresholds, that would let you tell "we're under attack" apart from "we shipped a broken rule" within 60 seconds.

**Deliverable:** one page of diagrams + numbers, plus the `token_bucket.lua` call you'd make (with the actual argument values for the Pro tier: what are `capacity` and `refill_rate` for 1,000/minute?).

---

## Quick reference cheat sheet

- **The whole problem in one line:** N servers with local counters means your "100/min" limit is really `N × 100/min`. Fix it with shared state or accept approximation — but *name the choice*.
- **Latency is the top NFR.** The limiter is in the path of every request. Adding 50ms doesn't add a limiter — it makes your entire API 50ms slower. Budget **< 5ms p99**.
- **Put it in the API Gateway.** After auth (identity is free), before routing (rejected traffic touches zero app resources), no extra hop.
- **Never rely on client-side limiting for enforcement.** It's a UX feature. Attackers use curl.
- **Token bucket wins:** allows legitimate bursts up to capacity, enforces the average rate, and costs **two numbers per key**.
- **GET-then-SET races.** Two servers read the same stale count and both allow. Fix with **one atomic Lua script** doing check-and-decrement in a single round trip. **Not a distributed lock.**
- **Fail open by default.** A protective layer must never be more dangerous than the threat. Degrade to a *local* fallback limiter, not to nothing.
- **Fail closed** only for money, hard third-party quotas, and login attempts. Make it a **per-rule flag**, not a global policy.
- **Rules are data, not code.** Config store → cached in memory on every limiter → refreshed by poll + pub/sub. The config store is **never** in the hot path.
- **Most restrictive rule wins.** A request matching 3 rules must pass all 3 — pipelined into one round trip.
- **Ship every rule in shadow mode (`log_only`) first.** Measure what it *would* have blocked before it blocks anything.
- **Good 429s are load shedding.** `Retry-After` + `X-RateLimit-*` on *every* response (200s too) makes clients self-throttle. A bare 429 makes them retry in a tight loop and *increases* your load. Add jitter or you get a thundering herd.
- **Hot keys don't shard away.** One whale = one key = one slot = one node. Split the key into `key:{0..N}` with `limit/N` each.
- **Set a 10ms Redis timeout and a circuit breaker.** A *slow* Redis is worse than a dead one — it poisons your p50.
- **Alert on `fail_open` events.** When the limiter silently stops limiting, every request succeeds and nothing looks wrong. That's the danger.
- **A 429 spike is ambiguous.** Few keys = abuse (working as intended). Many distinct customers = you shipped a bad rule (roll it back).
- **At 1M DAU the whole thing is ~750 MB and ~5,000 QPS.** That's 5% of one Redis node. Don't shard until you need to — and say so.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [93 — HLD Approach Framework](./93-hld-approach-framework.md) — the 5-step method we just executed end to end |
| **Next** | [105 — HLD Distributed Cache](./105-hld-distributed-cache.md) — the Redis layer this design leans on, designed properly: sharding, replication, eviction |
| **Related** | [70 — Rate Limiting](./70-rate-limiting.md) — the five algorithms in depth; read it first, this doc assumes it |
| **Related** | [58 — API Gateway](./58-api-gateway.md) — where the limiter actually lives, and what else the gateway does in the same hop |
| **Related** | [123 — LLD Rate Limiter](./123-lld-rate-limiter.md) — the same problem zoomed in: classes, strategy pattern, and a full single-node implementation |
