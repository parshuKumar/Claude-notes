# 04 — Functional vs Non-Functional Requirements
## Category: HLD Fundamentals

---

## What is this?

Every system has two kinds of requirements. **Functional requirements** are what the system must **DO** — "a user can upload a photo," "a user can follow another user." **Non-functional requirements (NFRs)** are how **WELL** it must do it — "the photo appears in under 200ms," "the service stays up 99.99% of the time," "we never lose an uploaded photo."

Think of ordering a pizza. *"I want a large pepperoni pizza"* is functional — it describes the thing. *"It must arrive in 30 minutes, still hot, and not cost more than $20"* is non-functional — it describes the quality of the delivery. Two pizzerias can both make you a correct pepperoni pizza; only one of them stays in business.

---

## Why does it matter?

Here is the sentence to tattoo on your brain:

> **Functional requirements tell you WHAT to build. Non-functional requirements tell you HOW to build it.**

The features are almost never the hard part. "A user can post a tweet" is a `INSERT INTO tweets` statement — a junior engineer writes that in an afternoon. What makes Twitter a hard engineering problem is *"and it must serve 200 million people with a 100:1 read:write ratio at sub-200ms latency without ever going down."*

**Every interesting architecture decision you will ever make is driven by an NFR, not a feature.**

| You added... | Because of which requirement? |
|-------------|------------------------------|
| A cache | NFR — latency + high read QPS |
| A load balancer | NFR — availability + scalability |
| Database replication | NFR — availability + durability |
| A message queue | NFR — availability under spikes, decoupling |
| Sharding | NFR — scalability (data exceeds one machine) |
| A CDN | NFR — latency for geographically distant users |

Notice: **not a single one is a feature.** This is why an interviewer who hears you say "I'd add a cache" will immediately ask "why?" — and the only acceptable answer is a non-functional one.

**The interview angle:** Candidates who skip NFRs design a system that works for 100 users and collapses at 100 million. Candidates who gather NFRs first design a system whose every component is justified.

**The real-work angle:** The classic production disaster is a feature that works perfectly in staging with 10 rows and melts in production with 10 million. That gap is *always* an NFR nobody wrote down.

---

## The core idea — explained simply

### The Restaurant Analogy

You're opening a restaurant. Someone asks you what you need.

**Functional requirements (what the restaurant DOES):**
- Customers can see a menu
- Customers can place an order
- The kitchen cooks the food
- A waiter delivers the food to the table
- Customers can pay

Fine. Now — is that enough to design the restaurant? **No.** You have no idea how big the kitchen should be, how many staff to hire, or whether you need one oven or twelve. Because you haven't asked the *real* questions:

**Non-functional requirements (how WELL it does it):**

| The restaurant question | The system design term | What it forces you to build |
|------------------------|----------------------|----------------------------|
| "How many customers per night — 20 or 2,000?" | **Scalability** | A bigger kitchen, more cooks, or a second location |
| "Food must arrive within 15 minutes of ordering" | **Latency** | Pre-chopped ingredients (a *cache*), more parallel stations |
| "We open 365 days a year, no exceptions" | **Availability** | A backup generator, a second chef on call (*redundancy*) |
| "Every customer at a table gets their food together" | **Consistency** | Coordination between kitchen stations |
| "A paid order must never be lost, even if the POS crashes" | **Durability** | Write the order on paper too (*a write-ahead log*) |
| "Only staff can enter the kitchen" | **Security** | Locks, badges (*authN/authZ*) |
| "We can't spend $2M on equipment" | **Cost** | Cheaper hardware, fewer replicas |

Now look at the two lists again. The functional list is **identical** for a 20-seat family diner and a 2,000-cover stadium restaurant. Both take orders, cook, deliver, and take payment.

**The NFRs are the ONLY thing that makes them different buildings.**

That is the entire lesson of this document. The features tell you what boxes to draw. The NFRs tell you how many of each box, where they go, and what glue holds them together.

---

## Key concepts inside this topic

