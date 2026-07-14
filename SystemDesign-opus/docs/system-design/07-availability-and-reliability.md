# 07 — Availability and Reliability — Uptime Guarantees

## Category: HLD Fundamentals

---

## What is this?

**Availability** is the fraction of time your system is up and answering requests. It's usually written as a percentage with an embarrassing number of nines: 99.9%, 99.99%, 99.999%.

Think of a corner shop. **Availability** = "what fraction of the hours it *claims* to be open is the door actually unlocked?" **Reliability** = "when I walk in, does it actually have the milk, and is the milk not expired?" **Durability** = "if the shop burns down, does my stuff in their storage locker survive?" A shop can be open (available) and still hand you spoiled milk (unreliable). These are three different promises, and confusing them is one of the fastest ways to look junior in an interview.

---

## Why does it matter?

Every extra nine costs roughly **10x more money and complexity**, and buys you **10x less downtime**. If you can't do this arithmetic, you will either burn a year of engineering time chasing a nine nobody asked for, or you'll promise a customer 99.99% in a contract and discover — after signing — that your architecture physically cannot deliver it.

**What breaks without this knowledge:** you chain 5 services in series, each with a lovely 99.9% availability, and confidently promise 99.9% overall — the real number is **99.5%**, which is 3.6 hours of downtime a month instead of 43 minutes, and you just missed your SLA by 5x. Or you have a "99.99% SLA" and no idea whether you're meeting it, because you never defined *what counts as down*. Or your team blocks every deploy "for stability," ships nothing for a quarter, and still has an outage — because you were spending an error budget you never measured.

**Interview angle:** "Design a system with 99.99% availability" is a real prompt. You must be able to (a) say what 99.99% means in minutes, (b) do the series/parallel arithmetic, (c) name the redundancy that gets you there, and (d) explain why you'd push back on 99.999%.

**Real-work angle:** This is the language of contracts, on-call, and post-mortems. SLA breaches cost real money (service credits). Error budgets are how mature teams decide whether to ship features or fix reliability — it turns "should we deploy on Friday?" from a vibe into a number.

---

## The core idea — explained simply

### The Airline Analogy

You run an airline.

**Availability** is: *what fraction of scheduled flights actually take off?* If you scheduled 1,000 flights this month and 999 took off, you're at 99.9% availability. Notice this says nothing about whether the flights were any good.

**Reliability** is: *does the plane consistently do its job correctly?* A flight that took off, flew to the wrong city, and lost your luggage was **available** but not **reliable**. In software: your API returned HTTP 200 (available!) with corrupted JSON (unreliable).

**Durability** is: *if the plane crashes, is your checked luggage still findable?* It's about **not losing data, ever** — even through catastrophe. S3 famously advertises "eleven nines" of durability (99.999999999%), which means if you store 10 million objects you'd expect to lose one every 10,000 years. But S3's *availability* SLA is only 99.9% — the objects are safe, but you might not be able to reach them for a few minutes.

**Now the redundancy insight.** How does the airline get more available? **One plane, one route:** if the plane breaks, the route is dead. **Two planes, one on standby:** the route only dies if *both* break — if each plane is 95% reliable, the chance both are broken is `0.05 × 0.05 = 0.25%`, so the route is **99.75%** available. You turned a mediocre component into a great system just by having a spare. **But a *connecting* flight is the opposite:** you need flight A *and* flight B to work, so at 99% each your journey is `0.99 × 0.99 = 98.01%` available. **Chaining makes things worse. Duplicating makes things better.** That's the whole game.

### Mapping the analogy back

| Airline | System | The rule |
|---|---|---|
| Flight takes off | Service responds | Availability |
| Flight lands at the right city, with your bags | Service returns *correct* results | Reliability |
| Your checked bag survives a crash | Your data survives disk/DC failure | Durability |
| One plane on the route | Single instance | Single point of failure (SPOF) |
| A spare plane on standby | Redundant replica | **Parallel** → multiply the *failure* rates |
| A connecting flight (A then B) | Service A calls service B | **Series** → multiply the *availability* rates |
| The contract promising a refund if delayed | **SLA** (legal) | Has teeth (money) |
| The internal target: "95% of flights on time" | **SLO** (internal target) | Stricter than the SLA |
| The measured on-time % this month | **SLI** (the measurement) | The raw number |
| "We may run 5% late; we've used 3%" | **Error budget** | Ship features, or freeze? |

---

## Key concepts inside this topic

### 1. The nines table — memorize this

Availability is `uptime / (uptime + downtime)`, expressed as a percentage. Here is what each level actually *costs you in downtime*:

