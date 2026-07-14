# 73 — Circuit Breaker Pattern
## Category: HLD Components

---

## What is this?

A **circuit breaker** is a piece of code that sits between your service and a dependency (another service, a database, a third-party API) and **stops calling that dependency when it looks broken.** It watches the last N calls, and if too many of them fail, it "trips" — every subsequent call is rejected instantly, without even trying, for a cooldown period.

The name comes from the metal box in your house. When there's a short circuit, the breaker **trips and cuts the power**. It doesn't try harder. It doesn't keep pushing current into the fault. It fails fast — because the alternative is that the wiring heats up and the house burns down. Then, after you fix the fault, you flip it back on and the electricity flows again.

Your service is the house. The dependency is the faulty appliance. The breaker's job is to make sure one broken appliance doesn't take down the whole building.

---

## Why does it matter?

Because **one slow, non-critical dependency can take down your entire product**, and almost every engineer's first instinct — "just retry it" — makes that outcome *more* likely, not less.

**What breaks if you don't understand this:**
- A recommendations service (which nobody would die without) gets slow, and your checkout page starts timing out. Revenue goes to zero because of a feature nobody was even looking at.
- Your on-call gets paged for "the API is down," and it takes 40 minutes to trace the failure four services deep to the actual culprit.
- Your retries turn a small blip into a permanent outage, because the struggling service now gets 3x the traffic and can never catch up.

**The interview angle:** "What happens if service X goes down?" is one of the most common follow-ups in any HLD interview. "We add a circuit breaker" is a good answer. "We add a circuit breaker, with a timeout, retries with exponential backoff and jitter, a bulkhead per dependency, and a fallback that serves stale cache" is a *senior* answer. Interviewers listen for whether you say **timeout** in the same breath — a circuit breaker without timeouts is decoration.

**The real-work angle:** Every production service that makes a network call to something it doesn't own needs this. You will configure one within your first year at any company running microservices.

---

## The core idea — explained simply

### The Blocked Kitchen Analogy

You run a restaurant. Waiters take orders from tables and pass them to the kitchen. The kitchen has one door.

One night, the **dessert station** has a problem. Not broken — just *slow*. Every dessert takes 10 minutes instead of 30 seconds.

Here's what happens if nobody intervenes:

1. A waiter takes a dessert order and walks to the dessert station. He **waits**. He's now out of circulation for 10 minutes.
2. Another table orders dessert. Another waiter goes and waits. Now two waiters are frozen.
3. You have 8 waiters. Within twenty minutes, **all 8 are standing at the dessert station**, holding trays, doing nothing.
4. Now nobody is taking orders. Nobody is serving mains. Nobody is bringing drinks.
5. **The restaurant is dead** — not because the kitchen exploded, but because a slow *dessert* station absorbed every waiter you had.

Notice the cruel detail: if the dessert station had been **completely on fire** — visibly, obviously destroyed — this would never have happened. The waiter would have come back immediately and said "no desserts tonight." He'd be free to serve the next table.

**Slow is worse than down.** A fast failure gives you your resource back. A slow call holds it hostage.

The fix is a manager standing at the kitchen door with a clipboard. He counts. "The last 20 dessert orders — 14 of them took over a minute. Right. **No more dessert orders go through.** If a waiter asks, I tell him 'no dessert' immediately, and he goes back to his tables." That's the circuit breaker in the OPEN state. Every 30 seconds he lets *one* waiter through to test whether the dessert station has recovered. That's HALF-OPEN.

### Mapping the analogy back

| Restaurant | Distributed system |
|---|---|
| Waiter | A thread / a slot in your connection pool / an in-flight request holding memory |
| Dessert station | A downstream dependency (service, DB, third-party API) |
| Waiter standing there for 10 minutes | A request blocked on a call with no timeout |
| All 8 waiters stuck | Resource exhaustion — pool drained, event loop saturated with pending promises |
| Restaurant goes dead | **Cascading failure** — your service stops answering *anyone* |
| Manager with the clipboard | The circuit breaker |
| "No dessert" said instantly | Fail fast — return an error/fallback in microseconds |
| Letting one waiter test | HALF-OPEN probe |
| "Have the ice cream instead" | Fallback / graceful degradation |

---

## Key concepts inside this topic

### 1. Cascading failure — the thing you are actually preventing

Let's make it concrete with real services. You have a product page:

```
Browser  ──▶  B: API Gateway  ──▶  C: Product Service  ──▶  D: Recommendations DB
```

- **D** (the recommendations store) develops a problem. It is **not down**. It just got slow — a bad query plan, a hot shard, whatever. Responses now take **10 seconds** instead of 20ms.
- **C** (Product Service) calls D on every single product page render, and *waits* for it. C is a Node process. Each pending call to D holds a socket, a promise, and a chunk of the request context in memory.
- C serves 500 requests/sec. Each one now sits for 10s. So the number of in-flight requests stacks up: 500/s × 10s = **5,000 concurrent pending requests**. C's HTTP agent has `maxSockets: 100`. The other 4,900 are queued in the agent, waiting for a free socket, and *they* are also holding memory.
- C's event loop is now drowning. Even its `/health` endpoint responds in 8 seconds. **C has effectively stopped answering.**
- **B** (API Gateway) calls C, and B has no timeout either. B's own connection pool fills with pending calls to C.
- Now B is dead. **The whole site is down.**

