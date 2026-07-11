# 35 — Network in System Design

> **Phase 6 — Topic 1 of 5**
> Prerequisites: `01-how-the-internet-works.md`, `33-network-latency.md`

---

## ELI5 — The Simple Analogy

Imagine you're planning a birthday party and someone asks: **"How much pizza do we need?"**

You don't count every slice. You reason: *"About 30 people coming, each eats ~3 slices, that's 90 slices, a pizza has 8 slices, so ~12 pizzas — round up to 15 to be safe."* Nobody expects an exact number. They expect a **sensible estimate you can defend**.

System design is the same reasoning applied to computers instead of pizza:

- **"How many servers?"** → How many requests per second, divided by what one server handles.
- **"How fast will it be?"** → Add up the delays of each step, especially the ones where data crosses the ocean.
- **"How much will bandwidth cost?"** → Requests per second × size of each response × how many seconds in a month.

And just like the party planner instantly knows *"a pizza has 8 slices"* without looking it up, a backend engineer must instantly know a handful of **latency and bandwidth numbers** — because a request to memory takes 100 nanoseconds, but a request across the Pacific Ocean takes 150 *milliseconds*. That's **1.5 million times slower.** If you don't feel that difference in your bones, you'll design systems that are accidentally slow.

This topic is about *back-of-envelope reasoning* — not exact answers, but the ability to say a number and justify it.

---

## Where This Lives in the Network Stack

Unlike most topics, this one isn't a single protocol at a single layer. It's a **way of reasoning across the whole stack** — you take everything from topics 01–34 and turn it into numbers you can multiply.

```
┌──────────────────────────────────────────────────────────────────────┐
│  DESIGN / ESTIMATION LAYER  (this topic — not in the OSI model)       │
│  "How many servers? How fast? How much bandwidth? Where's the         │
│   bottleneck?" — you reason about the layers below as budget lines.   │
├──────────────────────────────────────────────────────────────────────┤
│  L7 Application  (HTTP payload size, API chattiness, fan-out)         │ ← payload & round-trip budget
│  L6 Presentation (TLS handshake cost, compression ratio)             │ ← 1 extra RTT, ÷3 payload
│  L4 Transport    (TCP handshake = 1 RTT, connection pooling)         │ ← reuse or pay the RTT again
│  L3 Network      (IP routing, distance = latency floor)              │ ← speed of light, the hard limit
│  L1 Physical     (fiber, undersea cables, ~200,000 km/s)             │ ← the number you cannot beat
└──────────────────────────────────────────────────────────────────────┘
```

**Key idea:** every layer contributes a *cost* — a number of milliseconds or bytes. Design is the art of adding those costs up *before* you build, so you don't discover a 300 ms cross-region round trip in production.

---

## What Is This?

**Networking in system design** is the skill of *estimating* a system's behavior from a few known constants and simple arithmetic — before writing any code. It has three pillars:

1. **The latency numbers every engineer memorizes.** A short table spanning ~9 orders of magnitude, from L1 cache (0.5 ns) to a US↔Asia round trip (~150 ms). You memorize these so you can *feel* which operations are cheap and which are catastrophically expensive.

2. **Bandwidth/throughput math.** Converting between `Mbps` (bits, how links are sold) and `MB/s` (bytes, how files are measured — divide by 8), computing transfer time, and figuring out how many requests a link can carry.

3. **Back-of-envelope estimation.** A repeatable method: estimate **QPS** (queries/sec), **storage**, **bandwidth**, and **server count** — then find where the **network becomes the bottleneck**, and reason about the trade-offs (sync vs async, chatty vs chunky, single-region vs multi-region, cache vs cross-region round trip).

This is the language of **system design interviews** *and* of real capacity-planning meetings. The math is deliberately loose — you round aggressively, you use `100,000` seconds/day instead of `86,400`, and you carry one significant figure. The goal is an answer that's right to within a factor of ~2, delivered in 30 seconds.

---

## Why Does It Matter for a Backend Developer?

Because the network is almost always the part of your system you *can't* fix with faster code, and it's the part people forget to budget for.

- **You'll design capacity in meetings.** "We're launching to 5M users — how many API servers, how much DB, how much egress?" is a Tuesday-afternoon question, not just an interview trope.
- **You'll catch the fatal mistakes.** A design that makes a synchronous cross-region call inside a request handler pays ~80 ms *per call, per request, forever*. Multiply by fan-out and you have a dead product. You catch this on the whiteboard, not in the postmortem.
- **You'll pass the interview.** Every senior/staff backend loop has a design round. The interviewer is explicitly listening for "≈100 ms across the Atlantic" and "Mbps ÷ 8 = MB/s". Fumbling these reads as *junior*.
- **You'll size cloud bills correctly.** Egress bandwidth is one of the most expensive line items in AWS/GCP. `QPS × payload × seconds` tells you the bill before you get surprised by it.
- **You'll know when to reach for a CDN, a cache, a queue, or a replica** — because you can compare the round-trip cost of each option numerically instead of by vibes.

