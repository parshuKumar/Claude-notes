# 136 — Cost Optimization in System Design
## Category: Advanced

---

## What is this?

**Cost optimization** is the discipline of building systems that meet their performance and reliability goals for the *least amount of money*. Every architecture decision — which instance type, which database, how many replicas, where the data lives — is also silently a decision about the monthly cloud bill.

Think of it like running a restaurant. A great chef can cook a beautiful meal. A great *owner* can cook that same meal, keep the tables full, and still turn a profit — because they know the cost of every ingredient, notice when the walk-in fridge is left open overnight, and don't buy a commercial-grade oven to fry one egg. In software, "it scales" is worthless if it bankrupts the company along the way.

---

## Why does it matter?

Cost is the dimension juniors ignore and seniors obsess over. A system that works but costs 3x what it should is a system that gets you a budget freeze, a re-architecture mandate, and an uncomfortable meeting with finance.

- **The business runs on margins, not on uptime.** If serving each user costs more than that user pays you, growth *loses* money. More users make the problem worse, not better.
- **Cloud bills grow silently.** Nobody approves a $40,000/month bill in one go. It creeps: an over-provisioned cluster here, a forgotten staging environment there, an egress pattern nobody measured. Six months later it's real money.
- **An engineer who cuts the bill 40% without hurting performance is enormously valuable.** That is a direct, measurable contribution to profit. Very few engineers can do it, because it requires understanding the whole system *and* the pricing sheet.

**Interview angle:** Mentioning cost trade-offs *unprompted* in a design interview is one of the strongest signals of seniority you can send. Anyone can add a cache; a senior says "I'll add a cache here, and it also cuts our database bill because a cache hit is an order of magnitude cheaper than a query." That sentence gets you hired.

**Real-work angle:** At some point a director will drop a spreadsheet on your desk and say "our AWS bill doubled, find out why." The engineers who can read that bill, tie line items back to architecture, and propose fixes become indispensable.

**The mindset shift:** stop thinking in absolute dollars and start thinking in **unit cost** — cost *per request*, cost *per user*, cost *per GB*. A $50,000/month bill serving 50 million users ($0.001/user) is healthy. The same bill serving 5,000 users ($10/user) is a fire. The unit is what tells you whether you're winning.

---

## The core idea — explained simply

### The Household Utility Bill Analogy

Imagine your cloud bill is a household's monthly expenses. When money gets tight, a smart household doesn't move to a cheaper city (re-architect everything) as the first move. They look at *where the money actually goes* and attack the biggest, dumbest line items first.

- **The heating bill (compute).** The furnace runs 24/7 even when nobody's home. Fix: a smart thermostat that turns it down when the house is empty — that's **autoscaling**. And you realize you bought a furnace sized for a mansion when you live in a cottage — that's **right-sizing**.
- **The storage unit across town (cold storage).** You're paying premium climate-controlled rates to store boxes you haven't opened in three years. Move them to a cheap long-term unit — that's **archival storage tiers**.
- **The water bill from a running tap (data egress).** A tap left running in the yard that you never see on the meter until the bill arrives. This is the silent killer: **cross-region and internet egress**.
- **Buying vs. renting an appliance (build vs. buy).** You could buy a cheap washing machine and fix it yourself every month, or rent a serviced one that costs more but never eats your Saturdays. The "expensive" one might be cheaper once you count *your time*.
- **The subscription you forgot to cancel (orphaned resources).** The gym membership nobody uses. In the cloud these are unattached disks, idle load balancers, and that dev environment someone spun up in 2023.

Map it back:

| Household thing | System concept |
|---|---|
| Furnace running when house is empty | Instances running 24/7 at low utilization |
| Smart thermostat | Autoscaling — pay for peak only during peak |
| Furnace sized for a mansion | Over-provisioned instances → right-sizing |
| Cheap long-term storage unit | Archival tiers (S3 Glacier, cold blob storage) |
| Running tap you don't see metered | Data egress — the silent killer |
| Rent vs. buy an appliance | Managed service vs. self-hosted (count engineer-hours) |
| Forgotten gym membership | Idle/orphaned cloud resources |
| Reading the itemized bill | FinOps: tagging, budgets, cost attribution |

