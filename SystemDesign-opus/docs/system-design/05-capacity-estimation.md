# 05 — Capacity Estimation and Back-of-the-Envelope Math
## Category: HLD Fundamentals

---

## What is this?

Capacity estimation is doing **rough arithmetic on a whiteboard** to figure out how big your system actually needs to be: how many requests per second, how many terabytes of storage, how much bandwidth, how many servers.

It's exactly what a caterer does before a wedding. Nobody counts individual grains of rice. They think: *"300 guests, about 400g of food each, that's 120 kg. Call it 150 kg with a safety margin. That's six large pots and four cooks."* Nobody checks that against a scale afterwards — and nobody needs to. It got them within 20% of the truth in thirty seconds, and 20% is close enough to know whether you need four cooks or forty.

**You are not looking for the right answer. You are looking for the right order of magnitude.**

---

## Why does it matter?

Order of magnitude is the whole game. Consider two possible answers to "how much storage do we need over five years?"

- **200 GB** → one machine. A laptop, honestly. No sharding. Boring, and *correct*.
- **200 TB** → dozens of machines. Sharding is now mandatory. Your entire architecture changes.

Those two answers demand completely different systems. If you can't tell which one you're in, **you cannot design anything.** You will either over-engineer (sharding a database that fits on a phone) or under-engineer (one Postgres box melting under 200 TB).

**The interview angle:** When you say "we need a cache," the interviewer will ask "why?" The only winning answer is a number: *"Because peak read QPS is 460,000. A single Postgres primary handles maybe 5,000 reads per second. The cache isn't optional — the math forces it."* Estimation is what turns your design from opinion into argument.

**The real-work angle:** You cannot open an AWS console and provision infrastructure without these numbers. "How many Redis nodes?" "How much will S3 cost per month?" "Will one Kafka partition keep up?" Every one of those is a back-of-the-envelope calculation, and getting it wrong by 10x is either a wasted budget or an outage.

Recall from [04 — Functional vs Non-Functional Requirements](./04-functional-vs-non-functional-requirements.md) that an NFR without a number is just a wish. **This document is how the wish becomes a number.**

---

## The core idea — explained simply

### The Wedding Caterer Analogy

The caterer's method is four moves, and they are *exactly* the four moves of capacity estimation:

**1. Start from the one number you actually know.** *300 guests.* Everything else derives from it. In system design that number is almost always **DAU — daily active users.** If the interviewer doesn't give it, ask. If they say "you tell me," pick something sane and say it out loud: *"I'll assume 100 million DAU."*

**2. Multiply by a per-unit rate.** *400g of food per guest.* In system design: *2 tweets posted per user per day; 50 photos viewed per user per day.* These are assumptions, and that's fine — **state them out loud.** A stated assumption is a design input; an unstated one is a landmine.

**3. Round aggressively.** *"120 kg... call it 150."* The caterer doesn't compute 118.4 kg. You don't compute 4,629.6 requests per second. You say **"about 5,000."** One significant figure, then move on.

**4. Add headroom for the peak.** The caterer knows some guests eat double. You know traffic isn't flat across 24 hours — everyone opens the app at 9am and 8pm. **Peak is 2–3x the average.** Design for the peak, always.

| Caterer | System designer |
|---------|----------------|
| 300 guests | 100M daily active users |
| 400g of food per guest | 2 writes + 100 reads per user per day |
| 120 kg of food | 200M writes/day, 10B reads/day |
| "Call it 150 kg" | "Call it 2,500 QPS write, 120,000 QPS read" |
| Six pots, four cooks | 40 app servers, 20 cache nodes, 12 DB shards |
| Some guests eat double | Peak QPS = 3x average |

The caterer who skips this ends up with 300 hungry guests and two trays of samosas. The engineer who skips it ends up on a 3am incident call.

---

## Key concepts inside this topic

### 1. The Numbers You Must Memorise

**Powers of two → data sizes.** This is the single most useful table in system design.

| Power | Approximate value | Full name | Short name |
|-------|------------------|-----------|-----------|
| 2^10 | 1 thousand (1,024) | 1 Kilobyte | 1 KB |
| 2^20 | 1 million | 1 Megabyte | 1 MB |
| 2^30 | 1 billion | 1 Gigabyte | 1 GB |
| 2^40 | 1 trillion | 1 Terabyte | 1 TB |
| 2^50 | 1 quadrillion | 1 Petabyte | 1 PB |

Read it as: **KB → MB → GB → TB → PB, each step is ×1,000.** That's all you need. Nobody will ever ask you for 1,024 vs 1,000.