| Availability | Nickname | Downtime per **year** | per **month** (30d) | per **week** | per **day** |
|---|---|---|---|---|---|
| **90%** | "one nine" | 36.5 days | 72 hours | 16.8 hours | 2.4 hours |
| **99%** | "two nines" | 3.65 days | 7.2 hours | 1.68 hours | 14.4 minutes |
| **99.9%** | "three nines" | 8.77 hours | 43.8 minutes | 10.1 minutes | 1.44 minutes |
| **99.95%** | — | 4.38 hours | 21.9 minutes | 5.04 minutes | 43 seconds |
| **99.99%** | "four nines" | 52.6 minutes | 4.38 minutes | 1.01 minutes | 8.6 seconds |
| **99.999%** | "five nines" | 5.26 minutes | 26.3 seconds | 6.05 seconds | 0.86 seconds |
| **99.9999%** | "six nines" | 31.5 seconds | 2.63 seconds | 0.6 seconds | 0.086 seconds |

**How to derive any row on a whiteboard, in your head:**

```
A year is ~365 × 24 × 60 = 525,600 minutes.  (Memorize: ~525,600. Half a million.)

99%     → 1% down    → 525,600 × 0.01    = 5,256 min  = 3.65 days
99.9%   → 0.1% down  → 525,600 × 0.001   = 525.6 min  = 8.76 hours
99.99%  → 0.01% down → 525,600 × 0.0001  = 52.56 min  ≈ 1 hour
99.999% → 0.001%     → 525,600 × 0.00001 = 5.26 min
```

**Feel the numbers.** **99.9% is not very good** — 43 minutes a month is one bad deploy, and most internal tools live here. **99.99% (52 min/year) means you cannot fix things by hand:** a human paged at 3am takes 15 minutes just to open a laptop, so four nines *requires automated failover*. **99.999% (5 min/year) means a single machine reboot blows your entire annual budget** — you cannot achieve it with a human in the loop at all. That's telecom/payments territory, and it costs a fortune.

### 2. Availability vs Reliability vs Durability — the three-way distinction

This trips up almost everyone. Get it crisp.

| | Question it answers | Failure looks like | Typical metric |
|---|---|---|---|
| **Availability** | "Is it up right now?" | HTTP 503, connection refused, timeout | % uptime (99.9%) |
| **Reliability** | "Does it do the right thing, consistently, over time?" | HTTP 200 with wrong/corrupt data; a job that silently drops 3% of records | MTBF, error rate, "successful requests / total requests" |
| **Durability** | "Will my data still exist next year?" | Data loss — the write was acknowledged, then vanished | % durability (99.999999999%) |

**Three worked scenarios to lock it in.** **Available but unreliable:** your API is up 100% of the time and returns 200 OK on every call — but 5% of the time it serves a stale price because a cache never invalidated, and the user gets charged the wrong amount. **Reliable but unavailable:** your service is flawlessly correct whenever you can reach it, but a bad BGP route makes it unreachable for 3 hours a month. **Available and reliable but not durable:** your Redis-only session store answers every request correctly and instantly — then the node reboots and every session on earth is gone. 100% availability, 100% reliability, **0% durability.**

> **Availability = can I reach it? Reliability = can I trust it? Durability = will it still be there tomorrow?**

### 3. SLA vs SLO vs SLI — with real examples

These three are nested, from the outside in.

```
┌──────────────────────────────────────────────────────────────────────────┐
│  SLA — Service Level AGREEMENT   (the legal contract, with penalties)     │
│  "99.9% monthly uptime, or you get a 10% service credit."                 │
│                                                                          │
│   ┌────────────────────────────────────────────────────────────────┐     │
│   │  SLO — Service Level OBJECTIVE  (the internal target, stricter) │     │
│   │  "99.95% of requests succeed, measured over 30 days."           │     │
│   │   We aim HIGHER than the SLA so we never actually breach it.    │     │
│   │                                                                │     │
│   │    ┌──────────────────────────────────────────────────┐        │     │
│   │    │  SLI — Service Level INDICATOR  (the measurement) │        │     │
│   │    │  "count(status < 500) / count(all) over 5 min"    │        │     │
│   │    │  Right now: 99.973%                              │        │     │
│   │    └──────────────────────────────────────────────────┘        │     │
│   └────────────────────────────────────────────────────────────────┘     │
└──────────────────────────────────────────────────────────────────────────┘
```

- **SLI (Indicator)** — the raw number you actually measure. It must be a *ratio of good events to total events*. `successful_requests / total_requests`. Or `requests_faster_than_300ms / total_requests`. It's a fact.
- **SLO (Objective)** — the target you hold that SLI to, internally. "SLI ≥ 99.95% over a rolling 30 days." No lawyers involved. This is the number the on-call rotation defends.
- **SLA (Agreement)** — the promise you make to a paying customer, **with financial consequences** for breaking it. "99.9%, or we refund 10% of your monthly bill."

