# 12 — Numbers Every Engineer Should Know
## Category: HLD Fundamentals

---

## What is this?

This is a small set of numbers — about thirty of them — that let you predict how a system will behave *before you build it*. Things like "reading from memory takes ~100 nanoseconds", "a network round trip across the Atlantic takes ~150 milliseconds", "one Redis box handles ~100,000 operations per second".

Think of it like a carpenter knowing that a standard door is about 2 metres tall and a sheet of plywood is 8 feet by 4 feet. She doesn't measure every time. She glances at the room and says "that's three sheets, maybe four with offcuts." You want that same reflex for software: glance at a design and say "that's 5,000 QPS, ~2 TB a year, and the cache is what saves us."

These numbers are famously called **Jeff Dean's latency numbers** (Jeff Dean is a Google engineer who published the original list). You don't memorise them to be pedantic. You memorise them so that when someone proposes "let's just query the database inside that loop", you *feel* the cost immediately.

---

## Why does it matter?

**If you don't know these numbers, you cannot design.** You can only guess and hope.

Here's what goes wrong without them:

- You put a database call inside a loop that runs 1,000 times per request. Each call is a 500µs round trip. Your endpoint now takes 500ms minimum, and you have no idea why it's slow.
- You propose "we'll keep all user sessions in memory" without checking that 50 million sessions × 2 KB = 100 GB, which does not fit on one machine.
- You promise "99.99% uptime" in a design review without realising that's 52 minutes of downtime per *year* — meaning a single 1-hour outage blows your whole annual budget.
- You cache something that takes 200ns to compute, adding a 500µs network hop to "make it faster". You just made it 2,500× slower.

**The interview angle.** Every system design interview has a capacity estimation phase, usually 5-10 minutes in. The interviewer asks "how much storage will this need?" or "how many servers?". You are not being tested on arithmetic — you are being tested on whether you have *calibrated intuition*. A candidate who says "reads from SSD are roughly 100 microseconds, so at 10k QPS a single SSD-backed box is fine for this" sounds like an engineer. A candidate who says "um, disks are fast?" does not.

**The real-work angle.** These numbers are what you use to decide whether to add a cache, whether to shard, whether a feature is even feasible, and where to look first when something is slow. Performance debugging is mostly pattern-matching an observed latency against the table below. If a function takes 10ms and it "just does some math", the number tells you: it isn't just doing math. Something is hitting disk or the network.

Recall from [06 — Latency and Throughput](./06-latency-and-throughput.md) that latency is *time per operation* and throughput is *operations per second*. This doc gives you the actual values for both.

---

## The core idea — explained simply

### The "Everything Is Distance" Analogy

Imagine you are working at a desk, and you need information to do your job. Where the information lives determines how long you wait for it:

| Where the info is | How you get it | Real-world wait | The computer equivalent |
|---|---|---|---|
| **In your head** | You just know it | Instant | CPU register / L1 cache |
| **On the sticky note on your monitor** | Glance right | ~1 second | L2 cache |
| **In the drawer of your desk** | Reach down, open, look | ~2 minutes | Main memory (RAM) |
| **In the filing cabinet across the office** | Stand up, walk over, find the folder | ~1.5 days | SSD |
| **In the warehouse across town** | Drive there, search the shelves | ~4 months | Spinning hard disk seek |
| **In an archive in Amsterdam** | Book a flight, fly, retrieve, fly back | ~5 years | Network round trip to Europe |

That is not a cute exaggeration. That is **literally the ratio**, if you scale one nanosecond up to one second.

The whole discipline of system design is a fight against that table. Every cache, every replica, every CDN, every index — all of it is one idea: **move the data closer to the thing that needs it, because distance is time.**

### Making it visceral: the human-scale ladder

Multiply every latency by 1 billion (10⁹). Nanoseconds become seconds. Now a human can feel it:

```
  COMPUTER TIME          ×1,000,000,000        HUMAN TIME
 ───────────────────────────────────────────────────────────────────
  L1 cache read      0.5 ns      ────────▶     1 second   (a blink)
  Branch mispredict    5 ns      ────────▶     5 seconds
  L2 cache read        7 ns      ────────▶     7 seconds
  Mutex lock/unlock   25 ns      ────────▶     25 seconds
  Main memory (RAM)  100 ns      ────────▶     100 seconds  (~2 min)
  Read 1MB from RAM    3 µs      ────────▶     ~1 hour
  SSD random read    150 µs      ────────▶     1.7 DAYS
  Datacenter RTT     500 µs      ────────▶     6 days
  Read 1MB from SSD    1 ms      ────────▶     11 days
  Disk seek           10 ms      ────────▶     4 MONTHS
  Read 1MB from disk  20 ms      ────────▶     8 months
  CA → Netherlands   150 ms      ────────▶     4.8 YEARS
 ───────────────────────────────────────────────────────────────────
```

Now re-read this sentence: *"the API call makes 20 sequential database queries."* At datacenter-RTT scale, you just asked a human to make 20 six-day trips — four months of work — to answer one question. That's why N+1 queries kill you.

### The only three numbers you truly must memorise

If you forget everything else, keep these. They are three orders of magnitude apart, and almost every design decision falls out of them:

```
   ┌──────────────────────────────────────────────────────────┐
   │  MEMORY      ~100 ns        1×                           │
   │  SSD         ~100 µs        1,000× slower than memory    │
   │  CROSS-      ~100 ms        1,000,000× slower than memory│
   │  CONTINENT                  (1,000× slower than SSD)     │
   └──────────────────────────────────────────────────────────┘
```

Memory → SSD → intercontinental network. Each step is a thousand times worse. Everything else is detail.

---

## Key concepts inside this topic

### 1. The latency table (the canonical numbers)

These are the numbers to know. The "classic" column is Jeff Dean's original list, widely quoted in interviews. The "modern" column is what current hardware (NVMe SSDs, DDR4/DDR5) actually does. **Quote the classic numbers in interviews — everyone knows them — but know that modern hardware is faster in places.**

| Operation | Classic | Modern (2020s) | Notes |
|---|---|---|---|
| L1 cache reference | 0.5 ns | 0.5–1 ns | Effectively free |
| Branch mispredict | 5 ns | 3–5 ns | Why hot loops avoid unpredictable `if`s |
| L2 cache reference | 7 ns | 4–7 ns | ~14× L1 |
| Mutex lock/unlock | 25 ns | 15–25 ns | Uncontended. Contended is far worse |
| Main memory reference | 100 ns | 80–100 ns | **The anchor number** |
| Compress 1 KB with Snappy | 3 µs | 2–3 µs | Compression is cheap vs. network |
| Send 1 KB over 1 Gbps network | 10 µs | 10 µs | Pure serialization time |
| Read 1 MB sequentially from memory | 10 µs | ~3 µs | Memory bandwidth ~10-30 GB/s |
| SSD random read | 150 µs | 16–100 µs (NVMe) | **~1000× slower than RAM** |
| Round trip within same datacenter | 500 µs | 250–500 µs | The cost of *any* service-to-service call |
| Read 1 MB sequentially from SSD | 1 ms | 200–500 µs | ~1 GB/s+ on NVMe |
| Disk seek (spinning HDD) | 10 ms | 10 ms | Physics. Doesn't improve |
| Read 1 MB sequentially from disk | 20 ms | 20 ms | ~50-100 MB/s |
| Packet California → Netherlands → California | 150 ms | 150 ms | **Speed of light. Never improves.** |

Two things never get better, no matter what you buy:

1. **Disk seek time** — a physical arm has to move.
2. **Cross-continent latency** — light in fibre travels ~200,000 km/s, and the Earth is a fixed size. California to Amsterdam and back is ~18,000 km round trip, which is ~90ms of pure light-speed, plus routing overhead → ~150ms. **No amount of money fixes this.** The only fix is to not make the trip: put a CDN or a replica near the user.

### 2. Sequential vs random: the most under-appreciated distinction

Look again: SSD *random* read = 150µs for a small block. SSD *sequential* read = 1 MB in ~1ms. That means sequentially, you're pulling roughly 1 GB/s. But if you read 1 MB as 250 random 4 KB reads, you pay 250 × 150µs = **37.5ms** — nearly 40× worse for the same bytes.