**Typical object sizes** (memorise the ballpark, not the exact figure):

```
A UUID / an ID                      16 bytes
A timestamp                          8 bytes
A tweet (text + metadata)          ~500 bytes    (round UP — metadata always surprises you)
A row of user metadata               ~1 KB
A URL                              ~100 bytes
A JPEG thumbnail                    ~20 KB
A compressed photo (web-sized)     ~200 KB
An original phone photo             ~3 MB
1 minute of 1080p video            ~50 MB
```

**Latency numbers** (covered fully in [12 — Numbers Every Engineer Should Know](./12-numbers-every-engineer-should-know.md)):

```
Main memory (RAM) reference           ~100 ns
SSD random read                       ~100 µs   (1,000x slower than RAM)
Round trip within the same datacenter  ~0.5 ms
Disk (HDD) seek                        ~10 ms
Round trip California → Netherlands   ~150 ms
```

**The one takeaway:** RAM is roughly **1,000,000x** faster than a cross-continent network round trip, and ~1,000x faster than an SSD read. That single fact is why caches exist.

### 2. The Rounding Tricks (these make you fast)

**Trick 1 — the 100,000-second day. This is the most important trick in the document.**

```
1 day = 24 × 60 × 60 = 86,400 seconds.
DO NOT divide by 86,400 on a whiteboard. Divide by 100,000 = 10^5.
You land ~15% low — which is FINE, and safe, because you're about to
multiply by a 2-3x peak factor anyway. The margin swallows the error.

QPS ≈ (requests per day) / 100,000

    1 million/day  →      10 QPS
   10 million/day  →     100 QPS
  100 million/day  →   1,000 QPS
    1 billion/day  →  10,000 QPS
   10 billion/day  → 100,000 QPS
```

Learn that ladder. Someone says "1 billion requests a day" and you answer **"so about 10,000 QPS"** without pausing. It's a small thing that makes you look extremely comfortable with scale.

**Trick 2 — one significant figure, always.** `86,400 × 4,630 = 400,032,000` → just say **400 million.**

**Trick 3 — round the scary direction.** A 280-char tweet could be 280 bytes. Call it 500 and cover the ID, userId, timestamp, and indexes you forgot. Over-estimating storage wastes money; under-estimating it causes an outage.

**Trick 4 — ×365×5 ≈ ×2,000.** Worth memorising: **daily storage × 2,000 ≈ 5-year storage.**

### 3. The Formula Sheet

Every capacity estimate you will ever do is these seven lines.

```
1. WRITES/DAY   = DAU × (writes per user per day)
2. READS/DAY    = DAU × (reads  per user per day)
3. QPS          = (requests per day) / 100,000
4. PEAK QPS     = average QPS × 2  (or × 3 for spiky consumer apps)
5. STORAGE/DAY  = writes/day × (bytes per record)
6. STORAGE/5YR  = storage/day × 365 × 5   ≈  storage/day × 2,000
   (then × replication factor, usually × 3)
7. BANDWIDTH    = QPS × (bytes per payload)
```

Plus two derived ones:

```
8. CACHE MEMORY = (daily read volume in bytes) × 20%     [the 80/20 rule]
9. SERVERS      = peak QPS / (QPS one server handles)  × 2   [×2 for headroom + failures]
```

**The 80/20 rule (Pareto):** roughly **20% of your data serves 80% of your traffic.** Hot tweets, popular URLs, celebrity profiles. So you don't cache everything — you cache the hot 20%, and you get 80%+ of the benefit for 20% of the RAM. In practice teams tune this to whatever hit rate the budget buys, but 20% is the right starting assumption on a whiteboard.

**How many QPS does one server handle?** Rough, defensible whiteboard figures — say these as estimates, not gospel:

```
One Node.js app server (I/O-bound, cache hits):     ~1,000 QPS
One PostgreSQL primary (simple indexed reads):    ~5,000 QPS
One Redis node (in-memory, single-threaded):    ~100,000 QPS
```

That last one is why a cache is not a nice-to-have. **One Redis node absorbs the work of twenty Postgres boxes.**

### 4. Worked Example 1 — Twitter

> "Design Twitter for 200 million daily active users."

**Step 1 — State the assumptions out loud.**
```
DAU                       = 200 million
Tweets posted per user    = 2 per day
Tweets READ per user      = 100 per day  (they scroll a lot)
Media                     = 10% of tweets have an image, avg 200 KB
Tweet record size         = 500 bytes  (text + id + userId + timestamp + indexes)
```

