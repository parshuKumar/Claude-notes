# 133 — The System Design Trade-Off Matrix
## Category: Advanced

---

## What is this?

There are **no free lunches** in system design. Every single choice you make — every database, every cache, every queue, every consistency setting — buys you one good thing by paying for it with another. Faster reads cost you write complexity. Stronger consistency costs you latency. More reliability costs you money.

This document is the **capstone**: it gathers every major trade-off you have met across this entire course into one reasoning framework. Think of it as the moment where all the scattered lessons snap together and you finally see the wiring behind them. If the rest of the course taught you the *pieces*, this teaches you the *physics* — the conservation laws that no clever engineering can escape.

A trade-off matrix is like a **restaurant menu with prices**. You never ask "what's the best dish?" in a vacuum. You ask "what's the best dish *for my budget, my hunger, and my dietary constraints*?" System design is identical: there is no best architecture, only the best architecture *for these specific requirements*.

---

## Why does it matter?

**Interview angle:** Naming trade-offs out loud is the **single biggest signal of senior-level thinking**. A junior candidate says "I'll use a cache." A senior candidate says "I'll use a cache, which trades memory and a risk of staleness for read latency — and since this data is read 100x more than it's written and can tolerate being a few seconds stale, that trade is clearly worth it." Same decision. Completely different signal. Interviewers are explicitly listening for the second voice. They will often *push back* on your choices precisely to see whether you understand what you gave up.

**Real-work angle:** In production, unnamed trade-offs become 3 a.m. pages. The team that "just turned on eventual consistency" because it was faster, without saying so out loud, is the team that ships a double-charged-customer bug six months later. Every architecture decision record (ADR) worth writing has a "Consequences" section — and that section *is* the trade-off you accepted. Engineers who reason in trade-offs write systems that fail in *expected* ways instead of surprising ones.

If you internalize only one document from this entire course, internalize this one. Everything else is an instance of it.

---

## The core idea — explained simply

### The Blanket That's Always Too Short

Imagine sleeping under a blanket that is slightly too short. Pull it up to cover your shoulders, and your feet stick out into the cold. Pull it down to cover your feet, and your shoulders freeze. You cannot cover both — the blanket has a fixed amount of fabric. Your only real decision is **which part you're willing to leave cold tonight**, given how you're sleeping.

System design is the too-short blanket. The "fabric" is fixed by hard limits — the speed of light, the cost of a byte, the impossibility (proven!) of having consistency *and* availability during a network partition. You do not get to cover everything. You get to *choose what to leave exposed*, on purpose, for good reasons.

The amateur pretends the blanket is big enough and gets surprised by cold feet. The professional says out loud: "Tonight I'm sleeping on my back, so I'll leave my feet exposed and cover my shoulders — here's why."

| Analogy | System design concept |
|---------|----------------------|
| The blanket | Your finite engineering budget (latency, money, complexity) |
| Fabric is fixed | Hard limits: CAP, speed of light, cost per byte |
| Shoulders vs feet | Two things you want but can't fully have at once |
| Which part to leave cold | The trade-off you consciously accept |
| How you're sleeping tonight | The system's specific requirements |
| Saying it out loud | Naming the trade-off — the senior signal |
| Getting surprised by cold feet | An unnamed trade-off becoming a production incident |

The rest of this document is a catalogue of every blanket-edge you'll ever have to choose to leave cold.

---

## Key concepts inside this topic

Each trade-off below follows the same shape: **you gain X, you pay Y, lean this way when Z** — plus where in the course it first appeared.

### 1. Consistency vs Availability (and the honest version, PACELC)

