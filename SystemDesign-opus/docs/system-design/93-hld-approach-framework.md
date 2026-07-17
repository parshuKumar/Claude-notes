# 93 — How to Approach Any HLD Problem — The 5-Step Framework
## Category: HLD Case Study

---

## What is this?

This is the **recipe**. Every one of the 17 case studies that follow (94 through 110 — URL shortener, Twitter, WhatsApp, Uber, Netflix, and the rest) is the same five steps executed with different ingredients. Learn the steps once, and a "design X" question stops being a scary blank page and becomes a form you fill in.

Think of it like a **pilot's pre-flight checklist**. A pilot with 10,000 hours still reads the checklist out loud before every takeoff — not because they forgot how to fly, but because under pressure, humans skip steps. In a 45-minute interview with a stranger watching you, you *will* skip steps unless you have a checklist burned into your muscle memory.

Topic 03 gave you the general philosophy of approaching design. This doc is the **drilled, timed, interview-grade procedure** — with a clock, with literal scripts of what to say, and with a menu of what to do when the interviewer pushes.

---

## Why does it matter?

**Without the framework**, here's how the interview goes: the interviewer says "design Twitter," you draw a box labeled "API Server," then a box labeled "Database," then you freeze. You have 40 minutes left, no idea what to say, and you start nervously adding Kafka and Redis to boxes that don't need them. The interviewer writes "unstructured, no requirements gathering" in their notes. This is the single most common way people fail.

**With the framework**, the same 45 minutes has a shape. You know that minutes 0-8 are questions, 8-13 is math, 13-28 is the diagram, 28-40 is a deep dive, 40-45 is failure modes. You always know what you're supposed to be doing right now. Silence stops being panic and starts being "let me do this multiplication."

**Interview angle:** The framework *is* the grading rubric. Interviewers at Google, Meta, and Amazon score you on roughly these buckets: requirements gathering, estimation, high-level architecture, depth in a sub-problem, and awareness of failure/scale. Follow the framework and you're literally walking down their score sheet.

**Real-work angle:** This is also how a real design doc gets written. Requirements → scale assumptions → API + data model → architecture → the risky bits → operational concerns. The interview format is a compressed, spoken version of a document your team would review for a week.

---

## The core idea — explained simply

### The Restaurant Menu Analogy

Imagine you're a chef and someone walks in and says: **"Cook me dinner."**

A bad chef immediately fires up the grill and starts searing a steak. Then the customer says "I'm vegetarian" and everything is ruined. Twenty minutes wasted.

A good chef asks first:
- "How many people are eating?" *(scale)*
- "Any allergies or dietary restrictions?" *(constraints)*
- "Do you want this fast, or do you want it perfect?" *(latency vs quality trade-off)*
- "Three courses or one?" *(scope)*

Only after that does the chef plan the menu, check the pantry (does the math on quantities), fire up the right stations, and cook.

**System design is exactly this.** "Design Twitter" is "cook me dinner." It's deliberately vague. The vagueness is the test.

| Restaurant | System Design Interview |
|---|---|
| "Cook me dinner" | "Design Twitter" |
| Asking how many people | Asking for DAU / QPS |
| Asking about allergies | Asking about consistency, latency, region constraints |
| "Three courses or one?" | "Core feed + tweet only, or also DMs, search, ads?" |
| Checking the pantry quantities | Capacity estimation |
| The plated dish | The architecture diagram |
| The tricky sauce that could split | The deep dive |
| "What if the oven dies mid-service?" | Failure modes and bottlenecks |
| The 90-minute service window | The 45-minute clock |

The chef who asks four questions and then cooks confidently beats the chef who starts searing immediately. **Every time.**

---

## Key concepts inside this topic

Here is the entire framework, with a hard time budget for a 45-minute interview. Memorize the minute markers. Say them to yourself during the interview.

```
┌────────────────────────────────────────────────────────────────────┐
│                    THE 45-MINUTE TIME BUDGET                       │
├──────────┬─────────────────────────────────────────────────────────┤
│ 0 ── 8   │ STEP 1  Requirements & Scope      (5-8 min)  TALK       │
│ 8 ── 13  │ STEP 2  Capacity Estimation       (5 min)    MATH       │
│ 13 ─ 28  │ STEP 3  High-Level Design         (10-15 min) DRAW      │
│ 28 ─ 40  │ STEP 4  Deep Dive                 (10-15 min) DEFEND    │
│ 40 ─ 45  │ STEP 5  Bottlenecks & Scale       (5 min)    STRESS     │
└──────────┴─────────────────────────────────────────────────────────┘
     ▲                                                          ▲
     │                                                          │
  Do NOT                                                   Leave time
  draw yet                                                 for this —
                                                           it's free points
```

---

### 1. Step 1 — Requirements & Scope (5-8 minutes). NEVER start drawing.

The interviewer gives you two sentences on purpose. Those two sentences are missing 90% of what you need. Your job in the first eight minutes is to **turn a vague prompt into a bounded problem** — and to do it out loud, so they can watch you do it.

**Do not touch the whiteboard. Do not say "so I'll have a load balancer and..." Ask questions.**

#### The literal question script

Run these in order. It takes about two minutes to ask them and three to write the answers down.