This single fact explains a huge amount of database design:

- **B-tree indexes** exist to turn a random scan of a whole table into a few targeted reads.
- **LSM trees** (used by Cassandra, RocksDB, LevelDB) batch writes in memory and flush them *sequentially* to disk, because sequential writes are so much cheaper.
- **Kafka is fast** despite writing everything to disk, precisely because it only ever appends sequentially.

```js
// Same amount of data, wildly different cost.
// This is the shape of the problem, not a benchmark you should run as-is.

const ROWS = 250;

// BAD: 250 random reads, each paying full SSD latency (~150µs)
async function loadUsersNaive(ids, db) {
  const users = [];
  for (const id of ids) {
    users.push(await db.query('SELECT * FROM users WHERE id = $1', [id]));
  }
  return users; // ~250 × (500µs DC round trip) ≈ 125 ms. Ouch.
}

// GOOD: one round trip, one index range scan
async function loadUsersBatched(ids, db) {
  // Why: collapses N network round trips into 1, and lets the DB do a
  // single sequential-ish index read instead of N random probes.
  return db.query('SELECT * FROM users WHERE id = ANY($1)', [ids]);
  // ~500µs + a few ms of scan ≈ 2-3 ms. ~50× faster for zero extra infra.
}
```

The lesson: **before you add a cache, remove the round trips.** Batching is free; caching costs you a whole new system to operate.

### 3. Powers of two and data sizes

You need to convert "how many things" into "how many bytes" in your head. Memorise the exponents:

| Power | Exact value | Approx | Name | What it holds in practice |
|---|---|---|---|---|
| 2¹⁰ | 1,024 | 1 thousand | 1 KB | A short JSON blob; one small HTTP header set |
| 2²⁰ | 1,048,576 | 1 million | 1 MB | A high-res JPEG thumbnail; ~500 pages of plain text |
| 2³⁰ | 1,073,741,824 | 1 billion | 1 GB | ~2 hours of MP3; a small database's working set |
| 2⁴⁰ | ~1.1 × 10¹² | 1 trillion | 1 TB | A single big SSD; ~5 million photos at 200 KB each |
| 2⁵⁰ | ~1.1 × 10¹⁵ | 1 quadrillion | 1 PB | The scale of "we need a distributed file system" |

The trick that makes mental math possible: **2¹⁰ ≈ 10³.** So:

- 2 bytes × 1 million records = 2 MB.
- 1 KB × 1 billion records = 1 TB.
- 1 MB × 1 billion records = 1 PB.

That last line is the one that saves you in interviews. *"A billion of anything at 1 MB each is a petabyte."*

Other size facts worth carrying:

| Thing | Size |
|---|---|
| `int` / `int32` | 4 bytes |
| `bigint` / timestamp (ms) | 8 bytes |
| UUID (binary) | **16 bytes** (36 bytes as a hyphenated string — 2.25× bigger; this matters at scale) |
| A URL | ~100 bytes |
| A tweet / short post (text + metadata) | ~300 bytes |
| A typical database row (user profile) | ~1 KB |
| A JPEG photo from a phone | ~2 MB |
| One minute of 1080p video | ~50 MB (i.e. ~7 Mbps) |
| One hour of 1080p video | ~3 GB |

### 4. Time cheats

Estimation is almost always "X per day → Y per second". Memorise one number:

```
  1 day = 86,400 seconds  ≈  10^5 seconds   (round it. 100,000. always.)
  1 month ≈ 2.5 million seconds
  1 year  ≈ 31.5 million seconds  ≈  3 × 10^7 seconds
```

Then:

```
  QPS = (requests per day) / 100,000

  1 million requests/day     →   10 QPS
  10 million requests/day    →   100 QPS
  100 million requests/day   →   1,000 QPS
  1 billion requests/day     →   10,000 QPS
```

Look at that table for ten seconds. Internalise it. **A billion requests a day is only ten thousand QPS.** That's the single most important sanity check in system design — it stops you from over-engineering. And it stops you from under-engineering, too: 10,000 QPS *will not* fit on one Postgres box.