**Step 2 — Writes per day, and write QPS.**
```
Writes/day = 200M users × 2 tweets = 400M tweets/day

Write QPS  = 400,000,000 / 100,000
           = 4,000 writes/sec           ← round: "about 4,000"

Peak write QPS = 4,000 × 3 = 12,000 writes/sec
```

**Step 3 — Reads per day, and read QPS.**
```
Reads/day = 200M users × 100 tweets = 20,000,000,000 = 20 BILLION reads/day

Read QPS  = 20,000,000,000 / 100,000
          = 200,000 reads/sec

Peak read QPS = 200,000 × 3 = 600,000 reads/sec
```

**Step 4 — The read:write ratio (the single most design-defining number).**
```
Read:Write = 20B reads : 400M writes
           = 20,000 / 400
           = 50 : 1        ← massively read-heavy
```
> **Design consequence, say it out loud:** "50:1 read-heavy at 600k peak read QPS. One Postgres primary does maybe 5,000 reads/sec — I'd need 120 of them, which is absurd. So the timeline **must** be served from an in-memory cache, and because reads dominate this heavily, I should **precompute** the timeline at write time (fanout-on-write) rather than compute it on every read."

That sentence is the entire architecture, and the math produced it.

**Step 5 — Storage.**
```
TEXT:
  400M tweets/day × 500 bytes = 200,000,000,000 bytes = 200 GB/day

  Per year:   200 GB × 365           = 73 TB/year
  Over 5 yrs: 73 TB × 5              = 365 TB
  × 3 replication                    = ~1.1 PB

MEDIA (this is where it gets real):
  10% of 400M = 40M images/day
  40M × 200 KB = 8,000,000,000,000 bytes = 8 TB/day

  Per year:   8 TB × 365             = 2,920 TB ≈ 2.9 PB/year
  Over 5 yrs: 2.9 PB × 5             = ~15 PB
```
> **Design consequence:** "Media is **40x** bigger than text. Text (1.1 PB) goes in a sharded database. Media (15 PB) does NOT — that goes in object storage like S3, and the database stores only the URL. And 365 TB of text will never fit on one machine, so sharding is mandatory, not a choice."

**Step 6 — Bandwidth.**
```
WRITE bandwidth:
  text:  4,000 QPS × 500 bytes  = 2 MB/s          (trivial)
  media: 40M images/day / 100,000 = 400 images/s
         400 × 200 KB           = 80 MB/s          (fine for one ingest tier)

READ bandwidth:
  text:  200,000 QPS × 500 bytes = 100 MB/s        (fine)
  media: assume 10% of tweet-reads load an image
         200,000 × 10% = 20,000 image reads/s
         20,000 × 200 KB        = 4,000 MB/s = 4 GB/s   ← !!
```
> **Design consequence:** "4 GB/s of image traffic will destroy any origin server. That traffic goes to a **CDN**. At a 95% CDN hit rate, my origin only sees 5% × 4 GB/s = **200 MB/s** — which is completely manageable. The CDN isn't a performance nicety here; it's the difference between a system that works and one that doesn't."

**Step 7 — Cache memory (80/20 rule).**
```
Daily read volume (text) = 20B reads × 500 bytes = 10,000 GB = 10 TB/day
Cache the hot 20%        = 10 TB × 0.20 = 2 TB of RAM

Redis nodes @ 64 GB usable each:
  2,000 GB / 64 GB = ~32 nodes
  × 2 for replication = ~64 Redis nodes
```

**Step 8 — Servers.**
```
Peak total QPS = 600,000 reads + 12,000 writes ≈ 612,000 QPS  → call it 600k

App servers @ 1,000 QPS each (mostly serving cache hits):
  600,000 / 1,000 = 600 servers
  × 2 for headroom + failure tolerance = ~1,200 app servers
```

**The summary you write on the board:**
```
┌─────────────────────────────────────────────────────────┐
│ TWITTER — 200M DAU                                       │
│  Write QPS:     4,000 avg    /  12,000 peak              │
│  Read  QPS:   200,000 avg    / 600,000 peak              │
│  Read:Write:  50:1  → cache + fanout-on-write REQUIRED   │
│  Storage:     200 GB/day text  → 365 TB / 5yr → SHARD    │
│               8 TB/day media   →  15 PB / 5yr → S3       │
│  Bandwidth:   4 GB/s media reads → CDN REQUIRED          │
│  Cache:       ~2 TB RAM (~64 Redis nodes)                │
│  App servers: ~1,200                                     │
└─────────────────────────────────────────────────────────┘
```

### 5. Worked Example 2 — A URL Shortener

> "Design bit.ly. 100 million new URLs created per day."