### 1. Functional Requirements — What the System DOES

A functional requirement is always a sentence of the form: **"A [user] can [verb] a [noun]."** If you can't phrase it that way, it isn't functional.

```
GOOD functional requirements (testable, user-visible, verb-shaped):
  - A user can post a tweet of up to 280 characters
  - A user can follow another user
  - A user can view a home timeline of tweets from people they follow
  - A user can delete their own tweet

NOT functional requirements (these are NFRs in disguise):
  - The timeline should load fast              → latency (NFR)
  - The system should handle many users        → scalability (NFR)
  - Tweets should never be lost                → durability (NFR)
  - The system should be secure                → security (NFR)
```

**How to gather them in an interview:** ask for the top 3–5 features and *nothing more*. You cannot design ten features in 45 minutes and neither can the interviewer's real team in a quarter.

**Prioritise ruthlessly.** Not all features are equal:

```
MUST HAVE   (the system is pointless without these) → design these
SHOULD HAVE (important, but the system works without them) → mention, don't design
NICE TO HAVE (mention and explicitly defer) → "out of scope for today"
```

### 2. Non-Functional Requirements — How WELL It Performs

The seven that show up in essentially every interview. Learn the definition and, crucially, **what each one forces you to build.**

**a) Scalability** — can it grow?
> "The system handles 200M DAU and must absorb 10x growth without a rewrite."

*Forces:* horizontal scaling, load balancers, sharding, stateless services.
*The key distinction:* **vertical scaling** = buy a bigger machine (simple, but there's a ceiling and it's a single point of failure). **Horizontal scaling** = buy more machines (no ceiling, but now you have a distributed system and all its problems).

**b) Latency** — how fast is one request?
> "Timeline loads in under 200ms at the p99."

*Forces:* caching, CDNs, precomputation, denormalization, geographic replication.
*Always ask for the percentile.* "p99 = 200ms" means 99% of requests finish in under 200ms. Averages lie — an average of 100ms can hide a p99 of 5 seconds, and that p99 is the experience of your most active users, because they make the most requests.

**c) Availability** — what fraction of the time is it up?

| Availability | Downtime per year | Downtime per month | Typical use |
|-------------|-------------------|-------------------|-------------|
| 99% ("two nines") | 3.65 days | 7.2 hours | Internal tools |
| 99.9% ("three nines") | 8.77 hours | 43.8 minutes | Most SaaS |
| 99.99% ("four nines") | 52.6 minutes | 4.4 minutes | Serious consumer products |
| 99.999% ("five nines") | 5.26 minutes | 26 seconds | Telecom, payments, life-critical |

*Forces:* redundancy everywhere, replication, failover, health checks, multi-AZ or multi-region, no single point of failure.
*The trap:* every extra nine costs roughly an order of magnitude more money. Ask whether you actually need it.

**d) Consistency** — do all readers see the same data?
> "Two users viewing the same tweet must see the same like count... eventually."

*Forces:* your entire database choice.
- **Strong consistency:** every read returns the latest write. Required for bank balances, inventory counts, seat bookings. Costs latency and availability.
- **Eventual consistency:** reads may be briefly stale, but converge. Perfectly fine for likes, view counts, feeds, follower lists. Buys you availability and speed.

You will meet this again in [08 — Consistency Models](./08-consistency-models.md) and [09 — CAP Theorem](./09-cap-theorem.md).

**e) Durability** — once written, is it safe forever?
> "An uploaded photo is never lost, even if a whole data centre burns down."

*Forces:* replication across failure domains, write-ahead logs, backups, checksums.
*Do not confuse durability with availability.* Data can be perfectly durable (safely on disk, three copies) and simultaneously unavailable (the service is down). Durability = "we still have it." Availability = "you can reach it right now."

**f) Security** — who can do what?

*Forces:* authentication (who are you — tokens, OAuth), authorization (what may you do — roles, ACLs), encryption in transit (TLS) and at rest, rate limiting, input validation.

**g) Cost** — what's the budget?