```
       BEFORE                              DURING THE CASCADE
  ┌──────────────┐                     ┌──────────────┐
  │  B: Gateway  │  healthy            │  B: Gateway  │  DEAD (pool full)
  └──────┬───────┘                     └──────┬───────┘
         │ 20ms                               │ blocked
  ┌──────▼───────┐                     ┌──────▼───────┐
  │  C: Product  │  healthy            │  C: Product  │  DEAD (pool full)
  └──────┬───────┘                     └──────┬───────┘
         │ 20ms                               │ blocked 10s
  ┌──────▼───────┐                     ┌──────▼───────┐
  │  D: Recs DB  │  healthy            │  D: Recs DB  │  SLOW (not even down!)
  └──────────────┘                     └──────────────┘

  Failure travels UPWARD, against the direction of the call arrows.
```

Sit with that for a second. **The recommendations widget is not important.** If it had returned an error instantly, C would have rendered the page without recommendations and nobody would have filed a ticket. Instead, a non-critical, optional, cosmetic feature killed the entire product — purely because it was slow rather than dead.

**The lesson, stated plainly: slow is worse than down.** A fast failure frees your resource immediately. A slow call holds your resource hostage, and resources are finite.

### 2. Why naive retries make it strictly worse

The natural reaction to "calls to D are failing" is:

```javascript
// This code is a loaded gun pointed at your own foot.
async function getRecs(userId) {
  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      return await http.get(`http://recs/users/${userId}`); // no timeout, either
    } catch (err) {
      // "just try again, it's probably transient"
    }
  }
  throw new Error('recs failed');
}
```

Do the arithmetic. D is struggling under its normal load of 500 req/s. Now every failing request is attempted **3 times**. D is now receiving **1,500 req/s** — three times the traffic — *at exactly the moment it is least able to handle it.*

This is a **retry storm**. It guarantees D never recovers. D was maybe 30 seconds away from working through its backlog; you have now made recovery mathematically impossible. And when D finally does come back for a moment, the wall of retries knocks it straight back over.

Retries are correct for a *single* transient blip — a dropped packet, one bad node behind a load balancer. They are catastrophic when the dependency is *saturated*, because saturation is exactly the condition where more traffic is poison. The circuit breaker is what tells the difference: **retry the individual failure, but if failures become the norm, stop calling entirely.**

### 3. The three states

A circuit breaker is a small state machine with exactly three states.

**CLOSED** — the normal state. (Confusing at first: "closed" means the circuit is complete, so current flows. Traffic goes through.) The breaker passes every call to the dependency and **records the outcome** in a rolling window of the last N calls. It trips to OPEN when:

- the **failure rate** in the window crosses a threshold — e.g. more than 50% of the last 20 calls failed — **AND**
- the window contains at least a **minimum request volume**.

That second condition is the **minimum-volume gate**, and it is not optional. Without it, the very first call of the morning failing means "100% failure rate" and your breaker trips on a sample size of one. You want at least ~10-20 calls in the window before the percentage means anything.

Note what counts as a "failure": a thrown error, a 5xx, **and a timeout**. A 404 or a 400 usually should *not* count — the dependency answered you correctly and quickly; it's your request that was wrong. Tripping a breaker because users keep asking for missing products would be absurd.

**OPEN** — the tripped state. Every call is **rejected instantly, without being attempted.** No socket is opened. No packet leaves your machine. You return a fallback, or throw a `CircuitOpenError`, in *microseconds*.

This is the entire point of the pattern, so let me say it plainly:

> **Failing fast protects YOU and protects THEM.**
> It protects **you** because your threads/sockets/event loop stay free — you can still serve every other part of your product at full speed.
> It protects **them** because the dying service suddenly stops being hammered and gets breathing room to actually recover — to work through its backlog, GC, refill its cache, finish its failover.

A struggling service under a wall of traffic cannot recover. You have to *take the load off it*. The circuit breaker is the only component in your architecture that does that automatically.

After a **cooldown timer** expires (say 5 seconds), the breaker moves to HALF-OPEN.

**HALF-OPEN** — the trial state. Let a **small, limited number of probe requests** through — often just 1 to 3 — while continuing to reject everything else instantly.

- If the probes **succeed** → the dependency has recovered → go to **CLOSED** and **reset the counters** (this matters: if you kept the old failure counts, you'd trip again on the next single failure).
- If **any probe fails** → back to **OPEN**, and **restart the cooldown — usually longer than last time** (exponential: 5s → 10s → 20s → 40s, capped at some maximum like 60s). A dependency that's still broken after the first cooldown probably needs a lot longer than 5 more seconds, and you don't want to keep poking it every 5 seconds forever.

The half-open state is what makes the breaker **self-healing**. Nobody has to page a human to flip it back on.

### 4. The state machine

```
                       failure rate > threshold
                       AND calls >= minVolume
        ┌──────────┐  ────────────────────────────▶  ┌──────────┐
        │          │                                  │          │
        │  CLOSED  │                                  │   OPEN   │
        │          │                                  │          │
        │ traffic  │  ◀────────────────────────────   │  reject  │
        │ flows;   │      all probes succeeded        │  every   │
        │ count    │      (reset all counters)        │  call    │
        │ failures │                                  │ instantly│
        └──────────┘                                  └────┬─────┘
              ▲                                            │
              │                                            │ cooldown
              │                                            │ timer
              │ probes OK                                  │ expires
              │                                            ▼
              │                                      ┌──────────────┐
              └───────────────────────────────────── │  HALF-OPEN   │
                                                     │              │
                     any probe fails                 │ let N probe  │
              ┌───────────────────────────────────── │ calls        │
              │  → OPEN, cooldown × 2 (backoff)      │ through;     │
              │                                      │ reject rest  │
              └──────────────────────────────────▶   └──────────────┘