**Golden rule: SLO must be stricter than SLA.** If you promise customers 99.9% and target 99.9% internally, you will breach constantly, because your target has zero margin. Target 99.95% internally so that by the time you're in trouble, you still have room before the lawyers get involved.

**Real examples (as publicly documented — always check current terms):** **AWS EC2** offers a 99.99% monthly uptime SLA per Region, with service credits scaling from 10% to 100% of the bill as uptime drops. **AWS S3 (Standard)** advertises **99.999999999% durability** but a much more modest **99.9% availability** SLA — an asymmetry that is deliberate: losing your data is unforgivable, being briefly unreachable is survivable. Google Cloud, Azure, and most SaaS publish similar tiered-credit SLAs.

**The dirty secret of SLAs:** a service credit is a *refund of a fraction of what you paid*, not compensation for what the outage cost your business. If a 4-hour outage costs you $2M in lost sales and your cloud bill was $10k, you get a $1k credit. **SLAs are not insurance** — they're a signal of how seriously the vendor takes uptime.

**Choosing a good SLI:** your availability SLI is usually built from errors — `1 - (5xx responses / total responses)`. Crucially, **measure it where the user is** (at the load balancer or client), not inside your app: an app that has crashed cannot report its own downtime.

### 4. The arithmetic — series (multiply) vs parallel (redundancy)

This is the part interviewers actually test.

#### Components in SERIES — availability MULTIPLIES (it always gets worse)

If a request must pass through A **and** B **and** C, all three must be up.

```
  [ Client ] ──▶ [ LB: 99.99% ] ──▶ [ App: 99.9% ] ──▶ [ DB: 99.95% ] ──▶ ✓

  A_total = 0.9999 × 0.999 × 0.9995 = 0.99840 ≈ 99.84%
```

**99.84% is worse than every single component in the chain.** That's 14 hours of downtime a year — even though your worst component was 99.9% (8.8 hrs).

**Work a bigger one.** A microservice request path with 5 hops, each at a respectable 99.9%:

```
  A_total = 0.999^5 = 0.99500 ≈ 99.5%

  WHITEBOARD SHORTCUT — for small failure rates, failures just ADD:
    each service fails 0.1% of the time
    5 in series → total failure ≈ 5 × 0.1% = 0.5%
    → availability ≈ 100% - 0.5% = 99.5%      ✓ matches the exact answer
```

99.5% = **43.8 hours of downtime a year**, up from 8.8. **You made your system 5x less available by splitting it into microservices.** This is the hidden tax of a distributed architecture, and it's why microservice architectures need circuit breakers (topic 73), retries, and graceful degradation — not because it's fancy, but because the multiplication is brutal.

#### Components in PARALLEL — failure rates MULTIPLY (it always gets better)

If you have redundant replicas and the request succeeds if **any one** is up:

```
  A_total = 1 - (failure_A × failure_B) = 1 - (1 - A) × (1 - B)

  TWO app servers, each 99% available:
    failure of one = 1 - 0.99 = 0.01
    both fail      = 0.01 × 0.01 = 0.0001
    A_total        = 1 - 0.0001 = 99.99%       ← two nines became FOUR nines

  THREE app servers, each 99% available:
    all three fail = 0.01³ = 0.000001
    A_total        = 1 - 0.000001 = 99.9999%   ← six nines!
```

That looks like magic. **It is not, and here's the catch that separates a good answer from a great one:** this math assumes the failures are **independent**. In reality they usually aren't — all three servers run the same buggy code you just deployed (all three die together); all three sit in the same rack on the same power strip (one PDU failure kills all three); all three depend on the same database (a shared single point of failure that no amount of app redundancy helps); or server 1 dies and dumps its traffic onto servers 2 and 3, which were already at 70%, so they fall over too — **correlated failure by cascade.**

**So the practical formula in an interview is:** parallel redundancy multiplies failure rates *for independent failures only*, and your job as a designer is to **make failures independent** — different availability zones, different racks, staggered/canary deploys, no shared dependency, and enough headroom that surviving nodes can absorb the dead one's traffic.

#### A full worked example — the whole system

```
                        ┌──── App-1 (99.9%) ────┐
 Client ─▶ LB (99.99%) ─┤                       ├─▶ DB primary (99.9%)
                        └──── App-2 (99.9%) ────┘    + replica  (99.9%)
                          (parallel/redundant)         (parallel/failover)

  Step 1 — collapse the parallel app tier:
     A_app   = 1 - (0.001 × 0.001) = 99.9999%
  Step 2 — collapse the parallel DB tier:
     A_db    = 1 - (0.001 × 0.001) = 99.9999%
  Step 3 — chain the tiers in SERIES (LB → app → DB):
     A_total = 0.9999 × 0.999999 × 0.999999
             = 0.99988 ≈ 99.988%   →  ~63 minutes of downtime/year
```