*Forces:* the entire shape of the design. Storing every raw video forever in hot storage is technically fine and financially insane. Tiered storage, TTLs, and sampling all exist because of this NFR.

### 3. The NFR Checklist — Always Ask These

Print this. Ask these in **every** design interview and in every design doc.

```
SCALE
  [ ] How many daily active users (DAU)?
  [ ] Expected growth over 1–2 years?
  [ ] Read-heavy or write-heavy? What's the read:write ratio?
  [ ] Peak-to-average traffic ratio? Any predictable spikes (Black Friday, game day)?

PERFORMANCE
  [ ] Target latency for the primary operation? At which percentile (p50/p95/p99)?
  [ ] Is any part real-time (chat, live location) or can everything be async?

AVAILABILITY
  [ ] Target uptime — 99.9%? 99.99%?
  [ ] What's the blast radius of one server dying? One region dying?
  [ ] Is planned downtime acceptable (maintenance windows)?

CONSISTENCY
  [ ] Is eventual consistency acceptable for the main read path?
  [ ] Which operations REQUIRE strong consistency? (money, inventory, seats)

DURABILITY & RETENTION
  [ ] Can we ever lose data? (For payments: no. For view counts: a little.)
  [ ] How long do we retain it — 30 days, 7 years, forever?

SECURITY & COMPLIANCE
  [ ] Is the data public, private, or regulated (PII, health, payment card)?
  [ ] Any data-residency rules (GDPR: EU data stays in the EU)?

COST
  [ ] Is there a budget ceiling that rules anything out?
```

### 4. How Each NFR Becomes an Architecture Decision

This mapping is the payoff of the whole document. Memorise the shape of it.

```
NFR                          →  COMPONENT YOU ADD              →  PRICE YOU PAY
─────────────────────────────────────────────────────────────────────────────────
High read QPS (231k/s)       →  Redis cache in front of DB     →  Stale reads; cache
                                                                  invalidation is hard
Low latency, global users    →  CDN + regional replicas        →  $$$; replication lag
99.99% availability          →  Multi-AZ replicas + failover   →  2-3x infra cost
Durability (never lose data) →  3x replication + WAL + backups →  3x storage cost;
                                                                  slower writes
Write spikes / slow work     →  Kafka queue + async workers    →  Eventual consistency;
                                                                  more moving parts
Data > one machine           →  Sharding by userId             →  Cross-shard joins die;
                                                                  rebalancing is painful
Strong consistency (money)   →  Single primary + transactions  →  Lower availability;
                                                                  the primary is a bottleneck
Security (PII)               →  TLS + encryption at rest + IAM →  CPU cost; key management
```

**Read the right-hand column carefully.** There is no free NFR. Every one you satisfy, you pay for — in money, in complexity, or in another NFR. Saying "I want strong consistency AND 99.999% availability across regions" is not ambition; it is a misunderstanding of the CAP theorem, and an interviewer will catch it.

### 5. Encoding Requirements as Code (a Node.js example)

Requirements aren't just prose — they become guardrails you actually enforce in code and monitoring.