> **1. Users.** "Who uses this and how many? Are we talking about a startup with 10,000 users, or Twitter-scale with 300 million?"
>
> **2. Core features.** "Let me name the 3-4 features I think are core, and you tell me if I'm missing something. For a URL shortener: create a short link, redirect to the original, custom aliases, and expiry. Is that the right core set?"
>
> **3. Explicit exclusions.** "I'm going to explicitly leave OUT analytics dashboards, user accounts, and rate-limit tiers so I can focus on the core read/write path. Sound good?"
>
> **4. Read-heavy or write-heavy?** "Do we expect far more reads than writes? For a URL shortener I'd guess 100:1 reads to writes. Does that match what you have in mind?"
>
> **5. Scale.** "What's the expected scale — daily active users, or requests per second, if you have a number in mind?"
>
> **6. Freshness / consistency.** "How fresh does the data need to be? If a user creates a link, is it OK if it takes a second to be visible everywhere — can we be eventually consistent — or does it need to be readable immediately from any region?"
>
> **7. Latency target.** "What's the latency budget? Redirects feeling instant means p99 under ~100ms. Is that the bar?"
>
> **8. Geography.** "Single region, or global users? That decides whether I need a CDN and multi-region replication."

You will not always get straight answers. When the interviewer says "you tell me," that is **not a dodge — it's the test.** Answer your own question and state it as an assumption: *"OK — I'll assume 100 million DAU and read-heavy at roughly 100:1. I'll design for that."*

#### Then WRITE IT DOWN and get confirmation

Put this in the corner of the board and leave it there for the whole interview. It is your contract.

```
FUNCTIONAL REQUIREMENTS                NON-FUNCTIONAL REQUIREMENTS
1. Create short URL from long URL      • Scale:        100M DAU, ~10K QPS peak
2. Redirect short → long               • Latency:      p99 < 100ms on redirect
3. Optional custom alias               • Availability: 99.9% (redirects must never 500)
4. Links expire after N days           • Consistency:  eventual OK on create
                                       • Durability:   never lose a mapping
OUT OF SCOPE (explicit)
- Analytics / click tracking
- User accounts and auth
- Spam / malware detection
```

Then say the magic sentence:

> **"Before I design anything — does this scoping look right to you? Anything you'd add or cut?"**

#### Why this step decides the interview

They are testing exactly one thing here: **do you scope before you build?** A senior engineer who is handed a vague task and immediately starts coding is a liability. A senior engineer who says "hold on, what are we actually building, and what are we NOT building" is who they want to hire.

**Cut scope aggressively, out loud.** Saying "I'll leave out analytics" is not weakness — it's judgment. You are showing you know that a 45-minute answer can only cover the core. And if the interviewer *wants* analytics, they'll tell you now, when it's cheap, instead of at minute 35 when it's expensive.

*(Recall from [04 — Functional vs Non-Functional Requirements] that functional = what the system does, non-functional = how well it does it. Both go on the board. Non-functional is where the architecture actually comes from.)*

---

### 2. Step 2 — Capacity Estimation (5 minutes). Do the math OUT LOUD.

Now you know roughly how big this is. Turn that into numbers, because **the numbers decide the architecture.**

This is the whole reason estimation exists. It is not a math quiz:

| Scale | What the architecture has to be |
|---|---|
| **100 QPS, 10 GB data** | One app server. One Postgres. Done. Anything more is over-engineering. |
| **10,000 QPS, 1 TB data** | Load balancer, a few app servers, a read replica, Redis cache. |
| **1,000,000 QPS, 500 TB data** | Sharded DB, multi-tier cache, CDN, async workers, multi-region. |

If you skip the math, you have no honest basis for choosing any of those. You'd just be guessing — and the interviewer can tell.

#### The estimation ladder (say each rung out loud)