The whole discipline is: **know where the money goes, exploit the pricing models, and pay only for what you actually use.**

---

## Key concepts inside this topic

### 1. Where the money actually goes — the big buckets

You can't optimize what you can't see. Almost every cloud bill breaks into five buckets. Rough relative magnitudes for a typical web/API company (yours will vary — *measure* before you assume):

| Bucket | What it is | Rough share | The trap |
|---|---|---|---|
| **Compute** | VMs, containers, serverless functions | 40–60% | Over-provisioned, running 24/7 |
| **Database** | Managed DB/cache (often the single priciest line) | 15–30% | Oversized instance, over-replicated |
| **Storage** | Block, object, backups | 5–15% | Cold data sitting on hot SSD tiers |
| **Data transfer / egress** | Bytes leaving a region or going to the internet | 5–20% | Invisible until the bill; cross-region chatter |
| **Third-party APIs** | LLM tokens, payments, SMS, maps | 0–30% | Scales linearly with usage, no economies of scale |

The two that surprise people are **database** (a managed Postgres or a big Redis cluster is often the most expensive single service you run) and **egress** — the silent killer. Recall from [59 — Caching in Depth](./59-caching-in-depth.md) and CDN concepts that moving bytes *out* of a cloud region to the internet, or *between* regions, is metered and priced per GB, while moving bytes *in* is usually free. It's easy to build a chatty cross-region architecture and never notice until a five-figure "Data Transfer Out" line appears.

### 2. The pricing models and how to exploit them

Cloud compute is sold under several pricing models. Knowing them is half the battle, because the same VM can cost wildly different amounts depending on *how* you buy it.

| Model | Discount vs on-demand | Best for | Catch |
|---|---|---|---|
| **On-demand** | baseline (0%) | Unpredictable, short-lived, spiky | Most expensive per hour |
| **Reserved / committed-use** | 30–60% off | Steady 24/7 **baseline** load | 1–3 year commitment, pay even if idle |
| **Spot / preemptible** | 60–90% off | Interruptible **batch** work | Can be reclaimed with ~2 min notice |
| **Serverless (per-invocation)** | pay-per-use | **Spiky**, low-average traffic | Expensive when load is steady & high |

The pattern that experienced teams converge on:

> **Reserved instances for your steady baseline + on-demand or spot for the peaks.**

Your traffic almost always has a floor — the load that's there at 3 a.m. — plus spikes on top. Cover the floor with cheap reserved capacity you've committed to, and handle the spikes with flexible on-demand that you only pay for when the spike happens.

**Spot** is a superpower for the batch and stream jobs from [87 — Batch vs Stream Processing](./87-batch-vs-stream-processing.md). A nightly analytics job or a video-encoding queue doesn't care if a worker dies mid-task — the work item just gets retried on another worker. Paying 80% less for compute that "might disappear" is a fantastic deal when your workload is designed to tolerate it.

**Serverless** (see [89 — Serverless Architecture](./89-serverless-architecture.md)) has a *crossover point*. Because you pay per invocation with zero idle cost, it's the cheapest option for spiky or low-volume workloads. But per unit of steady compute it's more expensive than a reserved VM. There's a break-even traffic level: below it, serverless wins; above it, a reserved fleet wins. Knowing roughly where that line sits for your workload is exactly the kind of judgment interviewers probe.