```

Three states, four transitions. Draw this on the whiteboard and you have already earned most of the marks for this question.

### 5. Full Node.js implementation

This is a complete, correct, runnable circuit breaker. No dependencies. Save it as `circuit-breaker.mjs` and run `node circuit-breaker.mjs`.

```javascript
// ─────────────────────────────────────────────────────────────
// circuit-breaker.mjs
// ─────────────────────────────────────────────────────────────

export const State = Object.freeze({
  CLOSED: 'CLOSED',
  OPEN: 'OPEN',
  HALF_OPEN: 'HALF_OPEN',
});

export class CircuitOpenError extends Error {
  constructor(name) {
    super(`Circuit "${name}" is OPEN — call rejected without being attempted`);
    this.name = 'CircuitOpenError';
    this.isCircuitOpen = true; // lets callers distinguish "we didn't try" from "it failed"
  }
}

export class CallTimeoutError extends Error {
  constructor(name, ms) {
    super(`Circuit "${name}": call exceeded timeout of ${ms}ms`);
    this.name = 'CallTimeoutError';
    this.isTimeout = true;
  }
}

/**
 * A fixed-size ring buffer of the last N outcomes.
 *
 * WHY a rolling window and not a plain counter: a plain lifetime counter can
 * never forget. A service that failed 1,000 times last Tuesday would keep the
 * breaker tripped forever. We only ever care about "how are things RIGHT NOW".
 */
class RollingWindow {
  #buf;
  #cursor = 0;
  #filled = 0;
  #failures = 0;

  constructor(size) {
    this.#buf = new Array(size).fill(null); // null = slot never used
  }

  record(ok) {
    const evicted = this.#buf[this.#cursor];
    if (evicted === null) this.#filled++;      // window is still growing
    else if (evicted === false) this.#failures--; // the outgoing entry was a failure
    this.#buf[this.#cursor] = ok;
    if (!ok) this.#failures++;
    this.#cursor = (this.#cursor + 1) % this.#buf.length;
  }

  reset() {
    this.#buf.fill(null);
    this.#cursor = 0;
    this.#filled = 0;
    this.#failures = 0;
  }

  get total() { return this.#filled; }
  get failures() { return this.#failures; }
  get failureRate() { return this.#filled === 0 ? 0 : this.#failures / this.#filled; }
}

export class CircuitBreaker {
  constructor({
    name = 'breaker',
    windowSize = 20,        // consider the last 20 calls
    failureThreshold = 0.5, // trip if >50% of them failed
    minimumRequests = 10,   // ...but only once we have >=10 samples (the volume gate)
    timeoutMs = 2000,       // NON-NEGOTIABLE. A breaker without this is useless.
    cooldownMs = 5000,      // how long to stay OPEN before the first probe
    maxCooldownMs = 60000,  // cap on the exponential backoff
    halfOpenProbes = 3,     // how many trial calls must succeed to close again
    fallback = null,        // (err, ...args) => value  — served instead of throwing
    isFailure = defaultIsFailure,
  } = {}) {
    Object.assign(this, {
      name, failureThreshold, minimumRequests, timeoutMs,
      cooldownMs, maxCooldownMs, halfOpenProbes, fallback, isFailure,
    });

    this.state = State.CLOSED;
    this.window = new RollingWindow(windowSize);
    this.nextAttemptAt = 0;   // when OPEN, the timestamp at which we may probe
    this.consecutiveTrips = 0; // drives exponential backoff of the cooldown
    this.probesRemaining = 0;  // budget of trial calls allowed in HALF_OPEN
    this.probeSuccesses = 0;
    this.metrics = { success: 0, failure: 0, timeout: 0, rejected: 0, trips: 0 };
  }

  /**
   * Run `fn(...args)` through the breaker.
   * Returns the function's value, the fallback's value, or throws.
   */
  async fire(fn, ...args) {
    if (!this.#allowRequest()) {
      this.metrics.rejected++;
      const err = new CircuitOpenError(this.name);
      if (this.fallback) return this.fallback(err, ...args);
      throw err; // fails in microseconds — no socket, no packet, no waiting
    }

    try {
      const result = await this.#callWithTimeout(fn, args);
      this.#onSuccess();
      return result;
    } catch (err) {
      if (this.isFailure(err)) this.#onFailure(err);
      else this.#onSuccess(); // e.g. a 404: the dependency is HEALTHY, our input was bad
      if (this.fallback) return this.fallback(err, ...args);
      throw err;
    }
  }

  // ── State machine ────────────────────────────────────────────

  #allowRequest() {
    const now = Date.now();

    if (this.state === State.CLOSED) return true;

    if (this.state === State.OPEN) {
      if (now >= this.nextAttemptAt) {
        this.#toHalfOpen();
        return this.#takeProbe();
      }
      return false; // still cooling down
    }

    // HALF_OPEN: admit only as many trial calls as we have budget for.
    return this.#takeProbe();
  }

  #takeProbe() {
    if (this.probesRemaining <= 0) return false;
    this.probesRemaining--;
    return true;
  }

  #onSuccess() {
    this.metrics.success++;
    if (this.state === State.HALF_OPEN) {
      this.probeSuccesses++;
      // Only close once ALL probes have come back clean. One lucky call proves nothing.
      if (this.probeSuccesses >= this.halfOpenProbes) this.#toClosed();
      return;
    }
    this.window.record(true);
  }