**Step 1 — Assumptions.**
```
New URLs/day    = 100 million
Read:write      = 100 : 1   (a short link is created once, clicked many times)
Record size     = 500 bytes (short key 7B + long URL ~100B + userId + timestamps + INDEX overhead)
Retention       = 5 years
```

**Step 2 — QPS.**
```
Write QPS = 100,000,000 / 100,000 = 1,000 writes/sec
Peak      = 1,000 × 3             = 3,000 writes/sec

Reads/day = 100M × 100            = 10,000,000,000 = 10 billion redirects/day
Read QPS  = 10,000,000,000 / 100,000 = 100,000 reads/sec
Peak      = 100,000 × 3           = 300,000 reads/sec
```

**Step 3 — Storage.**
```
Storage/day = 100M × 500 bytes = 50,000,000,000 bytes = 50 GB/day

Over 5 years = 50 GB × 365 × 5
             = 50 GB × 1,825
             ≈ 91 TB              (remember the shortcut: daily × 2,000 ≈ 100 TB. Close enough.)

× 3 replication = ~275 TB
```

**Step 4 — The key-space question (unique to this problem).**
```
Total URLs over 5 years = 100M/day × 365 × 5 = 182,500,000,000 ≈ 182 billion keys

How many characters does the short key need?
Base62 alphabet = [a-z A-Z 0-9] = 62 characters

  62^5 =         916,132,832  ≈  0.9 billion   ← way too few
  62^6 =      56,800,235,584  ≈   57 billion   ← still not enough for 182B
  62^7 =   3,521,614,606,208  ≈  3.5 TRILLION  ← plenty, with room to spare

→ A 7-character key is the answer.
```
This is a beautiful example of estimation *directly producing a design decision*. The number 7 is not a guess. It fell out of the arithmetic.

**Step 5 — Bandwidth.**
```
Write: 1,000 QPS × 500 bytes  = 500 KB/s     (nothing)
Read:  100,000 QPS × 500 bytes = 50 MB/s     (very manageable)
```
> **Note:** a redirect response is tiny — an HTTP 301 with a `Location` header, maybe 500 bytes on the wire. Bandwidth is a **non-issue** here. Contrast with Twitter's 4 GB/s of images. Different systems have different bottlenecks, and estimation is how you find out which one you're in.

**Step 6 — Cache (80/20).**
```
Assume ~500M distinct URLs are clicked on a given day.
Hot 20% = 100M URLs
100M × 500 bytes = 50,000,000,000 bytes = 50 GB of cache

→ Fits comfortably in a small Redis cluster (2-3 nodes with replication).
```

**Step 7 — Servers.**
```
Peak read QPS = 300,000
Redis handles ~100,000 QPS/node → 3 nodes minimum, ×2 for replicas = 6

App servers @ 1,000 QPS: 300,000 / 1,000 = 300 servers, ×2 headroom = ~600
```

**The takeaway you say out loud:** *"This system is read-heavy but tiny in bytes — 91 TB of text over five years, and only 50 MB/s of bandwidth. The hard problems here aren't storage or bandwidth at all; they're (a) generating 182 billion unique keys without collisions across many servers, and (b) absorbing 300k peak read QPS. So the design centres on a key-generation strategy and a big cache, not on sharding for size."*

### 6. Worked Example 3 — An Image Host (Instagram-style)

> "500 million DAU. Users upload photos and scroll a feed of photos."

**Step 1 — Assumptions.**
```
DAU                = 500 million
Uploads per user   = 0.2 per day   (i.e. the average user posts once every 5 days)
Photos viewed/user = 50 per day
Per upload we store 4 sizes: original(compressed) + large + medium + thumb
                   ≈ 1.5 MB total
Average photo SERVED (feed shows medium size) ≈ 200 KB
Metadata row       = 500 bytes
```

**Step 2 — QPS.**
```
Uploads/day = 500M × 0.2 = 100,000,000 = 100M uploads/day
Write QPS   = 100,000,000 / 100,000 = 1,000 uploads/sec
Peak        = 1,000 × 3             = 3,000 uploads/sec

Views/day   = 500M × 50 = 25,000,000,000 = 25 billion photo views/day
Read QPS    = 25,000,000,000 / 100,000 = 250,000 views/sec
Peak        = 250,000 × 3             = 750,000 views/sec

Read:Write  = 25B : 100M = 250 : 1     ← extremely read-heavy
```