```js
// A back-of-envelope helper: at what monthly request volume does a
// reserved VM become cheaper than serverless functions?
function crossoverRequestsPerMonth({
  reservedMonthlyCost,      // e.g. $70/mo for a small reserved VM
  serverlessCostPerMillion, // e.g. $0.40 per 1M invocations + compute
}) {
  // Below this volume, serverless is cheaper (you pay only per call).
  // Above it, the flat reserved cost wins.
  const millionsAtBreakeven = reservedMonthlyCost / serverlessCostPerMillion;
  return millionsAtBreakeven * 1_000_000;
}

// ~175M requests/month is the rough break-even here.
console.log(crossoverRequestsPerMonth({
  reservedMonthlyCost: 70,
  serverlessCostPerMillion: 0.40,
})); // 175000000
```

### 3. Right-sizing — stop paying for headroom you never use

The single most common waste in the cloud: instances provisioned "just in case," running at 8% CPU forever. Someone picked a big instance during a launch scare and nobody ever revisited it.

Right-sizing is the boring, high-ROI practice of *measuring real utilization and shrinking to fit*. If a service peaks at 25% CPU and 30% memory on an 8-vCPU box, it belongs on a 2-vCPU box — a 4x cost cut for zero performance loss.

```js
// Flag over-provisioned instances from utilization metrics.
function rightSizeRecommendation(instance) {
  const { vcpus, peakCpuUtil, peakMemUtil } = instance;
  // Target ~60% peak utilization: enough headroom for spikes,
  // no huge idle waste. Recall horizontal-vs-vertical scaling (56).
  const TARGET = 0.60;
  const cpuDrivenVcpus = Math.ceil((vcpus * peakCpuUtil) / TARGET);
  const recommended = Math.max(1, cpuDrivenVcpus);
  return {
    current: vcpus,
    recommended,
    action: recommended < vcpus ? `DOWNSIZE to ${recommended} vCPU` : "OK",
  };
}

console.log(rightSizeRecommendation({ vcpus: 8, peakCpuUtil: 0.25, peakMemUtil: 0.30 }));
// { current: 8, recommended: 4, action: 'DOWNSIZE to 4 vCPU' }
```

Pair right-sizing with **autoscaling** so you pay for peak capacity *only during peak*. A fleet sized for Black Friday but running at that size every Tuesday at 4 a.m. is burning money 360 days a year. Recall [56 — Horizontal vs Vertical Scaling](./56-horizontal-vs-vertical-scaling.md): autoscaling is horizontal scaling driven by a metric, and it's as much a cost tool as a performance tool.

### 4. The cost / performance / reliability triangle

You cannot maximize all three of cost, performance, and reliability at once — improving one usually costs you another. This is the trade-off framing from [133 — Trade-off Analysis] applied to money.

```
              PERFORMANCE
               (low latency,
                high throughput)
                    /\
                   /  \
                  /    \
                 /      \
        COST ---/--------\--- RELIABILITY
      (cheap bill)      (every extra nine
                         of availability)
```

Every "nine" of availability and every millisecond of latency has a price:

| Availability | Downtime/year | Rough cost signal |
|---|---|---|
| 99% | ~3.65 days | 1 region, minimal redundancy — cheap |
| 99.9% | ~8.8 hours | multi-AZ, some redundancy — moderate |
| 99.99% | ~52 min | multi-region, hot standbys — expensive |
| 99.999% | ~5 min | full active-active everywhere — very expensive |

The senior question is not "how do I get five nines?" but **"what does the business actually need?"** Chasing 99.999% for an internal analytics dashboard when 99.9% is fine burns money on redundancy nobody will ever benefit from. Match the spend to the actual requirement. A payment system genuinely needs the nines; a "trending topics" widget does not.

### 5. Caching as cost optimization

You already know from [59 — Caching in Depth](./59-caching-in-depth.md) that caching improves *latency*. What juniors miss is that it also directly cuts the *bill*: **a cache hit is far cheaper than a database query or a recompute.** Every request the cache absorbs is a request your (expensive) database never sees.

Worked example. Suppose:

- 100 million requests/month hit an endpoint.
- Each database query costs you ~$0.0000020 in DB load (a fraction of your managed-DB and read-replica spend, amortized).
- A Redis cache hit costs ~$0.0000002 — about 10x cheaper.