  #onFailure(err) {
    this.metrics.failure++;
    if (err?.isTimeout) this.metrics.timeout++;

    // A single failure during the trial means it is still sick. Straight back to OPEN.
    if (this.state === State.HALF_OPEN) return this.#trip();

    this.window.record(false);
    if (
      this.window.total >= this.minimumRequests &&      // the volume gate
      this.window.failureRate > this.failureThreshold   // the failure gate
    ) {
      this.#trip();
    }
  }

  #trip() {
    this.consecutiveTrips++;
    this.metrics.trips++;
    // Exponential backoff: 5s, 10s, 20s, 40s ... capped. A dependency that is still
    // broken after one cooldown needs a lot longer than one more cooldown.
    const backoff = this.cooldownMs * 2 ** (this.consecutiveTrips - 1);
    const cooldown = Math.min(backoff, this.maxCooldownMs);
    this.state = State.OPEN;
    this.nextAttemptAt = Date.now() + cooldown;
    this.probesRemaining = 0;
    this.probeSuccesses = 0;
    this.#log(`TRIPPED → OPEN for ${cooldown}ms ` +
      `(failures ${this.window.failures}/${this.window.total})`);
  }

  #toHalfOpen() {
    this.state = State.HALF_OPEN;
    this.probesRemaining = this.halfOpenProbes;
    this.probeSuccesses = 0;
    this.#log(`cooldown expired → HALF_OPEN (${this.halfOpenProbes} probes allowed)`);
  }

  #toClosed() {
    this.state = State.CLOSED;
    this.window.reset();      // fresh start — old failures must not re-trip us instantly
    this.consecutiveTrips = 0; // recovery resets the backoff ladder
    this.probesRemaining = 0;
    this.probeSuccesses = 0;
    this.#log('probes succeeded → CLOSED (recovered)');
  }

  // ── Timeout ──────────────────────────────────────────────────

  /**
   * Caveat worth knowing: Promise.race does not CANCEL the losing promise. The
   * underlying socket keeps going until the OS/agent gives up. In production,
   * pass an AbortSignal (AbortSignal.timeout(ms)) into fetch/undici so the
   * request is genuinely torn down, not merely ignored.
   */
  async #callWithTimeout(fn, args) {
    if (!this.timeoutMs) return fn(...args);
    let timer;
    try {
      return await Promise.race([
        fn(...args),
        new Promise((_, reject) => {
          timer = setTimeout(
            () => reject(new CallTimeoutError(this.name, this.timeoutMs)),
            this.timeoutMs,
          );
        }),
      ]);
    } finally {
      clearTimeout(timer); // otherwise the timer keeps the event loop alive
    }
  }

  #log(msg) {
    console.log(`  [breaker:${this.name}] ${msg}`);
  }
}

// A 4xx means the dependency answered you correctly and fast — it is healthy,
// your request was wrong. Do NOT let user typos trip your breaker.
function defaultIsFailure(err) {
  const status = err?.response?.status ?? err?.status;
  if (typeof status === 'number' && status >= 400 && status < 500) return false;
  return true;
}
```

### 6. A demo that trips it and shows it recovering

Append this to the same file and run it.

```javascript
// ─────────────────────────────────────────────────────────────
// DEMO — a recommendations service that goes SLOW, then recovers
// ─────────────────────────────────────────────────────────────

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));

let recsHealthy = true;
setTimeout(() => { recsHealthy = false; console.log('\n>>> RECS SERVICE GOES SLOW (3s responses)\n'); }, 300);
setTimeout(() => { recsHealthy = true;  console.log('\n>>> RECS SERVICE RECOVERS\n'); },  3000);

// Note it never THROWS — it just takes 3 seconds. That is the realistic failure
// mode, and the one that kills you if you have no timeout.
async function fetchRecommendations(userId) {
  if (!recsHealthy) {
    await sleep(3000);
    return [`slow-rec-for-${userId}`];
  }
  await sleep(20);
  return [`rec-a-for-${userId}`, `rec-b-for-${userId}`];
}

const breaker = new CircuitBreaker({
  name: 'recs',
  windowSize: 10,
  failureThreshold: 0.5,
  minimumRequests: 4,
  timeoutMs: 200,       // if recs can't answer in 200ms, we don't want it
  cooldownMs: 800,
  maxCooldownMs: 5000,
  halfOpenProbes: 2,
  fallback: () => ['popular-item-1', 'popular-item-2'], // graceful degradation
});

async function main() {
  for (let i = 1; i <= 45; i++) {
    const started = Date.now();
    const recs = await breaker.fire(fetchRecommendations, `user${i}`);
    const took = Date.now() - started;
    const degraded = recs[0].startsWith('popular') ? 'FALLBACK' : 'live';
    console.log(
      `#${String(i).padStart(2)} state=${breaker.state.padEnd(9)} ` +
      `${String(took).padStart(4)}ms  ${degraded}`,
    );
    await sleep(120);
  }
  console.log('\nmetrics:', breaker.metrics);
}