**Step 3 — Storage. This is where the system's real problem lives.**
```
BLOB storage:
  100M uploads/day × 1.5 MB = 150,000,000 MB = 150 TB/day  (!!)

  Per year:   150 TB × 365 = 54,750 TB ≈ 55 PB/year
  Over 5 yrs: 55 PB × 5    = ~275 PB

  3x replication would mean 825 PB. That is financially insane.
  → Use ERASURE CODING instead (~1.5x overhead for similar durability):
    275 PB × 1.5 = ~410 PB

METADATA (the database):
  100M rows/day × 500 bytes = 50 GB/day
  Over 5 years: 50 GB × 1,825 = ~91 TB
```
> **Design consequence:** "The blobs and the metadata are **four orders of magnitude apart** — 410 PB vs 91 TB. They absolutely do not belong in the same store. Photos go in object storage with erasure coding; the database holds only `photoId, userId, s3Url, caption, createdAt` at 91 TB, which is a very ordinary sharded database. The single biggest design decision in this system fell straight out of comparing two numbers."

**Step 4 — Bandwidth. The other real problem.**
```
UPLOAD (write) bandwidth:
  1,000 uploads/sec × 1.5 MB = 1,500 MB/s = 1.5 GB/s

VIEW (read) bandwidth:
  250,000 views/sec × 200 KB = 50,000,000 KB/s = 50 GB/s     ← at AVERAGE
  At peak (750k QPS):                          = 150 GB/s     ← !!!
```
> **Design consequence:** "150 GB/s at peak is far beyond what any origin fleet should serve. This traffic **must** be served by a CDN. At a 95% cache hit rate, origin egress drops to 5% × 150 GB/s = **7.5 GB/s** — still large, but a manageable origin fleet. Notice the CDN here isn't about latency; it's about the origin literally being unable to push those bytes."

**Step 5 — Cache (for metadata / feeds, not the images — the CDN caches those).**
```
Daily metadata read volume = 25B views × 500 bytes = 12,500 GB = 12.5 TB/day
Hot 20%                    = 2.5 TB of RAM
@ 64 GB per Redis node     = ~40 nodes, ×2 replication = ~80 nodes
```

**The three-system comparison — this is the lesson:**

| | Twitter | URL Shortener | Image Host |
|---|---|---|---|
| Peak read QPS | 600k | 300k | 750k |
| Read:write | 50:1 | 100:1 | 250:1 |
| 5-yr storage | 365 TB text + 15 PB media | **91 TB** | **410 PB** |
| Peak read bandwidth | 4 GB/s | **50 MB/s** | **150 GB/s** |
| **The hard problem** | Fanout at scale | Unique key generation | Storage + bandwidth cost |

Three systems, similar QPS, **wildly** different bottlenecks — and the arithmetic is the only thing that told you which one you were designing. That is why you never skip this step.

### 7. Doing the Math in Code (a Node.js estimator)

Useful for real work, and worth writing once so the formulas stick.

```javascript
// estimator.js — the seven formulas, as code.
// On a whiteboard you do this by hand. At work you script it once, so when the
// PM asks "what if we 5x?" the answer takes a second instead of a meeting.

const SECONDS_PER_DAY = 86_400;
const MB = 1e6, GB = 1e9, TB = 1e12;

function estimate({
  dau, writesPerUserPerDay, readsPerUserPerDay, bytesPerRecord,
  bytesPerReadPayload = bytesPerRecord,
  peakMultiplier = 3,     // consumer apps are spiky: everyone opens at 9am and 8pm
  replicationFactor = 3,
  years = 5,
  hotDataFraction = 0.2,  // the 80/20 rule: 20% of the data serves ~80% of reads
}) {
  const writesPerDay = dau * writesPerUserPerDay;
  const readsPerDay = dau * readsPerUserPerDay;
  const writeQps = writesPerDay / SECONDS_PER_DAY;
  const readQps = readsPerDay / SECONDS_PER_DAY;

  // Peak is what you must SURVIVE. Average is only what you pay for on a good day.
  const peakWriteQps = writeQps * peakMultiplier;
  const peakReadQps = readQps * peakMultiplier;

  const storagePerDay = writesPerDay * bytesPerRecord;
  const cacheMemory = readsPerDay * bytesPerReadPayload * hotDataFraction;

  return {
    readQps: Math.round(readQps),
    peakReadQps: Math.round(peakReadQps),
    readWriteRatio: Math.round(readsPerDay / writesPerDay),
    storagePerDayGB: +(storagePerDay / GB).toFixed(1),
    storage5yrTB: +((storagePerDay * 365 * years * replicationFactor) / TB).toFixed(0),
    // Bandwidth uses PEAK — the network carries the worst minute, not the mean one.
    peakReadBandwidthMBs: +((peakReadQps * bytesPerReadPayload) / MB).toFixed(0),
    cacheMemoryGB: +(cacheMemory / GB).toFixed(0),
    // ×2 everywhere: you never run at 100%, and losing a node must not cascade.
    appServers: Math.ceil((peakReadQps + peakWriteQps) / 1_000) * 2,
    redisNodes: Math.ceil(cacheMemory / (64 * GB)) * 2,
  };
}

console.log(estimate({ dau: 200e6, writesPerUserPerDay: 2, readsPerUserPerDay: 100, bytesPerRecord: 500 }));
// Twitter → readQps ~231k · peakReadQps ~694k · readWriteRatio 50
//           storage5yrTB ~1095 · cacheMemoryGB ~2000 · redisNodes ~64
```