```js
function cacheSavings({ monthlyRequests, hitRatio, dbCostPerQuery, cacheCostPerHit }) {
  const hits = monthlyRequests * hitRatio;
  const misses = monthlyRequests - hits;
  // Without cache: every request is a DB query.
  const withoutCache = monthlyRequests * dbCostPerQuery;
  // With cache: hits cost the cheap cache price, misses still hit the DB.
  const withCache = hits * cacheCostPerHit + misses * dbCostPerQuery;
  return {
    withoutCache: +withoutCache.toFixed(2),
    withCache: +withCache.toFixed(2),
    saved: +(withoutCache - withCache).toFixed(2),
  };
}

console.log(cacheSavings({
  monthlyRequests: 100_000_000,
  hitRatio: 0.90,          // 90% of reads served from cache
  dbCostPerQuery: 0.0000020,
  cacheCostPerHit: 0.0000002,
}));
// { withoutCache: 200, withCache: 38, saved: 162 }
```

A 90% hit ratio turned a $200/month database read cost into $38 — an 81% cut on that line, *and* everything got faster. That's the dream trade: cheaper and better at the same time. This is why "what's your cache hit ratio?" is really a cost question in disguise. CDNs (content delivery networks) do the identical trick for *egress*: a byte served from a CDN edge is a byte your origin never had to send, cutting both origin compute and expensive egress.

### 6. Storage cost optimization

Storage is cheap per GB but enormous in aggregate, and most of it is cold — written once, never read again. Recall the storage tiers from [91 — Data Lakes and Warehouses](./91-data-lakes-and-warehouses.md) and blob-storage tiering: hot SSD is pricey, cold/archive tiers are 5–20x cheaper.

Levers, in order of impact:

- **Lifecycle policies.** Automatically move objects from hot → infrequent-access → archive as they age. A 90-day-old log doesn't belong on the same expensive tier as today's writes.
- **Compression.** Text, logs, and JSON compress 5–10x. Compressing before you store (and before you *transfer*) cuts both storage and egress.
- **Deduplication.** Don't store the same backup or asset ten times.
- **Delete data nobody queries.** The cheapest byte is the one you don't store. Ruthlessly expire what has no business or legal reason to exist.

**The stealth cost: logs and observability data.** Recall [80 — Monitoring and Observability] — verbose logging, high-cardinality metrics, and full-fidelity traces can quietly become one of your largest storage *and* ingestion bills. The fixes: **sample** (keep 1-in-N traces, not all of them) and set aggressive **retention** (do you really need debug logs from 18 months ago?). Many teams discover their logging vendor bill rivals their compute bill.

### 7. Data transfer optimization

Egress is the line item people forget until it bites. Four defenses:

- **Keep chatty services in the same AZ/region.** Traffic *within* an availability zone is usually free; *across* regions is metered per GB. A microservice that makes 20 calls to another service per request should not be paying cross-region rates for each hop.
- **Compress payloads.** Fewer bytes on the wire = lower egress and faster responses.
- **Use a CDN.** Serve static and cacheable content from edges so your origin isn't paying premium egress to reach every user directly.
- **Avoid needless cross-region replication.** Replicating everything everywhere (see [86 — Data Replication Strategies]) is a reliability lever with a real egress price tag. Replicate what needs it, not reflexively everything.

### 8. Build vs. buy, and managed vs. self-hosted — count the people

Here's the calculus juniors get wrong: they compare only the *sticker price*. A managed database might cost $500/month while a self-hosted one on raw VMs costs $200/month, so self-hosting "saves $300." But self-hosting means *someone* patches it, tunes it, handles failover at 3 a.m., manages backups, and upgrades versions. If that eats even four engineer-hours a month at a loaded cost of ~$100/hour, the "cheap" option is now $600 — *more* expensive than the managed one, before counting the risk of a botched failover.

> The **total cost** of a component = infrastructure cost **+** the human cost of operating it.