main();
```

**What you will see, and why it is the whole point:**

```
#01 state=CLOSED     22ms  live          <- healthy, fast
...
#04 state=CLOSED    201ms  FALLBACK      <- recs is slow; TIMEOUT saves us at 200ms
#05 state=CLOSED    201ms  FALLBACK
#06 state=CLOSED    202ms  FALLBACK
  [breaker:recs] TRIPPED → OPEN for 800ms (failures 4/6)
#07 state=OPEN        0ms  FALLBACK      <- 0ms. No socket. No waiting. THIS.
#08 state=OPEN        0ms  FALLBACK
  [breaker:recs] cooldown expired → HALF_OPEN (2 probes allowed)
#14 state=HALF_OPEN 201ms  FALLBACK      <- probe fails, still sick
  [breaker:recs] TRIPPED → OPEN for 1600ms   <- backoff doubled
...
  [breaker:recs] cooldown expired → HALF_OPEN (2 probes allowed)
#30 state=HALF_OPEN  21ms  live          <- probe 1 OK
#31 state=HALF_OPEN  22ms  live          <- probe 2 OK
  [breaker:recs] probes succeeded → CLOSED (recovered)
#32 state=CLOSED     21ms  live          <- fully healed, no human involved
```

Look at line #07. **Zero milliseconds.** While recs is dying, your service answers every user instantly with a degraded-but-useful response. That's a survivable product. Without the breaker, every one of those 45 requests would have parked a socket for 3 full seconds.

### 7. The companion patterns — never mention the breaker without these

A circuit breaker on its own is one leg of a four-legged table. An interviewer expects all four.

**a) Timeouts — always. This is the root cause.**

Go back to the cascade in section 1 and ask: *what actually caused it?* Not the slow DB. The **unbounded wait**. If C had a 200ms timeout on D, C would have shrugged off the slow calls and stayed alive — the outage would have stopped at D and never spread.

Every network call gets a timeout. Every single one. Default HTTP clients in Node have effectively no timeout (`http.request` will happily wait indefinitely on a socket that's open but silent). You must set them yourself:

```javascript
// Node 18+: AbortSignal.timeout genuinely tears the request down.
const res = await fetch('http://recs/users/42', { signal: AbortSignal.timeout(200) });
```

Pick the number from your dependency's **p99 latency**, not its average: if recs normally answers in 30ms at p99, a 200ms timeout is generous and anything slower is genuinely abnormal. **A circuit breaker without timeouts is useless**, because the breaker only sees failures — and a call that never returns never becomes a failure.

**b) Retries — with exponential backoff AND jitter**

Retry, but back off: wait 100ms, then 200ms, then 400ms, then 800ms. That gives the dependency room to breathe between attempts.

And add **jitter** — randomness — which is the part everyone forgets:

```javascript
// WITHOUT jitter: every client that failed at t=0 retries at EXACTLY t=100,
// then EXACTLY t=200. You have built a synchronised sledgehammer.
const delay = base * 2 ** attempt;

// WITH full jitter: the same clients spread themselves out across the window.
const delay = Math.random() * Math.min(cap, base * 2 ** attempt);
```

Why it matters, with numbers: a service behind a load balancer restarts, and **10,000 clients** all get a connection error in the same 50ms. Without jitter, all 10,000 sleep exactly 100ms and then hit the freshly-booted, cold-cache instance **simultaneously**. It falls over again. All 10,000 sleep 200ms, and hit it simultaneously again. This is a **thundering herd**, and it is perfectly self-sustaining — you have accidentally built a distributed denial-of-service against yourself, synchronised by your own retry policy.

With full jitter, those 10,000 retries are smeared uniformly across a 100ms window, then a 200ms window. The service sees a gentle ramp instead of a wall. It survives.

**Also: only retry idempotent operations.** Retrying a `POST /charge-card` that actually succeeded but whose response got lost will bill the customer twice.

**c) Bulkheads — isolate the resource pools**

A ship is divided into watertight **compartments (bulkheads)**. Hole in one compartment? It floods — and *only it* floods. The ship stays up.

The equivalent bug: your service uses **one shared HTTP agent / connection pool** for every downstream. Recs gets slow, floods that shared pool with 100 pending sockets, and now calls to the **payments** service can't get a socket either. A cosmetic dependency has starved a critical one.

The fix is a separate, bounded pool per dependency:

```javascript
import { Agent } from 'undici';