Notice what happened: **the load balancer is now the bottleneck.** It's the only non-redundant component, so it dominates. That's the real lesson of this arithmetic — *the math finds your single point of failure for you.* Add a second LB (or use a managed, multi-AZ one) and your ceiling jumps again.

Here's that computation as code you can actually run:

```js
/**
 * Series   = every component must be up → multiply AVAILABILITIES.
 * Parallel = at least one must be up    → multiply FAILURE rates, subtract from 1.
 */
const series = (...as) => as.reduce((acc, a) => acc * a, 1);
const parallel = (...as) => 1 - as.reduce((acc, a) => acc * (1 - a), 1);
const fmt = (a) => `${(a * 100).toFixed(4)}% → ${((1 - a) * 525_600).toFixed(1)} min/year`;

// Naive single-instance stack: everything in series, nothing redundant.
console.log(fmt(series(0.9999, 0.999, 0.999)));
// 99.7902% → 1102.9 min/year   (~18 hours!)

// Same components, but the app and DB tiers are now redundant pairs.
console.log(fmt(series(0.9999, parallel(0.999, 0.999), parallel(0.999, 0.999))));
// 99.9880% → 63.1 min/year     ← the LB is now the ceiling

// The microservice tax: 5 hops in series, each a "good" 99.9%.
console.log(fmt(series(...Array(5).fill(0.999))));
// 99.5010% → 2622.4 min/year   (~44 hours!)
```

### 5. MTBF and MTTR — where availability actually comes from

Availability isn't a magic property. It's produced by two things: **how often you break** and **how fast you recover**.

- **MTBF — Mean Time Between Failures.** Average uptime between incidents. "Our checkout service fails once every 30 days" → MTBF = 720 hours.
- **MTTR — Mean Time To Recovery** (sometimes "Repair"). Average time from failure to restored service. "It takes us 2 hours to notice, diagnose, and fix" → MTTR = 2 hours.

```
                   MTBF
   Availability = ───────────────
                  MTBF + MTTR
```

**Worked example:**
```
  MTBF = 720 hours (fails monthly), MTTR = 2 hours
  A = 720 / (720 + 2) = 720 / 722 = 0.99723 = 99.72%   → ~24 hours down/year
```

Now, two ways to improve it. Which is easier?

```
  Option A — halve the failure rate (MTBF → 1440h).
    Very hard: better testing, better code, better hardware. MONTHS of work.
    A = 1440 / (1440 + 2)     = 99.861%   → ~12 hours down/year

  Option B — cut recovery from 2 hours to 5 minutes (MTTR → 0.083h).
    Achievable: alerting + automated rollback + a runbook. WEEKS of work.
    A = 720 / (720 + 0.083)   = 99.988%   → ~1 hour down/year
```

**Option B wins, by a lot, for less effort.** This is one of the most important operational insights in the field:

> **You cannot prevent failure. You can absolutely make recovery fast.**
> **Optimizing MTTR beats optimizing MTBF almost every time.**

This is why mature teams invest in: fast rollbacks, feature flags (turn it off in 5 seconds, no deploy), health checks and automated failover, good alerting (MTTD — time to *detect* — is often the biggest chunk of MTTR), and practiced runbooks. It's also the philosophical foundation of chaos engineering (topic 90): if you deliberately break things in production, you *train* your MTTR down.

Related metrics you'll hear: **MTTD** (detect), **MTTA** (acknowledge), **MTTF** (mean time to failure — for things that don't get repaired, like a disk).

### 6. Error budgets — how to stop the "ship vs. stability" war

Here's the elegant idea that makes SLOs actually *useful* instead of just a number on a dashboard.

If your SLO is **99.9%**, then you have explicitly declared that **0.1% unavailability is acceptable**. That 0.1% is not a failure — it's a **budget you are allowed to spend.**

```
  Error budget = 100% - SLO

  SLO 99.9% over 30 days:
    Total minutes in 30 days = 30 × 24 × 60 = 43,200
    Budget = 0.1% × 43,200 = 43.2 minutes of downtime this month
```

Now the budget becomes a **decision rule that both teams agree to in advance:**

| Budget remaining | What the team does |
|---|---|
| **Plenty left** (35 of 43 min) | **Ship aggressively.** Deploy often, take risks. Unused budget is *wasted opportunity* — you were too conservative. |
| **Nearly exhausted** (5 of 43 min) | **Slow down.** Only low-risk changes, more canarying, more review. |
| **Exhausted / negative** | **Feature freeze.** All effort goes to reliability until the budget resets. No exceptions, no debate. |

