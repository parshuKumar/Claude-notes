# 138 — Testing Distributed Systems
## Category: Advanced

---

## What is this?

**Testing a distributed system** is the practice of gaining confidence that a bunch of independent services, spread across many machines, still produce correct results *even when the machines and the network misbehave*. It's not just "does my function return the right number" — it's "does the whole system stay correct when a node dies mid-request, a message arrives twice, or the network splits in half."

Think of it like **crash-testing a car** versus **checking that the engine turns on**. Turning the engine on (a unit test) is necessary but tells you nothing about what happens in a 40 mph collision (a network partition). You only trust the car because someone deliberately smashed it into a wall in a controlled lab first.

---

## Why does it matter?

The failures that take down real systems are almost never the ones your unit tests cover. They are timing failures, partial failures, and failures that only appear at scale — and they are **non-deterministic**, meaning they show up one run in a thousand and vanish when you try to reproduce them.

**What breaks if you ignore this:**

- **You ship "works on my laptop" code.** On your laptop the database always responds in 2ms and never disappears. In production it occasionally takes 8 seconds or returns a connection error, and your code has no idea what to do.
- **A downstream team changes their API and you find out in production.** Service B renamed a field. Nothing in your build caught it because you tested Service A against a *mock* of B that still had the old field.
- **You double-charge a customer.** A retry after a timeout ran your payment logic twice because you never tested that the same request delivered twice produces one charge.
- **Your "resilient" fallback never actually worked.** You wrote a circuit breaker (recall [73 — Circuit Breaker](./73-circuit-breaker.md)) but never tested it under real failure, so the day the dependency died, the breaker didn't trip and the whole fleet hung.

**Interview angle:** "How would you test this?" is a follow-up on almost every design question, and strong candidates immediately talk about contract tests and fault injection — not just "I'd write unit tests." Naming *chaos engineering*, *consumer-driven contracts*, and *testing in production with guardrails* signals real distributed-systems maturity.

**Real-work angle:** In a microservices org (recall [71 — Microservices vs Monolith](./71-microservices-vs-monolith.md)), the testing strategy is what lets teams deploy independently without a giant coordinated integration phase. Bad testing strategy = every release is a company-wide fire drill.

---

## The core idea — explained simply

### The Aircraft Certification Analogy

You do not certify a new airplane by taking off and hoping. You certify it in **layers**, each layer testing more of the real thing, each one slower and more expensive than the last.

1. **Bench-test each part.** Does this one valve open at the right pressure? Fast, cheap, thousands of them, done in isolation on a workbench. → **Unit tests.**
2. **Assemble a subsystem and test it together.** Does the fuel pump actually feed the engine? You wire up two real parts and check the joint between them. → **Integration tests.**
3. **Verify the interface spec before mating two subsystems.** The wing team and the fuselage team agree on a bolt pattern *document*, and each side checks their part against that document — without physically bolting the plane together every time. → **Contract tests.**
4. **Fly the whole plane on a test flight.** Everything, end to end, one full journey. Extremely high confidence, extremely expensive, you do it rarely. → **End-to-end tests.**

But certification doesn't stop at "does it work when everything is fine." The valuable, terrifying tests are the **failure** tests:

5. **Deliberately break things.** Cut an engine at takeoff. Ice the wings. Cut a hydraulic line. Does the plane degrade *gracefully* or fall out of the sky? → **Fault injection & chaos engineering.**
6. **Run it flat-out for 10,000 hours.** Does anything crack, leak, or overheat under sustained stress? → **Load & soak testing.**

| Analogy | Technical concept |
|---------|-------------------|
| Bench-testing one valve | Unit test (one component, in isolation, with fakes) |
| Wiring the pump to the engine | Integration test (real DB, real queue) |
| The agreed bolt-pattern document | The **contract** between two services |
| Checking your part against the doc | Consumer-driven contract test (Pact) |
| Full test flight | End-to-end test across all services |
| Cutting an engine mid-flight | Fault injection |
| Cutting an engine on a *real* passenger flight (safely) | Chaos engineering (in production) |
| 10,000-hour endurance run | Soak test |
| Instruments in the cockpit | Observability (metrics, traces, logs) |