*(Recall from [05 — Capacity Estimation] the full method. Here's the compressed interview version.)*

**Rung 1 — DAU → QPS.**
```
Rounding trick: 1 day ≈ 86,400 seconds ≈ 100,000 seconds. Use 100K.

100M DAU × 10 actions/user/day = 1B requests/day
1B / 100,000 s                 = 10,000 QPS  (average)
Peak = 2-3× average            = ~25,000 QPS
```

**Rung 2 — Read:write split.**
```
Assume 100:1 read-heavy.
Writes: 10,000 / 101 ≈ 100 writes/sec      → tiny. Any DB handles this.
Reads:  ~9,900 reads/sec                    → this is where the design effort goes.
```
This single line is worth a lot: it tells you the write path can be simple and the read path needs cache + replicas.

**Rung 3 — Storage.**
```
Per record: short_code 7B + long_url 200B + user_id 8B + timestamps 16B
          ≈ 250 bytes → round to 500 bytes with index overhead.

Per day:   100 writes/sec × 100,000 s   = 10M new records/day
           10M × 500 B                  = 5 GB/day
Per year:  5 GB × 365                   ≈ 1.8 TB/year
5 years:   ≈ 9 TB
```
9 TB does not fit comfortably on one box with good performance → **you just justified sharding**, with arithmetic, not vibes.

**Rung 4 — Bandwidth.**
```
Reads: 10,000 req/s × 500 B  = 5 MB/s  → trivial, no CDN needed for the metadata.
(If it were 10,000 req/s × 2 MB images = 20 GB/s → CDN is mandatory.)
```

**Rung 5 — Cache size (the 80/20 rule).**
```
80% of traffic hits 20% of the data.
Daily reads: 9,900/s × 100,000 s ≈ 1B reads/day over ~10M hot objects
Cache the hot 20% of one day's working set:
   10M × 20% × 500 B = 1 GB   → fits in a single Redis node, easily.
```

#### Say why it matters, and know when to skip

Close the section with the punchline, out loud:

> "So: writes are trivial at ~100/s, reads are ~10K/s and that's the real problem, we'll hit ~9 TB in five years so a single DB won't cut it long-term, and the hot working set is ~1 GB so a Redis cache will absorb almost all the read load. That tells me: cache-first read path, sharded storage, and I don't need a CDN for this."

**When to skip:** if the interviewer says "don't worry about the numbers, let's get to the design" — **believe them.** They're managing the clock. Say "sure — I'll assume read-heavy at roughly 10K QPS peak and move on," and go. Grinding through math they told you to skip is a clock-management failure, and clock management is being graded.

---

### 3. Step 3 — High-Level Design (10-15 minutes). API → Data model → Boxes → Walk the paths.

Now you draw. But in this order, because each step constrains the next.

#### 3a. Start with the API (2-3 endpoints)

Writing endpoint signatures forces you to be concrete about what the system actually *does*. It takes 90 seconds and it prevents an entire class of hand-waving.

```js
// POST /api/v1/urls  — create a short link
// body: { longUrl: string, customAlias?: string, ttlDays?: number }
// 201 → { shortUrl: "https://sho.rt/aB3xY9z", expiresAt: "2027-01-01T00:00:00Z" }

// GET /:shortCode  — the hot path. 301/302 redirect to the long URL.
// 302 → Location: https://example.com/very/long/path

// DELETE /api/v1/urls/:shortCode  — owner deletes a link
// 204 → no content
```

Two things to say out loud here: **which endpoint is the hot one** ("the GET is 99% of traffic — that's what I'm optimizing"), and **301 vs 302** ("302 temporary, so the browser keeps coming back to us — if I used 301 the browser caches forever and I'd lose the ability to expire or count clicks").

#### 3b. Then the data model (2-4 entities, with a DB choice AND a reason)

```
urls
├── short_code    VARCHAR(7)   PRIMARY KEY   -- the lookup key, always
├── long_url      TEXT         NOT NULL
├── user_id       BIGINT       NULL          -- nullable: anonymous creates allowed
├── created_at    TIMESTAMP
└── expires_at    TIMESTAMP    NULL          -- NULL = never expires

users            (only if auth is in scope — here it isn't, so skip it)
```

Then the sentence that most candidates never say:

> **"I'd use a key-value store here — DynamoDB or Cassandra — because every single read is a point lookup by primary key. There are no joins, no range scans, no transactions across rows. I'm giving up ad-hoc querying, which I don't need. If we'd kept analytics in scope, I'd want something else for that path."**

That is a technology choice **with a justification tied to the access pattern**. An unjustified "I'd use Cassandra" is a red flag. A justified one is a green flag. Same words — the difference is the *because*.

#### 3c. Then draw the boxes — and keep it SIMPLE first

The generic architecture below is where **most HLD answers converge.** Draw this skeleton, then delete the boxes you don't need and add the one or two specific to this problem.

```
                            ┌───────────────┐
                            │    Clients    │  web / mobile
                            └───────┬───────┘
                                    │ HTTPS
                    ┌───────────────▼───────────────┐
                    │        CDN  (static/media)    │   ← only if you serve big blobs
                    └───────────────┬───────────────┘
                                    │
                    ┌───────────────▼───────────────┐
                    │      Load Balancer  (L7)      │   TLS, health checks
                    └───────────────┬───────────────┘
                                    │
              ┌─────────────────────┼─────────────────────┐
              ▼                     ▼                     ▼
        ┌───────────┐         ┌───────────┐         ┌───────────┐
        │ API Server│         │ API Server│         │ API Server│  stateless!
        └─────┬─────┘         └─────┬─────┘         └─────┬─────┘
              │                     │                     │
              └─────────┬───────────┴──────────┬──────────┘
                        │                      │
                 ┌──────▼───────┐       ┌──────▼───────┐
                 │    Cache     │       │    Queue     │
                 │   (Redis)    │       │(Kafka/SQS)   │
                 └──────┬───────┘       └──────┬───────┘
                        │ miss                 │
                        ▼                      ▼
                 ┌──────────────┐       ┌──────────────┐
                 │   Database   │◀──────│   Workers    │  async: emails,
                 │  (sharded)   │       │              │  thumbnails, fanout
                 │  + replicas  │       └──────────────┘
                 └──────────────┘
                        │
                        ▼
                 ┌──────────────┐
                 │ Blob Storage │  S3 — images, video, files
                 └──────────────┘
```

**Resist the urge to sprinkle Kafka everywhere.** If nothing in your requirements is asynchronous, there is no queue. A queue you can't justify is worse than no queue — it invites the question "why is that there?" and you have no answer. Add complexity only when a requirement forces it.

#### 3d. Then WALK the two paths, end to end

This is the part interviewers remember. Trace a single request with your finger.

> **WRITE path:** "A user POSTs a long URL. It hits the LB, lands on any API server — they're stateless so it doesn't matter which. The server generates a unique 7-character code (I'll deep-dive on how in a moment), writes `{code → longUrl}` to the DB, and writes it into the cache too, since a freshly created link is likely to be clicked immediately. Returns 201 with the short URL. Total: one DB write, one cache write."
>
> **READ path:** "A browser hits `GET /aB3xY9z`. LB → API server. Server checks Redis first. **Cache hit — ~1ms — return 302, done. That's 90%+ of traffic and it never touches the database.** On a miss, we read from the DB (~10ms), populate the cache with a TTL, and return the 302. If the code doesn't exist, 404."

**This diagram plus these two walkthroughs is what's actually being graded in Step 3.** If you get pulled off track, this is what you come back to.

---

### 4. Step 4 — Deep Dive (10-15 minutes). Pick the hard part and commit.

Either the interviewer says "let's go deeper on X," or — better — **you offer**: *"The most interesting problem here is generating unique short codes at 100 writes/sec without collisions. Want me to go into that, or is there something else you'd rather see?"* Offering shows you know where the difficulty actually is.

Deep dives are not random. They come from a small menu, and each system archetype has its usual suspects:

| Deep dive | Which systems it shows up in | The core tension |
|---|---|---|
| **Unique ID generation** | URL shortener, Pastebin, tweet IDs, order IDs | Global uniqueness vs coordination cost. Options: DB auto-increment (bottleneck), UUID (128 bits, ugly, not sortable), **Snowflake** (timestamp + machine ID + sequence — sortable and coordination-free), pre-generated key ranges handed out to servers. |
| **The hot key / celebrity problem** | Twitter, Instagram, YouTube, ticketing | One key gets 1M reads/sec and melts one shard. Fixes: replicate the hot key across N cache nodes, add a local in-process cache in front of Redis, key-suffix sharding (`user:123#0..9`). |
| **Fanout-on-write vs fanout-on-read** | Twitter/X feed, Instagram feed, news feed | Write-time work vs read-time work. Push (precompute each follower's feed — fast reads, catastrophic for a 100M-follower celebrity) vs Pull (build the feed on read — cheap writes, slow reads). **Real answer: hybrid.** Push for normal users, pull for celebrities, merge at read time. |
| **Read/write path optimization** | Basically everything | Where do you spend the latency budget? Cache-aside vs write-through, read replicas, denormalize to avoid joins. |
| **Concurrent-update consistency** | Ticketmaster, inventory, bank balances, collaborative editing | Two people buy the last seat. Fixes: DB transactions + row locks, optimistic concurrency (version numbers + compare-and-swap), distributed locks (Redlock), or make the operation idempotent. |
| **Geospatial indexing** | Uber, Yelp, DoorDash, "find nearby X" | "Find drivers within 2km" — you cannot do that with a B-tree on (lat, lng). Fixes: **geohash** (encode 2D → 1D prefix-searchable string), quadtree, or Uber's H3 hexagonal grid. |
| **Thundering herd / cache stampede** | Any cache-heavy read system | A hot key expires; 10,000 requests miss simultaneously and all slam the DB. Fixes: request coalescing (one request fetches, the rest wait), probabilistic early expiry, stale-while-revalidate, a lock on the recompute. |

#### How to actually perform a deep dive

1. **State the problem crisply.** "At 100 writes/sec, two servers could generate the same 7-char code. What do I do?"
2. **Name 2-3 options.** Don't just present the winner — show the search space.
3. **Name the trade-off explicitly.** This is the highest-value sentence in the entire interview. *"UUIDs are trivially collision-free but they're 36 characters — the whole point is a SHORT url. Auto-increment IDs are compact and sortable but the counter is a single point of contention."*
4. **COMMIT.** *"I'll use a counter-based approach: a Zookeeper (or Redis) service hands each API server a block of 10,000 IDs at a time. Servers burn through their block locally with zero coordination, and base62-encode the integer into a 7-char code. That gives me 62^7 ≈ 3.5 trillion codes, no collisions by construction, and one coordination call per 10,000 writes."*
5. **Say what it costs.** *"The cost is that if a server dies mid-block, we waste up to 10,000 IDs. At 3.5 trillion available, I don't care."*

Steps 3, 4, and 5 are where the points are. **Thinking aloud is not optional** — silence reads as "I don't know," even if you're thinking brilliantly. Narrate.

---

### 5. Step 5 — Bottlenecks, Failure Modes & Scale (5 minutes). Free points. Don't run out of clock.

Most candidates never get here because they blew 20 minutes on requirements or got lost in a deep dive. If you land this section, you look markedly more senior than someone who didn't.

#### Ask these out loud, and answer them

> **"What's the single point of failure?"**
> Go box by box on your own diagram and kill each one:
> - *LB dies* → run it in an active-passive pair with a floating IP; DNS failover as backstop.
> - *One API server dies* → they're stateless, the LB health check ejects it. Zero impact.
> - *Cache dies* → **this is the dangerous one.** All 10K QPS falls through to a DB sized for 100 QPS. It dies too. Mitigate: cluster the cache, replicate hot keys, and put a circuit breaker + rate limit in front of the DB so it degrades instead of collapsing.
> - *DB primary dies* → promote a replica. Accept a few seconds of write unavailability. Reads keep serving from replicas.
> - *A whole region dies* → multi-region with async replication, accepting a small RPO (potential data loss window).

> **"Where's the bottleneck at 10× traffic?"** Trace it: 250K QPS → the cache tier saturates first, then DB connections. Add cache nodes, add read replicas.

> **"Where does it break at 100×?"** 2.5M QPS → now you need CDN edge caching of the redirect itself, geo-sharded storage, and you're rethinking the data layout, not just adding boxes.

#### Then apply the scaling toolkit (see the checklist below), and close with ops

- **Monitoring:** "I'd alert on p99 latency > 200ms, cache hit rate dropping below 90% (that's the leading indicator of a DB meltdown), 5xx rate > 0.1%, and DB replication lag > 5s."
- **Security:** "Rate-limit creates per IP to stop abuse. Validate and scan long URLs against a malware blocklist — a URL shortener is a phishing tool if you don't. TLS everywhere. Don't make short codes sequential/guessable if the content is private."

---

### 6. The Scaling Toolkit — your checklist for "this is too slow / too big"

When any part of your design is under pressure, run down this menu out loud. One of these seven is almost always the answer.

| Move | Use it when | What it costs you |
|---|---|---|
| **Cache it** (Redis, CDN, in-process) | Reads dominate; the same data is read repeatedly | Staleness. Invalidation is genuinely hard. |
| **Shard it** (partition by key) | One DB can't hold the data or take the writes | Cross-shard queries get painful; resharding is a project. |
| **Replicate it** (read replicas, followers) | Read load is the problem, not data size | Replication lag → stale reads. |
| **Queue it** (Kafka, SQS + workers) | The work doesn't have to finish before you respond | Eventual consistency; more moving parts to operate. |
| **CDN it** (edge caching) | You serve large static blobs, or users are global | Cost; cache invalidation at the edge is slow. |
| **Denormalize it** (precompute, duplicate data) | Joins are killing your read latency | Write amplification; data can drift out of sync. |
| **Batch / async it** | Many small writes are hammering the DB | Latency on the individual item; complexity. |

Add two structural moves you should also name: **stateless app servers** (so you can scale horizontally at all — never keep session state in the app), and **rate limiting / backpressure** (so overload degrades gracefully instead of cascading).

---

### 7. The top mistakes that FAIL candidates

Read this list before every mock interview. Each one is common, each one is fatal, each one is avoidable.

| Mistake | What it looks like | The fix |
|---|---|---|
| **Drawing before asking** | Marker touches the board in minute 1 | Hands off the marker for 8 minutes. Ask the script. |
| **Over-engineering from minute one** | Microservices, Kafka, Kubernetes, and a service mesh — for 100 users | Start simple. Add complexity only when a *number* forces it. Complexity you can't justify is complexity you'll be interrogated on. |
| **Going silent** | 90 seconds of quiet staring while you "think" | Narrate. "I'm weighing SQL vs NoSQL here — let me think about the access pattern out loud." |
| **Never naming a trade-off** | Every choice is presented as obviously correct | Every single decision: "I chose X over Y, and the cost is Z." If a choice has no cost, it wasn't a choice. |
| **Losing track of the clock** | Minute 35, still discussing requirements | Watch the minute markers. If you're behind, say so and move: "I want to leave time for scale — let me lock the design and move on." |
| **Can't justify a technology** | "I'd use Cassandra." → "Why?" → "...it's scalable?" | Tie every tech choice to an access pattern or a requirement. If you can't finish the sentence "...*because* our access pattern is ___", pick something you can. |
| **Designing for a scale you never computed** | Sharding a 5 GB database | Let Step 2's numbers drive Step 3. That's what they're for. |
| **Ignoring the interviewer's hints** | They say "hmm, what about the cache dying?" and you keep going | A hint is a gift. Stop and take it immediately. |

---

### 8. A worked mini-example — Pastebin in 5 minutes

Here is the whole framework, executed end to end in miniature, on a deliberately simple system. This is what a compressed answer sounds like.

**Step 1 — Requirements (script, abbreviated).**
> "Users paste text, get a link, others read it. Core features: create a paste, read a paste, optional expiry. I'll exclude editing, comments, syntax highlighting, and user accounts. Read-heavy — I'd guess 10:1. Assume 1M DAU. Pastes are immutable once created, which is a huge simplification. Eventual consistency is fine. Sound right?"

```
FUNCTIONAL                       NON-FUNCTIONAL
1. POST a text blob → get URL    Scale:        1M DAU
2. GET the blob by URL           Latency:      p99 < 200ms read
3. Optional TTL / expiry         Availability: 99.9%
                                 Consistency:  eventual
OUT: editing, accounts, comments, highlighting
```

**Step 2 — Capacity (out loud).**
```
1M DAU × 1 write/day        = 1M writes/day  → 1M / 100,000s = 10 writes/sec   (tiny)
10:1 read ratio             = 100 reads/sec                                     (also tiny)
Avg paste = 10 KB
Storage/day  = 1M × 10 KB   = 10 GB/day
Storage/5yr  = 10 GB × 365 × 5 ≈ 18 TB       → too big for one box's disk comfortably
Bandwidth    = 100/s × 10 KB = 1 MB/s        → trivial
Cache (20% of a day's hot set) = 2 GB        → one Redis node
```
> "Punchline: QPS is small — a single app server could do this. **The problem is not traffic, it's storage: 18 TB of blobs.** That tells me: put the text in object storage (S3), not in the database. The DB only holds metadata."

That is capacity estimation *deciding the architecture*. That one insight is the whole design.

**Step 3 — HLD.**

```js
// POST /api/v1/pastes   { content: string, ttlDays?: number }
//   → 201 { id: "x7Kp2m", url: "https://pb.io/x7Kp2m", expiresAt }
// GET  /:id             → 200 { content, createdAt, expiresAt }  (or 404)
```

```
pastes  (metadata only — the blob lives in S3)
├── id          VARCHAR(8)  PK        -- base62, from a counter
├── s3_key      VARCHAR(64)           -- pointer to the actual text
├── size_bytes  INT
├── created_at  TIMESTAMP
└── expires_at  TIMESTAMP  (indexed — a cleanup job scans this)
```
> "DB: any KV store or even Postgres — metadata is tiny and every read is a point lookup by id. **The blob goes in S3 because 18 TB in a relational DB is expensive and pointless; S3 is cheap, durable, and infinitely scalable, and I'm never querying inside the text.**"

```
Client ──▶ LB ──▶ API Server ──▶ Redis (metadata + hot blobs)
                       │              │ miss
                       │              ▼
                       │          Postgres (metadata)
                       │              │
                       └──────────────┴──▶ S3 (paste bodies)

                   Cleanup Worker (cron) ──▶ scans expires_at ──▶ deletes from S3 + DB
```
> **Write path:** generate id → PUT body to S3 → INSERT metadata row → return URL.
> **Read path:** GET /:id → Redis hit (~1ms) return; on miss → read metadata from DB → fetch body from S3 → populate cache with TTL → return.

**Step 4 — Deep dive: unique ID generation.**
> "Options: UUID (too long — a Pastebin URL should be short), DB auto-increment (single point of contention, and it leaks how many pastes exist), or a counter service handing out blocks. **I'll commit to the block approach:** Redis `INCRBY key 1000` gives each server a range of 1,000 ids; the server base62-encodes them locally into 6-char codes. 62^6 ≈ 56 billion — plenty. Cost: wasted ids on server crash, which is irrelevant at that supply. Bonus: it's not sequential-looking from the outside, since servers interleave their blocks."

**Step 5 — Bottlenecks.**
> "SPOF is the Redis counter — I'd run it replicated, and on total failure fall back to UUIDs temporarily (ugly URLs beat a broken service). Cache death sends 100 QPS to the DB, which it can absolutely handle — so no cascade risk here, unlike the URL shortener. At 100×, the bottleneck becomes S3 GET latency and egress cost → put a CDN in front of public pastes. Expiry cleanup at scale shouldn't be a full table scan — I'd use S3 lifecycle policies for the blobs and a partitioned index on `expires_at`. Alerts: p99 read latency, cache hit rate, S3 error rate, cleanup-job lag."

**That's the whole framework in five minutes.** Every case study from 94 onward is this, with more depth in each box.

---

## Visual / Diagram description

### Diagram 1 — The one-page cheat-sheet flow (memorize this; reproduce it on the whiteboard)

This is the flow you should be able to draw from memory in 30 seconds at the start of any interview. Some candidates literally sketch the five boxes in the corner of the board as a visible agenda — interviewers love it, because it tells them you have a plan.

```
┌──────────────────────────────────────────────────────────────────────┐
│  STEP 1  ── REQUIREMENTS & SCOPE ────────────────────── 5-8 min      │
│  Ask: users? features? EXCLUSIONS? read/write? scale?                │
│       consistency? latency? global?                                  │
│  Write: [numbered FRs] + [NFRs] + [OUT OF SCOPE]  →  CONFIRM         │
└────────────────────────────────┬─────────────────────────────────────┘
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│  STEP 2  ── CAPACITY ESTIMATION ──────────────────────── 5 min       │
│  DAU → req/day → avg QPS → peak QPS (×2-3)                           │
│  read:write │ storage/day → /5yr │ bandwidth │ cache (80/20)         │
│  ►► THE NUMBERS DECIDE THE ARCHITECTURE ◄◄                           │
└────────────────────────────────┬─────────────────────────────────────┘
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│  STEP 3  ── HIGH-LEVEL DESIGN ───────────────────────── 10-15 min    │
│  (a) API      2-3 endpoint signatures                                │
│  (b) Data     2-4 entities + DB choice + WHY                         │
│  (c) Boxes    Client→CDN→LB→API→Cache→DB (+queue/workers if needed)  │
│  (d) WALK the WRITE path, then WALK the READ path                    │
└────────────────────────────────┬─────────────────────────────────────┘
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│  STEP 4  ── DEEP DIVE ───────────────────────────────── 10-15 min    │
│  Menu: unique IDs │ hot key │ fanout push-vs-pull │ read/write path  │
│        concurrency │ geospatial │ thundering herd                    │
│  Pattern: state problem → 2-3 options → NAME TRADE-OFF → COMMIT      │
└────────────────────────────────┬─────────────────────────────────────┘
                                 ▼
┌──────────────────────────────────────────────────────────────────────┐
│  STEP 5  ── BOTTLENECKS & SCALE ──────────────────────── 5 min       │
│  SPOF? What if THIS box dies? Bottleneck at 10×? Break at 100×?      │
│  Toolkit: cache │ shard │ replicate │ queue │ CDN │ denormalize      │
│  Close with: monitoring + alerts + security                          │
└──────────────────────────────────────────────────────────────────────┘
```

**What it shows:** the interview as a pipeline. Each stage produces an artifact the next stage consumes. Requirements produce numbers to estimate; estimates produce the scale that constrains the architecture; the architecture exposes the hard sub-problem to deep-dive; the deep dive plus the architecture is what you stress-test at the end. You cannot reorder these — that's why "drawing first" fails so badly. It's designing without inputs.

### Diagram 2 — The generic reference architecture

Redraw this from memory. Then, for each specific problem, **delete what you don't need** and add the one or two boxes that make this system special (a geospatial index for Uber, a fanout worker for Twitter, a WebSocket gateway for chat).

```
   ┌────────┐   ┌────────┐
   │ Mobile │   │  Web   │
   └────┬───┘   └───┬────┘
        └─────┬─────┘
              │  HTTPS
        ┌─────▼──────┐
        │    CDN     │  static assets, images, video segments
        └─────┬──────┘  (skip if you only serve small JSON)
              │
        ┌─────▼──────┐
        │  API GW /  │  TLS termination, auth, RATE LIMITING
        │    LB      │  health checks, round-robin
        └─────┬──────┘
              │
  ┌───────────┼───────────┐
  ▼           ▼           ▼
┌────┐     ┌────┐      ┌────┐
│API │     │API │      │API │   ◀── STATELESS. Scale by adding boxes.
└─┬──┘     └─┬──┘      └─┬──┘
  └──────┬───┴─────┬─────┘
         │         │
   ┌─────▼───┐  ┌──▼──────────┐
   │  CACHE  │  │  MSG QUEUE  │  ◀── only if there is async work
   │ (Redis) │  │(Kafka/SQS)  │
   └────┬────┘  └──┬──────────┘
        │ miss     │
        │      ┌───▼────────┐
        │      │  WORKERS   │  fanout, email, transcode, indexing
        │      └───┬────────┘
        ▼          │
   ┌─────────────┐ │      ┌──────────────┐
   │  PRIMARY DB │◀┘      │ BLOB STORAGE │
   │  (sharded)  │        │     (S3)     │
   └──────┬──────┘        └──────────────┘
          │ replication
   ┌──────▼──────┐
   │  REPLICAS   │  read scaling
   └─────────────┘
```

**What it shows:** the ~85% of any HLD answer that is the same every time. Traffic enters through a CDN and a load balancer, hits a fleet of stateless application servers, which read through a cache to a sharded database, push slow work onto a queue for background workers, and store large binary objects separately in blob storage. **The interview is about the 15% that's different** — but you have to be able to draw the 85% fluently and fast, or you'll spend your whole budget on plumbing.

---

## Real world examples

### Meta / Facebook — the same skeleton, at absurd scale

Meta's public engineering writing describes a stack that maps almost box-for-box onto the reference architecture: edge PoPs and CDN in front, load balancers, a huge fleet of stateless web servers, **TAO** (their distributed, cache-first graph data store) sitting between the web tier and sharded MySQL, and a message-queue-plus-workers path for async work like notifications. The interesting part is that TAO exists specifically because the read:write ratio for the social graph is overwhelmingly read-heavy — a number, driving an architecture. That is Step 2 → Step 3 in the real world.

### Amazon — "working backwards" is Step 1, institutionalized

Amazon's famous "working backwards" process requires you to write the **press release and the FAQ before you write any code** — literally forcing requirements and scope to precede design. The internal design doc that follows is a longer-form version of exactly this framework: what are we building, for whom, at what scale, what's explicitly out of scope, what's the architecture, what are the risks. If you've ever wondered why interviewers punish "drawing first" so harshly, it's because their entire engineering culture punishes it too.

### Netflix — Step 5 as a company-wide practice

Netflix built **Chaos Monkey** (and the broader Simian Army) to randomly kill production instances during business hours. The entire point is to force engineers to answer Step 5's question — *"what happens when THIS component dies?"* — continuously, in production, rather than hopefully in a design review. Their architecture is the reference diagram with Open Connect (their own CDN, embedded inside ISPs) doing the heavy lifting for video bytes, and stateless microservices with aggressive caching handling the metadata and playback-control traffic. Video is 100% CDN because the bandwidth math demands it; the control plane is not, because it doesn't.

---

## Trade-offs

**Following the framework rigidly vs adapting it:**

| Approach | Pros | Cons |
|---|---|---|
| **Rigid — always all 5 steps, in order** | You never freeze. You always cover the rubric. Great under stress and for your first ~20 practice problems. | Can feel robotic. Wastes time if the interviewer explicitly wants to skip a step. |
| **Adaptive — follow the interviewer's lead** | Matches what they actually want to grade. Feels like a conversation between peers. | Requires experience to know what to cut. Easy to get lost and blow the clock. |

**The sweet spot:** **Keep the skeleton rigid, keep the timing adaptive.** Always do all five steps — but let the interviewer expand or compress any one of them. If they say "skip the math," compress Step 2 to one sentence of assumptions and bank the time in Step 4. If they spend 20 minutes drilling one deep dive, that's *their* choice and it's fine — but you still spend the last 3 minutes on failure modes, because that's free points nobody else collects.

**Rule of thumb:** *The framework is a floor, not a ceiling.* It guarantees you never bomb. Getting a strong-hire signal comes from the quality of your trade-off reasoning inside Step 4 — but you never get to Step 4 in good shape unless Steps 1-3 were disciplined.

---

## Common interview questions on this topic

### Q1: "Design Twitter." (You have 45 minutes. Go.)

**Hint:** Do not draw. Ask: how many DAU (assume 300M, 150M active)? Which features — post a tweet, view a home timeline, follow a user? (Explicitly cut DMs, search, ads, trends, notifications.) Read-heavy? (Yes — massively, maybe 1000:1.) Then the math: 150M × ~2 reads/day ≈ 300M timeline reads/day ≈ 3K QPS avg, ~10K peak; writes maybe 5K/sec. Then API (`POST /tweets`, `GET /feed`), data model (tweets, follows), then the diagram — and then you *know* the deep dive is fanout-on-write vs fanout-on-read, because the numbers made it obvious. The framework navigated you there.

### Q2: "You have 5 minutes left and haven't finished the design. What do you do?"

**Hint:** Say it out loud: *"I have about 5 minutes — I want to spend it on failure modes rather than polishing the diagram, since I think that's higher value. Is that OK with you?"* Then hit SPOFs, the 10× bottleneck, and monitoring. Explicit clock management reads as *senior*. Silently running out of time reads as *disorganized*.

### Q3: "Why did you choose that database?"

**Hint:** The only acceptable answer shape is *"because our access pattern is ___."* E.g. "Every read is a point lookup by primary key, there are no joins and no multi-row transactions, and we need to scale writes horizontally — so a wide-column store like Cassandra fits. If we needed ad-hoc analytical queries or strong multi-row transactions, I'd pick Postgres instead and shard it later." Naming what you'd choose *in the opposite case* proves you understand the trade-off rather than reciting a favorite.

### Q4: "The interviewer says 'don't worry about capacity estimation.' Do you skip it?"

**Hint:** Yes — believe them. But do not skip the *conclusions* estimation would have given you. State them as assumptions in one breath: *"I'll assume read-heavy, roughly 10K peak QPS, and single-digit TB of storage — so: cache-first read path, and I'll plan for sharding but not build it on day one."* You get the architectural payoff without spending the clock.

### Q5: "How do you know when your design is 'done'?"

**Hint:** It never is — you're managing a time budget, not finishing a project. It's "done enough" when: every functional requirement on the board has a path through your diagram, every non-functional requirement has a mechanism (latency → cache; availability → replication; durability → replication + backups), you've named at least one trade-off per major decision, and you've said what breaks first under load. Walk the FR list on the board out loud at the end and tick each one off — it's a strong closing move.

---

## Practice exercise

### The Five-Step Drill (do this cold, on a timer)

Pick **one** system you have NOT studied yet: a Q&A site (Stack Overflow), a food-delivery tracker, or a group expense splitter (Splitwise).

Set a **45-minute timer** and run all five steps on paper or a whiteboard, out loud, alone in a room. Speaking out loud is not optional — the silence-in-the-interview problem only gets fixed by practicing the talking.

Produce, as artifacts:

1. **(8 min)** A written list: 3-4 numbered functional requirements, 5 non-functional requirements (scale, latency, availability, consistency, durability), and an explicit **OUT OF SCOPE** list of at least 3 things you cut. Write down the 8 clarifying questions you would have asked.
2. **(5 min)** The capacity math, showing every step: DAU → requests/day → avg QPS → peak QPS → read:write ratio → storage/day → storage over 5 years → cache size. End with a one-sentence punchline stating **what the numbers force the architecture to be.**
3. **(15 min)** 3 API endpoint signatures, 2-4 entities with fields, a DB choice **with a one-line justification**, and the architecture diagram. Then write out the WRITE path and the READ path as numbered steps.
4. **(12 min)** Pick the hardest sub-problem. Write: the problem, 2-3 options, the explicit trade-off, your committed choice, and what that choice costs you.
5. **(5 min)** For each box in your diagram, one line on what happens when it dies. Then: the bottleneck at 10×, what breaks at 100×, three alert conditions.

**Then grade yourself against the mistakes table in section 7.** Did you touch the marker before minute 8? Did you go silent? Did you name a trade-off — out loud, with the word "trade-off" in it? Did you finish Step 5 before the timer, or run out?

Repeat with a different system until you can hit all five steps within the budget without looking at this page. That usually takes about five reps.

---

## Quick reference cheat sheet

- **Never draw first.** Marker stays down for the first 5-8 minutes. Requirements before architecture, always.
- **The 8 questions:** users/scale, core features, explicit exclusions, read vs write heavy, QPS, consistency needs, latency target, single-region vs global.
- **Write the requirements on the board** as numbered FRs + NFRs + an OUT OF SCOPE list — then ask the interviewer to confirm. That list is your contract for the next 40 minutes.
- **Cut scope out loud.** "I'll leave out analytics and focus on the core read/write path — sound good?" is a senior move, not a retreat.
- **The numbers decide the architecture.** 100 QPS = one box and a Postgres. 1M QPS = shards, caches, and a CDN. Do the math so your design has a reason.
- **Rounding trick:** 1 day ≈ 100,000 seconds. Peak QPS = 2-3× average. Cache the hot 20%.
- **Skip estimation if told to** — but still state the conclusions as assumptions in one sentence.
- **HLD order:** API → data model (+ DB choice + WHY) → boxes → walk the write path → walk the read path.
- **Keep it simple first.** No Kafka unless something is genuinely asynchronous. Unjustified complexity is a liability you'll be cross-examined on.
- **Deep-dive menu:** unique IDs, hot/celebrity key, fanout push vs pull, read/write path optimization, concurrent-update consistency, geospatial indexing, thundering herd.
- **The highest-value sentence in the interview:** "I chose X over Y — the trade-off is Z." Then **commit**.
- **Scaling toolkit:** cache it, shard it, replicate it, queue it, CDN it, denormalize it, batch it. Plus: stateless servers, rate limiting.
- **Always leave 5 minutes for failure modes.** SPOFs, "what if the cache dies," bottleneck at 10×, alerts. Almost nobody does this — it's free differentiation.
- **Fatal mistakes:** drawing first, over-engineering, going silent, never naming a trade-off, losing the clock, and being unable to answer "why that database?"

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [03 — How to Approach System Design](./03-how-to-approach-system-design.md) — the general philosophy; this doc is the drilled, timed, interview-grade version of it |
| **Next** | [94 — HLD: URL Shortener](./94-hld-url-shortener.md) — the first case study, and the cleanest possible execution of these five steps |
| **Related** | [05 — Capacity Estimation](./05-capacity-estimation.md) — the full method behind Step 2; the numbers that decide your architecture |
| **Related** | [04 — Functional vs Non-Functional Requirements](./04-functional-vs-non-functional-requirements.md) — the vocabulary you need to execute Step 1 properly |
| **Related** | [140 — System Design Interview Playbook](./140-system-design-interview-playbook.md) — the meta-level interview strategy: signals, levels, and how you're actually scored |