Why this is brilliant: it **ends the political argument.** The PM wants to ship; the SRE wants stability. Instead of arguing about vibes, both sides agreed *up front* on a number, and the number decides. It also — counterintuitively — **encourages risk-taking** when the budget is healthy: if you're at 99.99% against a 99.9% SLO, you over-delivered, which means you spent engineering effort on reliability customers never asked for.

```js
/** Error budget tracker: feed it SLI events, it tells you to ship or freeze. */
class ErrorBudget {
  constructor({ sloTarget }) {
    this.sloTarget = sloTarget; // e.g. 0.999
    this.total = 0;
    this.failed = 0;
  }

  record({ ok }) {
    this.total++;
    if (!ok) this.failed++;
  }

  /** How many bad requests we are ALLOWED in this window. */
  get allowedFailures() {
    return this.total * (1 - this.sloTarget);
  }

  /** Fraction of budget unspent. Negative = we've breached the SLO. */
  get remaining() {
    return this.allowedFailures === 0 ? 1 : 1 - this.failed / this.allowedFailures;
  }

  /** The policy, encoded — the number makes the decision, not the loudest voice. */
  get deployPolicy() {
    if (this.remaining <= 0) return 'FREEZE: budget exhausted. Reliability work only.';
    if (this.remaining < 0.25) return 'CAUTION: <25% left. Low-risk, canaried changes only.';
    return 'SHIP: healthy budget. Deploy freely.';
  }
}

const budget = new ErrorBudget({ sloTarget: 0.999 }); // 99.9%
// A month of traffic: 10,000,000 requests, of which 6,000 failed.
for (let i = 0; i < 10_000_000; i++) budget.record({ ok: i >= 6_000 });

console.log(budget.allowedFailures); // 10000  ← we were allowed 10k failures
console.log(budget.failed); // 6000   ← we used 6k
console.log((budget.remaining * 100).toFixed(1) + '%'); // 40.0%
console.log(budget.deployPolicy); // SHIP: healthy budget. Deploy freely.
```

### 7. Why 100% is never the goal

Say it out loud: **the correct availability target is never 100%.** Three reasons.

**1. It is physically impossible.** Your users reach you through ISPs, DNS resolvers, undersea cables, and home wifi — none of which you own. If their wifi drops, your 100% is irrelevant *to them*. User-perceived availability is capped by the weakest link in a chain you don't control.

**2. The cost curve is exponential; the value curve is flat.** Each nine costs roughly 10x more:

| Target | What it takes | Rough cost |
|---|---|---|
| 99% | One server, someone notices when it's down | $ |
| 99.9% | Redundant servers, monitoring, an on-call rotation | $$ |
| 99.99% | Multi-AZ, automated failover, **no manual steps in recovery** | $$$$ |
| 99.999% | Multi-region active-active, chaos engineering, a large SRE org | $$$$$$$$ |

Does your user *notice* the difference between 52 minutes and 5 minutes of downtime a year? Almost certainly not — their phone loses signal for longer on the train.

**3. A 100% target kills velocity.** If any downtime is a failure, the only rational move is to never change anything. But you have to ship to have a business. The error budget exists precisely to give you **permission to change things.**

**How to pick a target:** work backwards from the business. What does a minute of downtime cost? What do competitors offer? What do customers contractually demand? Pick the *lowest* number that satisfies those, and spend the savings on features. "As available as it needs to be, and not one nine more."

---

## Visual / Diagram description

### Diagram 1: Series vs Parallel — the two arithmetics

```
 ══════════════════ SERIES: all must work → MULTIPLY AVAILABILITIES ══════════════════

  ┌────────┐     ┌────────┐     ┌────────┐     ┌────────┐
  │  Auth  │────▶│  API   │────▶│ Order  │────▶│   DB   │
  │ 99.9%  │     │ 99.9%  │     │ 99.9%  │     │ 99.9%  │
  └────────┘     └────────┘     └────────┘     └────────┘

     A = 0.999 × 0.999 × 0.999 × 0.999 = 0.9960 = 99.60%
                                                   ▲
         ✗ WORSE than every single component. 35 hours down/year.
         Shortcut: failure rates ADD → 4 × 0.1% = 0.4% down → 99.6%


 ═════════════ PARALLEL: any one works → MULTIPLY FAILURE RATES ═════════════════════

                  ┌────────────────┐
              ┌──▶│  App-1  99.9%  │──┐
  ┌────────┐  │   └────────────────┘  │   ┌────────┐
  │   LB   │──┤                       ├──▶│ Result │
  └────────┘  │   ┌────────────────┐  │   └────────┘
              └──▶│  App-2  99.9%  │──┘
                  └────────────────┘

     Failure of one = 0.001
     Both fail      = 0.001 × 0.001 = 0.000001
     A = 1 - 0.000001 = 99.9999%
                          ▲
         ✓ BETTER than either component. 31 seconds down/year.

     ⚠ ONLY IF FAILURES ARE INDEPENDENT.
       Same rack? Same bad deploy? Same DB? → the math is a lie.
```