The rest of this document is those layers, in order, with code.

---

## Key concepts inside this topic

### 1. The testing pyramid, adapted for distributed systems

The classic pyramid says: **many fast tests at the bottom, few slow tests at the top.** The shape is about *cost and confidence*. A unit test runs in 2ms; an end-to-end test runs in 2 minutes and fails flakily. If you invert the pyramid (mostly E2E tests), your build takes an hour and nobody trusts it.

```
              ▲  slower, costlier, flakier, higher confidence
        ┌───────────┐
        │  E2E      │   a handful — only critical user journeys
        ├───────────┤
        │ Contract  │   one per service-to-service dependency
        ├───────────┤
        │Integration│   real DB / queue for each service
        ├───────────┤
        │   Unit    │   thousands — pure logic, milliseconds
        └───────────┘
              ▼  faster, cheaper, deterministic, lower confidence
```

The distributed twist: we **insert a contract layer** between integration and E2E, and we add an entire **second axis** (failure testing) that the classic pyramid never mentions.

**Unit tests** — test one component's logic with every dependency faked.

```js
// A shipping-cost calculator. No network, no DB — pure logic.
class ShippingCalculator {
  constructor(rateProvider) {   // dependency INJECTED (recall DIP, 18)
    this.rateProvider = rateProvider;
  }
  async cost(weightKg, zone) {
    const rate = await this.rateProvider.ratePerKg(zone);
    return Math.ceil(weightKg) * rate;
  }
}

// Test: we inject a FAKE rateProvider — no real service is called.
const fakeRates = { ratePerKg: async () => 5 };
const calc = new ShippingCalculator(fakeRates);
console.assert((await calc.cost(2.1, "EU")) === 15, "3kg * 5 = 15");
```

Notice the reason we *can* unit-test this: `rateProvider` is injected, not `new`-ed inside the class. **Dependency injection (recall [18 — Dependency Inversion])** is what makes a component testable in isolation. Unit tests catch logic bugs cheaply, but by design they cannot catch "the real rate service returns cents, not dollars" — that's an integration concern.

**What each layer catches:**

| Layer | Catches | Misses | Cost per test |
|-------|---------|--------|---------------|
| Unit | logic, branches, edge cases | wiring, serialization, real I/O | ~1ms |
| Integration | SQL, serialization, config, real driver quirks | cross-service contracts | ~100ms–2s |
| Contract | API mismatches between two services | whole-flow correctness | ~50ms |
| E2E | the full journey actually works | *which* component broke; rare timing bugs | seconds–minutes |

### 2. Integration tests — real components, real wiring

An integration test uses **real infrastructure** for the pieces immediately around your code — most commonly a real database in a throwaway container (via **Testcontainers**, a library that spins up Docker containers for the duration of a test).

```js
// Integration test: real Postgres in a disposable container.
import { PostgreSqlContainer } from "@testcontainers/postgresql";
import { OrderRepository } from "../src/OrderRepository.js";

let container, repo;
beforeAll(async () => {
  container = await new PostgreSqlContainer().start();   // real DB, real port
  repo = new OrderRepository(container.getConnectionUri());
  await repo.migrate();
});
afterAll(() => container.stop());

test("saves and reads an order with its real JSON column", async () => {
  const id = await repo.create({ items: ["sku-1"], total: 4200 });
  const found = await repo.findById(id);
  // This catches bugs a mock hides: JSON serialization, numeric precision,
  // the actual column names, NOT NULL constraints, timezone handling.
  expect(found.total).toBe(4200);
  expect(found.items).toEqual(["sku-1"]);
});
```

A mock of the database would have happily returned whatever you told it to. The real DB catches the bug where your `total` column is `integer` but your code passes `42.00` and Postgres rounds it. Integration tests are where **serialization, query, and config** bugs die.

### 3. Contract testing — the key technique for microservices

This is the most valuable idea in the document. Read it twice.

**The problem.** Service A (the *consumer*) calls Service B (the *provider*). To unit-test A, you mock B. But your mock is *your guess* of how B behaves. If B's team changes their API — renames a field, makes an optional field required, changes a status code — your mock still has the old shape, your tests stay green, and **A breaks in production.** Running both real services together in every build (E2E) catches it but is slow and couples the teams.