Sometimes the "expensive" managed service is genuinely cheaper than a team babysitting a self-hosted one. Sometimes, at large scale, self-hosting a mature workload pays off because the per-unit managed premium finally exceeds the human cost. The point is to *do the whole sum*, not just read the price tag.

### 9. FinOps — making cost visible and owned

FinOps is the practice of running cost like any other engineering metric. The core moves:

- **Tag every resource** by team, service, and feature. Untagged resources are unattributable — you can't fix what you can't blame on a chart. Tagging turns "our bill is $40k" into "recommendations service is $12k, and 60% of that is one over-provisioned cache."
- **Budgets and alerts.** Set a monthly budget per team and fire an alert at 80%, so you find the runaway *this week*, not on next month's invoice.
- **Hunt idle and orphaned resources.** Unattached disks, idle load balancers, unassociated IPs, and forgotten dev/staging environments that run nights and weekends for no one. These are pure waste and usually 5–15% of a bill.
- **Make cost visible to engineers.** When the team that spins up a resource sees its price, behavior changes on its own. Cost dashboards next to the latency dashboards.

---

## Visual / Diagram description

A monthly cloud bill, decomposed, with an optimization lever attached to each bucket:

```
              MONTHLY CLOUD BILL  ($100k, illustrative)
   ┌──────────────────────────────────────────────────────────┐
   │                                                            │
   │  COMPUTE          ██████████████████████  50%   $50k       │◀─ right-size,
   │                                                            │   autoscale,
   │                                                            │   reserved+spot
   │  DATABASE         ████████████  25%           $25k         │◀─ cache in front,
   │                                                            │   right-size, fewer
   │                                                            │   read replicas
   │  EGRESS           ██████  12%                 $12k         │◀─ CDN, compress,
   │                                                            │   same-AZ, no
   │                                                            │   cross-region chatter
   │  STORAGE          ████  8%                    $8k          │◀─ lifecycle tiers,
   │                                                            │   compress, delete,
   │                                                            │   sample logs
   │  3RD-PARTY APIs   ██  5%                       $5k         │◀─ cache responses,
   │                                                            │   batch calls
   └──────────────────────────────────────────────────────────┘
                                │
                                ▼
        FinOps layer (cuts across everything):
   ┌──────────────────────────────────────────────────────────┐
   │  TAG by team/feature  │  BUDGET + ALERTS  │  KILL orphans  │
   └──────────────────────────────────────────────────────────┘
```

Read it top to bottom: compute is almost always the biggest bar, so it's where the biggest absolute savings live (a 20% cut on a 50% bucket beats a 50% cut on a 5% bucket). Each bar has a lever pointing at it — the *right* tool for *that* bucket. Underneath, the FinOps layer isn't a bucket; it's the cross-cutting visibility that lets you *find* which bars to attack. Without the bottom layer you're optimizing blind.

The key insight the diagram teaches: **attack the biggest bucket first.** Beginners love shaving pennies off the small bars because they're easy; the money is in the big ones.

---

## Real world examples

### Netflix (representative)

Netflix runs one of the largest cloud footprints in the world and is famously public about cost engineering. They lean heavily on **reserved capacity for baseline** streaming load and flexible capacity for spikes, and they push enormous volumes of video through their own **Open Connect CDN** — placing caching appliances inside ISP networks so that video bytes are served as close to viewers as possible. That is egress and origin-compute optimization at planetary scale: the byte served from an ISP-local cache is a byte the core never has to send.

### Dropbox (representative)

Dropbox famously *reversed* a build-vs-buy decision. In its early years it stored user files on managed cloud object storage. As it reached exabyte scale, the per-unit managed premium finally exceeded the cost of running its own storage infrastructure ("Magic Pocket"), and it migrated much of its storage off the public cloud, reportedly saving very large sums. The lesson is *directional*, not universal: at small scale, buy; at extreme scale with a stable, well-understood workload, self-hosting can win. The crossover is real and worth computing.