The script's numbers land ~15% **higher** than the whiteboard versions, because it divides by the true 86,400 while your head divides by 100,000. **That gap is exactly why the whiteboard method is safe:** you round traffic slightly down, then multiply by a 3x peak factor — the margin swallows the error whole.

---

## Visual / Diagram description

### Diagram 1: The Estimation Funnel — one input, everything else derived

```
                    ┌──────────────────┐
                    │       DAU        │  ← THE ONLY NUMBER YOU ACTUALLY
                    │  (200,000,000)   │     NEED. Ask for it. Everything
                    └────────┬─────────┘     below is derived from it.
                             │
          ┌──────────────────┴──────────────────┐
          │ × writes/user/day    × reads/user/day│
          ▼                                     ▼
  ┌──────────────────┐                ┌──────────────────┐
  │  WRITES / DAY    │                │   READS / DAY    │
  │      400M        │                │       20B        │
  └────────┬─────────┘                └────────┬─────────┘
           │ ÷ 100,000                         │ ÷ 100,000
           ▼                                   ▼
  ┌──────────────────┐                ┌──────────────────┐
  │ WRITE QPS  4,000 │                │ READ QPS 200,000 │
  │ peak ×3  = 12k   │                │ peak ×3  = 600k  │
  └────────┬─────────┘                └────────┬─────────┘
           │ × bytes/record      ┌─────────────┼─────────────┐
           ▼            × bytes  ▼      × 20%  ▼   ÷ QPS/srv ▼
  ┌──────────────────┐  ┌────────────┐ ┌────────────┐ ┌────────────┐
  │ STORAGE/DAY      │  │ BANDWIDTH  │ │   CACHE    │ │  SERVERS   │
  │    200 GB        │  │  4 GB/s    │ │   2 TB     │ │   1,200    │
  └────────┬─────────┘  └────────────┘ └────────────┘ └────────────┘
           │ × 365 × 5 × 3 (replication)     ▲
           ▼                                 └── the 80/20 rule:
  ┌──────────────────┐                           20% of the data serves
  │ STORAGE / 5 YRS  │                           80% of the reads
  │     ~1.1 PB      │
  └──────────────────┘
```

**How to read it:** every number in the system flows down from **one** input. Get DAU, and the rest is multiplication and division. Redraw this from memory — it is genuinely all there is.

### Diagram 2: Peak vs Average — why you multiply by 3

```
  QPS
   │
600k├ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─╭─╮─ ─ ─ ─ ─  ← PEAK. You must SURVIVE this.
   │                      ╭╮         │ │             Provision for THIS line.
   │                     ╭╯╰╮       ╭╯ ╰╮
400k├                   ╭╯   ╰╮     ╭╯   ╰╮
   │                   ╭╯     ╰─╮  ╭╯     ╰╮
200k├─ ─ ─ ─ ─ ╭──╮─ ─╭╯ ─ ─ ─ ─╰╮╭╯ ─ ─ ─ ─╰─╮─ ─  ← AVERAGE. What the naive
   │     ╭─────╯  ╰───╯          ╰╯           ╰──╮    calculation gives you.
   │╭────╯                                       ╰──  Provision for THIS = outage.
  0└┴────┴────┴────┴────┴────┴────┴────┴────┴────┴──▶
   0    3am  6am   9am  noon  3pm  6pm   9pm  midnight
                    ▲               ▲
             morning commute   evening scroll (the real peak)
```

**What it shows:** traffic is never flat. Provision 200k QPS because that's the "average," and every evening at 8pm your system takes 600k and falls over — nightly, at exactly the hour your users care most. **Provision for the peak: ≈2x for enterprise/B2B (usage spreads over a workday), ≈3x for consumer apps, far more for event-driven spikes** — a stadium ticket drop can be 100x normal in the first sixty seconds.