**The fix: a consumer-driven contract.** A writes down *exactly* what it needs from B as a machine-readable **contract** (using a tool like **Pact**). That contract does two jobs:

1. On **A's** build, the contract acts as the mock — A is tested against the recorded expectations, so A can't drift from what it claims to need.
2. On **B's** build, the same contract is **replayed against the real B** — B must prove it still satisfies every expectation A recorded.

If B removes a field A depends on, **B's build goes red**, at build time, before deploy. The break is caught by the team that caused it, on the day they caused it.

```js
// CONSUMER side (Service A): declare what A expects from B, and get a mock.
const provider = new Pact({ consumer: "OrderService", provider: "PricingService" });

await provider.addInteraction({
  state: "sku-1 exists",                       // a precondition B must set up
  uponReceiving: "a price lookup for sku-1",
  withRequest: { method: "GET", path: "/prices/sku-1" },
  willRespondWith: {                            // A's EXACT expectation of B
    status: 200,
    body: { sku: "sku-1", cents: 4200, currency: "USD" }
  }
});

// A's real client code is tested against the Pact mock server.
const price = await new PricingClient(provider.mockService.baseUrl).get("sku-1");
expect(price.cents).toBe(4200);
// This interaction is exported as a "pact file" and shipped to B's pipeline,
// where it is REPLAYED against the real PricingService. If B no longer returns
// `cents`, B's build fails — not A's production traffic.
```

**Why this is the microservices superpower:** it lets teams **deploy independently** (recall [71 — Microservices vs Monolith](./71-microservices-vs-monolith.md)). No giant synchronized integration environment, no "release train." Each team's build knows, at compile time, whether it broke a consumer or a provider. Contract testing gives you most of the safety of E2E at a fraction of the cost and flakiness.

### 4. End-to-end tests — high confidence, use sparingly

An E2E test drives a **complete user journey** through all real services: sign up → add to cart → check out → receive confirmation. It's the closest thing to a real user, so it gives the highest confidence — but it is **slow** (spin up everything), **flaky** (any of 8 services, the network, or timing can fail it), and **expensive** (a full environment).

The rule: **E2E tests are a scalpel, not a hammer.** Cover only your handful of revenue-critical paths (checkout, login, payment). Everything else should be caught cheaper below. A common failure mode is 400 E2E tests that take 45 minutes and fail randomly 10% of the time — the team stops trusting red builds, which is worse than having no tests.

### 5. Fault injection — test that failure degrades gracefully

Correctness tests ask "does it work?" Fault-injection tests ask **"does it fail *well*?"** You deliberately break a dependency and assert the system does the right thing: circuit breaker trips, fallback kicks in, error is surfaced cleanly instead of hanging.

```js
// A dependency wrapper you can make "misbehave" on command in a test.
class FlakyDependency {
  constructor(real) { this.real = real; this.mode = "ok"; }
  async call(req) {
    if (this.mode === "down")    throw new Error("ECONNREFUSED");
    if (this.mode === "slow")    { await sleep(5000); }     // slow, not down
    if (this.mode === "errors")  return { status: 503 };
    return this.real.call(req);
  }
}

test("circuit breaker trips and fallback serves cached price", async () => {
  const dep = new FlakyDependency(realPricing);
  const svc = new PricingClientWithBreaker(dep, { threshold: 3 });

  dep.mode = "down";
  for (let i = 0; i < 3; i++) await svc.price("sku-1").catch(() => {});
  // After 3 failures the breaker should be OPEN (recall 73).
  expect(svc.breaker.state).toBe("OPEN");
  // And a subsequent call must NOT hit the dead dependency — it falls back.
  const p = await svc.price("sku-1");
  expect(p.source).toBe("cache");
});
```

The `"slow"` mode matters most: **a slow dependency is more dangerous than a dead one**, because a dead one fails fast while a slow one ties up your threads/connections until you exhaust the pool. Always fault-test *latency*, not just outright failure.

### 6. Chaos engineering — fault injection in production