### Airbnb / typical FinOps shops (representative)

Large product companies commonly run dedicated cost/FinOps efforts: resource tagging enforced at deploy time, per-team cost dashboards, automated hunts for idle resources, and spot instances for their data pipelines. The common thread across mature orgs is not one clever trick — it's *making cost a visible, owned, continuously-managed metric* rather than a quarterly surprise.

---

## Trade-offs

**Reserved vs. on-demand:**

| | Pros | Cons |
|---|---|---|
| **Reserved / committed** | 30–60% cheaper for steady load | Locked in 1–3 yrs; pay even if idle; risky if usage shrinks |
| **On-demand / spot** | Flexible, no commitment; spot is 60–90% off | On-demand pricey; spot can vanish mid-task |

**Optimizing cost vs. other goals:**

| | Pros | Cons |
|---|---|---|
| **Aggressive cost cutting** | Lower bill, better margins, higher unit efficiency | Can hurt latency/reliability if you cut too deep; engineer time to do it |
| **"Just throw money at it"** | Fast, simple, safe headroom | Waste compounds; unsustainable margins; eventual painful re-architecture |

**Managed vs. self-hosted:**

| | Pros | Cons |
|---|---|---|
| **Managed service** | Saves engineer-hours; ops handled; fast | Higher per-unit price; less control |
| **Self-hosted** | Cheaper per unit at scale; full control | Real human ops cost + 3 a.m. failovers |

**The sweet spot:** Reserved capacity for your steady baseline, spot for interruptible batch, on-demand/serverless for the spiky edges — right-sized, cache-fronted, CDN-served, and tagged so you can *see* the bill. Match every dollar to an actual business requirement, and never chase a nine or a millisecond the business didn't ask for.

**Rule of thumb:** Optimize the biggest bucket first, measure before you cut, and always express cost as a *unit* (per request / per user / per GB) so growth doesn't hide waste.

---

## Common interview questions on this topic

### Q1: "This design works — how would you make it cheaper without hurting the user?"

**Hint:** Walk the buckets. Add a cache to offload the expensive database (cheaper *and* faster). Add a CDN to cut egress and origin load. Right-size over-provisioned instances using real utilization. Move steady baseline to reserved instances and batch jobs to spot. Emphasize you'd *measure* first to find the biggest bucket, then attack it.

### Q2: "When would you choose serverless over a reserved fleet, purely on cost?"

**Hint:** Serverless wins for spiky or low-average traffic because idle cost is zero — you pay per invocation. A reserved fleet wins for steady, high-volume load because the flat committed price beats per-call pricing above a crossover point. Sketch the break-even: below X requests/month, serverless; above it, reserved. Reference the crossover from [89 — Serverless Architecture].

### Q3: "Our cloud bill doubled but traffic didn't. Where do you look?"

**Hint:** Start with tagging/cost-explorer to find the bucket that grew. Common culprits: a new cross-region data path spiking *egress*, an over-provisioned cluster from a launch that never got scaled back down, exploding *log/observability* storage, orphaned resources (unattached disks, idle load balancers, forgotten staging envs), or a third-party API (e.g., LLM tokens) scaling with a new feature. Measure, attribute, then fix the biggest first.

### Q4: "How do you decide managed database vs. self-hosting it?"

**Hint:** Compute *total* cost = infra + human ops. Managed costs more per unit but saves engineer-hours, patching, backups, and 3 a.m. failovers. Self-hosting is cheaper per unit but only pays off at scale with a stable workload and a team to run it. Do the full sum including loaded engineer hourly cost; don't compare sticker prices alone.

### Q5: "What availability target would you design for, and why?"

**Hint:** Flip it into a cost question. Ask what the business actually needs — every extra nine multiplies redundancy cost. 99.9% (multi-AZ) is fine for most services; 99.99%+ (multi-region active-active) is expensive and only justified for revenue-critical or safety-critical paths. Chasing nines nobody needs is burning money.