This is the granddaddy, from [09 — CAP Theorem](./09-cap-theorem.md). During a **network partition** (some nodes can't talk to others), you must choose: keep answering requests with possibly-wrong data (**Availability**), or refuse to answer until you're sure the data is correct (**Consistency**). You cannot have both while partitioned — that's a mathematical proof, not an engineering weakness.

- **Strong consistency** buys correctness. It costs you **latency** (coordination round trips) and **availability** (nodes go silent rather than serve stale data).
- **Eventual consistency** buys availability and low latency. It costs you a window where different users see different data.

**PACELC** is the more honest framing, and mentioning it in an interview is a flex. It reads: **P**artition → **A** or **C**; **E**lse (normal operation) → **L**atency or **C**onsistency. The insight PACELC adds: even when there's *no* partition, you *still* trade latency against consistency every single day. There is no "off" switch on this trade.

**Lean strong (CP) when:** the data is money, inventory, bookings, or auth — anywhere a wrong answer causes a real-world loss. **Lean eventual (AP) when:** the data is social (likes, feeds, follower counts, view counts) and a few seconds of staleness is invisible or harmless. See [08 — Consistency Models](./08-consistency-models.md) for the full spectrum between these poles.

```js
// The same "add a like" operation, two ways.

// CP: refuse rather than risk a wrong count. Correct, but can block.
async function likeStrong(postId, userId) {
  // A distributed transaction / quorum write — waits for agreement.
  return db.transaction(async (tx) => {
    await tx.insert("likes", { postId, userId }); // fails if partitioned
    await tx.increment("posts", postId, "likeCount");
  });
}

// AP: accept it locally now, reconcile later. Fast and always available.
async function likeEventual(postId, userId) {
  await localQueue.push({ type: "LIKE", postId, userId }); // returns instantly
  return { ok: true }; // count converges within seconds via background merge
}
```

### 2. Latency vs Throughput

From [06 — Latency and Throughput](./06-latency-and-throughput.md). **Latency** is how long *one* request takes. **Throughput** is how many requests you handle *per second*. They pull against each other.

- **Batching** raises throughput — group 100 writes into one disk flush and you do far less per-item work — but it **adds latency**, because early items wait for the batch to fill.
- Aggressively optimizing **tail latency** (the slow 1% of requests) often *lowers* throughput, because you hold spare capacity idle and refuse to queue work that would make the slow requests slower.

**Lean throughput when:** the work is background, bulk, or analytical — log ingestion, ETL, video encoding — where nobody is staring at a spinner. **Lean latency when:** a human is waiting synchronously — search, checkout, page load.

```js
// Batching: great throughput, worse per-item latency.
class BatchWriter {
  constructor(flushSize = 100, flushMs = 50) {
    this.buf = [];
    this.flushSize = flushSize;
    setInterval(() => this.flush(), flushMs); // items may wait up to 50ms
  }
  write(row) {
    this.buf.push(row);
    if (this.buf.length >= this.flushSize) this.flush(); // one big disk write
  }
  flush() {
    if (this.buf.length) { db.bulkInsert(this.buf); this.buf = []; }
  }
}
```

### 3. Latency vs Consistency

A close cousin of #1, but worth isolating because you make this choice *per read*. Do you read from the **nearest replica** (fast, but it may be a few hundred milliseconds behind the leader) or from the **leader** (guaranteed current, but possibly a continent away)?

- **Nearest-replica read:** low latency, possible staleness.
- **Leader read:** fresh data, higher latency (and more load concentrated on the leader).

**Lean nearest-replica when:** rendering a mostly-static profile, a product description, a cached feed. **Lean leader when:** the user is reading data *they themselves just wrote* ("read-your-own-writes") — seeing your own comment vanish after posting feels like a bug even when the count elsewhere is fine.

### 4. Read optimization vs Write optimization

Almost every performance tool speeds **reads** by doing **more work at write time**. You can't escape it — you're just moving the cost.

- **Indexes** make lookups fast but every write must also update every index.
- **Denormalization** makes reads join-free but every update must touch every duplicated copy.
- **Caching** makes reads instant but every write must invalidate or update the cache.
- **Fanout-on-write** (used in feeds) precomputes each follower's timeline at post time — reads become a single lookup, but a celebrity's one post triggers millions of writes (fanout amplification).

Storage engines encode this trade too: **B-trees** favor reads and in-place updates; **LSM-trees** favor high write throughput by appending and compacting later (the classic read-vs-write engine choice).

**Lean read-optimized when:** the read/write ratio is high (feeds, catalogs, dashboards — often 100:1 or more). **Lean write-optimized when:** you ingest a firehose and read rarely (metrics, logs, event streams).

| Technique | Speeds up | Cost paid at write time |
|-----------|-----------|-------------------------|
| Secondary index | Reads/filters | Every write updates the index |
| Denormalization | Reads (no joins) | Update anomalies, duplicated writes |
| Cache | Reads | Invalidation complexity, staleness |
| Fanout-on-write | Feed reads | Write amplification (celebrity problem) |
| Materialized view | Aggregations | Refresh cost, staleness |

### 5. Space vs Time

The most classic trade in all of computing: **spend memory to save time, or spend time to save memory.** Caching, precomputation, materialized views, memoization, and the Flyweight pattern all buy speed with space. Compression and recomputing-on-demand buy space back at the cost of CPU.

- **Precompute and store** → fast reads, more storage and stale-data risk.
- **Compute on demand** → minimal storage, slower reads and more CPU per request.

**Lean space-for-time when:** the same expensive result is read many times (a leaderboard, a daily report). **Lean time-for-space when:** storing everything is infeasible or the result is almost never reused.

```js
// Space-for-time: memoize an expensive pure computation.
const cache = new Map();
function fib(n) {
  if (n < 2) return n;
  if (cache.has(n)) return cache.get(n); // spend RAM...
  const v = fib(n - 1) + fib(n - 2);     // ...to skip recomputation
  cache.set(n, v);
  return v;
}
```

### 6. Normalization vs Denormalization

The database-modeling face of #4 and #5.

- **Normalized** (data stored once, referenced by ID): no duplication, cheap and *consistent* writes, but reads need **expensive joins**.
- **Denormalized** (data copied where it's read): fast join-free reads, but **duplicated data** and **update anomalies** — change a user's name and you must hunt down every copy.

**Lean normalized when:** writes are frequent and correctness matters (an OLTP orders table). **Lean denormalized when:** reads dominate and you can tolerate reconciliation (a read-model / feed / analytics store). Many mature systems keep a normalized "source of truth" *and* denormalized read-optimized projections — accepting extra storage and sync machinery to get both fast reads and correct writes.

### 7. Strong isolation vs Performance

From [66 — Transaction Isolation Levels]. **Serializable** isolation makes concurrent transactions behave as if they ran one-at-a-time — perfectly correct, but slow, because it takes heavy locks or aborts-and-retries under contention. Weaker levels (Read Committed, Repeatable Read) are faster but admit anomalies (dirty reads, non-repeatable reads, phantoms, write skew).

- **Serializable:** correct, lower throughput, more contention/retries.
- **Weaker isolation:** high throughput, but you must reason about which anomalies you just allowed.

**Lean serializable when:** money moves, or two transactions could corrupt each other (transfers, double-booking). **Lean weaker when:** rows rarely conflict and a stale-ish read is harmless (a comment feed under Read Committed).

### 8. SQL vs NoSQL

From [61 — SQL vs NoSQL].

- **SQL (relational):** ACID transactions, flexible ad-hoc queries and joins, strong schema — but harder to scale writes horizontally.
- **NoSQL:** effortless horizontal scale, flexible/evolving schema, high write throughput — but weaker cross-entity transactions and query patterns you must design for *up front*.

**Lean SQL when:** relationships are rich, you need transactions, and query patterns will change (most business apps start here — and should). **Lean NoSQL when:** you have one or two known access patterns at massive scale, or a schema that genuinely varies per record.

### 9. Monolith vs Microservices

From [71 — Monolith vs Microservices].

- **Monolith:** one deployable, simple local calls, easy transactions and debugging, one thing to run — but every part scales together and teams step on each other.
- **Microservices:** independent scaling and deployment, team autonomy, fault isolation — paid for with **distributed-systems complexity**: network failures, distributed transactions, versioning, and a much harder debugging story.

**Lean monolith when:** the team is small, the domain is young, and you don't yet know the boundaries. **Lean microservices when:** the org is large enough that team-independence is worth the operational tax, and boundaries are stable. The common failure is reaching for microservices *first* — buying distributed-systems pain before you have the scale that justifies it.

### 10. Synchronous vs Asynchronous

- **Synchronous** (request/response): simple mental model, easy to trace end-to-end, immediate errors — but the caller is **blocked**, and a slow dependency slows everyone (coupling, cascading failures).
- **Asynchronous** (queues, events): decouples producer from consumer, absorbs traffic spikes, survives downstream outages — but brings **eventual consistency**, harder debugging (no single stack trace), and duplicate/out-of-order handling.

**Lean sync when:** the caller genuinely needs the answer now (checkout must confirm payment). **Lean async when:** the work can happen "soon" and you want to absorb load and decouple (send the receipt email, resize the image, update the search index).

```js
// Sync: caller waits, sees the error immediately, but is blocked.
async function placeOrderSync(order) {
  await charge(order);        // if this is slow, the user waits
  await sendEmail(order);     // and this
  await updateInventory(order);
  return { status: "done" };
}

// Async: return fast, do the rest via a queue. Decoupled but eventually consistent.
async function placeOrderAsync(order) {
  await charge(order);                 // only the critical step is sync
  await queue.publish("ORDER_PLACED", order); // email/inventory happen later
  return { status: "accepted" };       // returns before side-effects finish
}
```

### 11. Cost vs Performance vs Reliability — the eternal triangle

You cannot maximize all three; pushing any corner out pulls budget from the others. From [07 — Availability and Reliability] and [90 — Disaster Recovery]: each extra "nine" of availability (99.9% → 99.99% → 99.999%) demands redundancy, multi-region replication, and standby capacity — real money for capacity that mostly sits idle. Likewise, shaving the last milliseconds off latency means more machines, more caches, more edge locations.

**Lean cost when:** the workload is internal, non-critical, or early-stage — three nines is plenty and a few seconds of latency is fine. **Lean performance/reliability when:** downtime or slowness directly loses revenue or trust (payments, trading, healthcare). The senior move is to state the target explicitly — "we need 99.95%, not 99.999%, and here's the money that saves."

### 12. Build vs Buy, and Simplicity vs Flexibility

**Build vs Buy:** building gives you exact fit and full control but costs engineering time and ongoing maintenance forever; buying (a managed service, a SaaS, an off-the-shelf queue) gives you speed-to-market and someone else's on-call, paid for in dollars, lock-in, and less control. **Lean buy** for anything that isn't your core differentiator (auth, email, payments infra); **lean build** for the thing that *is* your product.

**Simplicity vs Flexibility (YAGNI vs extensibility):** from [19 — SOLID and design principles]. Every abstraction you add "for future flexibility" is complexity you pay for *today* for a future that may never arrive (**YAGNI** — You Aren't Gonna Need It). But too little structure and the first real change requires a rewrite. **Lean simple** by default — you can always add abstraction when the second real use case actually shows up; premature flexibility is just complexity with good intentions.

---

## Visual / Diagram description

**The CAP / PACELC triangle** — the most-drawn diagram in system design:

```
                    CONSISTENCY (C)
                        /\
                       /  \
                      /    \        Pick any TWO...
                     /      \       but during a PARTITION,
                    /  CAP   \      P is forced on you, so you
                   /  space   \     really pick between C and A.
                  /            \
                 /______________\
   AVAILABILITY (A)          PARTITION-TOLERANCE (P)

   PACELC — the honest, fuller version:

   ┌─────────────────────────────────────────────────────┐
   │  IF there is a Partition (P):   choose  A   or   C   │
   │  ELSE (normal operation, E):    choose  L   or   C   │
   │                                (Latency)  (Consistency)│
   └─────────────────────────────────────────────────────┘

   DynamoDB:  PA/EL   (available + low-latency; eventual by default)
   MongoDB :  PC/EC   (consistent + consistent)  [config-dependent]
   Classic RDBMS: PC/EC (a single-leader SQL DB)
```

Read the triangle like this: you can be near any *edge* (favoring the two corners it connects), but never at all three corners at once. Since network partitions *will* happen (P is not optional in the real world), your live choice is which of the other two corners to sacrifice while partitioned.

**The trade-off dial** — the mental model to carry into every decision:

```
   Slow / Correct / Cheap          Fast / Approximate / Expensive
          |                                          |
          |   ●━━━━━━━━━━━━━━━●━━━━━━━━━━━━━━━━━━━━━●   |
          |   Strong         Bounded            Eventual
          |   consistency    staleness          consistency
          |                                          |
          └──────────────  YOU CHOOSE  ──────────────┘
             the dial position that fits the REQUIREMENTS
```

Every trade-off in this document is a dial, not a switch. It is almost never "on or off" — it's "how far, and *why that far*." The engineer's job is not to find the dial's "best" setting (there isn't one) but to place it deliberately given the requirements, and to say where they placed it and why.

---

## Real world examples

### Amazon DynamoDB — chose AP, on purpose

DynamoDB (and the original Dynamo paper that inspired it) is the textbook **AP** system. Faced with the CAP choice for Amazon's shopping cart, they picked **availability**: the cart must *always* accept "add to cart," even during a partition, because a rejected add is lost revenue. The price they paid — and named explicitly — is eventual consistency and the possibility of **conflicting versions** (an item you deleted reappearing), which they resolve with vector clocks and application-level merges. It reads as PA/EL in PACELC: available under partition, low-latency otherwise. This is representative of the documented design philosophy.

### A retail bank's core ledger — chose CP, on purpose

A bank's ledger makes the opposite call, and correctly. If the system is partitioned and can't confirm your balance, it **refuses the withdrawal** rather than risk letting you overdraw — sacrificing availability to protect consistency (**CP**). Combined with **serializable** isolation (concept #7) on transfers, the bank accepts lower throughput and occasional "please try again" errors as the fair price of never creating or destroying money. This is a conceptual illustration of standard financial-systems design.

### Twitter's timeline — fanout-on-write, then hybrid

Twitter's home timeline is the canonical **read-vs-write** story (concept #4). Early on they used **fanout-on-write**: when you tweet, they push it into every follower's precomputed timeline, so reading a timeline is one cheap lookup — read-optimized. But a celebrity with 100M followers turns one tweet into 100M writes (catastrophic write amplification). Their well-documented fix was a **hybrid**: fanout-on-write for ordinary users, and **fanout-on-read** (fetch-and-merge at read time) for the handful of mega-accounts — explicitly trading a bit of read latency on celebrity content to escape the write-amplification cliff. This is a widely-cited, representative account.

---

## Trade-offs

The whole document is trade-offs, so here is the **master matrix** — the single table to memorize:

| Trade-off axis | You gain | You pay | Lean this way when... |
|----------------|----------|---------|-----------------------|
| Consistency vs Availability | Correct data | Latency + downtime under partition | Strong for money/bookings; Eventual for feeds/likes |
| Latency vs Throughput | Fast single response | Fewer requests/sec (or vice versa) | Latency for humans waiting; Throughput for bulk/background |
| Latency vs Consistency | Fast nearest-replica reads | Possible staleness | Nearest replica for static reads; Leader for read-your-writes |
| Read vs Write optimization | Instant reads | Write amplification/complexity | Read-opt at high read:write ratios; Write-opt for firehoses |
| Space vs Time | Speed (precompute/cache) | Memory + staleness | Space-for-time when a result is re-read often |
| Normalization vs Denorm. | Consistent, small writes | Expensive joins | Normalize for write-heavy OLTP; Denorm for read models |
| Isolation vs Performance | Correctness (serializable) | Throughput, contention | Serializable for money; weaker when rows rarely conflict |
| SQL vs NoSQL | Transactions + flexible queries | Harder horizontal scaling | SQL for rich/uncertain domains; NoSQL for known huge-scale patterns |
| Monolith vs Microservices | Independent scaling + autonomy | Distributed complexity | Monolith when small/young; micro when org + boundaries mature |
| Sync vs Async | Simplicity + traceability | Coupling / eventual consistency | Sync when answer needed now; Async to decouple + absorb load |
| Cost/Perf/Reliability | More nines, more speed | More money | Spend only for the nines the business actually needs |
| Build vs Buy | Exact fit vs speed-to-market | Maintenance vs lock-in | Buy the non-core; build the differentiator |
| Simplicity vs Flexibility | Less complexity now | Rework later (or the reverse) | Stay simple (YAGNI) until a real second use case appears |

**Rule of thumb:** the right position on every dial is a *function of the requirements*, never a universal constant. If someone tells you their favorite database/pattern/architecture without asking about your requirements first, they're selling, not engineering.

---

## How to reason about a trade-off in an interview

This four-step move is what converts "I know things" into "I think like a senior." Do it out loud, every time you make a non-trivial choice. Recall the interview method from [93 — HLD Approach Framework](./93-hld-approach-framework.md) — this slots directly into its decision moments.

1. **Name it explicitly.** "This is a consistency-vs-availability decision." Just naming the axis already puts you ahead of most candidates.
2. **State both sides.** "If I go strong, I get correctness but pay latency and I'm unavailable during partitions. If I go eventual, I get availability and speed but users may see stale counts."
3. **Tie it to the specific requirements.** "The requirement said this is a social feed, read-heavy, and a few seconds of staleness in a like-count is invisible to users."
4. **Commit with a justification.** "So I'll trade consistency for availability and latency here, and use eventual consistency — I'd only revisit that if we later put money on this path."

The full sentence sounds like: *"Given that this system is read-heavy and can tolerate staleness, I'll trade consistency for availability and latency here."* That one sentence contains the axis, both sides, the requirement, and the commitment. It is the sound of senior thinking.

**The two failure modes to avoid:**
- **Not naming the trade-off** — making a choice silently, as if it were free. Interviewers read this as "doesn't know what they gave up."
- **Refusing to commit** — listing options forever without picking. Interviewers read this as "can't make decisions." Always land the plane: name the axis, then *choose*, with a reason tied to the requirements. A committed, justified choice beats a "perfect" waffle every time. See [140 — System Design Interview Playbook](./140-system-design-interview-playbook.md) for how this fits the whole 45-minute arc.

---

## Common interview questions on this topic

### Q1: "You're designing a payment system — which trade-off would you make on consistency, and why?"
**Hint:** Name it as consistency-vs-availability. Payments are money: a wrong balance or double-charge is unacceptable. Lean **strong / CP** with **serializable** isolation on the transfer path — you accept higher latency and occasional "try again" errors to guarantee money is never created or destroyed. Note that you can still make *non-critical* paths (receipt email, analytics) async and eventual.

### Q2: "You're designing a social media feed — which trade-off, and why?"
**Hint:** Read-heavy, staleness-tolerant. Lean **AP / eventual consistency** for like and view counts, **read-optimized** with fanout-on-write for ordinary users, and a **hybrid** (fanout-on-read) for celebrities to dodge write amplification. Say out loud that a few seconds of stale count is invisible, so trading consistency for availability and latency is clearly correct here.

### Q3: "PACELC — what does it add over CAP, and why should I care?"
**Hint:** CAP only describes the *partition* case. PACELC adds the **Else** clause: even in normal operation you trade **Latency vs Consistency** on every read (nearest replica vs leader). It matters because partitions are rare but the latency/consistency choice is *constant* — CAP alone makes you think the trade-off only exists during failures.

### Q4: "When would you deliberately choose a monolith over microservices?"
**Hint:** Name simplicity-vs-distributed-complexity. Choose the monolith when the team is small, the domain boundaries are still unknown, and you don't yet have scale that justifies the operational tax. Committing to microservices early buys distributed-systems pain before you have the scale that pays for it — a classic premature-optimization trap.

### Q5: "Your design is read-heavy at 100:1. Walk me through the trade-offs you'd make."
**Hint:** Lean hard into read optimization: cache (space-for-time), denormalized read models (read-vs-write), indexes, materialized views, nearest-replica reads (latency-vs-consistency). Explicitly acknowledge the bill: write amplification, invalidation complexity, and staleness — all acceptable because reads dominate 100:1.

---

## Practice exercise

### Name the dominant trade-off, pick a side, and justify it

For each of the three mini-scenarios below, write **three sentences**: (1) name the dominant trade-off axis, (2) state both sides briefly, (3) commit to a side with a justification tied to the scenario's requirements. Spend ~25-30 minutes. The goal is to build the *reflex* of reasoning in trade-offs out loud.

**Scenario A — Flash-sale inventory counter.** An e-commerce site sells 500 limited-edition sneakers. Ten thousand people hit "buy" in the same 3 seconds. What is the dominant trade-off, and which side do you pick? *(Think: what happens if you oversell? What does that force about consistency vs availability?)*

**Scenario B — IoT sensor ingestion.** One million temperature sensors each send a reading every 10 seconds; the data is read only occasionally, in daily analytics batches. What's the dominant trade-off, and which side? *(Think: read-vs-write optimization, latency-vs-throughput, SQL-vs-NoSQL.)*

**Scenario C — A "who viewed your profile" counter on a professional network.** Millions of profile views per minute; the count shown can lag reality by a minute without anyone noticing. Dominant trade-off, and which side? *(Think: consistency-vs-availability and space-vs-time.)*

**What to produce:** nine sentences total (three per scenario). Then, for one scenario, write the single "given that... I'll trade X for Y here" sentence in full. If you can do that fluently, you have the core senior skill this entire course was building toward.

---

## Quick reference cheat sheet

- **No free lunch:** every choice trades one property for another. There is no "best" — only "best *for these requirements*."
- **CAP:** during a partition, pick Consistency **or** Availability — proven, not negotiable.
- **PACELC:** the honest version — Partition → A/C; **Else → Latency/Consistency**. The trade never fully turns off.
- **Latency vs Throughput:** batching helps throughput, hurts latency. Optimize per the workload: humans → latency, bulk → throughput.
- **Read vs Write:** indexes, caches, denormalization, and fanout-on-write all speed reads by paying at write time.
- **Space vs Time:** cache/precompute to spend memory for speed; recompute to spend CPU for memory.
- **Normalize vs Denormalize:** normalized = clean writes, costly joins; denormalized = fast reads, update anomalies.
- **Isolation vs Perf:** serializable = correct but slow; weaker = fast but admits anomalies.
- **SQL vs NoSQL:** transactions/queries vs horizontal scale/flexible schema.
- **Monolith vs Micro:** simplicity vs independent scaling, at the cost of distributed complexity. Don't reach for micro too early.
- **Sync vs Async:** simple + traceable vs decoupled + load-absorbing but eventually consistent and harder to debug.
- **Cost/Perf/Reliability triangle:** every extra "nine" and every millisecond costs money. Target only the nines the business needs.
- **Build vs Buy / Simple vs Flexible:** buy the non-core, build the differentiator; stay simple (YAGNI) until a real need appears.
- **The senior move:** name the trade-off, state both sides, tie it to requirements, and COMMIT with a justification.

---

## Connected topics

| Direction | Topic | Why |
|-----------|-------|-----|
| **Previous** | [09 — CAP Theorem](./09-cap-theorem.md) | The foundational trade-off (C vs A) that this document generalizes into a full framework |
| **Next** | [140 — System Design Interview Playbook](./140-system-design-interview-playbook.md) | Puts this trade-off reasoning to work inside the full 45-minute interview arc |
| **Related** | [06 — Latency and Throughput](./06-latency-and-throughput.md) | Defines the latency/throughput axis (trade-off #2) in depth |
| **Related** | [08 — Consistency Models](./08-consistency-models.md) | The full spectrum between strong and eventual — the dial behind trade-offs #1 and #3 |
| **Related** | [93 — HLD Approach Framework](./93-hld-approach-framework.md) | The interview method these trade-off decisions slot into at every design step |