**Peak vs average.** Traffic is never flat. Use a multiplier:

- Peak QPS ≈ 2× average for a globally spread service.
- Peak QPS ≈ 5–10× average for a service concentrated in one timezone (or one with spikes: ticket sales, sports, news).

State your assumption out loud: *"I'll assume peak is 3× average, so 30,000 QPS at peak."* Interviewers love a stated assumption far more than a precise-sounding guess.

### 5. Throughput rules of thumb (what one box does)

This is how you turn QPS into a server count. These are deliberately rough — **order of magnitude only**, and they assume decent hardware and a workload that isn't pathological.

| Component | Rough capacity (one node) | Notes |
|---|---|---|
| **App server (Node.js)** | 1,000–10,000 QPS | Depends entirely on what the handler does. I/O-bound with async: high. CPU-bound (crypto, image resize): could be 100 QPS |
| **Nginx / reverse proxy** | 50,000+ QPS | Proxying is cheap; it's rarely your bottleneck |
| **Postgres / MySQL** | 5,000–15,000 simple QPS | Indexed point reads. Complex joins and aggregations: 10–100× less |
| **Postgres writes** | 1,000–5,000 writes/s | Bounded by fsync and WAL. Batch to go higher |
| **Redis / Memcached** | ~100,000 ops/s | Single-threaded but in-memory. Can hit 1M/s with pipelining |
| **Kafka broker** | ~100,000+ msg/s, ~100s of MB/s | Sequential append. Bounded by disk bandwidth and network, not IOPS |
| **Cassandra / DynamoDB node** | ~10,000 writes/s | Write-optimised (LSM) |
| **1 Gbps NIC** | ~125 MB/s | Divide bits by 8 |
| **10 Gbps NIC** | ~1.25 GB/s | Standard in cloud today |
| **RAM per commodity server** | 64–512 GB | 1 TB+ is available but pricey |
| **Cross-AZ bandwidth** | Charged per GB — treat as expensive | AWS charges for it; it shows up in bills fast |

The move in an interview: **QPS ÷ per-box capacity = number of boxes, then double it for headroom and redundancy.**

> "We need 30,000 read QPS. Postgres does ~10,000, so that's 3 boxes minimum — but I don't want a single primary handling that, so I'd put Redis in front. If we get an 80% cache hit rate, only 6,000 QPS reaches Postgres, and one primary plus one read replica comfortably covers it."

That paragraph is the entire skill. Notice it's just division plus the table above.

### 6. Storage cost — order of magnitude only

Prices change; ratios don't. Carry the **ratios**, and use the rough figures only to sanity-check whether an idea is affordable or insane.

| Tier | Rough $/GB/month | Relative cost | Use it for |
|---|---|---|---|
| **Blob / object storage** (S3, GCS) | ~$0.02 | **1×** | Photos, videos, backups, logs, anything big and cold |
| **Cold/archive tier** (Glacier) | ~$0.004 | 0.2× | Compliance archives you'll almost never read |
| **Network disk / SSD** (EBS gp3) | ~$0.08–0.10 | **~5×** | Database volumes |
| **RAM** (in a managed Redis/cache) | ~$3–5 | **~150–250×** | Hot working set only. Never "just keep it all in memory" |

Bandwidth (egress out of the cloud to the internet) is roughly **$0.05–0.09/GB** — which means *serving* data is often more expensive than *storing* it. That is the entire business case for a CDN.

**The one ratio to remember: RAM is ~100× more expensive per byte than blob storage.** So when you say "let's cache it", you are saying "let's pay 100× per byte for the 1% that's hot". That's a great deal — for the 1%. It's a catastrophe for the other 99%.

### 7. The availability "nines"

Availability is the fraction of time the system works. Engineers say "three nines" meaning 99.9%.