### Diagram 2: A 99.99% architecture, and where each nine comes from

```
                         ┌──────────────┐
                         │  Route 53 /  │   DNS failover between regions
                         │     DNS      │   (health-checked, ~60s TTL)
                         └──────┬───────┘
                ┌───────────────┴───────────────┐
    ════════════▼═══════════════   ════════════▼════════════
    ║  REGION us-east-1         ║   ║  REGION eu-west-1     ║
    ║                           ║   ║  (warm standby)       ║
    ║   ┌────────────────────┐  ║   ║   ┌──────────────┐    ║
    ║   │ Load Balancer      │  ║   ║   │ Load Balancer│    ║
    ║   │ (managed, multi-AZ)│  ║   ║   └──────┬───────┘    ║
    ║   └─────────┬──────────┘  ║   ║      ┌───▼───┐        ║
    ║      ┌──────┴──────┐      ║   ║      │  App  │        ║
    ║  ┌───▼───┐    ┌────▼──┐   ║   ║      └───┬───┘        ║
    ║  │ App   │    │ App   │   ║   ║      ┌───▼────┐       ║
    ║  │ AZ-a  │    │ AZ-b  │   ║   ║      │   DB   │       ║
    ║  └───┬───┘    └───┬───┘   ║   ║      │ replica│       ║
    ║      └──────┬─────┘       ║   ║      └───▲────┘       ║
    ║        ┌────▼─────┐       ║   ║          │            ║
    ║        │ DB       │───────╬═══╬══════════┘            ║
    ║        │ primary  │  async replication                ║
    ║        │  (AZ-a)  │       ║   ║  (RPO: seconds of     ║
    ║        └────┬─────┘  sync ║   ║   data may be lost —  ║
    ║        ┌────▼─────┐       ║   ║   see topic 90)       ║
    ║        │ DB       │       ║   ║                       ║
    ║        │ standby  │       ║   ║                       ║
    ║        │  (AZ-b)  │       ║   ║                       ║
    ║        └──────────┘       ║   ║                       ║
    ════════════════════════════   ═════════════════════════

  WHERE THE NINES COME FROM:
   ├─ App tier redundant across 2 AZs     → survives one AZ dying
   ├─ DB has a SYNC standby in another AZ → auto failover, ~30s, zero data loss
   ├─ Managed multi-AZ load balancer      → the LB is not a SPOF
   ├─ Cross-region async replica          → survives an entire region dying
   └─ MTTR is the real lever: failover is AUTOMATED, not a human at 3am
```

Redraw this on a whiteboard and you can answer "how would you build a 99.99% system?" — the answer is always: *remove every single point of failure, then automate recovery so MTTR is measured in seconds.*

---

## Real world examples

### 1. Amazon S3 — durability and availability are separate products

S3 Standard advertises **99.999999999% (11 nines) durability** and a **99.9% availability SLA**. That gap is the entire lesson of this topic in one product. Durability comes from writing each object redundantly across multiple devices in multiple Availability Zones, with continuous checksumming and automatic repair of corrupted copies. Availability is a *different* problem — a network partition or a control-plane bug can make objects temporarily unreachable without endangering them at all. AWS also sells storage tiers (e.g. One Zone-IA) that keep the same 11-nines durability *claim within a single AZ* but explicitly accept lower availability and AZ-loss risk in exchange for a lower price — a direct, purchasable trade-off between availability and cost.

### 2. Google SRE — the error budget as company policy

Google's SRE practice, described in their public SRE book, is where the error budget idea was popularized. Product teams and SRE teams agree on an SLO. The gap between 100% and that SLO is the error budget. If the service is *inside* budget, the product team is free to launch. If the budget is exhausted, launches stop and engineering effort shifts to reliability until the window rolls over. The key insight they emphasize: this makes the reliability/velocity trade-off **a data-driven decision rather than a political one**, and it explicitly *discourages* over-achieving on reliability, since an unspent budget means you shipped too slowly.

### 3. Netflix — engineering for MTTR, not MTBF

Netflix's Chaos Monkey (and the broader Simian Army) deliberately terminates production instances during business hours. The philosophy is a direct application of `A = MTBF / (MTBF + MTTR)`: you cannot make cloud instances stop failing, so instead **make failure boring**. By constantly injecting failures while engineers are awake and watching, they force the system's recovery paths to be exercised, automated, and fast — driving MTTR toward zero. Their architecture leans hard on redundancy across AZs and regions, aggressive fallbacks (a degraded homepage beats no homepage), and circuit breakers so that a failing dependency doesn't propagate through the series-multiplication of a microservice chain.

---

## Trade-offs