Chaos engineering takes fault injection and runs it **against the real production system**, continuously, as proof that your resilience actually holds under real conditions. Netflix's **Chaos Monkey** famously kills random production instances during business hours so engineers are *forced* to build services that survive it. (Recall [90 — Disaster Recovery](./90-disaster-recovery.md): chaos engineering is how you verify your DR assumptions are real instead of wishful.)

The disciplined loop:

1. **Define the steady-state hypothesis.** A measurable "system is healthy" signal — e.g. "checkout success rate stays above 99% and p99 latency below 300ms."
2. **Inject a fault.** Kill an instance, add 200ms latency to the payments call, drop 5% of packets to a Redis node.
3. **Observe.** Did steady state hold? If yes, your resilience works. If no, you found a real weakness — before a real outage did.
4. **Limit the blast radius.** Start in staging, then 1% of production traffic, one region, with an automatic abort ("stop the experiment if success rate drops below 95%").

**Start small.** The point is controlled, reversible experiments — not "delete the prod database and see what happens."

### 7. Load, stress, and soak testing

These verify your **non-functional requirements** (throughput, latency) rather than correctness.

- **Load testing** — apply *expected* traffic and confirm you meet your NFRs. Watch **p99, not the average** (recall [06 — Latency, Throughput, Percentiles]): an average of 50ms can hide a p99 of 4 seconds that's ruining 1% of users.
- **Stress testing** — push *past* peak to find the breaking point. Where does it fall over, and does it fail cleanly (shed load, return 503) or catastrophically (cascade, corrupt data)?
- **Soak testing** — sustained load for **hours or days**. This is the only way to catch **slow leaks**: a memory leak that OOM-kills the process at hour 30, or a **connection leak** (recall [65 — Connection Pooling]) where you forget to release DB connections and exhaust the pool overnight.

```js
// A k6-style load test (conceptual). k6 scripts ARE JavaScript.
import http from "k6/http";
import { check } from "k6";

export const options = {
  stages: [
    { duration: "2m", target: 200 },   // ramp up to 200 virtual users
    { duration: "5m", target: 200 },   // hold (this is your load test)
    { duration: "2m", target: 1000 },  // spike (this is your stress test)
  ],
  thresholds: { http_req_duration: ["p(99)<300"] }, // FAIL build if p99 ≥ 300ms
};

export default function () {
  const res = http.get("https://api.example.com/checkout/quote");
  check(res, { "status 200": (r) => r.status === 200 });
}
```

Other tools you'll hear named: **Locust** (Python, define load as code) and **JMeter** (older, GUI-driven, XML). Test at **realistic scale** — 10 users on your laptop proves nothing about 10,000 users hitting a shared connection pool.

### 8. Testing specific distributed hazards

These are the bugs that *only* exist in distributed systems. Each needs a targeted test.

**Idempotency** — send the same request twice, assert the effect happens **once** (recall [85 — Idempotency](./85-idempotency.md)).

```js
test("charging with the same idempotency key charges once", async () => {
  const key = "idem-abc-123";
  const first  = await payments.charge({ amount: 4200, idempotencyKey: key });
  const second = await payments.charge({ amount: 4200, idempotencyKey: key }); // retry!
  expect(second.chargeId).toBe(first.chargeId);        // same charge returned
  expect(await ledger.countCharges(key)).toBe(1);      // money moved ONCE
});
```

**Duplicate delivery & retries** — most message systems are *at-least-once*, so your consumer *will* see duplicates. Test that processing the same message twice is safe.

**Ordering** — deliver messages out of order and assert the final state is still correct (or that you correctly reject stale ones via a version/sequence number).

**Race conditions** — fire concurrent requests at the *same resource* and assert only one wins (recall [52 — Concurrency & Locks]). The classic: two people booking the last seat.

```js
test("two concurrent bookings of seat 14A — exactly one succeeds", async () => {
  const results = await Promise.allSettled([
    booking.reserve({ seat: "14A", user: "alice" }),
    booking.reserve({ seat: "14A", user: "bob" }),   // fired at the SAME time
  ]);
  const ok = results.filter(r => r.status === "fulfilled");
  expect(ok.length).toBe(1);                          // never 2, never 0
});
```

### 9. Deterministic testing of nondeterminism