| Availability | Name | Downtime / year | Downtime / month | Downtime / week | What it implies |
|---|---|---|---|---|---|
| 90% | one nine | 36.5 days | 72 hours | 16.8 hours | A hobby project |
| 99% | two nines | 3.65 days | 7.2 hours | 1.68 hours | Internal tool |
| 99.9% | three nines | **8.77 hours** | 43.8 min | 10.1 min | Typical SaaS SLA |
| 99.95% | | 4.38 hours | 21.9 min | 5 min | Good SaaS SLA |
| 99.99% | four nines | **52.6 min** | 4.38 min | 1.01 min | Serious infra (needs multi-AZ, automated failover) |
| 99.999% | five nines | **5.26 min** | 26 sec | 6 sec | Telco-grade. Extremely expensive. Rarely needed |

Two things that catch people out:

**(a) Components in series multiply.** If your request passes through a load balancer (99.99%), an app server (99.9%), and a database (99.9%), your end-to-end availability is:

```
0.9999 × 0.999 × 0.999 = 0.9979  →  99.79%  ≈ 18 hours of downtime a year
```

Adding dependencies *lowers* availability. Every extra synchronous service call is a new way to fail.

**(b) Redundancy adds nines.** Two independent replicas, each 99% available, both down at once has probability 0.01 × 0.01 = 0.0001:

```
Availability = 1 − (0.01)² = 99.99%
```

Two mediocre things in parallel beat one good thing. That's why redundancy is the primary tool for availability. (The catch: "independent" is doing a lot of work in that sentence. If both replicas are in the same rack, or share a config file, or get the same bad deploy, they are not independent.)

See [07 — Availability and Reliability](./07-availability-and-reliability.md) for the deeper treatment.

---

## Visual / Diagram description

### Diagram 1 — The latency ladder

Draw this on the whiteboard when someone asks "where should this data live?". Each rung is roughly 10× the one above.

```
                          THE LATENCY LADDER
        (each step down ≈ 10× slower — the ladder spans 9 orders of magnitude)

  0.5 ns  ┌──────────────────────┐
          │  L1 CACHE            │  free
    7 ns  ├──────────────────────┤
          │  L2 CACHE            │  basically free
   25 ns  ├──────────────────────┤
          │  MUTEX LOCK          │  the cost of coordination
  100 ns  ├══════════════════════┤ ◀── ANCHOR: MAIN MEMORY (RAM)
          │  MAIN MEMORY         │
    3 µs  ├──────────────────────┤
          │  READ 1MB FROM RAM   │
   10 µs  ├──────────────────────┤
          │  SEND 1KB ON NETWORK │
  150 µs  ├══════════════════════┤ ◀── ANCHOR: SSD RANDOM READ (1,000× RAM)
          │  SSD RANDOM READ     │
  500 µs  ├──────────────────────┤
          │  SAME-DC ROUND TRIP  │  ← every microservice hop costs this
    1 ms  ├──────────────────────┤
          │  READ 1MB FROM SSD   │
   10 ms  ├──────────────────────┤
          │  DISK SEEK           │  ← physics: a moving arm
   20 ms  ├──────────────────────┤
          │  READ 1MB FROM DISK  │
  150 ms  ├══════════════════════┤ ◀── ANCHOR: CROSS-CONTINENT (1,000,000× RAM)
          │  CA ⇄ NETHERLANDS    │  ← physics: speed of light
          └──────────────────────┘

  RULE: A human notices ~100ms. You have ~200ms of total budget for a
        "snappy" API response. Count your rungs and see what fits.
```

The three `══` rungs are your anchors: **100ns, 100µs, 100ms.** If you remember only those three lines, you can reconstruct most decisions.

### Diagram 2 — Where the time actually goes in one web request

This is what the ladder looks like in a real request. Draw it to explain to a PM why "just add an index" isn't always the answer.