```javascript
// requirements.js
// Writing the NFRs down as executable config keeps the team honest.
// If a change violates the budget, a test or an alert fires — not a customer.

export const NFR = Object.freeze({
  scale: {
    dau: 200_000_000,
    readWriteRatio: 100,          // 100 reads per write → cache is MANDATORY, not optional
    peakMultiplier: 2.5,          // peak QPS = 2.5x average
  },
  latency: {
    // We budget the p99, never the average. Averages hide the users who suffer.
    timelineReadP99Ms: 200,
    tweetWriteP99Ms: 500,         // writes may be slower — fanout happens async anyway
  },
  availability: {
    targetPercent: 99.99,         // ≈ 52 minutes of downtime per year
  },
  consistency: {
    timeline: 'eventual',         // a 2-3s stale feed is invisible to users
    followCount: 'eventual',
    payments: 'strong',           // money is NEVER eventually consistent
  },
  durability: {
    tweets: 'never-lose',         // → 3x replication across availability zones
    viewCounts: 'best-effort',    // → sampling and approximation are fine here
  },
});

/**
 * A latency budget is only real if it is measured and enforced.
 * This middleware turns the NFR above into an alert, so violations are
 * caught in production instead of discovered in a postmortem.
 */
export function latencyBudget(budgetMs, operationName, metrics) {
  return async (req, res, next) => {
    const start = process.hrtime.bigint();

    res.on('finish', () => {
      const elapsedMs = Number(process.hrtime.bigint() - start) / 1e6;
      metrics.histogram(`${operationName}.latency_ms`, elapsedMs);

      if (elapsedMs > budgetMs) {
        // Not an error for the user — but it IS a violated promise. Track it.
        metrics.increment(`${operationName}.budget_exceeded`);
      }
    });

    next();
  };
}

// Usage in an Express app:
// app.get('/v1/timeline',
//   latencyBudget(NFR.latency.timelineReadP99Ms, 'timeline.read', metrics),
//   timelineHandler);
```

### 6. How to Push Back and Scope Down (the senior move)

The interviewer *wants* you to cut scope. Handing you ten features is a test of judgment, not generosity.

**The three-part script:**

> **1. Acknowledge:** "Twitter has tweets, timelines, DMs, search, trends, ads, and notifications."
> **2. Prioritise with a reason:** "In 45 minutes I want to spend time where the scale problems actually are. Posting a tweet and generating the home timeline carry the hardest problems — fanout at 200M users. Search and ads are big systems but they're largely independent."
> **3. Get agreement:** "So I'll design tweet + timeline + follow, and treat DMs, search, and ads as out of scope. Does that work for you?"

**Push back on NFRs too.** When an interviewer (or a PM) demands something expensive, make the cost visible:

> **Them:** "The feed must be strongly consistent — every follower sees a tweet the instant it's posted."
> **You:** "I can do that, but here's the cost: strong consistency across regions means every read coordinates with a primary, so p99 latency goes from ~20ms to ~200ms+, and a network partition takes the feed *down* rather than serving slightly stale data. For a social feed, users cannot perceive a 2-second delay. I'd propose eventual consistency for the timeline and reserve strong consistency for things where staleness is a real bug — like DM read receipts or payments. Does that trade-off sound acceptable?"

That answer demonstrates you understand the *price* of an NFR — which is exactly what separates a senior engineer from someone reciting buzzwords.

---

## Visual / Diagram description

### Diagram 1: How NFRs Reshape the Same Feature Set

Both systems below implement the **identical functional requirements**: post a message, read a feed. Only the NFRs differ.

```
  SAME FEATURES, 1,000 USERS              SAME FEATURES, 200,000,000 USERS
  (NFRs: 10 QPS, 99% uptime,              (NFRs: 460k peak read QPS, 99.99%
   staleness fine, 1 GB data)              uptime, p99 < 200ms, 220 TB data)

      ┌──────────┐                            ┌──────────┐
      │  Client  │                            │  Client  │
      └────┬─────┘                            └────┬─────┘
           │                                       │
           ▼                                  ┌────▼─────┐   ← NFR: availability
   ┌───────────────┐                          │   CDN    │   ← NFR: global latency
   │  Node server  │                          └────┬─────┘
   │  (1 machine)  │                          ┌────▼─────┐
   └───────┬───────┘                          │   Load   │   ← NFR: scalability
           │                                  │ Balancer │      + availability
           ▼                                  └────┬─────┘
   ┌───────────────┐                          ┌────▼─────────────┐
   │  PostgreSQL   │                          │  API Gateway     │  ← NFR: security
   │  (1 machine)  │                          │ (authN, ratelimit)│     + abuse control
   └───────────────┘                          └──┬────────────┬──┘
                                                 │            │
                                     ┌───────────▼──┐  ┌──────▼────────┐
   THAT'S IT. And it is CORRECT.     │Write Service │  │ Read Service  │ ← NFR: 100:1
   Adding a cache here would be                 │            │              read:write
   over-engineering — you'd be                  │        ┌───▼────────┐
   paying complexity for an NFR                 │        │Redis Cluster│ ← NFR: p99 latency
   you don't have.                              │        └───▲────────┘
                                                │            │
                                     ┌──────────▼──┐  ┌──────┴───────┐
                                     │   Kafka     │─▶│Fanout Workers│ ← NFR: absorb
                                     └─────────────┘  └──────────────┘     write spikes
                                                │
                                     ┌──────────▼───────────────┐
                                     │ Cassandra (sharded, 3x   │ ← NFR: 220 TB storage
                                     │ replicated, multi-AZ)    │      + durability
                                     └──────────────────────────┘        + availability
```