Races and timeouts are non-deterministic — which makes them nearly impossible to test if you leave timing to chance. The trick: **make the nondeterministic thing injectable** so a test controls it exactly.

- **Injectable clocks.** Never call `Date.now()` directly; take a `clock` dependency. Then a test can jump time forward to fire a timeout *instantly and reliably* (recall the injectable-clock lesson from the scheduler and rate-limiter LLDs).
- **Seeded randomness.** Pass a seeded RNG so "random" jitter/backoff is reproducible run to run.
- **Controllable scheduler.** Control *which* concurrent operation runs first, so you can reproduce a specific interleaving instead of hoping the race shows up.

```js
class FakeClock {
  constructor() { this.now = 0; }
  time() { return this.now; }
  advance(ms) { this.now += ms; }   // test moves time by hand
}

test("token bucket refills after exactly 1 second", () => {
  const clock = new FakeClock();
  const bucket = new TokenBucket({ capacity: 5, refillPerSec: 5, clock });
  for (let i = 0; i < 5; i++) bucket.take();     // drain it
  expect(bucket.take()).toBe(false);             // empty — denied

  clock.advance(1000);                           // jump 1s WITHOUT waiting
  expect(bucket.take()).toBe(true);              // refilled — deterministic
});
```

No `sleep(1000)`, no flakiness. The test runs in microseconds and gives the same answer every single time.

### 10. Observability as a testing tool

You cannot test what you cannot observe (recall [80 — Monitoring & Observability](./80-monitoring-and-observability.md)). Good **metrics, traces, and logs** are what let you *assert on behavior* in staging and production — they turn "I think it worked" into "the trace shows the request hit all 5 services and the p99 metric held."

This unlocks **testing in production**, which for distributed systems isn't reckless — it's necessary, because some conditions (real traffic shapes, real data, real scale) *only* exist in prod. You do it with **guardrails**:

- **Feature flags** — ship the code dark, flip it on for internal users first, roll back instantly by flipping the flag (no redeploy).
- **Canary releases** — send 1% of traffic to the new version; compare its error rate and latency metrics against the old version; promote only if healthy.
- **Shadow traffic (dark launch)** — duplicate real production requests to the new service *without using its responses*. It gets real load and real data, but a bug can't hurt a user.

---

## Visual / Diagram description

**The adapted pyramid with its two axes** — correctness (vertical) and failure (the whole right side):

```
   CORRECTNESS AXIS                         FAILURE AXIS
   (does it work?)                          (does it fail well?)

        ┌─────────┐                    ┌────────────────────────┐
        │   E2E   │  few, critical     │  Chaos engineering     │
        ├─────────┤  paths only        │  (fault injection in   │
        │Contract │◀── the microservice│   PRODUCTION, guarded) │
        ├─────────┤    superpower      └───────────┬────────────┘
        │Integrat.│  real DB/queue                 │ verifies
        ├─────────┤                    ┌───────────▼────────────┐
        │  Unit   │  thousands, fast   │ Load · Stress · Soak   │
        └─────────┘                    │ (meets NFRs at scale?) │
                                       └────────────────────────┘
```

**The fault-injection / chaos loop** — the cycle you run over and over:

```
   ┌────────────────────────────────────────────────────────┐
   │                                                        │
   ▼                                                        │
┌─────────────────────┐   1. state the healthy signal      │
│ STEADY-STATE        │      "checkout success > 99%"       │
│ HYPOTHESIS          │                                     │
└──────────┬──────────┘                                     │
           │ 2. inject a fault                              │
           ▼    (kill node / add latency / drop packets)    │
┌─────────────────────┐                                     │
│ INJECT FAULT        │──── blast radius limited: 1 region, │
│ (blast radius small)│      1% traffic, auto-abort armed   │
└──────────┬──────────┘                                     │
           │ 3. observe via metrics/traces (recall 80)      │
           ▼                                                 │
┌─────────────────────┐   held?  → resilience confirmed ────┘
│ OBSERVE STEADY STATE │
└──────────┬───────────┘   broke? → you found a real weakness
           │                          → fix it, then loop
           ▼
     LEARN & HARDEN
```

