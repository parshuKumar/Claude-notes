# 134 — Multi-Region Architecture and Global Systems
## Category: Advanced

---

## What is this?

**Multi-region architecture** means running your system in more than one geographic location — say `us-east` (Virginia), `eu-west` (Ireland), and `ap-southeast` (Singapore) — at the same time, so that users are served by the copy closest to them and the whole system survives an entire region going dark.

Think of a global newspaper. It could print every copy in one city and airmail them worldwide (slow, and if that one printing plant burns down, nobody gets a paper). Or it could run a printing plant on every continent, each with its own copy of today's edition. Readers get their paper faster, and a fire in one plant doesn't stop the others. The hard part is keeping every plant printing *the same edition* — that synchronization problem is the entire subject of this document.

---

## Why does it matter?

If you only ever deploy to one region, three things eventually hurt you:

- **Latency for far-away users.** Data cannot travel faster than light. A round trip from Sydney to Virginia is physically ~160-200ms *before* your server does any work. Recall the speed-of-light limit: light in fiber travels ~200,000 km/s, and Sydney↔Virginia is ~16,000 km each way. A user on the other side of the planet feels every page load drag.
- **No disaster resilience.** Recall [90 — Disaster Recovery](./90-disaster-recovery.md): multi-AZ (multiple datacenters in one region) survives a *datacenter* failure, but a region-wide outage — a fiber cut, a bad config push, a natural disaster, an AWS control-plane meltdown — takes your entire product down. It has happened to every major cloud.
- **Data-residency law.** GDPR and similar laws (India's DPDP, China's PIPL, Russia's localization law) can require that citizens' personal data physically stays inside their country or economic zone. A single US region is illegal for those users.

**Interview angle:** "How would you make this global?" is the natural follow-up to any HLD once you've nailed the single-region design. Interviewers want to hear you reach for *active-passive vs active-active* and immediately confront *cross-region write consistency* — not hand-wave "just add more regions."

**Real-work angle:** The day your company signs its first big EU customer, or the day `us-east-1` has a 6-hour outage and your CEO asks "why were we down," multi-region stops being theoretical.

---

## The core idea — explained simply

### The Global Restaurant Chain Analogy

You run a restaurant famous for one thing: a live-updated specials menu that changes through the day.

**One location (single region).** Everyone in the world must fly to your city to eat. Locals love it; someone in Tokyo spends 14 hours traveling for lunch. And if your building floods, the whole chain is closed — there is no other building.

**One kitchen, many pickup windows in the same city (multi-AZ).** You open three pickup windows across town, all fed by the same central kitchen. If one window closes, the others keep serving. But a Tokyo customer still has to reach your city, and if the *whole city* loses power, every window is dark. This is a single region with multiple availability zones — resilient to a local failure, useless for global latency or a regional disaster.

**A full restaurant on every continent (multi-region).** Now there's a kitchen in Tokyo, one in London, one in New York. Locals eat fast, and a flood in London doesn't touch Tokyo. But here's the problem that defines everything: **when the New York chef changes today's special, how do the Tokyo and London kitchens find out — and what happens if two chefs change the same dish at the same time?** That coordination across kitchens, over a slow intercontinental phone line, is the real engineering problem.

| Analogy | Technical concept |
|---------|-------------------|
| One restaurant, everyone flies in | Single region — global users pay huge latency |
| Multiple pickup windows, one city | Multi-AZ — survives a datacenter, not a region |
| A restaurant on each continent | Multi-region deployment |
| The live specials menu | Your shared, mutable data (the database) |
| Chef changes today's special | A write |
| Telling the other kitchens | Cross-region replication |
| Two chefs edit the same dish | A write conflict |
| The slow intercontinental phone line | ~100-200ms cross-region network round trip |
| One flagship kitchen sets all menus | Single-region writes (home region) |
| Every kitchen can change menus | Multi-master / active-active writes |

---

## Key concepts inside this topic

### 1. The single-region baseline and its limits

Almost every system should *start* here. One region, multiple availability zones (AZs) inside it for local redundancy:

```
Region: us-east-1
  AZ-a: web servers + DB primary
  AZ-b: web servers + DB standby replica
  AZ-c: web servers + DB standby replica
```

Multi-AZ gives you a lot: if AZ-a's datacenter loses power, a standby in AZ-b is promoted and you keep running, usually with sub-minute impact. Replication *within* a region is cheap and fast (AZs are ~1-2ms apart), so you can even replicate **synchronously** — the write isn't acknowledged until a second AZ has it — with almost no latency cost.

What multi-AZ does **not** solve:

- **Global latency.** Every AZ is in the same geographic area. A user in Sydney still crosses the planet.
- **Region-wide failure.** A bad deploy, a region-wide network partition, or a disaster affects all AZs together.

That is the entire reason to consider multi-region. If you have neither problem — no far-flung users, no compliance mandate, and multi-AZ availability is enough — you should **stop here.** More on that honesty at the end.

### 2. Active-Passive vs Active-Active — the central axis

Once you decide to run more than one region, the first fork is: do both regions serve traffic, or does one just wait?

**Active-Passive (failover).** One region ("active") serves all live traffic. A second region ("passive" / standby) runs the same stack but takes no user traffic; it only receives replicated data. When the active region fails, you *fail over* — promote the passive region to active and point users at it.

```
   Users ──▶ [ ACTIVE: us-east ] ──async replication──▶ [ PASSIVE: eu-west ]
                    │                                          (idle, warm)
              serves 100% traffic                     serves 0% until failover
```

- Simpler: only one region ever takes writes, so there is **no write-conflict problem at all**.
- Wasteful: the passive region's compute mostly sits idle, burning money.
- Has **failover time** (detect failure → promote DB → repoint DNS): seconds to minutes.
- Has a **data-loss window**: because replication is async, writes made in the last moment before the crash may not have reached the passive region yet. That gap is your RPO (Recovery Point Objective — recall [90](./90-disaster-recovery.md)).

**Active-Active.** All regions serve live user traffic simultaneously. Each region has a full stack and a copy of the data.

```
   Users(US) ──▶ [ ACTIVE: us-east ] ◀──bidirectional async──▶ [ ACTIVE: eu-west ] ◀── Users(EU)
                        │                                              │
                  serves US traffic                            serves EU traffic
```

- Best latency: everyone hits a nearby region.
- Best utilization: no idle standby; you paid for the region, it does work.
- Built-in resilience: if one region dies, routing just sends its users to another live region — no cold-start promotion.
- **But** writes now happen in multiple regions at once, and you must solve consistency across them. This is the hard part, and it's the next concept.

| Dimension | Active-Passive | Active-Active |
|-----------|----------------|---------------|
| Regions serving traffic | One | All |
| Global read/write latency | Poor for far users | Good everywhere |
| Standby cost | Wasted (idle) | None (all used) |
| Write conflicts | None (single writer) | Must be solved |
| Failover on region loss | Manual/auto promotion, minutes | Automatic reroute, seconds |
| Complexity | Moderate | High |
| Data-loss window (RPO) | Yes (async gap) | Yes (async gap) |

**Rule of thumb:** reach for active-passive first (it buys you disaster resilience cheaply); only pay the complexity tax of active-active when latency or utilization genuinely demands it.

### 3. The core hard problem — data consistency across regions

Here is the physics that ruins the simple dream of "just replicate synchronously everywhere."

A cross-region round trip is **~100-200ms**. If a write in Virginia had to wait for Ireland and Singapore to confirm before returning success (synchronous replication), every single write would take a fifth of a second or more. Users would revolt. Worse, if any one region is briefly unreachable, all writes would block — you'd have traded region *independence* for region *interdependence*.

So you are forced toward **asynchronous cross-region replication**: the local region acknowledges the write immediately, then ships the change to other regions in the background. Recall [63 — Database Replication](./63-database-replication.md) and [86 — Data Replication Strategies](./86-data-replication-strategies.md) — this is the async-replication trade-off, now stretched across oceans.

The consequence is unavoidable: **eventual consistency across regions** (recall [76 — Eventual Consistency](./76-eventual-consistency.md)). For a window of tens to hundreds of milliseconds (sometimes seconds under load), different regions hold slightly different views of the data. Your job is to *choose which write model* makes that acceptable. There are three well-worn options — the next three sub-sections.

Quick numbers to make it concrete:

```
Synchronous cross-region write (US→EU→SG, wait for both):
  write time ≈ max(RTT_to_EU, RTT_to_SG) + local commit
            ≈ 180ms + 5ms  ≈ 185ms  PER WRITE   ← unacceptable

Asynchronous cross-region write (ack locally, ship later):
  write time ≈ local commit ≈ 5ms
  replication lag to other regions ≈ 100-300ms (invisible to the writer)
```

### 4. Option A — Single-region writes, multi-region reads

The most common starting point, and the one you should default to.

- **All writes** go to one designated **home region** (say `us-east`). That region owns the single source of truth. Because there is exactly one writer, the data is **strongly consistent and conflict-free** — no two regions can ever disagree about a write order.
- **Reads** are served **locally** from a replica in every region. Reads are fast everywhere, but eventually consistent — a user might read data that's a few hundred milliseconds stale.

```
     writes                                    reads (local, fast)
  US user ─────▶ [ us-east  PRIMARY (writes) ] ◀───── US user
  EU user ──┐                    │ async
            └──write forwarded──▶│ replication
                                 ▼
  EU user ─────▶ [ eu-west   REPLICA (reads) ] ◀───── EU user (reads local)
```

A write from an EU user is **forwarded** to the US primary (paying one cross-region hop), but that user's *reads* stay local and fast.

```js
// Every region can serve reads locally; writes route to the home region.
class RegionalRouter {
  constructor(localRegion, homeRegion) {
    this.localRegion = localRegion;   // e.g. "eu-west"
    this.homeRegion = homeRegion;     // e.g. "us-east" — the single writer
  }

  async read(key) {
    // Reads are served from the nearest replica: fast, eventually consistent.
    return this.localReplica.get(key);
  }

  async write(key, value) {
    // Writes always go to the home region so there is a single source of truth.
    if (this.localRegion === this.homeRegion) {
      return this.localPrimary.put(key, value);
    }
    // Cross-region forward: one ~150ms hop, but correctness is preserved.
    return this.forwardTo(this.homeRegion, "write", { key, value });
  }
}
```

**Best for:** read-heavy systems (most systems) where writes are a minority of traffic and a small write penalty for distant users is acceptable. The classic 90%-reads/10%-writes profile loves this model.

### 5. Option B — Multi-master / active-active writes (and the conflict problem)

Here every region accepts writes locally and replicates them to the others afterward. Writes are fast *everywhere* — but now two regions can modify the same record before either has heard about the other. That is a **write conflict**, and it is genuinely hard.

```
  t=0ms   US sets  price = $10        EU sets  price = $12   (neither knows of the other)
  t=150ms US learns of EU's write     EU learns of US's write
  t=??    Which value wins? $10 or $12? One user's change is about to vanish.
```

Ways to resolve conflicts:

- **Last-Write-Wins (LWW).** Attach a timestamp; the latest wins. Dead simple, and every distributed database offers it — but it **silently loses data** (one of the two legitimate writes is discarded), and clock skew across regions can pick the "wrong" winner.
- **CRDTs (Conflict-free Replicated Data Types).** Data structures mathematically designed so concurrent updates *merge* deterministically without a central coordinator — e.g. a grow-only counter, an add/remove set. Recall [76 — Eventual Consistency](./76-eventual-consistency.md). Great for things like "likes count" or collaborative documents; not a fit for arbitrary business data.
- **Application-level merge.** Keep both versions and let business logic (or the user) decide — like a git merge conflict. Powerful, but you must write and maintain that merge logic for every conflicting field.

```js
// Last-Write-Wins: simple, but the losing write is gone forever.
function resolveLWW(a, b) {
  // Two concurrent versions of the same record arrive from two regions.
  return a.timestamp >= b.timestamp ? a : b;  // b's change silently discarded
}

// A CRDT counter instead MERGES — no update is ever lost.
class GCounter {              // grow-only counter, one slot per region
  constructor(regions) { this.counts = Object.fromEntries(regions.map(r => [r, 0])); }
  increment(region) { this.counts[region] += 1; }
  value() { return Object.values(this.counts).reduce((a, b) => a + b, 0); }
  merge(other) {              // take the max per region — commutative, conflict-free
    for (const r in other.counts) {
      this.counts[r] = Math.max(this.counts[r] ?? 0, other.counts[r]);
    }
  }
}
```

**Best for:** globally-writable, latency-critical data where you can either tolerate LWW loss or express the data as a CRDT. It is the most complex model — reach for it last.

### 6. Option C — Geographic partitioning / the sticky home-region model

This is the pragmatic middle path, and it's how a huge number of real global systems (messaging, social, ride-hailing) actually work.

The insight: most data has a **natural owner** — a user. A user's own posts, messages, profile, and settings are written overwhelmingly by that one user, from roughly one place on Earth. So **pin each user to a home region** (usually the one nearest them at sign-up), and store their data there.

- **That user's writes are local and strongly consistent** — they hit their home region, no cross-ocean hop, no conflicts (still a single writer *for their data*).
- **Only what needs to be global is replicated globally** — for example a lightweight directory of "which region owns user X," or a shared read-only cache of public content.
- When a user in Europe views a US user's profile, Europe either reads a replicated copy or fetches it from the US home region on demand.

```
  ┌─────────────── us-east (home for US users) ───────────────┐
  │  Alice (US): her messages, profile  → written & read here │
  └───────────────────────────────────────────────────────────┘
             ▲  small global "who-lives-where" directory  ▲
             │      replicated to all regions (tiny)       │
  ┌─────────────── eu-west (home for EU users) ────────────────┐
  │  Bob (EU): his messages, profile     → written & read here │
  └────────────────────────────────────────────────────────────┘

  Bob messages Alice → eu-west looks up "Alice lives in us-east"
                     → routes the message to us-east, where Alice reads it locally
```

You get local-latency, conflict-free writes for the common case (a user touching their own data), and you only pay cross-region cost for the genuinely cross-region interactions (Bob talking to Alice). This is the "sticky home region" model, and it neatly sidesteps multi-master conflicts by making sure each piece of mutable data still has exactly one writing region — just a *different* one per user.

The cost: you need a fast, globally-available way to answer "which region owns this user?", and moving a user between regions (they emigrate, or you rebalance) is a real migration operation, not a config flip.

### 7. Global traffic routing — getting users to the nearest healthy region

Multi-region is pointless if everyone still lands on one region. You need to route each user to the closest region that is currently healthy. Three common mechanisms, often layered:

- **GeoDNS / latency-based DNS routing.** Recall [54 — DNS Deep Dive](./54-dns-deep-dive.md). The DNS resolver returns a *different* IP depending on where the query comes from — EU users resolve `api.example.com` to the `eu-west` load balancer, US users to `us-east`. "Latency-based" routing picks the region with the lowest measured network latency rather than raw geography.
- **Anycast.** A single IP address is *announced* from many locations at once; the internet's own routing (BGP) delivers each user's packets to the topologically nearest announcement. One address, many physical endpoints. Common for DNS itself and for CDN/edge entry points.
- **Global load balancers** (e.g. cloud provider global HTTP load balancers) sit at the edge and forward each request to a healthy backend region, doing health checks and failover for you.

**Health-checked failover:** the router continuously probes each region; when a region fails its health check, the router stops sending traffic there and redistributes to healthy regions.

**The DNS TTL lag problem:** DNS answers are cached by resolvers and clients for the record's **TTL** (time-to-live). If you set a 300-second TTL and a region dies, some users keep resolving to the dead region for up to 5 minutes because their resolver cached the old answer. That's why failover-critical records use **short TTLs** (30-60s) — at the cost of more DNS queries — and why Anycast/global LBs (which fail over *without* waiting for client DNS caches to expire) are attractive for the fastest failover.

---

## Visual / Diagram description

**Active-active multi-region topology with global routing:**

```
                         ┌──────────────────────────┐
                         │   Latency-based DNS /     │
       user's browser ──▶│   Anycast / Global LB     │  health-checks every region
                         └───────────┬──────────────┘
              ┌──────────────────────┼──────────────────────┐
              ▼                      ▼                       ▼
     ┌────────────────┐    ┌────────────────┐     ┌────────────────┐
     │   us-east      │    │   eu-west      │     │  ap-southeast  │
     │ ┌────────────┐ │    │ ┌────────────┐ │     │ ┌────────────┐ │
     │ │ web + app  │ │    │ │ web + app  │ │     │ │ web + app  │ │
     │ ├────────────┤ │    │ ├────────────┤ │     │ ├────────────┤ │
     │ │ DB (R/W)   │◀┼────┼▶│ DB (R/W)   │◀┼─────┼▶│ DB (R/W)   │ │
     │ └────────────┘ │    │ └────────────┘ │     │ └────────────┘ │
     └────────────────┘    └────────────────┘     └────────────────┘
         async, bidirectional cross-region replication (~100-300ms lag)
```

Each region is a full, self-sufficient stack: web/app servers plus a database that both reads and writes. The DNS/Anycast/global-LB layer at the top steers each user to the nearest healthy region and health-checks them all. The double-headed arrows between the databases are **asynchronous** replication streams — each region ships its writes to the others in the background, which is exactly why the data is only *eventually* consistent across regions.

**The home-region / partitioned-write model:**

```
   Alice (US) writes locally          Bob (EU) writes locally
        │                                   │
        ▼                                   ▼
  ┌───────────────┐                  ┌───────────────┐
  │   us-east     │                  │   eu-west     │
  │  OWNS Alice   │                  │   OWNS Bob    │
  │  (single      │                  │   (single     │
  │   writer for  │                  │    writer for │
  │   Alice)      │                  │    Bob)       │
  └──────┬────────┘                  └───────┬───────┘
         │        ┌───────────────────┐      │
         └───────▶│  Global directory │◀─────┘
                  │  user → home region│  (tiny, replicated everywhere)
                  └───────────────────┘
   Cross-region interaction (Bob → Alice) uses the directory to route.
```

Here there is **no bidirectional DB replication of user data** — each user's data has one owning region, so writes are always conflict-free and local for that user. The only globally-shared thing is the small "who owns whom" directory. Contrast this with the active-active diagram above, where every DB replicates to every other and you must solve write conflicts.

---

## Real world examples

### Netflix (representative)

Netflix runs actively in multiple AWS regions and is famous for **regional evacuation**: they can shift all traffic out of a failing region into healthy ones, and they rehearse it regularly. Their control plane and routing are built so that losing an entire region degrades but does not stop the service. They pioneered **chaos engineering** (Chaos Monkey / the Simian Army) partly to make sure multi-region failover actually works when it's needed rather than only in theory.

### AWS DynamoDB Global Tables (factual, product behavior)

DynamoDB Global Tables offer a managed **active-active, multi-master** model: you write in any region and DynamoDB replicates asynchronously to the others, resolving conflicts with **last-write-wins** on a per-item timestamp. It's a clean real-world example of Option B — and a reminder that even the managed version accepts LWW's "the losing write is discarded" semantics as the price of writing anywhere.

### A global messaging / social platform (conceptual, sticky home region)

Large messaging and social systems commonly **pin each user to a home region** and keep their mailbox/timeline there, replicating only lightweight routing metadata globally. When two users in different regions interact, the system looks up the recipient's home region and routes across. This is the geographic-partitioning model from concept 6, and it's why such systems can be globally fast while still giving each user's own data a single, conflict-free writer.

---

## Trade-offs

**Single vs multi-region:**

| | Single region (multi-AZ) | Multi-region |
|---|---|---|
| Global latency | Poor for far users | Good — nearby region |
| Survives region outage | No | Yes |
| Data residency compliance | Only one jurisdiction | Can place data per-law |
| Cost | 1x | N× infra + cross-region egress |
| Complexity | Low | High |
| Consistency | Easy (strong) | Hard (eventual across regions) |

**Write-model comparison:**

| Model | Write latency (far user) | Conflicts | Complexity | Use when |
|-------|--------------------------|-----------|------------|----------|
| Single-region writes | High (cross-region hop) | None | Low | Read-heavy, default choice |
| Multi-master (active-active) | Low (local) | Yes — LWW/CRDT/merge | High | Latency-critical global writes |
| Geographic partition (home region) | Low for own data | None (single writer per user) | Medium | Per-user-owned data (messaging, social) |

**Costs people forget:**

- **Cross-region egress is expensive.** Cloud providers charge real money per GB moved *between* regions; a chatty replication stream can dominate your bill.
- **N regions ≈ N× the infrastructure.** Every region needs the full stack. Active-passive means paying for an idle standby too.
- **Operational surface multiplies.** N regions means N× the deploys, dashboards, on-call surprises, and failure modes.

**The sweet spot:** Most systems should run **single-region, multi-AZ**, and add regions only when a concrete driver — global latency, a compliance mandate, or a strict availability requirement — forces it. When you do go multi-region, start **active-passive** (or single-region-writes/multi-region-reads), and only escalate to active-active/multi-master for the specific data that truly needs local writes everywhere.

---

## Common interview questions on this topic

### Q1: "Why can't you just replicate synchronously across regions and keep strong consistency everywhere?"

**Hint:** Physics. Cross-region round trips are ~100-200ms; making every write wait for remote regions makes writes unbearably slow and couples all regions (one slow region blocks all writes). So you go async, which forces eventual consistency across regions. Name the RTT number and the speed-of-light reason.

### Q2: "Active-active writes let two regions edit the same record at once. How do you resolve the conflict?"

**Hint:** Three tools — Last-Write-Wins (simple, silently loses data, clock-skew risk), CRDTs (merge deterministically, but only for data you can model that way — counters, sets, docs), and application-level merge (keep both, decide in business logic). Then mention the escape hatch: partition so each record has a single owning region and there *are* no conflicts.

### Q3: "How do users reach the nearest region, and what happens when a region dies?"

**Hint:** GeoDNS/latency-based DNS, Anycast, or a global load balancer, all health-checked. On failure the router stops sending traffic to the dead region. Then raise the **DNS TTL lag** gotcha — cached DNS answers keep pointing at the dead region until the TTL expires, which is why failover records use short TTLs and why Anycast/global LB fail over faster than DNS.

### Q4: "GDPR requires EU user data to stay in the EU. How does that shape your architecture?"

**Hint:** Data residency constrains your replication topology — you cannot freely replicate EU personal data to `us-east`. It pushes you toward the geographic-partition/home-region model: EU users' data is written and stored only in EU regions, and you replicate globally only non-personal or explicitly-allowed data. Compliance is a design driver, not a footnote.

### Q5: "When should a system NOT go multi-region?"

**Hint:** When there's no global user base (latency fine), no compliance mandate, and multi-AZ availability already meets the SLA. Multi-region multiplies cost (N× infra + egress) and complexity (eventual consistency, conflict handling, N× ops). The honest senior answer is that most systems shouldn't — do it only when latency, law, or availability genuinely demand it.

---

## Practice exercise

**Design the region strategy for a global note-taking app.**

Assume an app like a personal notes/journal product with users in North America, Europe, and Australia. In ~20-40 minutes, produce a one-page design document (text + one ASCII diagram) that answers:

1. **Baseline:** Would you even go multi-region? State the specific driver(s) that justify it (or argue for staying single-region).
2. **Write model:** Pick one of the three models (single-region writes / multi-master / home-region partition) for users' notes, and justify it. Notes are almost always written by their one owner — does that suggest a model?
3. **Routing:** How do users reach the nearest region? Pick GeoDNS, Anycast, or a global LB and say how failover works. What TTL would you set and why?
4. **Consistency story:** Write one sentence describing what a user might observe due to cross-region replication lag (e.g. logging in from a second device in another region), and argue why it's acceptable.
5. **Compliance:** Add the constraint "EU users' notes must stay in the EU." Show how your write model already handles it — or what you'd change.

Deliverable: the document, with your chosen write model clearly named and the trade-off you accepted stated in one line.

---

## Quick reference cheat sheet

- **Multi-region** = running your full stack in multiple geographic regions at once, for latency, disaster resilience, and data residency.
- **Multi-AZ ≠ multi-region.** Multi-AZ survives a datacenter failure and can replicate synchronously (cheap, fast); it does nothing for global latency or a region-wide outage.
- **Speed of light is the constraint.** Cross-region round trips are ~100-200ms — the reason synchronous cross-region replication is impractical.
- **Async cross-region replication ⇒ eventual consistency across regions.** You cannot avoid it; you choose how to make it acceptable.
- **Active-Passive:** one region serves, one stands by; simple, no write conflicts, but wastes the standby and has failover time + a data-loss window (RPO).
- **Active-Active:** all regions serve; best latency and utilization, but you must solve multi-region write conflicts.
- **Single-region writes, multi-region reads:** all writes to one home region (consistent), reads local everywhere (fast, stale). The common default.
- **Multi-master:** write anywhere, reconcile after. Conflicts resolved by **LWW** (loses data), **CRDTs** (merge, limited data shapes), or **app-level merge**.
- **Home-region partition (sticky region):** pin each user's data to one owning region → local, conflict-free writes for the common case; replicate globally only what must be global. How real messaging/social systems scale.
- **GeoDNS / latency routing / Anycast / global LB** send users to the nearest healthy region; all should be health-checked.
- **DNS TTL lag:** cached DNS answers point at a dead region until TTL expires — use short TTLs for failover records; Anycast/global LB fail over faster.
- **Data residency (GDPR etc.)** can legally force data to stay in-region — a real topology constraint, best served by the home-region model.
- **Costs:** cross-region **egress** is pricey, and N regions ≈ N× infra + N× operational surface. **Test failover** with chaos engineering (recall [90](./90-disaster-recovery.md)).
- **Honest rule:** most systems should NOT go multi-region — do it only when latency, compliance, or availability genuinely demand it.

---

## Connected topics

| Direction | Topic | Why |
|-----------|-------|-----|
| **Previous** | [90 — Disaster Recovery](./90-disaster-recovery.md) | Multi-region is the ultimate DR strategy; RPO/RTO, failover, and chaos-testing all carry over directly. |
| **Next** | [54 — DNS Deep Dive](./54-dns-deep-dive.md) | Global traffic routing rides on GeoDNS, latency-based routing, and TTL behavior — the mechanics of getting users to the right region. |
| **Related** | [76 — Eventual Consistency](./76-eventual-consistency.md) | The unavoidable consequence of async cross-region replication, and where CRDTs/LWW conflict resolution live. |
| **Related** | [63 — Database Replication](./63-database-replication.md) & [86 — Data Replication Strategies](./86-data-replication-strategies.md) | The sync-vs-async and single-vs-multi-master trade-offs, now stretched across oceans. |