**What the diagram shows:** every single box on the right exists because of a *non-functional* requirement, annotated beside it. The left system does exactly the same things for the user. If you ever add a component you cannot annotate with an NFR like this, **delete it** — you are over-engineering.

### Diagram 2: The Requirement-to-Architecture Pipeline

```
  ┌──────────────────┐     ┌──────────────────┐
  │   FUNCTIONAL     │     │  NON-FUNCTIONAL  │
  │  "post a tweet"  │     │ "200M DAU, p99   │
  │  "view timeline" │     │  <200ms, 99.99%" │
  └────────┬─────────┘     └────────┬─────────┘
           │                        │
           │                        ▼
           │               ┌──────────────────┐
           │               │    CAPACITY      │  ← the NFRs become NUMBERS
           │               │   ESTIMATION     │    (see doc 05)
           │               │  460k read QPS,  │
           │               │  220 TB, 4.6k wps│
           │               └────────┬─────────┘
           │                        │
           ▼                        ▼
  ┌─────────────────┐      ┌──────────────────┐
  │  WHICH BOXES    │      │  HOW MANY OF     │
  │  EXIST          │      │  EACH BOX, AND   │
  │  (services,     │      │  WHAT GLUE       │
  │   endpoints,    │      │  (cache, queue,  │
  │   tables)       │      │   shards, CDN)   │
  └────────┬────────┘      └────────┬─────────┘
           │                        │
           └───────────┬────────────┘
                       ▼
              ┌──────────────────┐
              │  THE ARCHITECTURE│
              └──────────────────┘
```

**What the diagram shows:** functional requirements decide *which* boxes appear. Non-functional requirements decide *how many* of each and what infrastructure connects them. Both branches must be walked, and the NFR branch passes through estimation first — you cannot size a system from adjectives like "fast" and "big."

---

## Real world examples

### 1. Netflix — Availability as the Dominant NFR

Netflix's functional requirement is unremarkable: play a video. What makes their engineering famous is a non-functional obsession with availability — the service should keep working even when infrastructure underneath it fails.

This NFR is why Netflix engineering is known for **chaos engineering** (their Chaos Monkey tool deliberately kills production instances so that engineers are forced to build services that survive instance death) and for aggressive use of fallbacks and graceful degradation. When a personalization service fails, you don't get an error page — you get a generic, non-personalized row. The feature degrades; the service stays up.

No feature drove that. One NFR did.

### 2. Payment Systems — Consistency and Durability Above All

For a payment platform, the functional requirement ("transfer money from A to B") is trivially describable. The NFRs are brutal:

- **Strong consistency:** a balance can never be read stale in a way that permits double-spending.
- **Durability:** an acknowledged payment must survive any single machine, rack, or datacenter loss.
- **Idempotency:** a retried request must not charge the customer twice.
- **Auditability:** every state change must be reconstructible years later — hence double-entry ledgers, which are append-only by design.

These NFRs are why payment systems typically use relational databases with ACID transactions and append-only ledgers, and *deliberately accept lower throughput* than a social feed would tolerate. They are trading throughput for correctness, on purpose. (See [107 — Design a Payment System](./107-hld-payment-system.md).)

### 3. WhatsApp — Latency and Scale on a Tiny Team