```
  USER (Mumbai)                                    SERVERS (us-east-1)
      │
      │  ── DNS lookup (cached: 0ms, cold: ~50ms) ──────────▶
      │
      │  ══ TCP + TLS handshake: 2-3 round trips ═══════════▶
      │     Mumbai → Virginia RTT ≈ 200ms  ×3  ≈ 600ms  ◀── THE KILLER
      │
      ├──────────────── HTTP request ──────────────────────▶┌───────────┐
      │                                                     │ LOAD BAL. │  ~1 ms
      │                                                     └─────┬─────┘
      │                                                           │ 0.5 ms (DC RTT)
      │                                                     ┌─────▼─────┐
      │                                                     │ APP SERVER│
      │                                                     └──┬─────┬──┘
      │                                       cache hit 0.5ms │     │ 5 ms
      │                                                  ┌────▼──┐ ┌▼────────┐
      │                                                  │ REDIS │ │ POSTGRES│
      │                                                  └───────┘ └─────────┘
      │◀─────────────── HTTP response ─────────────────── 200 ms (return trip)
      ▼
   TOTAL: ~800ms cold, ~210ms warm connection

   THE INSIGHT: the *server* work is ~7ms. Over 95% of the time is the
   speed of light between Mumbai and Virginia. Optimising the SQL query
   from 5ms to 2ms is invisible. Putting a CDN/edge POP in Mumbai —
   dropping RTT from 200ms to 10ms — is a 20× win.
```

**What this diagram teaches:** always find out *which rung of the ladder* dominates before you optimise. Engineers waste months tuning the 5ms while ignoring the 600ms.

---

## Real world examples

### Netflix — Open Connect, because 150ms is a law of physics

Netflix cannot make a packet cross the Atlantic faster. So instead of fighting the number, they moved the data. Netflix builds and ships its own caching appliances (**Open Connect Appliances**) and places them *inside internet service providers' networks* — physically close to subscribers. Popular titles are pre-loaded onto these boxes overnight, before anyone requests them.

The numbers drove the architecture: a video stream needs sustained bandwidth and low startup latency; a trans-continental fetch costs ~150ms per round trip and enormous egress bandwidth. Putting the bytes in the same city as the viewer collapses both. This is the "distance is time, so reduce distance" principle applied at national scale.

### Google — the numbers exist because Google needed them

Jeff Dean's list wasn't an academic exercise. Google's engineers were designing systems where the difference between a memory lookup and a disk seek, multiplied across billions of queries, decided whether a product was possible at all. The famous constraint: **a Google search must return in a few hundred milliseconds**, and it fans out to hundreds of machines to do it. If each of those fan-out calls cost a 10ms disk seek, the budget is blown instantly. So the search index is held **in RAM across thousands of machines** — deliberately paying the ~150× cost premium of RAM over disk, because the latency table said disk could not meet the requirement.

That's the lesson: the numbers don't just describe systems, they *dictate* them.

### Kafka — engineered directly around "sequential is 100× faster than random"

Kafka writes every message to disk, yet routinely handles hundreds of thousands of messages per second per broker. That seems to contradict "disk is slow" — until you look at *which* disk number applies. Kafka never seeks. It only ever **appends to the end of a log file** and reads forward. It also uses the OS page cache and `sendfile` (zero-copy) to avoid shuffling bytes through userspace.

So Kafka never pays the 10ms seek. It pays the sequential-throughput number (hundreds of MB/s), which is competitive with the network anyway. The entire design is a bet on one row of the latency table.

---

## Trade-offs

Knowing the numbers is free. **Acting** on them has costs.

| Decision | You gain | You give up |
|---|---|---|
| **Cache in RAM** (memory over SSD) | ~1,000× lower read latency | ~150× the cost per byte; a whole new system to operate; **cache invalidation** — now you can serve stale data |
| **Replicate near users** (CDN, geo-replicas) | Kills the 150ms cross-continent trip | Consistency headaches (see [08 — Consistency Models](./08-consistency-models.md)); more infra; more money |
| **Batch requests** | Removes N−1 round trips at ~500µs each | Higher latency for the *first* item in a batch; more complex code |
| **Add another 9 of availability** | Less downtime | Cost roughly *multiplies* per nine. 99.9% → 99.99% often means multi-AZ, automated failover, and an on-call rotation |
| **Denormalise / precompute** | Turns a slow join into a fast key lookup | Storage duplication; write amplification; data can drift out of sync |
| **Compress before sending** (Snappy: 3µs/KB) | 1 KB over 1 Gbps costs 10µs — compression pays for itself if it shrinks the payload meaningfully | CPU cost; useless (or harmful) on already-compressed data like JPEG/MP4 |