---

## Real world examples

### 1. Jeff Dean's "Numbers Every Programmer Should Know"

Jeff Dean, one of Google's most influential engineers, popularised a now-famous list of latency numbers (RAM ~100ns, disk seek ~10ms, cross-continent round trip ~150ms). The point of the list was that Google engineers should be able to *estimate the performance of a design before building it* — to know, from arithmetic alone, whether a proposal was even plausible.

That is the philosophy behind this entire document. Estimation isn't busywork you do before the design; **estimation is a form of design.** It kills bad architectures on the whiteboard, where they're free to kill, rather than in production, where they aren't.

### 2. The Object-Storage Split, Everywhere

Look again at the image-host example: 410 PB of photos, 91 TB of metadata. Every large photo and video platform lands on the same conclusion because the same arithmetic applies to all of them — **blobs go in object storage (S3, GCS, or an in-house equivalent), and the database stores only a pointer to them.**

Nobody reached that by intuition. They reached it by noticing the two numbers are four orders of magnitude apart, and that a relational database is a catastrophically expensive place to keep 410 PB of JPEGs.

### 3. CDNs Exist Because of a Bandwidth Calculation

A CDN is usually explained as a latency optimisation — "serve content from close to the user." True, and it's the *second* reason.

The first reason is the one our image-host math surfaced: **150 GB/s at peak.** No reasonable origin fleet can push that. Edge caches absorb 90-99% of it and only the misses reach your origin. Every large media platform on the internet is CDN-fronted for this reason — and you can derive the necessity of it in four lines of arithmetic, which is exactly what you should do out loud in an interview.

---

## Trade-offs

| Choice | Pro | Con |
|--------|-----|-----|
| **Estimate in detail** | Every architectural decision is defensible with a number | Eats interview time; can rabbit-hole into arithmetic theatre |
| **Skip estimation** | Straight to the fun diagrams | "We need sharding" becomes unsupported hand-waving |
| **Round aggressively (÷100k)** | Fast, no calculator, ~15% error absorbed by the peak multiplier | Not precise — never use it for a real capacity purchase order |
| **Compute exactly (÷86,400)** | Precise | Slow on a whiteboard; false precision on top of guessed inputs |
| **Over-estimate (round up)** | Safe — you over-provision, users never notice | Wasted money |
| **Under-estimate (round down)** | Cheaper | Outage. Always round the scary direction. |

| Peak multiplier | When to use it |
|----------------|---------------|
| **× 2** | Enterprise / B2B apps — usage spread across a workday |
| **× 3** | Consumer apps — sharp morning and evening peaks |
| **× 10+** | Event-driven — ticket sales, product launches, live sports, breaking news |

**Rule of thumb:** spend **5 minutes**, get within **one order of magnitude**, and always **state your assumptions out loud**. An interviewer will never fault you for assuming 2 tweets/user/day instead of 3. They will absolutely fault you for a silent, unstated, load-bearing guess — because at work, that's the guess that takes down production.

---

## Common interview questions on this topic

### Q1: "Estimate the storage needed for a system storing 500M photos per day for 5 years."
**Hint:** Ask what an average photo weighs *after* compression, and whether you store multiple sizes. Assume ~1.5 MB across all variants. Then: `500M × 1.5 MB = 750 TB/day`. `750 TB × 365 × 5 = ~1,370 PB ≈ 1.4 EB`. Then immediately say the important part: "That rules out 3x replication on cost grounds — I'd use erasure coding at ~1.5x overhead, and this goes in object storage, never in a database."

### Q2: "How many servers do you need for 1 million QPS?"
**Hint:** Ask what the servers actually *do* first — that's the real question hiding inside this one. If it's a cache read, one Redis node does ~100k QPS → ~10 nodes, ×2 for replicas = 20. If it's an app server doing I/O-bound work, ~1,000 QPS each → 1,000 servers, ×2 for headroom = 2,000. **The answer differs by 100x depending on the work.** Never give a server count without first stating the per-server QPS assumption.

### Q3: "Why do you multiply average QPS by 2 or 3?"
**Hint:** Because traffic isn't flat. Users cluster at predictable hours (commute, evening). If you provision for the average, you're down every single evening at exactly your busiest hour. Use ×2 for enterprise apps, ×3 for consumer apps, and much more for event-driven spikes like a ticket drop.