WhatsApp famously served an enormous user base with a remarkably small engineering team. That was possible because they were fanatically clear about their requirements: the functional core was small (send a message, deliver it, show delivery state) and the NFRs were sharp — extremely low delivery latency, extremely high connection concurrency, and minimal server-side message retention.

That last one is the interesting NFR. By choosing **not** to retain messages on the server after delivery, they turned a gigantic storage problem into a small one. A *requirements decision* eliminated an entire class of infrastructure. That is the highest form of system design: the cheapest component is the one you proved you don't need.

---

## Trade-offs

| NFR you optimize for | What you gain | What you give up |
|---------------------|--------------|------------------|
| **Strong consistency** | Correctness; no stale reads | Latency, availability under partition (CAP), throughput |
| **High availability** | Survives failures; users never see errors | Cost (2–3x infra), and usually consistency |
| **Low latency** | Great UX; higher engagement | Cost (caches, CDN, replicas), staleness, cache-invalidation complexity |
| **High durability** | Never lose data | Slower writes (must fsync/replicate before ack), 3x storage cost |
| **High scalability** | Handles growth | Distributed-system complexity: sharding, rebalancing, no easy joins |
| **Tight security** | Compliance, user trust | CPU overhead, latency, developer friction, key management |
| **Low cost** | Cheap to run | Everything above gets worse |

| Requirements approach | Pro | Con |
|----------------------|-----|-----|
| Gather all NFRs up front | Design is right the first time | Slow; may over-engineer for scale you never reach |
| Gather only functional, ignore NFRs | Ship fast | Melts in production; expensive rewrite later |
| Design for 10x current scale | Room to grow without a rewrite | Some wasted complexity and cost today |
| Design for 1000x current scale | Feels future-proof | Almost always premature; you'll be wrong about which axis grows |

**The sweet spot:** design for roughly **10x your current scale**, and make sure nothing in the design *blocks* the next 10x. Ask about every NFR; only build for the ones with real numbers behind them. And remember: **an NFR without a number is not a requirement, it's a wish.** "Fast" is a wish. "p99 under 200ms" is a requirement.

---

## Common interview questions on this topic

### Q1: "What's the difference between functional and non-functional requirements?"
**Hint:** Functional = what the system does (features, user-visible verbs — "a user can post a tweet"). Non-functional = how well it does it (scalability, latency, availability, consistency, durability, security, cost). The killer line to deliver: *"Functional requirements decide which components exist; non-functional requirements decide how many of each and what infrastructure connects them."*

### Q2: "Which NFRs would you ask about for a ride-sharing app like Uber?"
**Hint:** Latency is the star — driver location updates and matching must be near real-time (sub-second), so you need geospatial indexing and in-memory location stores. Availability must be very high; a rider stranded at night is a real-world failure. Consistency is *mixed*: driver locations can be eventually consistent (a location 2 seconds stale is fine), but a ride assignment must be strongly consistent — two riders must never be matched to the same driver. Point out that different parts of one system can have different consistency requirements; that nuance scores.

### Q3: "The interviewer lists 8 features. What do you do?"
**Hint:** Don't try to design 8 features. Acknowledge all of them, pick the 2–3 with the hardest scale characteristics, explain *why* you picked those, and get explicit agreement to treat the rest as out of scope. Write OUT OF SCOPE on the board. This is a judgment test, and picking is passing.

### Q4: "Why can't we just have strong consistency AND high availability everywhere?"
**Hint:** The CAP theorem. In a distributed system, network partitions *will* happen — they're not optional. When one does, you must choose: refuse writes to stay consistent (CP), or accept writes and reconcile later (AP). You cannot have both during a partition. In practice you also pay a *latency* tax for strong consistency even when there's no partition, because reads and writes must coordinate. Real systems mix: strong consistency for money and inventory, eventual for feeds and counters.