Read it as: you never inject a fault without first defining what "healthy" means and how you'll observe it. The loop's output is either a confirmation ("our circuit breaker really does trip") or a defect found in a controlled experiment instead of a 3 a.m. outage.

---

## Real world examples

### Netflix — Chaos Monkey and the Simian Army

Netflix pioneered chaos engineering. **Chaos Monkey** randomly terminates production instances during working hours; the broader **Simian Army** added tools that inject latency (Latency Monkey) and even simulate an entire AWS region going dark (Chaos Kong). The philosophy: if failure is *constant and expected*, engineers have no choice but to build services that tolerate it. This is representative of how a large streaming platform continuously verifies resilience rather than hoping.

### Amazon / AWS — Fault Injection Service and game days

AWS offers **Fault Injection Service (FIS)**, a managed way to inject faults (kill instances, throttle APIs, add latency) into your own workloads as controlled experiments. Internally, Amazon is known for **"game days"** — scheduled exercises where teams deliberately break part of a system to rehearse the human and automated response. Conceptually, this is chaos engineering plus incident-response practice.

### Financial and payments systems — contract tests and idempotency

Payment providers (representatively, companies like Stripe or PayPal-scale platforms) lean heavily on **idempotency keys** so that a network retry never double-charges — and they test that guarantee explicitly. Across microservice estates in banking and fintech, **consumer-driven contract testing (Pact)** is widely used so that independent teams can evolve APIs without a shared integration bottleneck. These are conceptual/representative descriptions of common industry practice.

---

## Trade-offs

**Where to invest across the pyramid:**

| Layer | Pros | Cons |
|-------|------|------|
| Unit | Fast, deterministic, cheap, precise failure location | Can't catch integration or failure-mode bugs |
| Integration | Catches real I/O, serialization, config bugs | Slower; needs Docker/containers |
| Contract | Independent deploys; catches API drift at build time | Setup cost; both teams must adopt it |
| E2E | Highest confidence; real user journey | Slow, flaky, expensive; vague failures |

**Correctness vs. failure testing:**

| Approach | Buys you | Costs you |
|----------|----------|-----------|
| Correctness tests only | Confidence it works when all is well | Zero confidence under partition/latency/death |
| + Fault injection | Proof the fallbacks/breakers actually fire | Extra test scaffolding |
| + Chaos in prod | Proof resilience holds under *real* conditions | Requires maturity, guardrails, buy-in |
| Testing in production | Real traffic, real data, real scale | Needs flags/canaries/observability first |

**Rule of thumb:** put the *bulk* of your test count in unit and contract tests (cheap, catch the most common bugs — logic and API drift), reserve E2E for a few critical journeys, and add fault injection for every resilience mechanism you claim to have. Only run chaos in production once you have observability and guardrails in place. **If you built a circuit breaker or a fallback and never fault-tested it, assume it doesn't work.**

---

## Common interview questions on this topic

### Q1: "Why can't unit tests catch distributed-systems bugs?"

**Hint:** Unit tests are deterministic and in-process — they mock away the network, the clock, and other services. The bugs that matter in distributed systems live *precisely* in what you mocked away: partial failure, latency, duplicate delivery, partitions, races. You need integration, contract, and fault-injection tests to reach them.

### Q2: "Two of my services are owned by different teams. How do I stop one from breaking the other without a giant shared integration environment?"

**Hint:** Consumer-driven contract testing (Pact). The consumer records exactly what it needs; that contract is replayed against the real provider in the provider's own build. If the provider breaks the expectation, *their* build goes red at build time. This gives independent deployability without an E2E bottleneck.

### Q3: "What's the difference between fault injection and chaos engineering?"

**Hint:** Fault injection is the *technique* (deliberately break a dependency and observe). Chaos engineering is running that technique **in production, continuously**, as ongoing verification — with a steady-state hypothesis, a limited blast radius, and an abort switch. Chaos engineering is fault injection with production stakes and scientific discipline.

### Q4: "How would you test that a retry doesn't double-charge a customer?"

**Hint:** Idempotency test (recall 85). Send the same request twice with the same idempotency key; assert the second call returns the *same* charge id and that exactly one entry landed in the ledger. Combine with fault injection: force a timeout on the first attempt so the retry path is genuinely exercised.