The engineers who get promoted are the ones who can say "that'll be about 30,000 QPS at peak, ~12 Gbps egress, roughly 40 app servers, and the bottleneck is the cross-region write — here's how I'd fix it" — and be *right within a factor of two*.

---

## The Packet/Protocol Anatomy

There's no single packet here. The "anatomy" of system-design reasoning is the **latency ladder** — the set of numbers, laid out on a log scale so you can *see* the gaps.

### The latency numbers every engineer must memorize

```
Operation                          Time          Log scale (each ▓ = ~10×)
─────────────────────────────────────────────────────────────────────────
L1 cache reference          ~0.5   ns   (10^-9) ▓
L2 cache reference          ~5-7   ns           ▓▓
Branch mispredict           ~5     ns           ▓▓
Mutex lock/unlock           ~20    ns           ▓▓
Main memory (RAM) ref       ~100   ns           ▓▓▓
Compress 1 KB (snappy)      ~2     µs   (10^-6) ▓▓▓▓
Send 1 KB over 1 Gbps       ~8     µs           ▓▓▓▓
SSD random read (NVMe)      ~16    µs           ▓▓▓▓▓
Read 1 MB from RAM          ~100   µs           ▓▓▓▓▓▓
── the network wall ─────────────────────────────────────────
Round trip, same datacenter ~0.5   ms   (10^-3) ▓▓▓▓▓▓▓
Round trip, same region/AZ  ~1-2   ms           ▓▓▓▓▓▓▓
Disk seek (spinning HDD)    ~2-10  ms           ▓▓▓▓▓▓▓▓
Read 1 MB from SSD          ~1     ms           ▓▓▓▓▓▓▓
Send 1 MB over 1 Gbps       ~8     ms           ▓▓▓▓▓▓▓▓
Cross-country US RTT        ~30-70 ms           ▓▓▓▓▓▓▓▓▓
US ↔ Europe RTT             ~70-90 ms           ▓▓▓▓▓▓▓▓▓
Read 1 MB from disk (seq)   ~20    ms           ▓▓▓▓▓▓▓▓▓
US ↔ Asia RTT               ~150-200 ms         ▓▓▓▓▓▓▓▓▓▓
US ↔ Australia RTT          ~180-250 ms         ▓▓▓▓▓▓▓▓▓▓
```

### The "human time" trick — scale so 1 ns = 1 second

Nanoseconds are impossible to feel. Multiply every number by ~1 billion and it becomes human-scale time:

```
If an L1 cache hit took 1 second, then...

L1 cache        0.5 ns  →  0.5 seconds      "grab it off the desk"
RAM             100 ns  →  1.7 minutes      "walk to the kitchen"
SSD read         16 µs  →  4.4 hours        "wait for a delivery"
Datacenter RTT  0.5 ms  →  5.8 days         "mail a letter across town"
Disk seek        10 ms  →  ~4 months        "order from overseas"
US↔EU RTT        80 ms  →  ~2.5 years       "college degree"
US↔Asia RTT     150 ms  →  ~4.75 years      "raise a kindergartner"
```

**This is the whole point of the topic.** RAM vs a cross-region call isn't "a bit slower" — it's *1.7 minutes vs 2.5 years.* When you design, you are constantly deciding whether an operation lives on the "seconds" side or the "years" side of this ladder.

### The two hard walls

```
WALL 1 — The network wall (~0.5 ms)
   Everything above it is on-box (CPU/RAM/SSD), measured in ns–µs.
   Everything below it involves another machine, measured in ms.
   Crossing this wall is 1000× slower than staying on-box.

WALL 2 — The speed of light (geography)
   Light in fiber travels ~200,000 km/s.
   RTT_min (ms) ≈ distance_km / 100.
   No cache, no code, no money buys you below this floor.
   The ONLY fix is to move the data physically closer (CDN / regional deploy).
```

---

## How It Works — Step by Step

The core mechanic is the **back-of-envelope estimation method**. Here it is as a five-step recipe you run in your head. We'll use one running example: **design the backend for a social news feed.**

**Given:** 200M daily active users (DAU). Each opens the app ~10 times/day. Each user posts ~2 items/day. Each feed response is ~20 posts ≈ 30 KB. Keep data for 2 years.

### Step 1 — Estimate QPS (queries per second)

```
Feed reads/day = DAU × opens_per_user
               = 200,000,000 × 10
               = 2,000,000,000  (2 billion reads/day)

QPS_avg = reads_per_day / seconds_per_day
        = 2,000,000,000 / 86,400
        ≈ 23,000 QPS         (use /100,000 for mental math → 20,000)

QPS_peak = QPS_avg × peak_factor
         = 23,000 × 2         (traffic clusters in daytime hours)
         ≈ 46,000 QPS
```

> Rule of thumb: real traffic is *not* uniform. Peak is usually **2–3×** the daily average; for spiky consumer apps use up to 10×. Always size for **peak**, not average.

### Step 2 — Estimate bandwidth (egress)

```
Egress at peak = QPS_peak × payload_size
              = 46,000 req/s × 30 KB
              = 1,380,000 KB/s
              ≈ 1.38 GB/s
              ≈ 11 Gbps         (× 8 to go from bytes to bits)

Monthly egress = 1.38 GB/s × 2,592,000 s/month  (30 days)
              ≈ 3.6 PB/month    ← this is a serious cloud bill
```