### Q5: "How do you handle a stakeholder who says every requirement is critical?"
**Hint:** Convert requirements into costs and make them choose. "Five nines instead of four means multi-region active-active, roughly triple the infrastructure spend, and a harder consistency story. Four nines is 52 minutes of downtime a year. Is the extra 47 minutes of uptime worth that?" Nobody says "everything is P0" once each P0 has a dollar figure attached.

---

## Practice exercise

### The Requirements Interrogation: WhatsApp

You are the interviewer's candidate. The prompt is: **"Design WhatsApp."** You get nothing else.

Produce a one-page requirements document containing these five sections. Do **not** draw any architecture — the point of this exercise is to prove you can resist that urge.

**Part 1 — Clarifying questions (write at least 10).**
Split them: at least 4 functional ("Are group chats in scope? Voice calls? Media? Is it end-to-end encrypted?") and at least 6 non-functional (use the checklist in section 3 — DAU, read:write ratio, latency target, availability, retention, consistency).

**Part 2 — Functional requirements.**
Write 4–6 in the form *"A user can ___."* Then classify each as MUST / SHOULD / NICE-TO-HAVE. Then write an explicit OUT OF SCOPE list.

**Part 3 — Non-functional requirements — with numbers.**
For each of the seven NFRs (scalability, latency, availability, consistency, durability, security, cost), write one line with an actual number or an explicit decision. `"Latency: message delivered in <500ms p99 when the recipient is online."` **A line without a number is a failed line** — rewrite it.

**Part 4 — The mapping table.**
Build a two-column table: `NFR → the architectural component or decision it forces`. Aim for at least 6 rows. Example row: `"Message must not be lost if the recipient is offline → server-side pending-message queue with at-least-once delivery, plus client-side dedupe by message ID."`

**Part 5 — The push-back.**
Your interviewer insists: *"Messages must be readable by the user forever, from any device, with full search."* Write a 4–6 sentence response that (a) states what that NFR costs — storage growth, encryption trade-offs, search index cost, (b) proposes a cheaper alternative, and (c) asks for a decision.

**What to produce:** one page, five sections, no boxes and no arrows. If you find yourself drawing a database, stop — you've skipped ahead, and skipping ahead is exactly the habit this exercise is designed to break.

---

## Quick reference cheat sheet

- **Functional = what it DOES.** Phrase every one as *"A user can [verb] a [noun]."*
- **Non-functional = how WELL it does it.** Scalability, latency, availability, consistency, durability, security, cost.
- **NFRs drive architecture, features don't.** Every cache, queue, shard, and replica exists for an NFR.
- **An NFR without a number is a wish.** "Fast" is meaningless; "p99 < 200ms" is a requirement.
- **Always ask for the percentile.** Averages hide disasters. Design to p99, not the mean.
- **The four questions to always ask:** DAU? Read:write ratio? Latency target? Availability target?
- **Availability arithmetic:** 99.9% = 8.8 hrs/year down. 99.99% = 52 min/year. Each nine costs ~10x more.
- **Durability ≠ availability.** Durable = "we still have your data." Available = "you can reach it now."
- **Consistency is per-operation, not per-system.** Feeds can be eventual; money must be strong.
- **Scope down out loud.** Name the features you're skipping — that's a senior signal, not a gap.
- **Push back with costs, not opinions.** Translate every NFR demand into latency, dollars, or complexity.
- **Design for 10x, not 1000x.** Just don't paint yourself into a corner that blocks the next 10x.
- **The cheapest component is the one you proved you don't need** — a requirements decision can delete an entire subsystem.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [03 — How to Approach a System Design Problem](./03-how-to-approach-system-design.md) — the framework in which requirements are Step 1 |
| **Next** | [05 — Capacity Estimation](./05-capacity-estimation.md) — turning the NFRs you just gathered into hard numbers |
| **Related** | [07 — Availability and Reliability](./07-availability-and-reliability.md) — the availability NFR in full depth (SLA, SLO, SLI) |
| **Related** | [09 — CAP Theorem](./09-cap-theorem.md) — why consistency and availability cannot both be maximised |