---

## Practice exercise

### Estimate and then cut a system's monthly bill

Take this simple system and produce a **before/after** cost estimate on paper (or in a small JS script):

**The system (before):**
- 10 web servers on **on-demand** general-purpose VMs, `$0.10/hr` each, running 24/7.
- 1 managed Postgres, oversized: `$800/month`.
- 5 TB of object storage, *all* on the hot tier at `$0.023/GB-month`, but 4 TB of it hasn't been read in 90 days.
- 50 TB/month of internet egress at `$0.09/GB`, served **directly from origin** (no CDN).
- No cache.

**Step 1 — compute the current monthly bill.** Show your arithmetic for each bucket (hours × rate, GB × rate). Add them up.

**Step 2 — apply four optimizations and recompute:**
1. Move the 10 servers to **reserved** instances at a 40% discount, and **right-size** to 6 servers (real peak needs 6).
2. Add a **cache** with a 90% hit ratio in front of Postgres, letting you **right-size** the DB down to `$400/month`.
3. Add a **CDN** with an 80% offload ratio, so only 20% of egress comes from origin (and CDN egress at, say, `$0.02/GB`).
4. Apply a **lifecycle policy** moving the 4 TB of cold data to an archive tier at `$0.004/GB-month`.

**What to produce:** a two-column table (Before / After) per bucket, the two totals, and the overall percent saved. You should land somewhere around a 50–70% reduction. Write one sentence on which single optimization saved the most, and why it was the biggest bucket that made it so.

---

## Quick reference cheat sheet

- **Unit cost is the metric.** Always think cost *per request / per user / per GB*, not absolute dollars — it survives growth.
- **Five buckets:** compute (biggest), database (often priciest single service), egress (silent killer), storage, third-party APIs. Attack the biggest first.
- **Egress is the one people forget** — cross-region and internet-out are metered per GB; ingress is usually free.
- **Reserved for baseline + on-demand/spot for peaks.** Commit to your floor, stay flexible for spikes.
- **Spot/preemptible = 60–90% off** for interruptible batch/stream work — design jobs to tolerate being killed.
- **Serverless has a crossover point** — cheapest when spiky, expensive when steady and high-volume.
- **Right-size everything** — most instances run at single-digit utilization; measure real load and shrink.
- **Autoscale** so you pay for peak *only during peak*, not 24/7.
- **A cache hit is ~10x cheaper than a DB query** — a good hit ratio cuts your database bill directly.
- **CDN cuts egress AND origin load** — an edge byte is a byte the origin never sends.
- **Lifecycle policies** move cold data to archive tiers (5–20x cheaper); compress, dedupe, delete.
- **Logs/observability are a stealth cost** — sample traces and set aggressive retention.
- **Total cost = infra + people.** The "expensive" managed service can be cheaper than babysitting a self-hosted one.
- **FinOps basics:** tag by team/feature, set budgets + alerts, kill orphaned resources (unattached disks, idle LBs, forgotten dev envs).
- **Match spend to need** — every extra nine and every saved millisecond costs money; don't buy what the business didn't ask for.

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [59 — Caching in Depth](./59-caching-in-depth.md) | Caching is a primary cost lever: a cache hit is far cheaper than a DB query or recompute. |
| **Next** | [137 — Performance Optimization](./137-performance-optimization.md) | The other side of the trade-off triangle — cost and performance are optimized together. |
| **Related** | [89 — Serverless Architecture](./89-serverless-architecture.md) | Per-invocation pricing and its crossover point vs. reserved fleets. |
| **Related** | [56 — Horizontal vs Vertical Scaling](./56-horizontal-vs-vertical-scaling.md) | Autoscaling and right-sizing are scaling decisions that are also cost decisions. |
| **Related** | [91 — Data Lakes and Warehouses](./91-data-lakes-and-warehouses.md) | Storage tiering and lifecycle policies for cutting cold-data storage cost. |