### Q5: "A race condition only shows up 1 in 1000 runs. How do you get it into a reliable test?"

**Hint:** Make the nondeterminism injectable. Use an injectable clock and a controllable scheduler so you can force the exact interleaving, or fire the concurrent operations with `Promise.all` against the same resource and assert exactly one wins. Seeded randomness makes any jitter/backoff reproducible. Deterministic control turns a flaky 1-in-1000 into a test that fails the same way every time.

---

## Practice exercise

### Build and fault-test a resilient client (~30–40 min)

Write a small Node.js module `ResilientPriceClient` that fetches a price from a "pricing service," and then *prove it fails well*. Produce:

1. **The client** — it wraps a dependency (injected, not `new`-ed) and has:
   - a **circuit breaker** that opens after 3 consecutive failures (recall 73),
   - a **fallback** to a last-known cached price when the breaker is open,
   - an **injectable clock** so timeouts are testable.
2. **A `FlakyDependency` fake** with modes `ok`, `down`, `slow` (5s), and `errors` (503) that you can switch at runtime.
3. **Four tests** (plain `console.assert` is fine):
   - happy path returns the live price,
   - after 3 `down` calls the breaker is `OPEN` and the next call returns the *cached* price with `source: "cache"`,
   - a `slow` call is aborted by your timeout instead of hanging,
   - firing two `charge` calls with the same idempotency key results in one effect.
4. **One paragraph**: which of these tests a unit test could *not* have caught on its own, and why.

Success = every assertion passes, and no test contains a real `sleep` (use the injectable clock / abort instead).

---

## Quick reference cheat sheet

- **Testing pyramid** — many cheap unit tests at the bottom, few slow E2E tests at the top; cost rises and determinism falls as you go up.
- **Unit test** — one component, all dependencies faked; needs dependency injection (recall 18) to be possible.
- **Integration test** — real DB/queue (e.g. Testcontainers); catches serialization, SQL, and config bugs mocks hide.
- **Contract test (Pact)** — the microservices superpower: catches "provider broke the consumer" at build time and enables independent deploys.
- **Consumer-driven** — the *consumer* writes the contract; the *provider* must keep satisfying it.
- **E2E test** — full journey, highest confidence, slowest and flakiest; reserve for critical paths only.
- **Fault injection** — deliberately break a dependency; assert graceful degradation (breaker trips, fallback fires).
- **Slow is worse than down** — always fault-test *latency*, not just outright failure; slow deps exhaust pools.
- **Chaos engineering** — fault injection in production: steady-state hypothesis → inject → observe → limit blast radius.
- **Load / stress / soak** — meet NFRs at expected load, find the breaking point past peak, catch leaks over days. Watch **p99, not average**.
- **Distributed hazards** — test idempotency (recall 85), duplicate delivery, ordering, and races (recall 52) explicitly.
- **Deterministic nondeterminism** — injectable clocks, seeded RNG, controllable schedulers turn flaky races into reliable tests.
- **Observability = testability** — you can't test what you can't observe (recall 80).
- **Testing in production** — legitimate and necessary, with guardrails: feature flags, canaries, shadow traffic.

---

## Connected topics

| Direction | Topic | Why |
|-----------|-------|-----|
| **Previous** | [73 — Circuit Breaker](./73-circuit-breaker.md) | The exact resilience mechanism you fault-test: does the breaker actually trip when a dependency dies? |
| **Next** | [90 — Disaster Recovery](./90-disaster-recovery.md) | Chaos engineering is how you verify your DR plan is real instead of a document nobody has ever exercised. |
| **Related** | [80 — Monitoring & Observability](./80-monitoring-and-observability.md) | You can't test what you can't observe; metrics and traces are how you assert behavior in staging and prod. |
| **Related** | [71 — Microservices vs Monolith](./71-microservices-vs-monolith.md) | Contract testing is what lets independent microservice teams deploy without a shared integration bottleneck. |
| **Related** | [85 — Idempotency](./85-idempotency.md) | The canonical distributed hazard you test: send the same request twice, assert the effect happens exactly once. |