// Recs can burn through at most 20 connections. Even if every one of them hangs,
// payments still has its own 50 waiting untouched. The flood is compartmentalised.
const recsAgent     = new Agent({ connections: 20, headersTimeout: 200 });
const paymentsAgent = new Agent({ connections: 50, headersTimeout: 2000 });
```

Same idea applies to thread pools, DB connection pools, and even to *deployment* — running the critical path on separate instances from the risky path is a bulkhead at the infrastructure level.

**d) Fallbacks / graceful degradation**

When the breaker is OPEN you have to return *something*. A 500 is the laziest possible choice. Better, in rough order of preference:

1. **Stale cache** — "here are the recommendations we computed for you an hour ago." The user cannot tell.
2. **A sensible generic default** — the global top-10 popular items instead of personalised ones.
3. **An empty but valid response** — render the page with the recommendations row simply absent.
4. **An honest partial failure** — "Recommendations are temporarily unavailable," with the rest of the page working perfectly.

The design rule: **decide, per dependency, whether it is critical or optional**, and write the fallback *before* you need it. Payments failing must fail the request. Recommendations failing must not. If you can't articulate the fallback, you haven't finished the design.

---

## Visual / Diagram description

### Diagram 1: Where the breaker physically sits

```
                    ┌──────────────────────────────────────────────┐
                    │        C: PRODUCT SERVICE (your code)        │
                    │                                              │
  request  ────────▶│  handler()                                   │
                    │     │                                        │
                    │     ├──▶ [ BULKHEAD: recs pool, max 20 ]     │
                    │     │      │                                 │
                    │     │      ▼                                 │
                    │     │   ┌──────────────────┐                 │
                    │     │   │ CIRCUIT BREAKER  │  state=OPEN     │
                    │     │   │   (recs)         │──┐              │
                    │     │   └────────┬─────────┘  │ reject in    │
                    │     │            │            │ ~0ms         │
                    │     │      timeout 200ms      ▼              │
                    │     │            │      ┌───────────┐        │
                    │     │            X      │ FALLBACK  │        │
                    │     │      (not sent)   │ top-10    │        │
                    │     │                   └─────┬─────┘        │
                    │     │                         │              │
                    │     ├──▶ [ BULKHEAD: payments pool, max 50 ] │
                    │     │      │                                 │
                    │     │      ▼  ── unaffected, still healthy ──┼──▶ payments
                    │     │                                        │
                    │     ▼                                        │
  200 OK   ◀────────│  render(product, fallbackRecs)               │
  in 25ms           └──────────────────────────────────────────────┘
                                          ╎
                                          ╎ no traffic reaches it —
                                          ╎ it gets to recover
                                          ▼
                                 ┌──────────────────┐
                                 │   D: RECS (sick) │
                                 └──────────────────┘
```

Read it as a story: the request comes in. The recs call is routed through the recs bulkhead (bounded pool), then the recs breaker. The breaker is OPEN, so the call is rejected in microseconds and the fallback fires. Meanwhile the **payments** call goes through a completely separate pool and is entirely unaffected. The user gets a 200 in 25ms with a slightly worse product page. And D, receiving no traffic at all, gets the quiet it needs to recover.

### Diagram 2: The rolling window tripping the breaker

```
  window size = 10,  threshold = 50%,  minimum volume = 4

  call:    1    2    3    4    5    6    7    8    9   10
         ┌────┬────┬────┬────┬────┬────┬────┬────┬────┬────┐
         │ ✓  │ ✓  │ ✓  │ ✗  │ ✗  │ ✗  │ ✗  │    │    │    │
         └────┴────┴────┴────┴────┴────┴────┴────┴────┴────┘
                              ▲         ▲
                              │         └─ after call 7:
                              │            total=7, failures=4 → 57% > 50%
                              │            AND total(7) >= minVolume(4)
                              │            ══▶ TRIP
                              │
                              └─ after call 4: total=4, failures=1 → 25%. No trip.

  Without the minimum-volume gate, a single failed call at position 1 would read
  as "100% failure rate" and trip the breaker on a sample size of ONE.
```

---

## Real world examples

### 1. Netflix — Hystrix, and what came after it

Netflix built and open-sourced **Hystrix**, the library that made this pattern famous. Every call from a Netflix API server to a downstream service was wrapped in a `HystrixCommand` that bundled exactly the four things above: a timeout, a circuit breaker, a bulkhead (a dedicated thread pool per dependency), and a `getFallback()` method. Their canonical example is the one from this doc: if the personalised-recommendations service is unhealthy, the breaker opens and the UI shows a **generic, non-personalised row** instead. You still get a working Netflix home page — just a slightly dumber one. Most users never notice, and that is the entire success criterion.

Netflix later put Hystrix into **maintenance mode**, and the reason is instructive for interviews. Hystrix's thresholds are *statically configured* — you, a human, pick "50% of 20 calls" and "10 threads" per dependency, for hundreds of dependencies, and then those numbers slowly go stale as traffic patterns change. Netflix moved toward **adaptive concurrency limits** (their `concurrency-limits` library, borrowing TCP congestion-control ideas like AIMD): instead of a human-set thread count, the system continuously measures latency and *infers* the right concurrency limit, shedding load automatically when latency climbs. The pattern didn't go away — the **manual tuning** did.

### 2. Resilience4j — the standard in the JVM world

The library that succeeded Hystrix. It's a lightweight, functional set of composable decorators: `CircuitBreaker`, `RateLimiter`, `Retry`, `Bulkhead`, `TimeLimiter` — you stack exactly the ones you need around a call. It's worth knowing by name because it's what a Java/Kotlin shop will actually be running, and because its config knobs are the vocabulary of this whole topic: `slidingWindowSize`, `failureRateThreshold`, `minimumNumberOfCalls`, `waitDurationInOpenState`, `permittedNumberOfCallsInHalfOpenState`. If those five names feel familiar, it's because you just implemented every one of them.

### 3. `opossum` — the Node.js option

In Node you would not hand-roll the class above for production; you'd use **`opossum`**. Same model, plus batteries:

```javascript
import CircuitBreaker from 'opossum';