### Q4: "How much cache memory do you need?"
**Hint:** Start with the 80/20 rule — roughly 20% of the data drives 80% of the reads. Compute the daily read volume in bytes (reads/day × payload size), take 20% of it, and that's your starting RAM figure. Then sanity-check the cost: 2 TB of RAM is ~32 nodes at 64 GB each, and if that's too expensive, say so and cache less with a lower hit rate. Showing the cost sanity-check is the part that scores.

### Q5: "1 billion requests per day — what's the QPS?"
**Hint:** Divide by 100,000 (not 86,400). **10,000 QPS.** Say it instantly, without hesitation. Then add: "and peak is likely 2-3x that, so I'd design for 30,000." That two-second answer signals more comfort with scale than five minutes of long division would.

---

## Practice exercise

### The Estimation Drill: Design-Size Three Systems in 30 Minutes

Do these **by hand, on paper, with no calculator.** You're building a reflex, not being precise. Ten minutes each.

**System A — YouTube.** 2B DAU; each watches 5 videos/day of ~10 min; 1 in 1,000 users uploads 1 video/day; a 10-min video in 5 quality levels totals ~500 MB; average streamed bitrate 5 Mbps.
> Calculate: uploads/day, upload QPS, videos watched/day, storage/day, storage over 5 years, and **peak streaming bandwidth in Gbps**. Then one sentence: what is the dominant cost, and what does it force into the architecture?

**System B — WhatsApp.** 500M DAU; each sends 40 messages/day; message = 200 bytes; **messages are deleted from the server once delivered**; media is 5% of messages at ~500 KB, retained 30 days.
> Calculate: messages/day, write QPS, peak write QPS, and **storage**. Then answer: why is text storage almost nothing here — and what does that teach you about a *requirements* decision being worth more than any optimisation?

**System C — Uber.** 5M active drivers; each app sends a GPS ping every 4 seconds; ping = 100 bytes; 1M rides/day.
> Calculate: location-update QPS (careful — this one is **not** derived from DAU), ping-stream bandwidth, and the RAM to hold every driver's current location (~1 KB per driver). Then answer: should these pings go to a disk database? Justify with the QPS number.

**What to produce:** for each system, a summary box in the format of Worked Example 1 (QPS, storage, bandwidth, cache, servers), plus **one sentence naming the dominant bottleneck** and the component it forces into your design. That sentence is the entire point — the numbers exist only to produce it.

---

## Quick reference cheat sheet

- **1 day ≈ 100,000 seconds.** Divide requests/day by 100,000 to get QPS. Never long-divide by 86,400.
- **The QPS ladder:** 1M/day = 10 QPS · 100M/day = 1,000 QPS · 1B/day = 10,000 QPS · 10B/day = 100,000 QPS.
- **Peak QPS = average × 3** for consumer apps (×2 enterprise, ×10+ for event spikes). Always design for the peak.
- **Powers of two:** KB → MB → GB → TB → PB, each step ×1,000. 2^30 = 1 GB, 2^40 = 1 TB, 2^50 = 1 PB.
- **5-year storage shortcut:** daily storage **× 2,000** (that's 365 × 5, rounded up). Then × replication factor.
- **Bandwidth = QPS × payload size.** Use *peak* QPS — the network must carry the worst minute.
- **Cache memory = 20% of daily read volume** (the 80/20 rule). Sanity-check the cost afterwards.
- **Servers = peak QPS ÷ per-server QPS, × 2** for headroom and failure tolerance.
- **Per-server rules of thumb:** Node app ~1,000 QPS · Postgres ~5,000 reads/s · Redis ~100,000 QPS.
- **Sizes to memorise:** tweet ≈ 500 B · URL ≈ 100 B · thumbnail ≈ 20 KB · web photo ≈ 200 KB · phone photo ≈ 3 MB.
- **Round the scary direction.** Over-estimate storage; under-estimating causes outages. One significant figure is enough.
- **The read:write ratio is the most design-defining number you will compute.** >10:1 means "cache, and probably precompute."
- **Compare your numbers to each other.** Blobs 410 PB vs metadata 91 TB → they belong in different stores. That comparison *is* the design.
- **State every assumption out loud.** A silent guess is the one that kills you in production.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [04 — Functional vs Non-Functional Requirements](./04-functional-vs-non-functional-requirements.md) — the NFRs that these numbers give teeth to |
| **Next** | [06 — Latency vs Throughput](./06-latency-and-throughput.md) — what QPS and response time actually mean, in depth |
| **Related** | [12 — Numbers Every Engineer Should Know](./12-numbers-every-engineer-should-know.md) — the full latency and size cheat sheet |
| **Related** | [94 — Design a URL Shortener](./94-hld-url-shortener.md) — the full design that Worked Example 2 sizes |