This is the number that tells you *you need a CDN* (offload static/image bytes to the edge so they never touch your origin's egress meter).

### Step 3 — Estimate storage

```
Posts/day = DAU × posts_per_user = 200M × 2 = 400,000,000/day
Bytes/day = 400M × 1 KB ≈ 400 GB/day
Per year  = 400 GB × 365 ≈ 146 TB/year
2-year raw ≈ 292 TB
× replication (3 copies) ≈ 876 TB ≈ ~0.9 PB of primary storage
```

### Step 4 — Estimate server count

```
servers = QPS_peak / per_server_QPS
        = 46,000 / 1,000          (a tuned app server ≈ 1,000 QPS)
        = 46 servers
        × redundancy (N+2, survive AZ loss) ≈ ~60 app servers
```

### Step 5 — Find where the network becomes the bottleneck

```
Now walk the request path and add up the latency budget:

  Client → Load balancer        (same region)     ~1 ms
  LB → App server               (same DC)         ~0.5 ms
  App → Cache (Redis) hit       (same DC)         ~0.5 ms
  App → DB (if cache miss)      (same region)     ~2 ms
  App → serialize 30 KB + send  (transfer)        ~1 ms
  ────────────────────────────────────────────────────────
  Backend budget (all same-region)               ~5 ms  ✓ fine

  BUT the user in Sydney → us-east-1 origin:      ~200 ms RTT ✗✗✗
```

**The bottleneck is not compute — it's the ocean.** A Sydney user pays ~200 ms *before your 5 ms of work even starts*, and pays it again for TCP + TLS + the request. The estimation surfaced the real problem: **geography, not capacity.** The fix (CDN + regional read replicas) is a network decision, not a code decision. That framing — *walk the path, find the cross-region hop, move the data closer* — is the whole game.

---

## Exact Syntax Breakdown

There are no CLI flags here — the "syntax" of this topic is a **small set of formulas**. Memorize these the way you'd memorize `curl` flags.

### Formula 1 — QPS from users

```
QPS_avg = (DAU × actions_per_user_per_day) / 86,400
  │         │      │                           │
  │         │      │                           └── seconds in a day
  │         │      │                               (use 100,000 for easy math)
  │         │      └── how many times each user does THIS action per day
  │         └── Daily Active Users
  └── average queries/sec the system must sustain

QPS_peak = QPS_avg × peak_factor        (peak_factor ≈ 2–10)
```

### Formula 2 — Bytes ↔ bits (the #1 interview trap)

```
MB/s  = Mbps / 8          ← links are sold in BITS, files measured in BYTES
Mbps  = MB/s × 8

1 Gbps  = 1000 Mbps = 125 MB/s
100 Mbps = 12.5 MB/s
10 Gbps  = 1250 MB/s = 1.25 GB/s
```

### Formula 3 — Transfer time (throughput term + latency floor)

```
transfer_time = (size / bandwidth) + RTT
  │              │                    │
  │              │                    └── latency floor: you pay ≥1 RTT no
  │              │                        matter how small the payload
  │              └── serialization/throughput term: dominates for BIG files
  └── total wall-clock time to move `size` bytes

Small payload  → RTT dominates      (fix: fewer round trips, edge/CDN)
Large payload  → bandwidth dominates (fix: compress, fatter pipe, closer)
```

### Formula 4 — Latency floor from distance (the speed-of-light wall)

```
RTT_min (ms) ≈ distance_km / 100

  (because light in fiber ≈ 200,000 km/s, so one-way = dist/200 ms,
   round trip = dist/100 ms. Real RTT is ~1.5–2× this due to routing
   detours, not straight-line paths.)

  NYC → London  ≈ 5,570 km  → floor ~56 ms  (real ~70–80 ms)
  NYC → Tokyo   ≈ 10,800 km → floor ~108 ms (real ~150–170 ms)
```

### Formula 5 — Egress bandwidth & cost

```
egress_bandwidth = QPS × payload_size
egress_per_month = egress_bandwidth × 2.592e6   (seconds in 30 days)
egress_cost      = egress_GB × price_per_GB      (~$0.05–0.09/GB on AWS)
```

### Formula 6 — Server count

```
servers = ceil(QPS_peak / per_server_QPS) × redundancy_factor
  │              │                             │
  │              │                             └── N+1 / N+2, survive AZ loss (~1.3–2×)
  │              └── benchmark it; ballpark 500–5,000 QPS for a JSON API
  └── round UP
```

### Formula 7 — Storage over time

```
storage = writes_per_day × record_size × retention_days × replication_factor
```

**The whole toolkit fits on an index card.** Every design estimate is these seven formulas plus the latency ladder.

---

## Example 1 — Basic

**Question:** *"How long does it take to transfer a 10 GB database backup over a 1 Gbps link, and is that link fast enough to also serve 2,000 API responses/sec at 50 KB each?"*

**Part A — transfer time (Formula 2 + 3):**

```
Step 1: convert the link to bytes/sec
   1 Gbps ÷ 8 = 125 MB/s

Step 2: transfer_time = size / bandwidth  (+ negligible RTT for a big file)
   10 GB = 10,000 MB
   10,000 MB / 125 MB/s = 80 seconds

Real-world: add ~10–20% for TCP overhead, ACKs, and the fact that you
rarely get 100% of a link → expect ~90–100 seconds.
```

So: **~80 seconds theoretical, ~1.5 minutes realistic.** (Note: this is why nightly backups of huge DBs use compression and incremental diffs — moving the full 10 GB every night saturates the pipe.)

**Part B — can the same link carry the API traffic? (Formula 5):**

```
API egress = QPS × payload
           = 2,000 req/s × 50 KB
           = 100,000 KB/s
           = 100 MB/s   (in BYTES)

Link capacity = 125 MB/s

100 MB/s of API traffic on a 125 MB/s link → 80% utilization.
```

**Answer:** The link *can* carry 2,000 QPS at 50 KB (100 MB/s < 125 MB/s), but you're at **80% utilization** — dangerously close to saturation, and you have *no room* to also run the backup at the same time (that needs the full pipe). At 80%+ utilization, queuing delay spikes and latency gets ugly. **Fix:** compress responses (gzip a 50 KB JSON to ~10 KB → drops to 20 MB/s, 16% utilization), or move to a 10 Gbps link, or schedule the backup for off-peak. This is the everyday bandwidth reasoning that separates a defensible design from a hopeful one.

---

## Example 2 — Production Scenario

**The brief:** *"Design the network and data flow for a globally-used product page (an e-commerce PDP — product detail page). Users are everywhere; our origin is in `us-east-1` (Virginia). Make it fast for a shopper in Singapore. Estimate the QPS and bandwidth, and identify the network bottleneck."*

### Step 1 — Classify the page's data by how fresh it must be

```
┌─────────────────────────────────────────────────────────────────────┐
│ COMPONENT            FRESHNESS NEEDED      WHERE IT SHOULD LIVE        │
├─────────────────────────────────────────────────────────────────────┤
│ Product images       days (immutable)      CDN edge  (cache forever)  │
│ CSS / JS bundles     until deploy          CDN edge  (hashed URLs)    │
│ Title, description   minutes–hours          CDN edge  (short TTL) OR   │
│                                             regional read replica      │
│ Price                seconds–minutes        regional replica / edge KV │
│ Inventory / "3 left" real-time-ish          origin or regional cache   │
│ "Add to cart" (write) strongly consistent   origin (single region)     │
└─────────────────────────────────────────────────────────────────────┘
```

The design principle: **push each piece of data as close to the user as its freshness requirement allows.** Immutable bytes go to the edge; the rare strongly-consistent write goes back to the origin and pays the RTT.

### Step 2 — The data-flow architecture

```
   Shopper in Singapore
          │
          │  ~2 ms to nearest CDN PoP (Singapore edge)
          ▼
   ┌──────────────────────┐   cache HIT (images, JS, cached PDP JSON)
   │  CDN EDGE (Singapore) │──────────────────────────────► served in ~2 ms ✓
   └──────────┬───────────┘
              │ cache MISS (or dynamic price/stock)
              │  ~2 ms to regional stack (ap-southeast-1)
              ▼
   ┌────────────────────────────┐
   │ REGIONAL STACK (Singapore) │
   │  App servers + Redis cache  │
   │  + Postgres READ REPLICA    │──── read price/description locally ~2 ms ✓
   └──────────┬─────────────────┘
              │ WRITE ONLY (add-to-cart, order)
              │  ~200 ms cross-region RTT to primary
              ▼
   ┌────────────────────────────┐
   │ PRIMARY DB (us-east-1)      │  ← single writer, source of truth
   └────────────────────────────┘
```

### Step 3 — Estimate the load (Formulas 1, 2)

```
Assume 50M DAU, each views ~20 product pages/day:
  PDP views/day = 50M × 20 = 1,000,000,000  (1B/day)
  QPS_avg = 1e9 / 100,000 ≈ 10,000 QPS
  QPS_peak = 10,000 × 3 ≈ 30,000 QPS   (retail is spiky: sales, launches)

Payload (the dynamic JSON, images served by CDN separately) ≈ 15 KB:
  Origin/regional egress = 30,000 × 15 KB = 450 MB/s ≈ 3.6 Gbps

But with a 95% CDN + regional-cache hit rate, only 5% reaches origin JSON:
  Origin QPS ≈ 30,000 × 0.05 = 1,500 QPS   ← 20× reduction
  Image bytes (the heavy part) never touch origin at all.
```

### Step 4 — Validate an assumption with real commands

Before trusting "~2 ms to the edge, ~200 ms to origin," measure it. From a box in Singapore:

```bash
# RTT to a nearby CDN edge (Cloudflare anycast → nearest PoP)
$ ping -c 4 1.1.1.1
64 bytes from 1.1.1.1: icmp_seq=1 ttl=57 time=1.90 ms
64 bytes from 1.1.1.1: icmp_seq=2 ttl=57 time=2.11 ms
--- 1.1.1.1 ping statistics ---
rtt min/avg/max = 1.90/2.03/2.11 ms          ← edge is ~2 ms away ✓

# RTT to the us-east-1 origin (cross-Pacific)
$ ping -c 4 ec2-us-east-1-host.example.com
64 bytes from 52.x.x.x: icmp_seq=1 ttl=44 time=214 ms
64 bytes from 52.x.x.x: icmp_seq=2 ttl=44 time=221 ms
rtt min/avg/max = 214/218/226 ms             ← origin is ~218 ms away ✗

# Compare a CDN-cached asset vs a dynamic origin call end-to-end
$ curl -o /dev/null -s -w "cdn asset TTFB: %{time_starttransfer}s\n" \
    https://cdn.example.com/img/product-123.jpg
cdn asset TTFB: 0.021s                        ← 21 ms (edge cache hit)

$ curl -o /dev/null -s -w "origin API TTFB: %{time_starttransfer}s\n" \
    https://origin.example.com/api/pdp/123
origin API TTFB: 0.664s                       ← 664 ms (TCP+TLS+req over Pacific)
```

The measurement *confirms the estimate*: the origin round trip really is ~218 ms, and a single cold call to origin is ~660 ms (roughly 3× RTT for TCP handshake + TLS + request). Serving from the edge is **~30× faster**.

### Step 5 — Name the bottleneck and the mitigation

```
BOTTLENECK: the "add to cart" WRITE. It must reach the single primary in
us-east-1, so a Singapore shopper pays ~200 ms RTT on every write. Reads
are solved (edge + regional replica); writes are not.

MITIGATIONS (trade-offs, pick per business need):
  • Accept it. A cart write is rare vs page views; 200 ms on click is OK.
  • Async it (topic 38): write to a regional queue, ack the user immediately,
    replicate to primary in the background. Fast UX, eventual consistency.
  • Regional write partitioning: shard carts by region so APAC carts have a
    local primary. More consistency complexity, no cross-region write.

READ bottleneck already handled:
  • CDN for images/JS/cached JSON  (topic 30) → bytes never cross the ocean
  • Regional read replica for price/description → local ~2 ms reads
  • Redis at the edge/region for hot products → avoid even the DB hop
```

**The lesson:** you made every read local and identified the *one* operation that inherently pays the cross-region tax — then chose consciously whether to eat it, hide it (async), or eliminate it (regional partitioning). That is exactly what an interviewer or a staff engineer wants to hear.

---

## Common Mistakes

### Mistake 1: Forgetting network round trips in the latency budget

```
WRONG: "The DB query is 2 ms and serialization is 1 ms, so the endpoint
        is ~3 ms."

Reality — you forgot every hop:
  client → LB           1 RTT
  LB → app              1 RTT
  app → cache miss      1 RTT
  app → DB              1 RTT   (+ the 2 ms query itself)
  ...and if any hop is cross-region, add ~80–200 ms EACH.
```
**Fix:** Draw the request path and put a number on **every arrow**, not just the compute. Latency is dominated by the *number of network round trips*, not by CPU. On the "human time" ladder, one cross-region round trip outweighs *millions* of RAM accesses.

---

### Mistake 2: Confusing bits and bytes (Mbps vs MB/s)

```
WRONG: "We have a 1 Gbps link, so we can push 1 GB/s."
Reality: 1 Gbps = 1000 Mbps ÷ 8 = 125 MB/s.  You were off by 8×.

WRONG: "This 100 Mbps connection downloads a 100 MB file in 1 second."
Reality: 100 Mbps = 12.5 MB/s → 100 MB / 12.5 = 8 seconds.
```
**Fix:** Network links are sold in **bits** (Mbps, Gbps). Files and RAM are measured in **bytes** (MB, GB). To go from a link speed to a transfer rate, **divide by 8**. Getting this wrong in an interview is an instant red flag; getting it wrong in production means your "fast enough" pipe is actually saturated.

---

### Mistake 3: Designing chatty cross-region APIs (N+1 over the network)

```
WRONG (chatty): render a page by calling the origin 20 times.
  20 sequential calls × 200 ms cross-region RTT = 4,000 ms.  Dead.

  for id in item_ids:              # ← N+1: one round trip PER item
      call_origin(f"/item/{id}")   # ← each pays the full ocean RTT

RIGHT (chunky): one call that returns everything.
  1 batched call × 200 ms = 200 ms.  20× faster for zero extra work.
  call_origin("/items?ids=1,2,3,...,20")
```
**Fix:** Every network call has a fixed RTT cost that dwarfs the payload cost when far away. **Batch.** Turn N calls into 1 (chunky). This is the classic N+1 problem (topics 33/39), and it's catastrophic across regions. If you *must* be chatty, at least keep the caller and callee in the same region/DC.

---

### Mistake 4: Ignoring tail latency in fan-out

```
A request that fans out to 100 services and WAITS for all of them:
  Each service is fast on average (p50 = 10 ms) but p99 = 100 ms.

  P(all 100 fast) = 0.99^100 ≈ 0.37
  → 63% of user requests hit AT LEAST ONE slow service.
  → Your user-facing p50 is now ~100 ms, not 10 ms.

The SLOWEST of N calls dominates. p99 of one service becomes p50 of the whole.
```
**Fix:** When a request depends on many parallel calls, the **tail (p99) of the slowest call defines your user experience**, not the average. Mitigate with: fewer dependencies, hedged/backup requests, per-call timeouts + partial responses, and always measuring **p99, not just the mean**. "Average is fine" is how fan-out systems die.

---

### Mistake 5: Over-provisioning bandwidth to "fix" latency

```
WRONG: "Users in Asia see 200 ms latency — let's upgrade to a 10 Gbps link!"
Reality: latency and bandwidth are DIFFERENT things.
  • Bandwidth = how WIDE the pipe is (bytes/sec).
  • Latency   = how LONG the pipe is (speed-of-light distance).
  A fatter pipe does NOTHING for a small request that's slow because of distance.
```
**Fix:** Diagnose first. If the payload is small and it's slow, it's a **latency (distance)** problem → move data closer (CDN, regional deploy). If the payload is huge and slow, it's a **bandwidth** problem → fatter pipe or compression. Throwing bandwidth at a latency problem is money set on fire.

---

### Mistake 6: Assuming a single region is fine for a global audience

```
WRONG: "us-east-1 serves everyone. It's fine, it works in my testing."
       (You tested from an office 10 ms from Virginia.)
Reality: every user outside North America eats an ocean crossing on every
         request. Sydney: ~200 ms RTT × (TCP + TLS + request) ≈ 600+ ms
         before your server does any work.
```
**Fix:** Test from where your users actually are (or from a cloud box in their region). For a global product you need **edge (CDN) + at least regional read replicas**, accepting that writes may still be single-region. "Works on my machine" hides geography because your machine is near the origin.

---

## Hands-On Proof

This topic proves itself with **calculators and measurement**, not a single tool. Do these to make the numbers real.

### Proof 1 — Validate the speed-of-light floor against reality

```bash
# 1. Measure real RTT to a far host
ping -c 5 1.1.1.1              # Cloudflare (anycast → likely near you)
# note the avg time, e.g. 12 ms

# 2. Measure to a deliberately distant host (pick one across an ocean)
ping -c 5 www.icann.org        # or any known far server

# 3. Compute the theoretical floor for that distance:
#    RTT_min (ms) ≈ distance_km / 100
#    e.g. you → Tokyo ≈ 10,000 km → floor ≈ 100 ms
#
# Compare: real RTT should be ~1.5–2× the floor (routing isn't a straight line).
# If real RTT is CLOSE to the floor, the path is well-optimized.
# If it's 3×+ the floor, packets are taking a scenic detour (see traceroute, topic 25).
```

### Proof 2 — Validate a transfer-time estimate with curl

```bash
# Download a known-size file and compare measured vs predicted time.
curl -o /dev/null -s -w \
"size:        %{size_download} bytes
speed:        %{speed_download} bytes/s
TTFB:         %{time_starttransfer}s
total:        %{time_total}s
transfer:     %{time_total}s - TTFB = pure download time
" https://speed.cloudflare.com/__down?bytes=10000000   # 10 MB

# Then check the math:
#   transfer_time ≈ size / speed_download
#   e.g. 10,000,000 bytes / 12,500,000 B/s ≈ 0.8 s
# Confirm speed_download ÷ 125,000 ≈ your link's Mbps portion in use.
```

### Proof 3 — Prove CDN/edge beats origin (round-trip elimination)

```bash
# A CDN-cached asset vs a dynamic origin endpoint. Watch TTFB collapse.
curl -o /dev/null -s -w "edge TTFB:   %{time_starttransfer}s\n" \
  https://cdn.jsdelivr.net/npm/react@18/package.json

curl -o /dev/null -s -w "origin TTFB: %{time_starttransfer}s\n" \
  https://httpbin.org/delay/0
# The edge asset is served from a PoP near you; the origin call pays the
# full distance. The gap IS the value of the CDN.
```

### Proof 4 — Back-of-envelope drill (do it on paper, then check)

```
Estimate these WITHOUT a calculator, one significant figure:

  a) 100M DAU, 5 actions/user/day → QPS_avg?
     100e6 × 5 = 500e6 /day; /100,000 = 5,000 QPS.

  b) That service returns 20 KB responses. Peak = 3× avg. Peak egress in Gbps?
     15,000 QPS × 20 KB = 300 MB/s × 8 = 2.4 Gbps.

  c) One server handles 2,000 QPS. How many for peak + N+1 redundancy?
     15,000 / 2,000 = 7.5 → 8, +1 = ~9 servers.

  d) A request fans out to 10 services, each p99 = 50 ms. Rough user p99?
     ≈ 50 ms (the slowest call dominates; ~1 - 0.99^10 ≈ 10% hit a tail).
```

---

## Practice Exercises

### Exercise 1 — Easy: The bits/bytes and transfer-time drill

```
No tools needed — do these on paper, then verify with a calculator.

1. Convert:  500 Mbps → MB/s ?   2 Gbps → MB/s ?   25 MB/s → Mbps ?
2. How long to transfer a 4 GB video file over a 500 Mbps link?
3. A mobile user is on a 40 Mbps connection. Your homepage is 6 MB of
   assets (uncompressed). How many seconds to load, ignoring latency?
4. You gzip those assets down to 1.5 MB. New load time? What did you save?

# Goal: internalize "÷8" and "size ÷ bandwidth" until they're reflexes.
```

### Exercise 2 — Medium: Size a real service end to end

```
Design target: a URL-shortener like bit.ly.
  • 100M new short links created per day (writes)
  • Reads (redirects) are 100× the writes
  • Each stored record ≈ 500 bytes
  • Each redirect response ≈ 500 bytes (a 301 + tiny body)
  • Keep links for 5 years
  • Peak factor = 2×

Compute, showing your formulas:
  1. Write QPS (avg and peak)
  2. Read QPS (avg and peak)
  3. Peak read egress bandwidth in Gbps
  4. Total storage after 5 years (× 3 replication)
  5. If one server handles 5,000 QPS, how many read servers at peak (N+2)?
  6. Where is the network bottleneck, and would you add a CDN? Why/why not?

# Then run a real one for calibration:
curl -o /dev/null -s -w "redirect TTFB: %{time_starttransfer}s\n" -I https://bit.ly
```

### Exercise 3 — Hard (Production Simulation): Global chat latency budget

```
You're designing a global group-chat feature. Users in Frankfurt, Virginia,
and Sydney are in the SAME chat room. Origin/primary DB is in us-east-1.
A message must appear for all members "instantly."

Part A — Measure the physics:
  • From a cloud box (or estimate from the ladder), get RTTs:
      Frankfurt ↔ us-east-1, Sydney ↔ us-east-1, Frankfurt ↔ Sydney.
  • Use RTT_min ≈ distance_km/100 as your floor; real ≈ 1.5–2×.

Part B — Budget the naive design:
  A Sydney user sends a message → written to primary (us-east-1) →
  fanned out to Frankfurt + Sydney members. Trace the round trips and
  compute the delay before a Frankfurt member SEES the message.
  Then compute it for the SENDER'S OWN echo (Sydney → us-east-1 → Sydney).

Part C — Redesign to cut the tail:
  • Where would you put write primaries / regional relays?
  • Would you use a message queue (topic 38) and async fan-out? What does
    that buy you, and what consistency guarantee do you give up?
  • What's the LOWEST 'other members see it' latency physically possible for
    the Frankfurt↔Sydney pair, and why can no architecture beat it?

# Deliverable: a latency budget table (each hop + number) for naive vs
# redesigned, and one sentence naming the hard physical floor.
```

---

## Mental Model Checkpoint

Answer from memory. If you can't, re-read the relevant section.

1. **On the "human time" ladder (1 ns = 1 second), roughly how long is a RAM access vs a US↔Asia round trip?** Why does this comparison matter when you're deciding whether to cache something vs fetch it cross-region?

2. **A colleague says "we have a 1 Gbps link, so we can serve a 1 GB backup in one second."** What's wrong, and what's the real transfer time?

3. **Your endpoint does a 2 ms DB query and 1 ms of serialization. A user reports 220 ms latency.** Give the most likely cause and how you'd confirm it in one command.

4. **Write the estimation chain from DAU to server count.** (DAU → QPS → ? → ? → servers.) What number do you always size for — average or peak — and by what factor?

5. **A request fans out to 50 microservices in parallel, each with p50 = 5 ms and p99 = 80 ms. Roughly what is the user-facing latency, and why isn't it 5 ms?**

6. **A shopper in Singapore hits your us-east-1 origin. Which parts of a product page should live at the CDN edge, which at a regional replica, and which must go back to the origin — and what's the one operation that inherently pays the cross-region RTT?**

7. **When is throwing more bandwidth at a slow system the right fix, and when is it useless?** Give the diagnostic question that tells the two apart.

---

## Quick Reference Card

### The latency ladder (memorize the bold ones)

| Operation | Time | Human scale (1 ns = 1 s) |
|-----------|------|--------------------------|
| L1 cache reference | ~0.5 ns | 0.5 s |
| **RAM (main memory) reference** | **~100 ns** | 1.7 min |
| Compress 1 KB (snappy) | ~2 µs | ~30 min |
| **SSD random read (NVMe)** | **~16 µs** | 4.4 hr |
| **Round trip, same datacenter** | **~0.5 ms** | 5.8 days |
| Round trip, same region/AZ | ~1–2 ms | ~11 days |
| Disk seek (spinning HDD) | ~2–10 ms | ~2–4 months |
| **Send 1 MB over 1 Gbps** | **~8 ms** | ~3 months |
| Cross-country US RTT | ~30–70 ms | ~1.5 years |
| **US ↔ Europe RTT** | **~70–90 ms** | ~2.5 years |
| **US ↔ Asia RTT** | **~150–200 ms** | ~4.75 years |
| US ↔ Australia RTT | ~180–250 ms | ~6 years |

### The seven formulas

| # | Formula |
|---|---------|
| QPS | `QPS_avg = DAU × actions/day ÷ 86,400` (use ÷100,000); `peak = avg × 2–10` |
| Bits↔Bytes | `MB/s = Mbps ÷ 8`; `1 Gbps = 125 MB/s` |
| Transfer | `time = size ÷ bandwidth + RTT` |
| Distance floor | `RTT_min(ms) ≈ distance_km ÷ 100` |
| Egress | `bandwidth = QPS × payload`; `cost = GB × ~$0.05–0.09` |
| Servers | `ceil(QPS_peak ÷ per_server_QPS) × redundancy` |
| Storage | `writes/day × record_size × retention_days × replication` |

### Bandwidth conversions cheat block

```
1 Gbps  = 125 MB/s          100 Mbps = 12.5 MB/s
10 Gbps = 1.25 GB/s         1 Mbps   = 125 KB/s

Transfer time (theoretical), ÷8 already applied:
  1 GB over 1 Gbps   ≈ 8 s        10 GB over 1 Gbps  ≈ 80 s
  1 MB over 1 Gbps   ≈ 8 ms       1 MB over 100 Mbps ≈ 80 ms
```

### Trade-off decision cheat block

```
Slow + SMALL payload   → LATENCY problem  → move data closer (CDN, region), fewer round trips
Slow + BIG payload     → BANDWIDTH problem → compress, fatter pipe, chunk
Many parallel deps     → TAIL problem      → measure p99, hedge, timeout+partial
Cross-region on hot path → round-trip tax  → cache/replica for reads, async for writes
Chatty API (N calls)   → N×RTT             → batch into 1 chunky call
```

### The two walls

```
Network wall  ~0.5 ms  : on-box (ns–µs) vs off-box (ms) — 1000× jump
Light wall  dist/100 ms: geography — the ONLY fix is physical proximity
```

---

## When Would I Use This at Work?

### Scenario 1: Capacity planning for a launch

Product says "we're launching to 3M users, expect ~15 actions each per day." In 60 seconds you produce: `3M × 15 ÷ 100,000 ≈ 450 QPS avg, ~1,350 peak; at 2,000 QPS/server that's 1 box but run 3 for redundancy; at 20 KB payload that's ~27 MB/s ≈ 0.2 Gbps egress — trivial.` You've just sized the launch and told finance the bandwidth bill, before anyone wrote code. If the numbers had come out to 30,000 QPS and 12 Gbps, you'd have flagged "we need a CDN and autoscaling" in the same meeting.

### Scenario 2: The design interview

The prompt is "design Twitter's timeline" or "make our product fast for global users." You reach for the exact framing this topic teaches: estimate QPS/storage/bandwidth out loud, then say **"for global users: CDN at the edge for static, regional read replicas for reads, single-region primary for writes, minimize round trips, and async the writes that can tolerate eventual consistency."** That sentence, plus defensible numbers, is what gets the senior/staff signal.

### Scenario 3: The "why is it slow in Europe" incident

The dashboard shows US p50 = 40 ms, EU p50 = 400 ms. You don't guess — you apply the diagnostic: small payload + slow = latency (distance). You confirm with `ping` (~90 ms RTT to origin) and `curl -w` (TTFB ≈ 3× RTT from TCP+TLS+request). Root cause: single-region origin. Fix: regional deploy or edge caching. You resisted the tempting-but-wrong "let's buy more bandwidth," because you know a fat pipe doesn't shorten the Atlantic.

### Scenario 4: The cross-region N+1 that's killing p99

A page got slow after a "small refactor." You spot a loop that calls a service in another region 15 times. `15 × 80 ms = 1.2 s` of pure round trips. You batch it into one call (or move the dependency in-region) and p99 drops from 1.3 s to 150 ms. You caught it because you instinctively multiply *round trips × RTT*, not because a profiler pointed at CPU (it wouldn't have — the CPU was idle, waiting on the network).

---

## Connected Topics

**Study before this:**
- `01-how-the-internet-works.md` — the physical path and why distance = latency; this topic turns that into numbers.
- `33-network-latency.md` — the physics of latency and the speed-of-light limit; here we *use* those numbers for estimation.

**This topic connects forward to:**
- `30-cdns.md` — the primary tool for beating the speed-of-light wall: cache bytes at the edge so they never cross the ocean.
- `38-message-queues-at-network-level.md` — the sync-vs-async trade-off: how to hide a cross-region write behind a queue and buy fast UX at the cost of eventual consistency.
- `39-database-connections-over-network.md` — connection pooling (reuse the TCP+TLS handshake instead of paying the RTT again) and why chatty N+1 queries destroy latency.
- `36-service-to-service-communication.md` — keeping internal calls in-region and cheap.
- `37-grpc-and-protocol-buffers.md` — shrinking the payload term of the transfer-time formula.

**The one-line summary:** memorize the latency ladder, keep `Mbps ÷ 8 = MB/s` reflexive, estimate QPS → bandwidth → servers from users, and on every design *walk the request path, put a number on each arrow, and move the data closer to whichever hop crosses an ocean.*

---

*Doc saved: `/docs/networking/35-network-in-system-design.md`*