### The estimation trade-off itself

| Approach | Pros | Cons |
|---|---|---|
| **Precise math** (86,400 seconds, exact bytes) | Feels rigorous | Slow, error-prone under pressure, and *false precision* — your input assumptions are ±10× anyway |
| **Order-of-magnitude** (100,000 seconds/day, round to powers of ten) | Fast, robust, good enough to make the decision | Off by up to 2×; will annoy a pedantic interviewer (rare) |

**Rule of thumb:** round aggressively, but always **state the rounding out loud**. "I'll call a day 100,000 seconds — close enough to 86,400 for this." That single sentence tells the interviewer you're rounding on purpose, not because you can't divide.

**The sweet spot:** be precise about **orders of magnitude**, sloppy about the leading digit. Nobody cares whether it's 4 TB or 6 TB. Everybody cares whether it's 5 TB or 5 PB, because those are completely different architectures.

---

## Common interview questions on this topic

### Q1: "Roughly how long does it take to read 1 MB sequentially from memory, from SSD, and from disk?"

**Hint:** Memory ~10µs (classic) or ~3µs (modern). SSD ~1ms. Spinning disk ~20ms. The shape of the answer matters more than the digits: **SSD is ~100× slower than RAM for bulk reads, and disk is another ~20× slower than SSD.** Then add the insight: "and for *random* reads the gap widens hugely, because disk pays a 10ms seek per access — which is why databases work so hard to turn random access into sequential access."

### Q2: "Should we cache this? It's a computation that takes 200 nanoseconds."

**Hint:** **No — this is a trap question.** A Redis lookup costs a datacenter round trip: ~500µs. That's **2,500× slower** than just doing the computation. Caching only makes sense when the thing you're avoiding is *more expensive than the cache lookup itself* — i.e. a DB query (~5ms), an external API call (~100ms), or a heavy computation (>10ms). Say the numbers out loud; that's the whole answer.

### Q3: "Our service is 99.9% available. We add a synchronous call to a partner API that's 99.5% available. What's our new availability?"

**Hint:** Serial dependencies **multiply**: 0.999 × 0.995 = **0.994 → 99.4%**, which is ~52 hours of downtime a year, up from ~9. Then give the fix: make the call **asynchronous** (queue it), add a **timeout with a fallback/default**, or wrap it in a **circuit breaker** so the partner's outage degrades your feature instead of taking down your request path.

### Q4: "We store 500 million photos, averaging 2 MB each. How much storage, and roughly what does it cost per month?"

**Hint:** 5 × 10⁸ × 2 × 10⁶ B = **10¹⁵ bytes = 1 PB**. At blob-storage prices (~$0.02/GB/month) and 1 PB ≈ 10⁶ GB, that's **~$20,000/month**. Then earn the extra point: mention replication (×3 → ~$60k, though S3-class storage already prices in redundancy), and note that **egress bandwidth will likely cost more than the storage** if these photos are actually being viewed — which is the argument for a CDN.

### Q5: "How many app servers do we need for 1 billion requests per day?"

**Hint:** Walk it: 10⁹ / 10⁵ s = **10,000 QPS average**. Assume peak = 3× → **30,000 QPS**. One Node app server handles ~2,000–5,000 QPS for a light I/O-bound handler; take 2,500 to be safe → **12 servers**. Round up to **~15-20 for headroom and rolling deploys**, spread across at least 3 availability zones so losing one AZ doesn't halve your capacity. Say every assumption out loud — that's what's being scored.

---

## Practice exercise

### Exercise: "Estimate a Pastebin" — 30 minutes, pen and paper only

No calculator. No laptop. Scroll back up to the tables when you need a number — the point is learning to *reach for the right table*, not memorising it cold.

**The system:** a Pastebin clone. Users paste text, get back a short URL, and others read it.

**Given assumptions:**
- 10 million new pastes per day
- Read:write ratio of 100:1
- Average paste size: 10 KB
- Data retained for 5 years
- Users are global; you serve from one region in the US

**Produce a single page of notes answering:**