| Availability target | Pros | Cons |
|---|---|---|
| **99% (2 nines)** | Cheap. One server, basic monitoring. Fine for internal tools, batch jobs. | 3.6 days/year down. Unacceptable for anything customer-facing. |
| **99.9% (3 nines)** | Achievable with redundancy + on-call. The pragmatic default for most SaaS. | 8.8 hours/year. A single bad deploy can blow the month's budget. |
| **99.99% (4 nines)** | Serious product-grade uptime. Multi-AZ, automated failover. | 10x cost. **Requires zero manual steps in recovery** — humans are too slow. |
| **99.999% (5 nines)** | Telecom / payments grade. | Astronomically expensive. Multi-region active-active, formal change control, huge SRE org. Usually the wrong answer. |

| Redundancy strategy | Pros | Cons |
|---|---|---|
| **Active-Passive (hot standby)** | Simpler; no split-brain risk; cheaper than active-active | Standby capacity is idle (you pay for nothing); failover takes time (raising MTTR); the failover path is rarely exercised, so it often *doesn't work* when needed |
| **Active-Active** | Zero failover time; all capacity is used; the path is exercised constantly | Much harder: needs conflict resolution, consistency decisions (topic 08/09), and **each region must be able to carry 100% of the load** — so you're still paying for 2x |
| **Multi-AZ (one region)** | Survives datacenter loss; low latency between AZs; relatively easy | Doesn't survive a whole-region outage or a global control-plane bug |
| **Multi-Region** | Survives a region loss; better global latency | Cross-region replication lag → data loss window (RPO); big consistency headaches; expensive |

**The sweet spot:** Target **99.9%** by default and only buy more nines when a specific business reason demands it. Get there with **multi-AZ redundancy** (which turns your failure rates into a product, not a sum) and above all by **crushing MTTR** — one-click rollback, feature flags, automated health-check failover. Then set an error budget and let it, not politics, decide when to ship and when to freeze. **You cannot prevent failure; you can make recovery boring.**

---

## Common interview questions on this topic

### Q1: "What does 99.99% availability actually mean, in minutes?"
**Hint:** ~52 minutes of downtime per year, ~4.4 minutes per month, ~8.6 seconds per day. Derive it live: a year is ~525,600 minutes, and 0.01% of that is 52.56. Then make the point that matters: **at 4 minutes a month, no human can be in the recovery loop.** Four nines is fundamentally a statement about *automation*, not about server quality.

### Q2: "Your system has 5 microservices in a chain, each 99.9% available. What's the overall availability?"
**Hint:** They're in **series**, so availabilities multiply: `0.999^5 ≈ 0.995 = 99.5%` — about 44 hours of downtime a year, 5x worse than any single service. Shortcut for the whiteboard: small failure rates add, so `5 × 0.1% = 0.5% down`. Then show you know the fix: add redundancy inside each tier (parallel → multiply *failure* rates), add circuit breakers and fallbacks (topic 73) so a dead dependency degrades rather than kills, and make non-critical calls async so they're not in the critical path at all.

### Q3: "What's the difference between an SLA, an SLO, and an SLI?"
**Hint:** SLI = the **measurement** (`good_requests / total_requests` = 99.973% right now). SLO = the internal **target** for that SLI (99.95% over 30 days). SLA = the **contract** with the customer, with financial penalties (99.9%, or you get a service credit). The relationship that matters: **SLO must be stricter than SLA**, so that you get an internal alarm long before you breach a contract. Bonus point: your SLI should be measured at the load balancer or client — a crashed app can't report its own downtime.

### Q4: "You have a service at 99.7% and you need to get to 99.99%. Where do you start?"
**Hint:** Don't start with "write better code." Start with `A = MTBF / (MTBF + MTTR)` and figure out *which term is hurting you*. Usually it's MTTR. If you fail once a month (MTBF = 720h) and take 2 hours to recover, you're at 99.72%; cut MTTR to 5 minutes and you're at 99.99% **without fixing a single bug**. So: automated health checks and failover, one-click rollback, feature flags, and alerting that pages in seconds not minutes. Then, separately, remove single points of failure — walk the request path and find every component that isn't redundant (usually it's the database or the load balancer).

### Q5: "You add a second server. Does that double your availability? And should we aim for 100%?"
**Hint:** Adding a server *squares your failure rate*, which is far better than doubling — two 99% servers give `1 - 0.01² = 99.99%`. But immediately raise the caveat that earns the point: **this only holds if the failures are independent.** Same rack, same power supply, same bad deploy, same shared database, or a cascade where server 1's traffic overloads server 2 — any of these makes the real number far worse. And no, never aim for 100%: it's impossible (the user's ISP and wifi are in the chain and you don't own them), each nine costs ~10x more while the perceived benefit flattens to zero, and a 100% target means never changing anything. Use the **error budget** framing: `100% - SLO` is a resource you are *supposed to spend* on shipping.