const breaker = new CircuitBreaker(fetchRecommendations, {
  timeout: 200,               // fail the call if it takes longer
  errorThresholdPercentage: 50,
  volumeThreshold: 10,        // the minimum-volume gate
  resetTimeout: 5000,         // cooldown before HALF_OPEN
});

breaker.fallback(() => TOP_TEN_POPULAR);

// Every transition is an event → wire these straight into your metrics.
breaker.on('open',     () => metrics.increment('breaker.recs.open'));
breaker.on('halfOpen', () => metrics.increment('breaker.recs.half_open'));
breaker.on('close',    () => metrics.increment('breaker.recs.close'));

const recs = await breaker.fire('user42');
```

Also note that **service meshes (Istio, Linkerd) and Envoy do this at the sidecar/proxy layer**, outside your code entirely — outlier detection ejects a failing upstream host from the load-balancing pool, which is a circuit breaker applied per *instance* rather than per *service*.

### 4. The observability angle — a trip is an ALERT

This is the point people forget, and it's a great thing to volunteer in an interview.

**A circuit breaker opening is one of the highest-signal events in your entire system.** It is a *leading* indicator: the breaker has detected a dependency degrading *before* your users have finished complaining, and — crucially — it has hidden the damage from them with a fallback. That means your user-facing error rate might look **fine** while a dependency is quietly on fire. If you're only alerting on 5xx, you're blind.

So:
- **Emit a metric on every state transition** (`open`, `half_open`, `close`) tagged with the dependency name.
- **Page on a breaker that has been OPEN for more than a minute**, or on any breaker that is flapping (repeated open → half-open → open) — flapping means it's genuinely, persistently broken, not a blip.
- **Put every breaker's state on one dashboard.** During an incident, "which breakers are open?" localises the fault in seconds instead of 40 minutes, because the breaker names *tell you which dependency broke*.

The breaker turns an invisible, slow-burning dependency failure into a single, loud, precisely-labelled signal.

---

## Trade-offs

| Pros | Cons |
|---|---|
| Prevents cascading failure — the outage stops at the broken service | You are now **deliberately failing requests that might have succeeded** |
| Frees your resources instantly (pool, event loop, memory stay healthy) | More moving parts: another state machine to reason about, test, and tune |
| Gives the dying dependency room to recover instead of hammering it | Bad tuning trips on normal traffic spikes and causes a **self-inflicted outage** |
| Self-heals via HALF-OPEN — no human has to flip anything back on | Fallback logic is real code that must be written, tested, and kept correct |
| Turns a diffuse "site is slow" into a precise "the recs breaker is open" | State is **per-process** — 50 pods learn the lesson independently, 50 times |

| Tuning knob | Too low | Too high |
|---|---|---|
| **Failure threshold** (e.g. 50%) | Trips on normal noise; you degrade users for nothing | Never trips; the cascade happens anyway |
| **Minimum volume** (e.g. 20) | 1 failure out of 1 = "100% failure" → trips on a sample of one | On a low-traffic endpoint, the window never fills and the breaker never engages |
| **Timeout** (e.g. p99 × 2) | You kill healthy-but-slightly-slow calls and manufacture failures | The unbounded wait comes back and the cascade returns |
| **Cooldown** (e.g. 5s) | You probe a dying service constantly and never let it rest | You keep serving degraded responses long after it recovered |
| **Half-open probes** (e.g. 3) | One lucky call closes the breaker, and it immediately re-trips (flapping) | Slow to recover; users stay degraded longer than necessary |

**Rule of thumb:** Put a breaker on **every synchronous network call to something you don't own**, but only bother with a *fallback* for the dependencies that are genuinely optional. Start with: timeout = p99 latency × 2 (rounded up), threshold = 50%, minimum volume = 20, cooldown = 5s with exponential backoff. Then tune from real traffic — and remember that if a dependency is *truly* critical (payments), the honest answer is often "no fallback, fail the request loudly" rather than pretending you succeeded.

---

## Common interview questions on this topic

### Q1: "Why is a slow dependency more dangerous than a dead one?"
**Hint:** A dead one fails instantly and gives you your resource back — the thread, the socket, the pool slot. A slow one *holds it hostage*. At 500 req/s with a 10s hang, you accumulate 5,000 in-flight requests and exhaust your pool. Resource exhaustion, not the error itself, is what kills you — and it propagates upward to your callers. Say the line: **slow is worse than down.**

### Q2: "Walk me through the three states and every transition."
**Hint:** CLOSED (traffic flows, count failures in a rolling window) → OPEN when failure rate > threshold **and** call volume ≥ minimum. OPEN (reject instantly, no call attempted) → HALF-OPEN when the cooldown expires. HALF-OPEN (let N probes through, reject the rest) → CLOSED if all probes succeed (**and reset the counters**), or straight back to OPEN on any single failure, with the cooldown now doubled. Mention the minimum-volume gate unprompted — it shows you've actually built one.

### Q3: "Isn't a circuit breaker enough on its own?"
**Hint:** No — it's one of four. Without a **timeout**, calls hang forever and never even register as failures, so the breaker never trips: the timeout is what *creates* the signal the breaker consumes. Without **bulkheads**, one dependency drains the shared pool and starves the others. Without a **fallback**, opening the breaker just converts hangs into 500s. And **retries without jitter** turn a blip into a thundering herd. Name all four.

### Q4: "Why does jitter matter in retry backoff?"
**Hint:** Because 10,000 clients that failed at the same instant will, without jitter, all sleep for exactly the same 100ms and retry at exactly the same moment — a synchronised wall of traffic that re-kills the service you were waiting for. Randomising the delay (`Math.random() * min(cap, base * 2**attempt)`) smears those retries across the window so the recovering service sees a ramp, not a wall. Then add: only retry idempotent calls.

### Q5: "Your breaker is per-process and you have 50 pods. Isn't that a problem?"
**Hint:** It's a genuine trade-off, and the honest answer is "mostly no." Each pod independently discovers the failure after its own ~20 calls, so there's some duplicated pain — but local state means **zero coordination latency, no shared-state dependency (which would itself need a breaker!), and no single point of failure.** A shared/distributed breaker in Redis sounds appealing and is almost always the wrong call. The real answer to fleet-wide coordination is at the load-balancer or mesh layer (Envoy outlier detection), which ejects bad *hosts* for everyone.

---

## Practice exercise

### Harden the payments call

You have this handler. It is a faithful reproduction of code that has taken down real companies.

```javascript
// checkout-handler.js  — DO NOT SHIP THIS
app.post('/checkout', async (req, res) => {
  const cart      = await cartService.get(req.user.id);        // critical
  const inventory = await inventoryService.check(cart.items);  // critical
  const recs      = await recsService.forUser(req.user.id);    // COSMETIC
  const charge    = await paymentsService.charge(cart.total);  // critical
  res.json({ orderId: charge.id, alsoYouMightLike: recs });
});
```

**Produce these five things (~30 minutes):**

1. **Name every failure mode.** For each of the four downstream calls, write one line: what happens to this handler *right now* if that service starts taking 30 seconds? Which of the four is the most absurd cause of a total outage?
2. **Classify each dependency** as CRITICAL or OPTIONAL, and for each one state the fallback (or "none — fail the request").
3. **Pick the numbers.** For each call, choose a timeout, a failure threshold, a minimum volume, and a cooldown. Justify the *timeout* specifically — what p99 are you assuming?
4. **Rewrite the handler** using the `CircuitBreaker` class from this doc. One breaker instance per dependency (never one shared breaker for everything — think about why). Give the optional one a fallback and the critical ones none.
5. **Add the observability.** Write the three lines of metric-emitting code you'd attach to the state transitions, and state the one alert you'd configure.

**Stretch:** the `paymentsService.charge` call is not idempotent. Your retry policy could double-charge a customer. In two sentences, say how you'd fix that (the answer is a concept from another doc in this curriculum).

---

## Quick reference cheat sheet

- **Slow is worse than down.** A fast failure returns your resource; a slow call holds it hostage until your pool is empty.
- **Cascading failure** = a downstream problem travels *upward* through blocked callers until the whole product is dead.
- **The breaker's job** is to fail fast — reject instantly instead of waiting on something that is clearly broken.
- **Failing fast protects you** (your pool and event loop stay free) **and protects them** (the dying service stops being hammered and can recover).
- **CLOSED** → traffic flows, failures counted in a rolling window of the last N calls.
- **OPEN** → every call rejected in microseconds, never attempted. After a cooldown, go HALF-OPEN.
- **HALF-OPEN** → a few trial probes. All succeed → CLOSED + reset counters. Any fail → OPEN, cooldown doubled.
- **Minimum-volume gate** is mandatory: without it, 1 failure out of 1 call reads as a 100% failure rate and trips the breaker.
- **A breaker without timeouts is useless** — a call that never returns never becomes a failure the breaker can see.
- **Retries need exponential backoff AND jitter.** No jitter → 10,000 clients retry in the same millisecond → thundering herd re-kills the service.
- **Only retry idempotent operations.** Retrying a charge can bill the customer twice.
- **Bulkheads** = a separate bounded pool per dependency, so a flood to one can't starve the others. (Ship compartments.)
- **Fallbacks** = stale cache > generic default > empty-but-valid > honest "unavailable" > 500. Decide critical vs optional *per dependency*.
- **A breaker tripping is an ALERT** — a leading indicator of an outage, and it may be *hiding* the damage from your error rate.
- **Node:** `opossum`. **JVM:** Resilience4j (was Hystrix). **Infra layer:** Envoy/Istio outlier detection.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [72 — Service Discovery](./72-service-discovery.md) — how services find each other; the breaker is what you do when the one you found is sick |
| **Next** | [80 — Monitoring and Observability](./80-monitoring-and-observability.md) — a tripped breaker is a leading indicator of an outage, and it only helps if someone is watching |
| **Related** | [71 — Microservices vs Monolith](./71-microservices-vs-monolith.md) — cascading failure is the tax you pay for splitting a monolith into network calls |
| **Related** | [67 — Message Queues](./67-message-queues.md) — the other answer to "don't wait on a slow dependency": don't call it synchronously at all |
| **Related** | [85 — Idempotency](./85-idempotency.md) — the prerequisite for safe retries, and the fix for the double-charge in the practice exercise |
| **Related** | [46 — State Pattern](./46-pattern-state.md) — the breaker is a three-state machine; that's the LLD pattern behind it |
| **Related** | [137 — Performance Optimization](./137-performance-optimization.md) — where the p99 numbers you use to pick your timeouts come from |