1. **Write QPS** (average and peak — state your peak multiplier and why).
2. **Read QPS** (average and peak).
3. **Storage after 5 years.** Show the arithmetic. Then add ~20% for metadata (URL key, user id, timestamps, expiry) and state your final number in the right unit.
4. **Bandwidth** in and out, in MB/s. Which direction dominates, and by how much?
5. **How many boxes?** Given your read QPS, decide: does this fit on one Postgres primary? Do you need a cache? If you add Redis at an 80% hit rate, how much read QPS actually reaches the database?
6. **The memory question.** If you wanted to cache the hottest 20% of pastes in Redis, how much RAM is that, and roughly what would it cost per month? Is that a good deal versus just reading from SSD? Justify with the latency numbers.
7. **The latency budget.** A user in Sydney reads a paste from your US-East servers. Write out the time budget rung by rung (RTT, TLS, app, DB/cache) and identify which single change would most improve their experience.

**Success criteria:** you should reach a defensible answer for every line, each traceable to a number in this doc, with every assumption written down. If your storage answer is in the right *order of magnitude* (tens of terabytes — not gigabytes, not petabytes), you've passed.

---

## Quick reference cheat sheet

- **The three anchors:** RAM ~**100 ns**, SSD ~**100 µs** (1,000×), cross-continent ~**100 ms** (1,000,000×). Everything else interpolates between these.
- **Human scale (×10⁹):** L1 = 1 second, RAM = 100 seconds, SSD read = 1.7 days, disk seek = 4 months, network trip to Europe = 4.8 years.
- **1 day ≈ 100,000 seconds** (10⁵). Therefore **1 billion requests/day = 10,000 QPS**. Divide by 100,000, always.
- **1 year ≈ 31.5 million seconds** ≈ 3 × 10⁷. 1 month ≈ 2.5 million seconds.
- **2¹⁰ ≈ 10³.** So 1 KB × 1 billion = 1 TB, and 1 MB × 1 billion = 1 PB.
- **Datacenter round trip ≈ 500 µs.** Every microservice hop, every DB call, every cache read pays this. Count your hops.
- **Sequential beats random by ~100×** on the same disk. This is why LSM trees, Kafka, and append-only logs exist.
- **Two things never improve:** disk seek (10 ms — physics of a moving arm) and speed of light (150 ms CA↔Europe). Don't optimise them — *avoid* them.
- **Server capacities:** app server ~1–10k QPS · Postgres ~5–15k simple reads/s · Redis ~100k ops/s · Kafka broker ~100k+ msg/s.
- **Object sizes:** UUID 16 B (binary) · URL ~100 B · tweet ~300 B · DB row ~1 KB · photo ~2 MB · minute of 1080p ~50 MB.
- **Cost ratios:** blob storage 1× · SSD ~5× · **RAM ~150×**. Cache the hot 1%, never everything.
- **Nines:** 99% = 3.65 days/yr · 99.9% = 8.8 hours/yr · 99.99% = 53 min/yr · 99.999% = 5 min/yr.
- **Serial dependencies multiply availability (worse); parallel redundancy squares the failure rate (better).**
- **Round aggressively, but say so.** "I'll call a day 100,000 seconds" beats fumbling with 86,400 every time.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [11 — How the Web Works](./11-how-the-web-works.md) — where the DNS, TCP, and TLS round trips in the latency budget actually come from |
| **Next** | [13 — OOP: The Four Pillars](./13-oop-four-pillars.md) — the HLD fundamentals are done; now the LLD track begins |
| **Related** | [05 — Capacity Estimation](./05-capacity-estimation.md) — the *method* for estimating; this doc is the *data* it runs on |
| **Related** | [06 — Latency and Throughput](./06-latency-and-throughput.md) — defines the concepts these numbers measure, including p50/p99 tails |
| **Related** | [07 — Availability and Reliability](./07-availability-and-reliability.md) — the nines table in depth: SLAs, SLOs, and error budgets |
| **Related** | [59 — Caching in Depth](./59-caching-in-depth.md) — the practical payoff: caching is just exploiting the RAM-vs-SSD-vs-network gap |
