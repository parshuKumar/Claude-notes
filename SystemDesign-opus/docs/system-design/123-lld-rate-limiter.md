# 123 — Design a Rate Limiter (Class Level)
## Category: LLD Case Study

---

## What is this?

A rate limiter is a small piece of code that decides, for each incoming request, "yes, you may proceed" or "no, you've asked too often — wait." Think of a **nightclub bouncer with a clicker counter**: they let people in up to a limit, and once the room is full (or you've shown up too many times this hour), you're turned away.

This document is the **class-level (LLD) companion** to the rate-limiting theory (topic 70) and the distributed HLD design (topic 95). Here we do not care about Redis, clusters, or millions of nodes. We care about one thing: writing a clean, testable, pluggable `RateLimiter` class in JavaScript where the algorithm is **swappable** without touching the limiter itself.

---

## Why does it matter?

**Interview angle.** "Design a rate limiter" is one of the most common LLD (low-level design) questions because it is a near-perfect showcase for the **Strategy pattern**. The interviewer is watching whether you can (a) define one clean interface, (b) hang three different algorithms off it, and (c) let the caller swap them at construction time with zero changes to the limiter. If you hard-code one algorithm with a pile of `if` statements, you've failed the point of the question.

**Real-work angle.** Every API you ship needs throttling: to stop abuse, to protect a fragile downstream service, to enforce paid tiers ("free = 100 req/min, pro = 10,000"). If your rate-limiting logic is a tangle glued into your HTTP middleware, you can never test it in isolation, never A/B two algorithms, and never reuse it. A well-factored limiter is a class you can unit-test in milliseconds with a fake clock.

If you skip this, you end up copy-pasting counter logic into every route, each subtly buggy, none tested.

---

## The core idea — explained simply

### The analogy: an arcade token machine

Imagine an arcade where each player has a **personal token cup**. There is a machine behind the counter that governs the cup. Every time you want to play a game, you hand the machine a request; it looks at your cup and either gives you a green light (a token is spent) or a red light (empty — come back later). Crucially, the arcade owner can **swap the machine's internal rulebook** without changing the counter, the cups, or the players:

- **Rulebook A ("token bucket"):** your cup holds up to 5 tokens and slowly refills 1 token per second. You can blow all 5 at once (a burst), then you're rate-limited to the refill trickle.
- **Rulebook B ("fixed window"):** the machine resets a shared tally every minute on the clock. You get 5 plays per minute — but the reset is abrupt.
- **Rulebook C ("sliding log"):** the machine keeps the exact timestamps of your last few plays and counts how many fall inside the last 60 seconds.

The counter, the cups, and the act of "ask the machine" never change. Only the rulebook does. That swappable rulebook is the **Strategy**.

| Analogy piece | Technical concept |
|---|---|
| The arcade machine | `RateLimiter` (the facade the caller talks to) |
| The swappable rulebook | `RateLimitStrategy` (the interface) |
| Rulebook A / B / C | `TokenBucketStrategy`, `FixedWindowStrategy`, `SlidingWindowLogStrategy` |
| Your personal token cup | per-client state (a bucket, a counter, or a log) |
| The shelf of everyone's cups | the `Map<clientId, state>` inside the strategy |
| "Give me a green/red light" | `allowRequest(clientId)` returning `true`/`false` |
| The wall clock the machine reads | an **injectable clock** (so tests can fake time) |

The whole design flows from one decision: **the algorithm is a strategy object, and the limiter holds a reference to it.**

---

## Key concepts inside this topic

### 1. Requirements & clarifying questions

Before writing code, pin down exactly what the class must do. The public contract is tiny:

```js
rateLimiter.allowRequest(clientId) // -> true (proceed) or false (reject)
```

**Functional requirements:**
- `allowRequest(clientId)` returns `true` if the request is within limits, `false` otherwise.
- Support **multiple algorithms** — token bucket, fixed window, sliding window — that are **swappable** without changing `RateLimiter`.
- **Per-client** limits: client "A" and client "B" have independent budgets.
- **Configurable** limit and window (e.g. "5 requests per 1 second").

**Non-functional / quality requirements:**
- Testable in isolation (no real sleeping in tests → injectable clock).
- Open for extension: a new algorithm should be a new class, not an edit to `RateLimiter` (the **Open/Closed Principle**).

**Clarifying questions to ask the interviewer (say these out loud):**

1. *Per-client or global?* We'll do **per-client** (keyed by `clientId`), which trivially covers global by using one fixed key.
2. *Single process or distributed?* **Single process, in-memory** here. If state must be shared across many servers, that is the **distributed** problem — see the HLD in topic 95 (Redis, atomic Lua scripts). We explicitly scope that out.
3. *Thread / async safety?* Node is single-threaded per event loop, but two `async` requests for the same client can interleave. We'll note where a race exists and how a per-client async mutex fixes it (recall topic 52), while keeping the demo single-threaded.
4. *What happens on reject?* We return `false`; the caller decides (HTTP 429, retry-after header, queue). The limiter only decides allow/deny.
5. *Fairness / burst tolerance?* This drives algorithm choice — token bucket allows bursts, fixed window is simplest, sliding log is most accurate.

### 2. Identify the core objects (nouns)

Read the requirements and pull out the nouns:

- **`RateLimiter`** — the facade the caller uses. Holds one strategy and delegates to it. This is the *only* class the rest of the app imports.
- **`RateLimitStrategy`** — the **interface** (an abstract base class in JS). Defines `allowRequest(clientId, now)`.
- **`TokenBucketStrategy`, `FixedWindowStrategy`, `SlidingWindowLogStrategy`** — the concrete algorithms. Each is fully self-contained.
- **Per-client state** — each strategy owns a `Map<clientId, state>`. The *shape* of the state differs per algorithm (a bucket has `{tokens, lastRefill}`; a fixed window has `{windowStart, count}`; a log has `number[]` of timestamps).
- **`Clock`** — a tiny abstraction over "what time is it now?" so tests can inject a `FakeClock`.

We do **not** need a config enum here, but limits (`capacity`, `refillPerSec`, `windowMs`) are passed into each strategy's constructor.

### 3. Behaviours (verbs) + who owns them

| Behaviour | Owner | Notes |
|---|---|---|
| `allowRequest(clientId)` | `RateLimiter` (public) | Reads the clock, delegates to the strategy. |
| `allowRequest(clientId, now)` | each `RateLimitStrategy` | Contains the actual allow/deny math. |
| Store & fetch per-client state | each strategy | Lazily creates fresh state for a first-seen client. |
| Refill / window-reset / log-prune | each strategy | Algorithm-specific bookkeeping, done lazily on read. |
| Tell the time | `Clock` | `SystemClock` in prod, `FakeClock` in tests. |

The key ownership rule: **the `RateLimiter` knows nothing about tokens, windows, or logs.** It only knows "I have a strategy; I ask it." Each strategy fully owns its state and its decision.

### 4. The design — Strategy is the whole point

We are using the **Strategy pattern** (recall topic 42): *define a family of interchangeable algorithms behind one interface, and let the client pick one at runtime.* Rate limiting is the textbook use case because all three algorithms answer the exact same question — "allow this request?" — with different internal logic.

Injecting the strategy into `RateLimiter` means adding a fourth algorithm (leaky bucket, sliding window counter) requires **zero changes** to `RateLimiter` — you write a new strategy class and pass it in. That is the **Open/Closed Principle** in action: open for extension, closed for modification.

Here are the three algorithms with their trade-offs (implemented cleanly in section 6):

- **`TokenBucketStrategy`** — a bucket of `capacity` tokens that refills at `refillPerSec`. Each request costs 1 token; refill is computed **lazily** from elapsed time (`tokensToAdd = elapsedSec * refillPerSec`) — no `setInterval` timer needed. **Trade-off:** allows short bursts up to `capacity`, then smooths to the refill rate. Memory: O(1) per client. The industry default.

- **`FixedWindowStrategy`** — a counter that resets every `windowMs`. Increment on each request; reject once the counter hits `limit`. **Trade-off:** dead simple and O(1) memory, but has the **boundary-burst flaw**: a client can send `limit` requests at 0:59 and another `limit` at 1:00, i.e. `2 × limit` in a two-second span straddling the reset.

- **`SlidingWindowLogStrategy`** — store the timestamp of every recent request; on each call, drop timestamps older than `windowMs`, then count the rest. **Trade-off:** perfectly accurate, no boundary burst, but **O(n) memory** per client (n = requests in the window) and O(n) work to prune. Great for low limits, expensive for high ones.

- *(Optional)* **`SlidingWindowCounterStrategy`** — a weighted blend of the current and previous fixed windows; O(1) memory, approximates the sliding log without storing every timestamp. Shown in Extensions.

### 5. Class diagram

```
                 ┌─────────────────────────────────┐
                 │           RateLimiter            │   the facade the app calls
                 ├─────────────────────────────────┤
                 │ - strategy : RateLimitStrategy   │
                 │ - clock    : Clock               │
                 ├─────────────────────────────────┤
                 │ + allowRequest(clientId) : bool  │
                 └────────────────┬────────────────┘
                                  │ delegates to (holds one)
                                  ▼
                 ┌─────────────────────────────────┐
                 │   «interface» RateLimitStrategy  │
                 ├─────────────────────────────────┤
                 │ # state : Map<clientId, ...>     │   per-client state lives here
                 ├─────────────────────────────────┤
                 │ + allowRequest(clientId, now)    │   abstract — subclasses implement
                 └──────┬──────────┬──────────┬─────┘
                        │          │          │   (implements ▲)
          ┌─────────────┘   ┌──────┘    └──────────────┐
          ▼                 ▼                          ▼
┌────────────────────┐ ┌──────────────────┐ ┌──────────────────────────┐
│ TokenBucketStrategy│ │FixedWindowStrategy│ │ SlidingWindowLogStrategy │
├────────────────────┤ ├──────────────────┤ ├──────────────────────────┤
│ capacity           │ │ limit            │ │ limit                    │
│ refillPerSec       │ │ windowMs         │ │ windowMs                 │
│ state:{tokens,     │ │ state:{windowStart│ │ state:number[] timestamps│
│   lastRefillMs}    │ │   , count}        │ │                          │
└────────────────────┘ └──────────────────┘ └──────────────────────────┘

        Clock  ◄─── injected into RateLimiter (SystemClock | FakeClock)
```

Read it top-down: the app touches only `RateLimiter`. `RateLimiter` holds one `RateLimitStrategy` and one `Clock`. The three concrete strategies each implement the same `allowRequest(clientId, now)` and each keep their own per-client `Map`. Swapping strategies swaps the whole algorithm; `RateLimiter` never changes.

---

## Visual / Diagram description

A sequence diagram of one `allowRequest` call for a client already known to a token-bucket limiter:

```
 caller            RateLimiter          Clock          TokenBucketStrategy      state Map
   │                    │                 │                    │                    │
   │ allowRequest("A")  │                 │                    │                    │
   │───────────────────>│                 │                    │                    │
   │                    │ now()           │                    │                    │
   │                    │────────────────>│                    │                    │
   │                    │   1000 (ms)     │                    │                    │
   │                    │<────────────────│                    │                    │
   │                    │ allowRequest("A", 1000)              │                    │
   │                    │────────────────────────────────────>│                    │
   │                    │                 │       get("A")     │                    │
   │                    │                 │───────────────────────────────────────>│
   │                    │                 │   {tokens:2, lastRefillMs:600}          │
   │                    │                 │<───────────────────────────────────────│
   │                    │        lazy refill: +0.4s*1/s = +0.4 tok → 2.4 tokens     │
   │                    │        2.4 >= 1 ? yes → tokens = 1.4, return true         │
   │                    │<────────────────────────────────────│                    │
   │      true          │                 │                    │                    │
   │<───────────────────│                 │                    │                    │
```

The important beat is the **lazy refill**: the strategy never runs a background timer. It reads "how long since I last touched this bucket?" and pours in exactly that many tokens right before deciding. This makes the whole thing a pure function of `(state, now)` — which is why a fake clock makes it perfectly testable.

---

## Real world examples

### Stripe (representative)

Stripe publicly documents using rate limiting to protect its API and famously blogged about using a **token-bucket-style** approach for burst tolerance plus a concurrency limiter. At the class level, the lesson mirrors ours: one limiter abstraction, an algorithm behind it, per-key (per-account) state. The exact production internals are proprietary, so treat specifics as conceptual.

### Nginx / `express-rate-limit` (factual)

Nginx's `limit_req` module implements the **leaky bucket** algorithm (a close cousin of token bucket) to smooth bursts. In the Node ecosystem, the widely used `express-rate-limit` middleware defaults to a **fixed-window** counter with pluggable stores (memory, Redis) — a direct real-world example of the same Strategy split we build here: the counting algorithm and the state store are both swappable.

### GitHub API (factual)

GitHub's REST API enforces per-user hourly limits and returns `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers — a fixed-window flavour where the "window" is an hour. Our `RateLimiter` returning a plain boolean is the minimal core; a production wrapper would additionally expose "remaining" and "reset" for those headers.

---

## Trade-offs

**Algorithm comparison** (the decision the Strategy pattern lets you defer):

| Algorithm | Memory / client | Burst behaviour | Accuracy | Best for |
|---|---|---|---|---|
| Token bucket | O(1) | Allows bursts up to capacity | Good | General-purpose API limiting |
| Fixed window | O(1) | **Boundary burst (2× limit)** | Coarse | Simplicity, rough limits |
| Sliding log | O(n) | No burst, exact | Perfect | Low limits, strict fairness |
| Sliding counter | O(1) | Minimal burst | Very good | High limits needing accuracy |

**Design-level trade-offs of the Strategy approach:**

| Pros | Cons |
|---|---|
| New algorithm = new class, zero edits to `RateLimiter` (OCP) | More classes / indirection than one hard-coded function |
| Each algorithm unit-tested in isolation | Slight overhead of a virtual method call (negligible) |
| Caller swaps algorithm at construction | Strategies must agree on one interface shape |
| Injectable clock → deterministic tests | Beginners may over-engineer tiny cases |

**The sweet spot:** default to **token bucket** for a general API (burst tolerance + O(1) memory), keep the **Strategy** seam so you can drop in sliding-window later, and always inject the clock so your tests never call `sleep`.

---

## Common interview questions on this topic

### Q1: "Why Strategy here and not just an `if/else` on an algorithm name?"

**Hint:** An `if/else` violates the Open/Closed Principle — adding an algorithm means editing `RateLimiter` and risking the existing branches. Strategy makes each algorithm a self-contained, independently testable class; the limiter is closed for modification. It also lets you swap algorithms per-instance at runtime.

### Q2: "How does the token bucket refill without a background timer?"

**Hint:** **Lazily.** Store `lastRefillMs`. On each request compute `elapsed = now - lastRefillMs`, add `elapsed/1000 * refillPerSec` tokens (capped at `capacity`), update `lastRefillMs`. No `setInterval`, no wasted CPU on idle clients, and it's a pure function of `(state, now)`.

### Q3: "What's the flaw in fixed window, and which algorithm fixes it?"

**Hint:** The **boundary burst**: `limit` requests just before the reset plus `limit` just after allows `2 × limit` in a short span. **Sliding window log** (exact) or **sliding window counter** (approximate, O(1)) fixes it by looking at a rolling window instead of a hard reset.

### Q4: "This runs in an async Node server. Where's the race, and how do you fix it?"

**Hint:** Two concurrent `async` requests for the same client can both read the state, both see a token available, and both decrement — a lost update. Since our decision is synchronous CPU work with no `await` in the middle, the event loop actually serializes it safely. The race appears only if state lives behind an `await` (e.g. Redis). Fix: a **per-client async mutex** (recall topic 52) around read-modify-write, or an atomic server-side operation (Redis Lua). See topic 95.

### Q5: "How would you make this work across 10 servers?"

**Hint:** That's the **distributed** problem, not this class design. In-memory `Map` state is per-process, so each server would allow the full limit → 10× the intended rate. Move state to a shared store (Redis) with atomic increments / Lua scripts, accept slight latency, and handle the store being down (fail-open vs fail-closed). Full treatment in topic 95.

---

## Practice exercise

**Build and run a `LeakyBucketStrategy`.** In ~30 minutes, add a fourth strategy to the code in section 6 without touching `RateLimiter`:

1. Model a leaky bucket: a queue "leaks" at a fixed rate `leakPerSec`; each request adds 1 unit of water; reject if the bucket would overflow `capacity`. Like token bucket but the metaphor is inverted (water in vs tokens out).
2. Store `{ level, lastLeakMs }` per client; lazily leak `elapsedSec * leakPerSec` before each decision.
3. Prove OCP: instantiate `new RateLimiter(new LeakyBucketStrategy(...), fakeClock)` — you should not edit `RateLimiter` at all.
4. Write a deterministic test with `FakeClock`: fire a burst, advance time, assert the bucket drained.

**Produce:** the `LeakyBucketStrategy` class plus a small `main()`-style script showing 8 rapid requests (some rejected), a time advance, and recovery — printing `true`/`false` per request.

---

## Quick reference cheat sheet

- **`RateLimiter`** — the facade; holds one strategy + one clock; exposes `allowRequest(clientId)`.
- **Strategy pattern** — the star of this design; interchangeable algorithms behind `RateLimitStrategy.allowRequest`.
- **Open/Closed Principle** — new algorithm = new class, no edits to `RateLimiter`.
- **Token bucket** — capacity + refill rate; lazy refill; allows bursts; O(1) memory; the default.
- **Fixed window** — counter reset per window; O(1); simplest; suffers the boundary burst.
- **Sliding log** — timestamps of recent requests; exact; O(n) memory.
- **Sliding counter** — weighted blend of two windows; O(1); accurate approximation.
- **Lazy refill** — compute tokens from elapsed time on read; no timers.
- **Injectable clock** — pass `now` / a `Clock` so tests are deterministic (no real sleep).
- **Per-client state** — a `Map<clientId, state>` inside each strategy; created lazily on first sight.
- **Async race** — concurrent read-modify-write; guard with a per-client mutex (topic 52) or atomic store op.
- **Distributed limiting** — shared state in Redis with atomic ops; that's topic 95, not this class design.
- **Returns boolean** — the limiter decides allow/deny; the caller does 429 / retry-after.

---

## Full JavaScript implementation

Real, complete, runnable. Save as `rate-limiter.js` and run `node rate-limiter.js`. It uses a **fake clock** so the demo is fully deterministic — no real sleeping.

```js
'use strict';

// ---------------------------------------------------------------------------
// Clock abstraction — the testability seam.
// Production uses SystemClock; tests/demo use FakeClock to control "now".
// ---------------------------------------------------------------------------
class Clock {
  now() { throw new Error('not implemented'); } // ms since epoch
}

class SystemClock extends Clock {
  now() { return Date.now(); }
}

class FakeClock extends Clock {
  constructor(startMs = 0) { super(); this._ms = startMs; }
  now() { return this._ms; }
  advance(ms) { this._ms += ms; return this._ms; } // move time forward in tests
}

// ---------------------------------------------------------------------------
// Strategy interface. Every algorithm implements allowRequest(clientId, now).
// It owns its own per-client state Map, so RateLimiter stays algorithm-agnostic.
// ---------------------------------------------------------------------------
class RateLimitStrategy {
  constructor() {
    // clientId -> algorithm-specific state (shape differs per subclass)
    this.state = new Map();
  }
  // Return true if the request is allowed, false if rejected.
  allowRequest(clientId, now) {
    throw new Error('RateLimitStrategy.allowRequest must be implemented');
  }
}

// ---------------------------------------------------------------------------
// 1. Token bucket: `capacity` tokens, refilling `refillPerSec` per second.
//    Each request costs 1 token. Refill is computed LAZILY from elapsed time —
//    no setInterval. Allows bursts up to capacity, then smooths to refill rate.
// ---------------------------------------------------------------------------
class TokenBucketStrategy extends RateLimitStrategy {
  constructor(capacity, refillPerSec) {
    super();
    this.capacity = capacity;
    this.refillPerSec = refillPerSec;
  }

  _bucketFor(clientId, now) {
    if (!this.state.has(clientId)) {
      // A first-seen client starts with a full bucket.
      this.state.set(clientId, { tokens: this.capacity, lastRefillMs: now });
    }
    return this.state.get(clientId);
  }

  allowRequest(clientId, now) {
    const b = this._bucketFor(clientId, now);

    // Lazy refill: pour in tokens for the time that has elapsed.
    const elapsedSec = (now - b.lastRefillMs) / 1000;
    if (elapsedSec > 0) {
      b.tokens = Math.min(this.capacity, b.tokens + elapsedSec * this.refillPerSec);
      b.lastRefillMs = now;
    }

    if (b.tokens >= 1) {
      b.tokens -= 1;   // spend one token
      return true;
    }
    return false;      // empty bucket → rejected
  }
}

// ---------------------------------------------------------------------------
// 2. Fixed window: a counter that resets every `windowMs`.
//    Simplest possible. Flaw: boundary burst (up to 2x limit across a reset).
// ---------------------------------------------------------------------------
class FixedWindowStrategy extends RateLimitStrategy {
  constructor(limit, windowMs) {
    super();
    this.limit = limit;
    this.windowMs = windowMs;
  }

  allowRequest(clientId, now) {
    // Which window does `now` fall into? Align to fixed boundaries.
    const windowStart = now - (now % this.windowMs);

    let s = this.state.get(clientId);
    if (!s || s.windowStart !== windowStart) {
      // New window (or first request): reset the counter.
      s = { windowStart, count: 0 };
      this.state.set(clientId, s);
    }

    if (s.count < this.limit) {
      s.count += 1;
      return true;
    }
    return false;
  }
}

// ---------------------------------------------------------------------------
// 3. Sliding window log: keep timestamps of recent requests, drop old ones,
//    count the rest. Exact — no boundary burst — but O(n) memory per client.
// ---------------------------------------------------------------------------
class SlidingWindowLogStrategy extends RateLimitStrategy {
  constructor(limit, windowMs) {
    super();
    this.limit = limit;
    this.windowMs = windowMs;
  }

  allowRequest(clientId, now) {
    if (!this.state.has(clientId)) this.state.set(clientId, []);
    const log = this.state.get(clientId); // array of timestamps (ms)

    // Drop timestamps that fall outside the sliding window.
    const cutoff = now - this.windowMs;
    while (log.length > 0 && log[0] <= cutoff) log.shift();

    if (log.length < this.limit) {
      log.push(now);
      return true;
    }
    return false;
  }
}

// ---------------------------------------------------------------------------
// The facade. Holds ONE strategy and ONE clock. Knows nothing about tokens,
// windows, or logs — it just reads the time and delegates. Swapping the
// strategy swaps the entire algorithm with zero changes here (OCP).
// ---------------------------------------------------------------------------
class RateLimiter {
  constructor(strategy, clock = new SystemClock()) {
    this.strategy = strategy;
    this.clock = clock;
  }

  allowRequest(clientId) {
    const now = this.clock.now();
    // NOTE (async safety): in a real server where the strategy awaits a shared
    // store (e.g. Redis), two concurrent calls for the same clientId could race
    // on read-modify-write. Guard with a per-client async mutex (see topic 52)
    // or an atomic store operation (Redis Lua). Our in-memory strategies do no
    // awaiting, so the single-threaded event loop serializes them safely.
    return this.strategy.allowRequest(clientId, now);
  }

  // Optional convenience: hot-swap the algorithm at runtime.
  setStrategy(strategy) { this.strategy = strategy; }
}

// ---------------------------------------------------------------------------
// main() — a deterministic demo using FakeClock (no real sleeps).
// ---------------------------------------------------------------------------
function main() {
  const clock = new FakeClock(0);

  // --- Part 1: Token bucket, capacity 5, refill 1 token/sec ---
  console.log('=== TokenBucketStrategy: capacity 5, refill 1/sec ===');
  const limiter = new RateLimiter(new TokenBucketStrategy(5, 1), clock);

  // Fire 8 rapid requests for client "A" at t=0. First 5 allowed, rest denied.
  const first = [];
  for (let i = 1; i <= 8; i++) first.push(limiter.allowRequest('A'));
  console.log('8 rapid requests for A:', first.map(String).join(', '));
  // Expect: true,true,true,true,true,false,false,false

  // Advance 3 seconds → refill 3 tokens. Next 3 requests allowed, 4th denied.
  clock.advance(3000);
  const afterWait = [];
  for (let i = 1; i <= 4; i++) afterWait.push(limiter.allowRequest('A'));
  console.log('after +3s, 4 requests for A:', afterWait.map(String).join(', '));
  // Expect: true,true,true,false

  // Client "B" is independent — its bucket is still full.
  console.log('B (independent) first request:', limiter.allowRequest('B')); // true

  // --- Part 2: SWAP to FixedWindow, showing swappability + boundary burst ---
  console.log('\n=== Swap to FixedWindowStrategy: 5 per 1000ms ===');
  const clock2 = new FakeClock(900); // start near a window boundary (t=900ms)
  limiter.setStrategy(new FixedWindowStrategy(5, 1000)); // same RateLimiter, new algo!
  limiter.clock = clock2;

  // Window [0,1000): fire 5 at t=900 → all allowed, 6th denied.
  const w1 = [];
  for (let i = 1; i <= 6; i++) w1.push(limiter.allowRequest('C'));
  console.log('t=900ms, 6 requests for C:', w1.map(String).join(', '));
  // Expect: true x5, false

  // Cross into window [1000,2000) at t=1000: counter resets → 5 more allowed.
  clock2.advance(100); // now t=1000
  const w2 = [];
  for (let i = 1; i <= 5; i++) w2.push(limiter.allowRequest('C'));
  console.log('t=1000ms, 5 requests for C:', w2.map(String).join(', '));
  // Expect: true x5  --> 10 allowed in ~100ms across the boundary = the flaw!

  // --- Part 3: SlidingWindowLog has no boundary burst ---
  console.log('\n=== SlidingWindowLogStrategy: 5 per 1000ms ===');
  const clock3 = new FakeClock(900);
  const slidingLimiter = new RateLimiter(new SlidingWindowLogStrategy(5, 1000), clock3);
  const s1 = [];
  for (let i = 1; i <= 5; i++) s1.push(slidingLimiter.allowRequest('D')); // 5 at t=900
  clock3.advance(100); // t=1000
  const s2 = [];
  for (let i = 1; i <= 5; i++) s2.push(slidingLimiter.allowRequest('D'));
  console.log('t=900 then t=1000, requests for D:', [...s1, ...s2].map(String).join(', '));
  // The 5 from t=900 are still inside the 1000ms window at t=1000 → t=1000 batch denied.
  // Expect: true x5, then false x5  --> NO boundary burst.
}

main();

module.exports = {
  Clock, SystemClock, FakeClock,
  RateLimitStrategy, TokenBucketStrategy, FixedWindowStrategy, SlidingWindowLogStrategy,
  RateLimiter,
};
```

**Expected output:**

```
=== TokenBucketStrategy: capacity 5, refill 1/sec ===
8 rapid requests for A: true, true, true, true, true, false, false, false
after +3s, 4 requests for A: true, true, true, false
B (independent) first request: true

=== Swap to FixedWindowStrategy: 5 per 1000ms ===
t=900ms, 6 requests for C: true, true, true, true, true, false
t=1000ms, 5 requests for C: true, true, true, true, true

=== SlidingWindowLogStrategy: 5 per 1000ms ===
t=900 then t=1000, requests for D: true, true, true, true, true, false, false, false, false, false
```

Notice Part 2 allows 10 requests within ~100 ms across the reset — the fixed-window boundary burst — while Part 3's sliding log correctly rejects the second batch because those five earlier timestamps are still inside the rolling 1000 ms window. Same `RateLimiter`, three algorithms, proven both correct and swappable.

**On the deterministic clock:** because every decision is a pure function of `(state, now)` and `now` comes from an injectable `Clock`, the demo needs no `setTimeout` or `await sleep`. A test can `clock.advance(3000)` to simulate three seconds instantly. This is a general testability lesson: **never read the wall clock directly inside logic you want to test** — inject time.

---

## Design patterns used and WHY

- **Strategy (the star).** `RateLimitStrategy` is the interface; `TokenBucketStrategy`, `FixedWindowStrategy`, and `SlidingWindowLogStrategy` are interchangeable implementations. **Why:** the three algorithms answer the identical question with different logic — the exact shape Strategy exists for. It gives us the Open/Closed Principle (new algorithm = new class), per-instance runtime selection, and isolated unit tests. Recall the pattern from topic 42.

- **Factory (per-client state creation).** Each strategy lazily *manufactures* fresh state the first time it sees a `clientId` (`_bucketFor`, the `windowStart` reset, `state.set(clientId, [])`). **Why:** every client must get its own independent bucket/window/log; a factory-on-first-touch keeps memory proportional to active clients and guarantees no cross-client leakage. You could hoist this into an explicit `stateFactory` function injected into the strategy — useful when the state object itself is complex.

- **Dependency injection of the clock (testability).** `Clock` is injected into `RateLimiter`; production passes `SystemClock`, tests pass `FakeClock`. **Why:** it removes the single biggest obstacle to testing time-based code — the real wall clock. Deterministic tests run in microseconds and never flake. This is arguably the highest-leverage decision in the whole design.

- *(Implicit)* **Facade.** `RateLimiter` is a thin facade hiding which algorithm and clock are in play, so the rest of the app depends on one stable method: `allowRequest(clientId)`.

---

## Extensions the interviewer asks for

**"Add a new algorithm — sliding window counter."** A new Strategy class; `RateLimiter` untouched. It blends the current and previous fixed windows by weight:

```js
class SlidingWindowCounterStrategy extends RateLimitStrategy {
  constructor(limit, windowMs) { super(); this.limit = limit; this.windowMs = windowMs; }
  allowRequest(clientId, now) {
    const cur = now - (now % this.windowMs);        // current window start
    let s = this.state.get(clientId);
    if (!s) { s = { curStart: cur, curCount: 0, prevCount: 0 }; this.state.set(clientId, s); }
    if (s.curStart !== cur) {
      // Rolled into a new window: today's "previous" is yesterday's "current".
      s.prevCount = (cur - s.curStart === this.windowMs) ? s.curCount : 0;
      s.curStart = cur; s.curCount = 0;
    }
    // Weight the previous window by how much of it still overlaps `now`.
    const overlap = 1 - (now - cur) / this.windowMs;  // fraction of prev window still in view
    const estimate = s.prevCount * overlap + s.curCount;
    if (estimate < this.limit) { s.curCount += 1; return true; }
    return false;
  }
}
```

O(1) memory, no timestamp log, and it smooths the boundary burst. Drop it in with `new RateLimiter(new SlidingWindowCounterStrategy(5, 1000), clock)`.

**"Make it distributed with Redis."** This is the **HLD**, not this class design — see topic 95. The seam is clean: replace the in-memory `Map` inside a strategy with Redis calls, and make `allowRequest` `async`. The catch is atomicity: a naive `GET`/`SET` from two servers races, so you use an atomic **Redis Lua script** or `INCR`+`EXPIRE` executed server-side. Contrast with our design: **here state is per-process and decisions are synchronous; distributed limiting shares state across processes and must handle atomicity, network latency, and store failure (fail-open vs fail-closed).** Same Strategy shape, different (harder) constraints.

**"Different limits per endpoint or per tier."** Compose limiters. Keep a `Map<key, RateLimiter>` where the key is `endpoint` or `tier` (`free`/`pro`), each configured with its own strategy and limits. `allowRequest` picks the right limiter. No change to any strategy — the design already isolates limits inside strategy constructors.

**"Add async-safe per-client locking."** Wrap the strategy's read-modify-write in a per-client async mutex (recall topic 52) so concurrent `await`ing calls for the same client serialize:

```js
// sketch: acquire a per-client lock, run the decision, release.
async allowRequest(clientId) {
  const release = await this.locks.acquire(clientId); // one lock object per clientId
  try { return this.strategy.allowRequest(clientId, this.clock.now()); }
  finally { release(); }
}
```

Because the strategy is a black box behind one method, the lock wraps it without knowing which algorithm runs inside — the design absorbs the change at the facade, exactly where concurrency should be handled.

Each extension lands as either a new Strategy class or a wrapper at the facade — never as surgery inside `RateLimiter` or the existing algorithms. That is the payoff of getting the Strategy seam right.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [70 — Rate Limiting](./70-rate-limiting.md) | The theory of the algorithms (token bucket, windows) this doc implements as clean classes. |
| **Next** | [95 — HLD: Rate Limiter](./95-hld-rate-limiter.md) | The distributed system version — Redis, atomic ops, multi-node state — that this class design deliberately scopes out. |
| **Related** | [42 — Strategy Pattern](./42-pattern-strategy.md) | The pattern that is the entire point of this design; swappable algorithms behind one interface. |
| **Related** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The 7-step nouns/verbs/classes method this doc follows end to end. |
| **Related** | [124 — LLD: LRU Cache](./124-lld-lru-cache.md) | Another compact LLD case study emphasizing clean data-structure design and testability. |