---

## Practice exercise

### The Availability Audit

You've inherited this architecture for an e-commerce checkout. Here are the published availabilities of each component:

```
  Client ─▶ CDN (99.99%) ─▶ Load Balancer (99.95%) ─▶ API Server (99.9%)
                                                          │
                            ┌─────────────────────────────┼──────────────────────┐
                            ▼                             ▼                      ▼
                   Auth Service (99.9%)        Inventory Service (99.5%)   Payment Gateway
                            │                             │                  (3rd party, 99.9%)
                            ▼                             ▼
                   User DB (99.95%)              Inventory DB (99.9%)
```

Every component in this diagram is a **single instance**, and the checkout flow requires **all** of them.

**Part A — Do the math (15 min).**
1. Compute the end-to-end availability of the checkout flow (all in series — multiply everything), then convert it to **downtime per year** and **per month** using `525,600 minutes/year`.
2. The business has a **99.9% SLA** with its enterprise customers. Are you meeting it? By how much are you missing?

**Part B — Fix it (15 min).**
3. Identify the **single worst component** — the one whose failure rate contributes most to the total.
4. You may make exactly **THREE** components redundant (duplicate them in parallel). Which three, and why? Recompute the end-to-end availability. Show the arithmetic.
5. The Payment Gateway is a **third party** — you cannot make it redundant or improve it. Propose **two design changes** that reduce its impact on your availability. (Does payment have to be *in* the synchronous checkout path? What does topic 73 suggest? What does topic 67 suggest?)

**Part C — Operate it (10 min).**
6. Your service fails about **twice a month** and takes **90 minutes** to recover. Compute availability with `A = MTBF / (MTBF + MTTR)`.
7. Your manager offers a choice: spend a quarter halving the failure rate, OR spend a quarter cutting MTTR to 10 minutes. Compute both. Which do you choose, and what engineering work does that imply?
8. Define one **SLI** (an exact ratio you could compute from access logs), one **SLO**, and the resulting monthly **error budget in minutes**.

**Produce:** a one-page write-up with all arithmetic shown, a redrawn architecture diagram with your redundancy added, and a three-sentence recommendation to the business.

---

## Quick reference cheat sheet

- **Availability** = "is it up?" · **Reliability** = "is it correct, consistently?" · **Durability** = "is my data still there?" — three different promises.
- **The nines:** 99% = 3.65 days/yr · **99.9% = 8.8 hrs/yr (43 min/month)** · **99.99% = 52 min/yr** · 99.999% = 5 min/yr. **Derive any of them:** a year is **525,600 minutes**; multiply by the failure fraction (`99.99% → 525,600 × 0.0001 = 52.6 min`).
- **Series (all must work) → MULTIPLY availabilities.** It always gets worse. `0.999^5 = 99.5%`. Shortcut: small failure rates just add.
- **Parallel (any one works) → MULTIPLY failure rates.** It always gets better. `1 − 0.01² = 99.99%` from two 99% servers.
- **Redundancy math assumes INDEPENDENT failures.** Same rack, same deploy, same shared DB, or a cascading overload → the math is a lie.
- **`A = MTBF / (MTBF + MTTR)`** — availability comes from failing rarely *and* recovering fast. **Optimize MTTR, not MTBF:** cutting recovery from 2 hours to 5 minutes beats halving your bug count, for a fraction of the effort.
- **SLI** = the measurement (`good/total`). **SLO** = your internal target. **SLA** = the customer contract with money attached. **SLO must be stricter than SLA.**
- **Measure the SLI at the edge** (load balancer or client). A crashed app cannot report its own downtime.
- **Error budget = 100% − SLO.** It's a resource you are *meant to spend*. Budget healthy → ship fast. Budget gone → feature freeze.
- **99.99% requires zero humans in the recovery loop.** 4 minutes/month is less than the time it takes to wake up and open a laptop.
- **Each nine costs ~10x more.** Never target 100% — it's impossible (you don't own the user's ISP), the value curve flattens, and it kills your ability to ship.
- **The load balancer is often the hidden SPOF.** Do the series math and it will find your single point of failure for you.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [06 — Latency vs Throughput](./06-latency-and-throughput.md) — the *performance* half of non-functional requirements; this doc is the *uptime* half |
| **Next** | [08 — Consistency Models](./08-consistency-models.md) — once you add the replicas that give you availability, you must decide what "correct data" even means across them |
| **Related** | [73 — Circuit Breaker Pattern](./73-circuit-breaker.md) — the standard defence against series-multiplication in a microservice chain |
| **Related** | [90 — Disaster Recovery and Backup Strategies](./90-disaster-recovery.md) — RPO/RTO, hot vs warm vs cold standby, and chaos engineering as MTTR training |
