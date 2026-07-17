# 103 — Design Twitter / Social Feed
## Category: HLD Case Study

---

## What is this?

Twitter (now X) is a firehose. Hundreds of millions of people post short text messages, follow each other, and expect to open the app and instantly see a fresh, ranked list of everything the people they follow just said. It's a global, real-time, append-only bulletin board where the bulletins are 280 characters and the board is different for every single person looking at it.

Think of it as a **newspaper that is printed fresh for each reader, one second before they pick it up** — your copy contains only the articles written by journalists you personally subscribe to. Now imagine 150 million readers, each demanding their personalised edition, twenty times a day, in under 200 milliseconds.

The hard part is not storing tweets. Text is tiny. The hard part is **delivery**: getting the right 800 tweets in front of the right person fast, when one of those writers has 200 million subscribers.

---

## Why does it matter?

Twitter is the canonical "social feed" interview. Once you can design it, you have designed Instagram's feed, LinkedIn's feed, Facebook's News Feed, Reddit's home page, TikTok's For You page, a Slack channel list, and the notification inbox of almost every product on earth. The shape repeats: **many writers, many more readers, a follow graph in between, and a per-user materialised view that has to be cheap to read.**

It matters in the interview because it forces you to confront the one trade-off that defines large-scale system design: **do you do work at write time or at read time?** Every candidate can say "fanout on write vs fanout on read." Very few can then say *why Twitter is the system where the pure versions of both approaches collapse*, and describe the hybrid that actually ships.

It matters in real work because you will build a feed. You will build a notifications table. And the first time a single account with a million followers does something, your naïve fanout job will fall over and you will learn this lesson at 3am instead of at a whiteboard.

Twitter also drags in two problems Instagram doesn't force you to solve: **trending topics** (a genuine stream-processing problem with an approximation algorithm at its heart) and **real-time search** (an inverted index that must be queryable seconds after a write). Those two deep dives are where strong candidates separate themselves.

---

## The core idea — explained simply

### The Newspaper Delivery Analogy

Two rival newspaper companies serve the same town.

**Company A — "Print it in advance" (fanout on write).** Every time a journalist files a story, an army of clerks immediately copies that story into the personal newspaper of every subscriber who follows that journalist. By 6am, every doorstep has a complete, personalised paper. Readers just pick it up — instant. The clerks did all the work overnight.

**Company B — "Print on demand" (fanout on read).** Nothing is prepared. When you knock on the office door and ask for your paper, a clerk grabs your subscription list, runs to each journalist's filing cabinet, pulls their latest stories, sorts them by time, staples them, and hands you the result. Writers do zero work. Readers wait.

Company A is great — until a journalist with **200 million subscribers** files a story. Now the clerks must make 200 million copies of one article before anybody's paper is complete. The presses jam. Every other journalist's story is stuck behind it.

Company B is great — until a reader who follows **5,000 journalists** shows up. The clerk sprints to 5,000 filing cabinets. That reader waits ten minutes for their paper.

The real answer, the one Twitter actually landed on: **Company A for normal journalists, Company B for the 200-million-subscriber superstars.** Ordinary stories get pre-copied to doorsteps. Superstar stories are *not* copied — instead, when you pick up your paper, a clerk quickly checks the (very short) list of superstars you follow, grabs their latest few stories, and staples them on top. You follow maybe 3 superstars. Stapling 3 stories on is fast. Copying 200 million is not.

| Analogy piece | Technical concept |
|---|---|
| Journalist files a story | User posts a tweet (write) |
| Subscriber list | The **follow graph** (`follows` table) |
| Personal newspaper on your doorstep | The **timeline cache** — a per-user Redis list of tweet IDs |
| Army of clerks copying stories | The **fanout workers** consuming a queue |
| Pre-printed paper | Fanout on write (precompute) |
| Print-on-demand | Fanout on read (compute at query time) |
| Superstar journalist | **Celebrity account** — 1M+ followers |
| Stapling superstar stories on at pickup | **Merge at read time** — the hybrid |
| The article text itself | Stored **once** in the tweet store; only IDs are copied |

That last row is the storage insight most people miss. The clerks don't copy the *text* onto your doorstep. They copy a **slip of paper with the article's ID number**. The text lives once, in the archive. Your paper is a list of ID numbers, and the text is fetched when you actually read it. Copying an 8-byte ID 200 million times is expensive. Copying 300 bytes of text 200 million times is 37× more expensive. Same idea, wildly different bill.

---

## Key concepts inside this topic

We'll run the standard 5-step framework: requirements → estimation → API → data model → architecture, then deep dives, then bottlenecks.

---

### 1. Requirements

**Clarifying questions to ask the interviewer** (ask these out loud — it's half the grade):

- "Are we designing the **home timeline** (tweets from people you follow) or the **user timeline** (one user's own tweets)? These are two completely different problems and I want to be explicit about both."
- "Is the home timeline strictly chronological, or ranked by relevance? Ranking changes the architecture."
- "Text only, or media too? I'll assume text-first and treat media as a blob-store + CDN concern."
- "How stale can a timeline be? Seconds? A minute?"
- "Do we need trending topics and search, or just the feed?"

**Functional requirements:**

| # | Requirement | Note |
|---|---|---|
| F1 | Post a tweet (≤280 chars) | The write path |
| F2 | Follow / unfollow a user | Mutates the graph — asymmetric, unlike Facebook friendship |
| F3 | **Home timeline** — tweets from everyone you follow, newest/ranked first | The hard one |
| F4 | **User timeline** — one user's own tweets, reverse-chronological | The easy one — a single-partition range scan |
| F5 | Retweet, like, reply | Retweet = a tweet pointing at a tweet; reply = a tweet with a parent |
| F6 | Trending topics | Global + per-region top hashtags |
| F7 | Search tweets | Full-text, near-real-time |

**The distinction candidates conflate (F3 vs F4).** Say this explicitly:

- **User timeline** = "give me tweets WHERE user_id = X ORDER BY id DESC LIMIT 50". One user, one shard, one index. It is a *database query*. Cache it and move on.
- **Home timeline** = "give me tweets WHERE user_id IN (the 400 people X follows) ORDER BY id DESC LIMIT 50". That IN-clause is the entire system design. You cannot run this query at 35,000 QPS across a sharded database. Everything in this doc exists to avoid running that query.

**Non-functional requirements:**

| # | Requirement | Consequence |
|---|---|---|
| N1 | Extremely read-heavy (~100:1 reads:writes) | Optimise reads. Do work at write time. Cache aggressively. |
| N2 | Home timeline p99 < 200ms | Rules out fanning out to a DB at read time. Timeline must be precomputed and in memory. |
| N3 | High availability > strong consistency | Feed being 5 seconds stale is fine. Feed being *down* is not. AP over CP. |
| N4 | Eventual consistency acceptable | Your tweet appearing in a follower's feed 3s late is invisible to users. |
| N5 | **Must survive extreme celebrity skew** | One account with 100M–200M followers. This single constraint dictates the whole design. |
| N6 | Predictable, extreme traffic spikes | World Cup goal, election night, an earthquake. 2× normal within seconds. |

N5 is the thesis of this doc. Twitter's follower distribution is not a bell curve; it is a brutal power law. The median account has a handful of followers. The top account has ~200 million. **Any design that treats the top account the same as the median account is broken.**

---

### 2. Capacity estimation

Show the arithmetic step by step. Round aggressively — the interviewer wants the shape, not the decimal.

**Users**
```
Monthly active users (MAU)          = 300,000,000
Daily active users (DAU)            = 150,000,000   (a 50% MAU→DAU ratio, generous but standard)
```

**Write QPS (tweets)**
```
Tweets per day                      = 500,000,000
Seconds per day                     = 86,400  (round to 100,000 for mental math)
Average write QPS = 500M / 86,400  ≈ 5,800  →  call it ~6,000 writes/sec
```

**Peak write QPS.** Social traffic is not smooth. A World Cup goal, a celebrity death, a market crash — everyone tweets in the same 10 seconds.
```
Peak multiplier                     ≈ 2×  (sustained peak; instantaneous bursts are worse)
Peak write QPS                     ≈ 12,000 writes/sec
```
Two things to say out loud here: the spikes are **extreme** (short bursts far above 2× are real) and they are **predictable** — you know the World Cup final's kickoff time. That means you can *pre-scale*: warm up fanout workers and add queue consumers before the match, rather than discovering the load. A system whose spikes are predictable is a system you can capacity-plan for.

**Read QPS (timeline reads)** — the number that actually sizes the fleet:
```
Refreshes per DAU per day           = 20
Timeline reads per day = 150M × 20  = 3,000,000,000  (3 billion)
Average read QPS = 3B / 86,400     ≈ 34,700  →  ~35,000 reads/sec
Peak read QPS (2×)                 ≈ 70,000 reads/sec

Read:write ratio = 35,000 / 6,000  ≈ 6:1 at the request level
```
(The *amplified* ratio is far higher — one tweet write triggers thousands of timeline-cache writes, and one timeline read hydrates 50 tweets. But at the request level, ~6:1, heavily read-skewed.)

**Storage — and the punchline**
```
Bytes per tweet ≈ 280 chars of UTF-8 text + metadata (ids, timestamps, counters)
               ≈ 300 bytes

Per day   = 500,000,000 × 300 B  = 150,000,000,000 B ≈ 150 GB/day
Per year  = 150 GB × 365          ≈ 55 TB/year
Over 5 yr = 55 TB × 5             ≈ 275 TB  (text only, before replication)
With 3× replication              ≈ 825 TB  ≈ under 1 PB
```

**Now the insight.** 275 TB over five years is... nothing. That's a modest rack of disks. A single well-provisioned Cassandra cluster eats that for breakfast. Text is *free*.

> **The expensive things in Twitter are not storage. They are (a) the FANOUT — the write amplification of copying one tweet into millions of timelines — and (b) the READ VOLUME — 35,000 timeline assemblies per second, each needing 50 tweets in under 200ms.**

If you spend your estimation section agonising over disk, you have optimised the cheapest resource in the building. Spend it on fanout and reads. (Media is the exception: add photos/video and storage explodes into petabytes and the answer becomes "blob store + CDN" — but that's topic 100's territory.)

**Bandwidth (sanity check):**
```
Timeline read = 50 tweets × 300 B ≈ 15 KB of JSON
Egress = 35,000 reads/s × 15 KB   ≈ 525 MB/s ≈ 4.2 Gbps  — fine behind a CDN/LB tier.
```

---

### 3. Recap: fanout on write vs fanout on read — and why Twitter is the extreme case

Recall from [100 — Design Instagram](./100-hld-instagram.md) that you have two ways to build a feed, and that the celebrity problem forces a hybrid. Quick refresher, then we go where Instagram didn't:

| | Fanout on **write** (push) | Fanout on **read** (pull) |
|---|---|---|
| When work happens | On post | On timeline load |
| What's stored | A precomputed timeline per user | Nothing extra |
| Read latency | Excellent (one cache lookup) | Poor (N queries + merge) |
| Write cost | O(followers) — explodes for celebrities | O(1) |
| Wasted work | Huge — you fan out to inactive users who never log in | None |

Instagram taught you: **push for normal users, pull for celebrities, merge at read.** Good. Now here's why Twitter is where that idea stops being an optimisation and becomes *survival*.

**Do the arithmetic on one celebrity tweet.**

```
Followers of a top-tier account     ≈ 150,000,000  (call it 100–200M)

Naïve fanout on write:
  Timeline insertions from ONE tweet = 150,000,000

  At an optimistic 1 ms per insertion, single-threaded:
      150,000,000 ms = 150,000 seconds = 41.6 HOURS

  So ONE tweet, done serially, is nearly TWO DAYS of work.
```

Parallelise it. With 1,000 fanout workers each doing 1,000 insertions/sec:
```
  Throughput = 1,000 workers × 1,000 ins/s = 1,000,000 insertions/sec
  Time for one celebrity tweet = 150M / 1M = 150 seconds
```
Two and a half minutes, *if that celebrity tweet monopolises the entire fanout fleet* — while 6,000 other tweets/sec keep arriving and queue behind it. And that account tweets dozens of times a day. And there are hundreds of accounts at that scale. The fanout queue doesn't just get slow; it **never catches up**.

And even if you *could* do it, you *shouldn't*: most of those 150M followers will not open the app in the next hour. You just burned 150 million writes to deliver a tweet to people who aren't looking.

**Therefore the rule:**

> **Never fan out celebrity tweets. Merge them in at read time.**

**The concrete hybrid algorithm.** On a home-timeline read for user U:

1. **Fetch U's precomputed timeline cache** — a Redis sorted set of tweet IDs, populated by fanout from the *normal* accounts U follows. One O(log N + K) call. Fast.
2. **Fetch recent tweets from the celebrities U follows** — look up U's celebrity-follow list (tiny; you follow 400 people but maybe 5 are celebrities), then read each celebrity's *user timeline* from a hot cache. 5 cache reads, all of which are shared across millions of users and therefore always warm.
3. **Merge and sort** the two ID lists by ID descending (Snowflake IDs are time-sortable — see §5, this is why it matters).
4. **Truncate to the page size, hydrate** the tweet objects from the tweet cache/store, and return.

The cost of step 2 is bounded by *how many celebrities you follow*, not by how many followers the celebrity has. That's the whole trick: you moved the cost from the celebrity's huge out-degree to the reader's tiny celebrity in-degree.

**Who counts as a "celebrity"?** This is a **tunable knob**, and saying so is a strong signal. Options:

- **Fixed threshold:** follower_count > 1,000,000 → celebrity. Simple, explainable, works.
- **Cost-based threshold:** fan out if `follower_count < K`, where K is chosen so fanout cost ≈ read-merge cost. Roughly: fanning out costs `followers` writes once; merging costs 1 extra read per follower per timeline load. If a follower loads their timeline 20×/day, pulling is *more* expensive than pushing unless followers are enormous — which pushes K high (hundreds of thousands to millions).
- **Activity-aware:** don't just count followers, count *active* followers. Fanning out to 5M followers of whom 50k are DAU is mostly waste. Twitter-style systems commonly fan out only to recently-active users and rebuild lazily for the rest.
- **Dynamic:** raise the threshold under load. If the fanout queue is backing up, temporarily demote more accounts into the "pull" bucket. This is a load-shedding valve.

There is no magic number. The honest answer is: "I'd start at ~1M followers, measure fanout queue depth and p99 timeline latency, and tune. It's a dial, not a constant."

---

### 4. The timeline cache — the key data structure

This is the heart of the system. Get it right and everything else is plumbing.

**What it is:** for each user, a **Redis sorted set** (`ZSET`) keyed `timeline:{user_id}`, holding the last **~800 tweet IDs**, scored by the tweet ID itself (which is time-sortable). Not the tweets. **Just the IDs.**

```
timeline:12345  →  ZSET {
   1723490001234567  (score = same)
   1723490001234512
   1723489998123001
   ... up to 800 entries, trimmed on every insert
}
```

**Why IDs and not the tweet content?** Suppose you stored full tweet objects in every timeline. A tweet from someone with 100,000 followers would be copied 100,000 times × 300 bytes = **30 MB of duplicated text for one tweet.** Multiply by 500M tweets/day and you have invented a storage catastrophe out of thin air.

```
Fan out full CONTENT:  100,000 followers × 300 bytes = 30 MB per tweet
Fan out just the ID:   100,000 followers ×   8 bytes = 800 KB per tweet
                                                        ────────────────
                                                        ~37× cheaper
```

Store the tweet **once** in the tweet store. Copy only the pointer. Then, at read time, **hydrate**: take the 50 IDs you need and do one batched multi-get against the tweet cache. Because popular tweets are read by everyone, that cache has a phenomenal hit rate. (Recall from [59 — Caching in Depth](./59-caching-in-depth.md) that this is a classic cache-aside pattern: the timeline holds keys, the cache holds values.)

**Memory math — why it's a cluster, not a server:**
```
Active users to keep timelines for   = 150,000,000
Tweet IDs per timeline               = 800
Bytes per ID (64-bit Snowflake)      = 8

Raw payload = 150,000,000 × 800 × 8 B
            = 960,000,000,000 B
            ≈ 960 GB  ≈ ~1 TB

Redis ZSET overhead (skiplist + dict + scores) is real — budget 2–3× raw:
Realistic footprint ≈ 2–3 TB across the cluster.
```

**~1 TB of raw IDs (2–3 TB dressed) does not fit on one machine.** So the timeline cache is a **sharded Redis cluster**: shard by `user_id` (consistent hashing), replicate each shard for availability, and accept that a shard loss means those users' timelines must be rebuilt (which you can do lazily — see the failure-modes section).

**Why 800?** Because nobody scrolls past ~800 tweets in a session. If they do, you fall back to a slower, DB-backed path. This is a deliberate trade: **bound the cache so it stays affordable, and make the rare deep-scroll pay for itself.**

**Don't fan out to everyone.** Of 300M MAU, only 150M are daily-active. Maintaining timelines for the 150M dormant users is pure waste. Real systems mark users inactive after N days, stop fanning out to them, and **rebuild their timeline on demand** the first time they come back (a one-off pull across everyone they follow). That halves your fanout bill and your Redis bill.

---

### 5. Data model, and Snowflake IDs

**DB choice:** the tweet store is a wide, append-only, key-range-scanned dataset with no need for joins or multi-row transactions. That's a **wide-column store** (Cassandra / HBase / DynamoDB) — horizontally scalable, tunable consistency, brilliant at "give me the last N rows for this partition key." The follow graph is a hot, high-fanout adjacency list, also served from a sharded store fronted by a cache. Nothing here wants a single-primary relational DB.

```sql
-- tweets  (wide-column; partitioned by tweet_id, plus a user-partitioned view)
tweets (
  id           BIGINT   PRIMARY KEY,   -- Snowflake ID, time-sortable
  user_id      BIGINT   NOT NULL,
  text         VARCHAR(280),
  created_at   TIMESTAMP,              -- derivable from id, kept for convenience
  reply_to     BIGINT   NULL,          -- parent tweet id, if this is a reply
  retweet_of   BIGINT   NULL           -- original tweet id, if this is a retweet
)

-- user_timelines  (the USER timeline — one user's own tweets)
user_timeline (
  user_id      BIGINT,                 -- partition key
  tweet_id     BIGINT,                 -- clustering key, DESC
  PRIMARY KEY ((user_id), tweet_id)
)   -- "last 50 tweets by user X" = one partition, one range scan. Trivially fast.

-- follows  (THE HOT GRAPH — read on every fanout, written on every follow)
follows (
  follower_id  BIGINT,
  followee_id  BIGINT,
  created_at   TIMESTAMP,
  PRIMARY KEY ((followee_id), follower_id)   -- "who follows X?"  → fanout needs this
)
follows_by_follower (
  follower_id  BIGINT,
  followee_id  BIGINT,
  PRIMARY KEY ((follower_id), followee_id)   -- "who does X follow?" → read-merge needs this
)
-- Note: you store the edge TWICE, once per query direction. That is normal and correct
-- in a wide-column store: you denormalise per access pattern. Writes are cheap, joins aren't.

-- likes
likes (
  tweet_id     BIGINT,
  user_id      BIGINT,
  created_at   TIMESTAMP,
  PRIMARY KEY ((tweet_id), user_id)
)
-- like COUNTS are a separate, approximate, heavily-cached counter — never COUNT(*) a
-- viral tweet's likes table at read time.

-- timeline cache (Redis, not a table)
timeline:{user_id}      → ZSET of ~800 tweet IDs, scored by tweet ID
celebs:{user_id}        → SET of celebrity user_ids this user follows (small)
usertl:{celeb_id}       → LIST of that celebrity's recent tweet IDs (hot, shared)
```

#### Snowflake IDs — and why time-sortability is the point

You need to generate 12,000 unique tweet IDs per second across many machines, with **no coordination** (a central sequence server is a single point of failure and a latency tax). And you want the IDs to be **sortable by time**.

**Why time-sortability matters here — this is the interview answer:**

1. **You can sort a timeline by ID alone.** No timestamp column, no secondary comparison. Merging the fanout list and the celebrity list is a plain numeric merge on 64-bit integers. Redis can even use the ID *as the ZSET score* — the sort is free.
2. **IDs generated on 100 different machines still sort chronologically**, because the high bits are a timestamp. No global clock, no consensus, no coordinator.
3. **Range scans become time scans.** "Tweets since 9am" = "IDs above X." Your primary key *is* your time index.
4. **They're 64-bit**, so 8 bytes — which is exactly what makes the ~1 TB timeline-cache math work. UUIDs are 16 bytes and *random*, which would double the memory AND destroy sortability AND wreck write locality on disk.

**Bit layout (64 bits):**

```
 1 bit    41 bits                            10 bits      12 bits
┌─┬────────────────────────────────────────┬───────────┬────────────┐
│0│         timestamp (ms since epoch)      │ machine   │  sequence  │
│ │                                         │    id     │            │
└─┴────────────────────────────────────────┴───────────┴────────────┘
 ▲          ▲                                    ▲             ▲
 │          │                                    │             │
 sign bit   41 bits of ms  = 2^41 ms             1024 machines 4096 IDs per
 (always 0, ≈ 69 years from a custom epoch       (2^10)        machine per
  so IDs are                                                    millisecond
  positive)                                                     (2^12)

Throughput ceiling = 1024 machines × 4096 IDs × 1000 ms = ~4.2 BILLION IDs/sec.
We need 12,000/sec. Comfortable.
```

```js
// snowflake.js — a 64-bit, time-sortable, coordination-free ID generator.
// We use BigInt because JS Numbers lose precision above 2^53 and these are 63-bit.

const EPOCH = 1288834974657n;      // Twitter's custom epoch (Nov 2010), in ms.
                                   // A custom epoch buys back years of the 41-bit range.

const MACHINE_BITS  = 10n;
const SEQUENCE_BITS = 12n;

const MAX_MACHINE_ID = (1n << MACHINE_BITS) - 1n;    // 1023
const MAX_SEQUENCE   = (1n << SEQUENCE_BITS) - 1n;   // 4095

const MACHINE_SHIFT   = SEQUENCE_BITS;               // 12
const TIMESTAMP_SHIFT = SEQUENCE_BITS + MACHINE_BITS;// 22

export class SnowflakeGenerator {
  constructor(machineId) {
    const id = BigInt(machineId);
    if (id < 0n || id > MAX_MACHINE_ID) {
      throw new RangeError(`machineId must be 0..${MAX_MACHINE_ID}`);
    }
    this.machineId = id;
    this.sequence = 0n;
    this.lastTimestamp = -1n;
  }

  nextId() {
    let now = BigInt(Date.now());

    // Clock went backwards (NTP correction, VM migration). We must NOT emit an ID
    // that could collide with one already issued. Refusing is safer than duplicating.
    if (now < this.lastTimestamp) {
      throw new Error(`Clock moved backwards by ${this.lastTimestamp - now}ms; refusing to generate`);
    }

    if (now === this.lastTimestamp) {
      // Same millisecond: bump the sequence.
      this.sequence = (this.sequence + 1n) & MAX_SEQUENCE;
      if (this.sequence === 0n) {
        // We burned all 4096 slots in this ms. Spin until the clock ticks over.
        now = this._waitNextMillis(this.lastTimestamp);
      }
    } else {
      this.sequence = 0n;
    }

    this.lastTimestamp = now;

    return ((now - EPOCH) << TIMESTAMP_SHIFT)
         | (this.machineId << MACHINE_SHIFT)
         | this.sequence;
  }

  _waitNextMillis(last) {
    let now = BigInt(Date.now());
    while (now <= last) now = BigInt(Date.now());
    return now;
  }

  // The reverse trip: because time is in the high bits, an ID *is* a timestamp.
  // You never need a created_at column to sort a feed.
  static timestampOf(id) {
    return Number((BigInt(id) >> TIMESTAMP_SHIFT) + EPOCH);
  }
}

// --- demo ---
const gen = new SnowflakeGenerator(7);
const a = gen.nextId();
const b = gen.nextId();
console.log(a < b);                                    // true — monotonically increasing
console.log(new Date(SnowflakeGenerator.timestampOf(a)));  // real wall-clock time
```

---

### 6. High-level architecture

```
                                 ┌──────────────┐
                                 │   CLIENTS    │  web / iOS / Android
                                 └──────┬───────┘
                                        │ HTTPS
                                        ▼
                                 ┌──────────────┐
                                 │ LOAD BALANCER│  + CDN for static/media
                                 └──────┬───────┘
                                        ▼
                                 ┌──────────────┐
                                 │  API GATEWAY │  auth, rate limit, routing
                                 └──┬────┬────┬─┘
              ┌────────────────────┘    │    └────────────────────┐
              │                         │                         │
      WRITE PATH                  READ PATH                 SEARCH / TRENDS
              │                         │                         │
              ▼                         ▼                         ▼
   ┌────────────────────┐   ┌────────────────────┐   ┌────────────────────┐
   │   TWEET SERVICE    │   │  TIMELINE SERVICE  │   │  SEARCH SERVICE    │
   │  - validate 280ch  │   │  - read tl cache   │   │  - inverted index  │
   │  - Snowflake ID    │   │  - pull celeb tl   │   │  - query parse     │
   │  - persist tweet   │   │  - MERGE + rank    │   └─────────┬──────────┘
   └─────────┬──────────┘   │  - hydrate         │             │
             │              └────┬──────────┬────┘   ┌─────────▼──────────┐
             │  emits               │          │        │  TRENDS SERVICE    │
             │  TweetCreated        │          │        │  - top-K hashtags  │
             ▼                      │          │        │  - CMS sliding win │
   ┌────────────────────┐           │          │        └─────────┬──────────┘
   │   KAFKA / QUEUE    │───────────┼──────────┼──────────────────┘
   │  topic: tweets     │           │          │        (both consume the same
   └─────────┬──────────┘           │          │         tweet stream)
             │                      │          │
             ▼                      │          │
   ┌────────────────────┐           │          │
   │  FANOUT SERVICE    │           │          │
   │  (worker pool)     │           │          │
   │                    │           │          │
   │  IF celebrity ────────► SKIP FANOUT       │
   │  ELSE:             │           │          │
   │   1. GraphSvc:     │           │          │
   │      who follows?  │           │          │
   │   2. ZADD tweet_id │           │          │
   │      to each       │           │          │
   │      follower's tl │           │          │
   │   3. ZREMRANGEBYRANK trim 800  │          │
   └────┬───────────┬───┘           │          │
        │           │               │          │
        ▼           ▼               ▼          ▼
 ┌────────────┐ ┌──────────────────────┐ ┌──────────────┐
 │ GRAPH SVC  │ │  TIMELINE CACHE      │ │ TWEET CACHE  │
 │ follows    │ │  (Redis cluster)     │ │ (Redis)      │
 │ + cache    │ │  timeline:{uid}      │ │ tweet:{id}   │
 └─────┬──────┘ │   → ZSET of ~800 IDs │ │  → tweet obj │
       │        │  ~2-3 TB, sharded    │ └──────┬───────┘
       │        └──────────────────────┘        │ miss
       ▼                                        ▼
 ┌────────────┐                          ┌──────────────┐   ┌──────────────┐
 │ GRAPH DB / │                          │ TWEET STORE  │   │ SEARCH INDEX │
 │ follows    │                          │ (Cassandra,  │   │ (Elastic /   │
 │ (sharded)  │                          │  sharded by  │   │  Lucene,     │
 └────────────┘                          │  tweet_id)   │   │  sharded)    │
                                         └──────────────┘   └──────────────┘
```

**Reading the diagram.**

*Write path:* client → LB → API → **Tweet Service**, which validates the 280 characters, mints a **Snowflake ID**, writes the tweet once to the **tweet store**, appends the ID to the author's own **user timeline**, and then — crucially — **returns 200 to the client immediately**. It does *not* wait for fanout. It publishes a `TweetCreated` event to **Kafka** and walks away. The user's tweet is durable and visible on their profile; delivery to followers happens asynchronously. This is what buys you a fast write path and lets the fanout queue absorb spikes.

*Fanout path:* **Fanout workers** consume the topic. For each tweet they ask the **Graph Service** "who follows this author?" If the author is a **celebrity**, they *skip fanout entirely*. Otherwise they `ZADD` the tweet ID into each follower's timeline ZSET and trim to 800. Fanout is embarrassingly parallel — it's a pool of stateless workers you scale horizontally.

*Read path:* client → **Timeline Service**, which does the four-step hybrid merge from §3, hydrates tweet bodies from the **tweet cache**, and returns. In the happy path this is *two Redis round-trips and zero database queries*. That is how you hit sub-200ms at 35,000 QPS.

*Side consumers:* **Search** and **Trends** subscribe to the same Kafka tweet stream. This is the crucial architectural move — the write path publishes an event *once*, and any number of downstream systems build their own materialised view of it without the Tweet Service knowing they exist.

**The hybrid fanout, drawn on its own:**

```
                      A TWEET IS POSTED
                             │
                             ▼
                 ┌───────────────────────┐
                 │ followers > threshold?│   (e.g. 1,000,000 — a TUNABLE knob)
                 └────┬─────────────┬────┘
                 NO   │             │  YES  (celebrity)
                      ▼             ▼
        ┌──────────────────┐   ┌────────────────────────────┐
        │  FAN OUT ON WRITE│   │  DO NOTHING. NO FANOUT.    │
        │  push ID to each │   │  Just store the tweet and  │
        │  follower's      │   │  append to the celeb's own │
        │  timeline ZSET   │   │  hot user-timeline cache.  │
        └────────┬─────────┘   └─────────────┬──────────────┘
                 │                           │
                 │      ┌────────────────────┘
                 ▼      ▼
        ══════════════════════════════════════════════
                  TIMELINE READ for user U
        ══════════════════════════════════════════════
                 │
    ┌────────────┴───────────────┐
    ▼                            ▼
┌─────────────────┐   ┌──────────────────────────┐
│ 1. GET U's      │   │ 2. GET U's celeb list    │
│    precomputed  │   │    (tiny — maybe 5 ids)  │
│    timeline     │   │    then read each celeb's│
│    (ZSET, 800)  │   │    recent tweets (HOT    │
│                 │   │    cache, shared by all) │
└────────┬────────┘   └────────────┬─────────────┘
         │                         │
         └──────────┬──────────────┘
                    ▼
         ┌─────────────────────┐
         │  3. MERGE + SORT    │  by Snowflake ID DESC
         │     (IDs sort by    │  — no timestamps needed
         │      time for free) │
         └──────────┬──────────┘
                    ▼
         ┌─────────────────────┐
         │  4. HYDRATE (mget)  │  IDs → tweet objects
         │     + return page   │
         └─────────────────────┘
```

The cost of the celebrity branch is proportional to **how many celebrities YOU follow** (≈5), never to **how many followers THEY have** (≈150,000,000). That inversion is the entire design.

**The timeline read + merge, in code:**

```js
// timeline-service.js
const PAGE_SIZE = 50;
const CELEB_LOOKBACK = 100;   // recent tweets to pull per celebrity before merging

export class TimelineService {
  constructor({ redis, graphService, tweetCache }) {
    this.redis = redis;
    this.graph = graphService;
    this.tweets = tweetCache;
  }

  async getHomeTimeline(userId, { limit = PAGE_SIZE, maxId = null } = {}) {
    // ---- Step 1: the precomputed timeline (fanned out from NORMAL accounts) ----
    // Scored by the tweet id itself, so ZREVRANGE gives newest-first for free.
    const pushedIds = await this.redis.zrevrange(
      `timeline:${userId}`, 0, limit * 2 - 1   // over-fetch: the merge will discard some
    );

    // ---- Step 2: pull from the CELEBRITIES this user follows ----
    // This list is tiny. That is the whole point of the hybrid.
    const celebIds = await this.graph.getFollowedCelebrities(userId);   // ~0-20 ids

    const celebTimelines = await Promise.all(
      celebIds.map(cid =>
        // Shared across every follower of this celebrity → near-100% cache hit rate.
        this.redis.lrange(`usertl:${cid}`, 0, CELEB_LOOKBACK - 1)
      )
    );

    // ---- Step 3: merge + sort ----
    // Snowflake IDs are time-sortable, so a numeric DESC sort IS a chronological sort.
    // No timestamp lookups, no joins, no second comparison key.
    const merged = [...pushedIds, ...celebTimelines.flat()]
      .map(BigInt);

    const deduped = [...new Set(merged)];                       // retweet of the same id, etc.
    const paged = deduped
      .filter(id => (maxId ? id < BigInt(maxId) : true))        // cursor pagination
      .sort((a, b) => (a > b ? -1 : a < b ? 1 : 0))             // DESC
      .slice(0, limit);

    // ---- Step 4: hydrate ----
    // One batched multi-get. Tweet bodies are stored ONCE and shared by every timeline
    // that references them — this is why we fanned out IDs and not content.
    const hydrated = await this.tweets.mget(paged.map(String));

    return {
      tweets: hydrated.filter(Boolean),                          // skip deleted tweets
      nextCursor: paged.length === limit ? String(paged.at(-1)) : null,
    };
  }
}
```

```js
// fanout-worker.js — consumes TweetCreated from Kafka.
const CELEBRITY_THRESHOLD = 1_000_000;   // the tunable knob
const TIMELINE_MAX = 800;
const BATCH = 1000;

export async function handleTweetCreated(evt, { redis, graph }) {
  const { tweetId, authorId } = evt;

  // The author's OWN user timeline is always updated — it's one write, and it's what
  // the read-time celebrity merge (and every profile page) reads from.
  await redis.lpush(`usertl:${authorId}`, tweetId);
  await redis.ltrim(`usertl:${authorId}`, 0, 999);

  const followerCount = await graph.getFollowerCount(authorId);

  // THE decision. A celebrity tweet is never fanned out. 150M inserts would take the
  // fanout fleet minutes and starve every other tweet in the queue.
  if (followerCount >= CELEBRITY_THRESHOLD) return;

  // Normal account: push the ID (8 bytes) — never the content (300 bytes) — to each
  // ACTIVE follower. Dormant users get their timeline rebuilt lazily on next login.
  for await (const followers of graph.streamActiveFollowers(authorId, BATCH)) {
    const pipe = redis.pipeline();
    for (const f of followers) {
      pipe.zadd(`timeline:${f}`, tweetId, tweetId);   // score = id → free chronological sort
      pipe.zremrangebyrank(`timeline:${f}`, 0, -(TIMELINE_MAX + 1));  // keep newest 800
    }
    await pipe.exec();   // batched: ~1000 timelines per round-trip, not 1000 round-trips
  }
}
```

---

### 7. Deep dives

#### 7a. Trending topics — the stream-processing problem

The naïve idea: keep a counter per hashtag, increment on every tweet, `ORDER BY count DESC LIMIT 10`. At 12,000 tweets/second, with millions of distinct hashtags, over a **rolling window**, this fails on three counts: you'd need a write to a shared counter on every tweet (a hot-key nightmare), you'd need to store a count for every hashtag that has *ever* appeared, and expiring old counts out of a sliding window means remembering *when* each increment happened.

**The approach:**

1. **Stream, don't query.** Trends consume the same Kafka tweet stream the fanout workers do. A streaming job (Flink / Kafka Streams / Spark Streaming) extracts hashtags and entities from each tweet.
2. **Sliding window.** Maintain counts over e.g. the last 15 minutes, advancing every minute. Old buckets fall off the back.
3. **Approximate counting — Count-Min Sketch.** You cannot store an exact count for every hashtag. A **Count-Min Sketch (CMS)** is a fixed-size 2D array of counters plus `d` hash functions. To count a hashtag, hash it `d` ways and increment one counter per row. To *query* it, hash it `d` ways and take the **minimum** of those counters. Collisions can only inflate a count, never deflate it — so taking the min gives you a count that is never an underestimate and is usually very close.

```js
// count-min-sketch.js — fixed memory, approximate counts, no per-key storage.
export class CountMinSketch {
  constructor(width = 2 ** 16, depth = 5) {
    this.width = width;
    this.depth = depth;
    // A few KB-to-MB of Int32s, TOTAL — regardless of how many distinct hashtags exist.
    this.table = Array.from({ length: depth }, () => new Int32Array(width));
    this.seeds = Array.from({ length: depth }, (_, i) => i * 0x9e3779b1);
  }

  _hash(key, seed) {
    let h = seed >>> 0;
    for (let i = 0; i < key.length; i++) {
      h ^= key.charCodeAt(i);
      h = Math.imul(h, 0x01000193) >>> 0;   // FNV-style mix
    }
    return h % this.width;
  }

  add(key, count = 1) {
    for (let r = 0; r < this.depth; r++) {
      this.table[r][this._hash(key, this.seeds[r])] += count;
    }
  }

  // Collisions only ever ADD to a bucket, so every row is an over-estimate.
  // The MINIMUM row is therefore the tightest upper bound — and usually exact.
  estimate(key) {
    let min = Infinity;
    for (let r = 0; r < this.depth; r++) {
      min = Math.min(min, this.table[r][this._hash(key, this.seeds[r])]);
    }
    return min;
  }
}
```

Pair the CMS with a small **top-K heap** (a min-heap of the current top 50 candidates): on each hashtag, `add()` to the sketch, `estimate()` it, and if the estimate beats the heap's minimum, insert it. You get top-K in **bounded memory** without ever storing a per-hashtag count.

4. **THE INSIGHT — trending is not volume, it's a RATE OF CHANGE.**

This is the point interviewers are actually fishing for. If you rank by raw count, `#love` and `#happybirthday` trend **forever**, because they are always high-volume. They are not *news*. Trending means "something unusual is happening right now."

So you score each topic by how far its current rate deviates from **its own baseline**:

```js
// A topic trends when it BREAKS ITS OWN PATTERN, not when it's merely popular.
function trendScore(topic, sketch, baselines) {
  const current = sketch.estimate(topic);            // count in the last 15 min

  // Baseline = this topic's typical rate for this time-of-day / day-of-week,
  // maintained as an exponentially-weighted moving average.
  const { mean, stddev } = baselines.get(topic) ?? { mean: 0, stddev: 1 };

  // A z-score: how many standard deviations above its OWN normal is it right now?
  //   #love     : mean 50,000, now 52,000 → z ≈ 0.4  → not trending (this is Tuesday)
  //   #earthquake: mean 5,    now  8,000  → z ≈ huge → TRENDING
  const z = (current - mean) / Math.max(stddev, 1);

  // Guard against noise: a hashtag going from 1 → 30 has a wild z-score but isn't news.
  const MIN_VOLUME = 500;
  if (current < MIN_VOLUME) return 0;

  return z;
}
```

Layer on top: **de-duplication** (spam bots posting the same hashtag 10,000 times — count *distinct users*, not tweets), **geo-partitioning** (trends are computed per region, so a trend in Mumbai isn't diluted by São Paulo), and a **freshness decay** so a topic that peaked two hours ago falls off.

The full pipeline:
```
Kafka tweet stream
      ▼
 extract hashtags/entities   →  spam & bot filter (distinct-user counting)
      ▼
 sliding-window Count-Min Sketch  (bounded memory, approximate counts)
      ▼
 top-K heap  →  score vs. per-topic BASELINE (z-score, not raw volume)
      ▼
 geo-partitioned trend list  →  Redis  →  served in ~1ms
```

#### 7b. Search — a near-real-time inverted index

Recall from [79 — Search Systems](./79-search-systems.md) that full-text search is powered by an **inverted index**: instead of "document → words," you store "word → list of documents containing it." Searching for `earthquake` becomes a hash lookup, not a scan of 500 million rows.

```
INVERTED INDEX
  "earthquake" → [ 1723490001234567, 1723490001199801, 1723489998123001, ... ]
  "tokyo"      → [ 1723490001234567, 1723490000456789, ... ]

Query "earthquake tokyo"  →  INTERSECT the two posting lists  →  {1723490001234567, ...}
                             (and because the doc IDs are Snowflake IDs, the intersection
                              is ALREADY in reverse-chronological order. Free again.)
```

**Keeping it in sync — via the stream.** The Search Service is another consumer of the Kafka `tweets` topic. When a tweet is written, an event is published; an indexer consumes it, tokenises the text, and updates the inverted index. The Tweet Service knows nothing about search. **This decoupling is the point** — you can rebuild the entire index from the log without touching the write path.

**Why real-time search is genuinely hard:** a tweet about an earthquake must be findable within *seconds*, not minutes. But traditional search indexes are built in large, immutable segments — batching writes is what makes them fast to read. So you need:

- **A two-tier index.** A small, in-memory, **real-time index** holding the last few minutes of tweets (mutable, fast to write, slower per-doc to search — but it's tiny). Behind it, the large **archival index** of everything older (immutable segments, heavily optimised). A query hits **both** and merges results.
- **Periodic flush.** Every few minutes, the real-time segment is sealed, optimised, and merged into the archival tier. Classic **LSM-tree** shape: fast mutable memtable in front, immutable sorted segments behind.
- **Sharding by time.** Most searches are for recent tweets. Shard the index by time range so the hot shards (today) are separate from the cold ones (2019), and you can throw hardware at the hot ones.
- **Ranking.** "Top" results aren't just keyword matches — they're weighted by engagement, author reputation, and recency. Which means the index must carry engagement signals that are *updated after* the tweet was indexed.

The eventual-consistency tax: your tweet is durable instantly, in your followers' timelines within seconds, and searchable within a few seconds more. Three different latencies for three different views of the same write. That's normal, and you should say so.

#### 7c. Ranking the timeline — chronological vs ML-ranked

**Chronological** is a sort by Snowflake ID. Free. Predictable. Users understand it. Everything above works perfectly.

**ML-ranked** ("here's what you missed") predicts *engagement probability* per tweet and sorts by that. It measurably increases time-on-app. But look at what it **forces you to change**:

| Concern | Chronological | ML-ranked |
|---|---|---|
| The merge step | A numeric sort on IDs | A scoring pass over every candidate |
| Timeline cache | The final answer | Only a **candidate list** — you must re-rank on read |
| Candidates needed | 50 (exactly what you show) | 500–1,500 (you must score more than you show) |
| Read latency | ~10ms | ~10ms + inference budget. Model must run in **<50ms** or the 200ms SLO dies. |
| Features needed at read time | None | User embeddings, tweet embeddings, engagement counts — all needing their own low-latency **feature store** |
| Debuggability | Trivial | "Why did I see this?" becomes a research project |
| Consistency | A refresh gives you the same order | A refresh may reorder everything. Users find this disorienting. |

The architectural consequence is the important bit: **ranking turns the timeline cache from an answer into a candidate generator.** You now need a whole ranking tier (candidate generation → light scoring → heavy scoring → diversity/business rules) sitting between the cache and the client, plus a feature store, plus a model-serving fleet. It roughly doubles the read-path complexity.

Interview line: *"I'd ship chronological first, because it's correct, cheap, and lets me prove the fanout architecture. Then I'd add ranking as a layer on top of the same timeline cache — the cache becomes a candidate pool rather than the final answer. The cache design doesn't change; I just fetch 500 IDs instead of 50 and score them."*

#### 7d. Sharding the tweet store and the follow graph

Recall from [64 — Database Sharding](./64-database-sharding.md) that your shard key determines your failure modes.

**Tweet store — shard by `tweet_id` (the Snowflake ID):**
- **Pro:** Snowflake IDs are effectively unique and evenly distributed across the low bits, so hashing them spreads load perfectly. No hot shards. Lookup-by-id (which is what timeline hydration does, and hydration is your highest-volume operation) hits exactly one shard.
- **Con:** "all tweets by user X" now scatters across every shard. Which is why you **keep a separate `user_timeline` table partitioned by `user_id`** — a small index of (user_id → tweet_ids) that makes the user-timeline query a single-partition scan. Two views of the same data, each shaped for its query. Denormalise per access pattern.
- **Anti-pattern:** sharding tweets by `user_id` alone. Now every tweet from a celebrity, and every read of their profile, lands on **one shard**. That shard melts. This is a hot-shard machine.

**Follow graph — shard by the vertex, stored twice:**
- `follows` partitioned by `followee_id` answers "who follows X?" (fanout's query) in one partition.
- `follows_by_follower` partitioned by `follower_id` answers "who does X follow?" (read-merge's query) in one partition.
- **The unavoidable hot partition:** a celebrity's follower list is 150M rows in a single partition. You cannot read it in one go. Mitigate by (a) **not needing to** — celebrities are never fanned out, which is exactly why the hybrid saves you here too, (b) sub-partitioning huge follower lists into buckets (`(followee_id, bucket)`), and (c) caching follower *counts* separately so the "is this a celebrity?" check is O(1) and never touches the graph.

Notice how the hybrid pays off twice: it saves the fanout fleet *and* it means you never have to read a 150-million-row partition.

---

### 8. Bottlenecks, failure modes, and how to scale further

**1. The thundering herd on a viral tweet.** A celebrity tweets; two million people open the app in the same 30 seconds. If that tweet isn't cached yet, two million requests miss the cache *simultaneously* and stampede the tweet store, which collapses.
- **Mitigation:** **request coalescing** (a.k.a. single-flight) — the first miss for a key takes a lock and fetches; the other 1,999,999 wait for its result rather than issuing their own query. Plus **proactive cache warming**: when a celebrity posts, immediately write the tweet into the tweet cache. And per-key **negative caching** for deleted tweets.

**2. Hot shards.** A viral tweet's ID hashes to one shard, and now that shard serves millions of reads/sec while its neighbours idle.
- **Mitigation:** replicate hot keys across many nodes (read replicas per shard), or **cache the hot key at the application tier** (an in-process LRU in front of Redis — a millisecond of local memory absorbs almost everything). The distribution of tweet popularity is a power law, so a tiny local cache captures a huge share of reads.

**3. The fanout queue backing up during a spike.** World Cup goal: writes jump to 12,000+/sec and fanout work jumps *super-linearly* (more tweets AND they're from high-follower accounts). The queue depth grows; timelines go stale by minutes.
- **Mitigation:** (a) **fanout is asynchronous by design** — the queue is the shock absorber, and the user's write already succeeded; (b) **autoscale workers on queue depth**, not CPU; (c) **pre-scale for predictable events** — you know when kickoff is; (d) **shed load by lowering the celebrity threshold under pressure**: temporarily treat 100k-follower accounts as celebrities, converting expensive write work into cheap read work. That last one is a beautiful, cheap valve. (e) **Prioritise the queue**: fan out to recently-active users first so the people actually staring at the app get fresh timelines, and backfill the rest later.

**4. Timeline cache node loss.** A Redis shard dies; a few million users' timelines vanish.
- **Mitigation:** timelines are a **cache, not a source of truth** — they can always be rebuilt from `follows` + `user_timeline`. Serve those users via a temporary **full fanout-on-read** path (slower, but correct), and rebuild their ZSET lazily on first request. Replicate shards so this is rare. Never treat the timeline cache as durable.

**5. The unbounded follow graph read.** Someone follows 50,000 accounts. Their read-time merge is huge.
- **Mitigation:** cap follows (Twitter does), and cap the celebrity-pull list, accepting that ultra-high-follow accounts get a slightly degraded (sampled) timeline.

**6. Scaling further.** Geo-replicate: put a full stack (API, cache, read replicas) in each region, route users to the nearest, and replicate the tweet store asynchronously across regions. Timelines are computed regionally. Accept cross-region lag of a few seconds — N3/N4 said eventual consistency was fine, and here is where you cash that cheque.

---

## Visual / Diagram description

The two diagrams you must be able to redraw from memory are in §6 above: the **full architecture** (client → LB → API → {Tweet, Timeline, Fanout+queue, Graph, Search, Trends} → caches → DBs) and the **hybrid fanout decision + read-merge** flow.

Here is the third — the write path's *timing*, which is what makes the whole thing feel fast:

```
 t=0ms     Client POSTs a tweet
              │
              ▼
 t=5ms     Tweet Service: validate → Snowflake ID → write to tweet store
              │                                     → append to author's user timeline
              ▼
 t=8ms     Publish TweetCreated to Kafka  ─────────┐
              │                                     │
              ▼                                     │  (async, in parallel,
 t=10ms    ◄── 200 OK returned to client ──►        │   nobody is waiting)
           The user sees their tweet. DONE.         │
                                                    ▼
                                        ┌───────────────────────────┐
 t=50ms  ── 3s                          │ Fanout workers: ZADD id   │
                                        │ into follower timelines   │
                                        ├───────────────────────────┤
                                        │ Search indexer: tokenise  │
                                        │ + update inverted index   │
                                        ├───────────────────────────┤
                                        │ Trends job: extract tags  │
                                        │ + update sliding CMS      │
                                        └───────────────────────────┘
```

The lesson the diagram teaches: **the synchronous write path is ~10ms and touches two systems.** Everything else — delivery, indexing, trend counting — happens off a queue, after the user has already been told "sent." Anything you can move below that dashed line, you should.

---

## Real world examples

### Twitter / X

Twitter published the canonical version of this design. Their timeline system (historically "Timeline Service" backed by a Redis-based store) precomputes home timelines by fanning out tweet IDs — not tweet bodies — into per-user in-memory lists on write, capped at roughly 800 entries. Tweets themselves are stored once and hydrated at read time. Snowflake, their distributed time-sortable ID generator, was open-sourced in 2010 and is the origin of the 41-bit-timestamp / 10-bit-machine / 12-bit-sequence layout used across the industry. They have publicly discussed the celebrity/high-fanout problem and the resulting hybrid of push for most accounts and merge-at-read for very-high-follower accounts.

### Facebook

Facebook's News Feed leans much harder toward **pull + rank**: rather than materialising a finished feed per user, it generates a candidate set at read time and runs a heavy ML ranking pass over it. This is coherent with their product — the Facebook feed was never promised to be chronological, so the freedom to rank buys more than the cost of read-time computation. It's the same trade from §7c, resolved in the other direction because the product requirements differ. (Conceptually; the precise internal architecture is not fully public.)

### Discord and Slack

Channel-based messaging looks nothing like Twitter until you notice the shape: many writers, many readers, a membership graph in between. But because a channel has bounded membership (thousands, not 200 million), **fanout on read is fine** — you fetch the channel's recent messages once and every member reads the same list. The lesson is precise and worth internalising: **fanout on write only becomes necessary when each reader's view is unique.** A channel is one view shared by many. A home timeline is a unique view per person. That difference, not the message volume, is what forces the fanout architecture.

---

## Trade-offs

**Fanout strategy:**

| Strategy | Read latency | Write cost | Wasted work | Verdict |
|---|---|---|---|---|
| Pure fanout on write | Excellent | O(followers) — fatal for celebrities | High (inactive users) | Broken at Twitter scale |
| Pure fanout on read | Poor (N queries + merge) | O(1) | None | Can't hit the 200ms SLO |
| **Hybrid (push normal, pull celeb)** | **Good** | **Bounded** | **Moderate** | **Ships** |

**Storing IDs vs. content in the timeline:**

| | Fan out IDs | Fan out full content |
|---|---|---|
| Bytes per follower | 8 | ~300 |
| Extra read step | Yes — a hydration mget | No |
| Storage for 100k-follower tweet | 800 KB | 30 MB |
| Deleted/edited tweet | Fixed in one place | Stale copies everywhere |
| Verdict | **Do this** | Only for tiny fanouts |

**Chronological vs ranked:** see the table in §7c. Chronological is free and honest; ranked is engaging and costs you a candidate-generation tier, a feature store, and an inference budget inside your 200ms.

**Exact vs approximate trend counts:**

| | Exact | Count-Min Sketch |
|---|---|---|
| Memory | O(distinct hashtags) — unbounded | O(width × depth) — **fixed** |
| Accuracy | Perfect | Never underestimates; slightly over |
| Fits at 12k tweets/s | No | Yes |

Nobody cares whether `#worldcup` had 84,412 or 84,589 mentions. They care that it's #1. **Approximate is the correct answer, and knowing when approximation is acceptable is a senior-engineer signal.**

**Rule of thumb:** In any feed system, ask *"is each reader's view unique, and does any writer have a pathological number of readers?"* If yes to both, you need the hybrid. If no to either, you're over-engineering.

**The sweet spot:** Fan out **IDs** (not content) on write for the ~99.9% of accounts below the celebrity threshold; **merge celebrities in at read time**; make the threshold a **tunable knob** you can turn under load; and let a queue absorb every spike so the user's write path stays a flat 10ms.

---

## Common interview questions on this topic

### Q1: "Why not just query the database at read time — `SELECT * FROM tweets WHERE user_id IN (...) ORDER BY created_at DESC LIMIT 50`?"

**Hint:** Because that `IN` clause fans out to 400 user_ids scattered across every shard of a sharded tweet store, and you'd run it 35,000 times per second. Each query becomes a scatter-gather across the cluster, and its latency is that of the *slowest* shard (tail-latency amplification — with 100 shards, a p99-slow shard is hit on almost every request). You cannot hit a 200ms p99 that way. Precompute the answer instead: do the work once at write time, on the ~6,000 writes/sec, rather than 35,000 times a second at read time. **That's the whole insight — this system is read-heavy, so you move work to the write side.**

### Q2: "What happens when a user with 150 million followers tweets?"

**Hint:** *Nothing fans out.* Do the arithmetic out loud: 150M insertions at 1ms each is 41 hours serially; even with 1,000 workers at 1,000 ins/sec it's 150 seconds of the entire fanout fleet, for one tweet, while 6,000 more arrive every second. So celebrity tweets are never pushed. They're written once to the tweet store and to the celebrity's own hot user-timeline cache, and **merged in at read time** by each follower's timeline read. The cost is now proportional to how many celebrities *you follow* (≈5), not how many followers *they have* (150M).

### Q3: "How do you compute trending topics without counting every hashtag?"

**Hint:** Two parts, and the second is what they're really testing. **(1)** A streaming job over a sliding window using a **Count-Min Sketch** — fixed memory, approximate counts, never underestimates — plus a top-K heap. **(2)** The insight: **trending is not raw volume, it's an unusual rate of change relative to that topic's own baseline.** Rank by z-score against an EWMA baseline, not by count — otherwise `#love` trends forever and nothing is ever news. Add a minimum-volume floor so a hashtag going 1 → 30 doesn't spike, and count distinct *users* not tweets, to defeat bots.

### Q4: "Why Snowflake IDs instead of auto-increment or UUIDs?"

**Hint:** Auto-increment needs a central coordinator — a SPOF and a latency tax at 12,000 writes/sec. UUIDs are 16 random bytes: they destroy sort order (so you'd need a separate timestamp comparison on every merge), double your timeline-cache memory, and wreck write locality. Snowflake gives you 64 bits, generated **independently on any machine with no coordination**, that still **sort chronologically** because the high 41 bits are a millisecond timestamp. That means your timeline merge is a plain numeric sort, Redis can use the ID *as the ZSET score*, and your primary key doubles as your time index.

### Q5: "Your fanout queue is 40 minutes behind during the World Cup final. What do you do?"

**Hint:** First, note that nothing is *broken* — writes still succeed in 10ms; delivery is delayed. Then, in order: **autoscale workers on queue depth** (not CPU); **prioritise active users** in the queue so people actually looking at the app get fresh timelines; and the clever valve — **dynamically lower the celebrity threshold**, converting expensive write-time work into cheap read-time merges for more accounts. Longer term: **pre-scale**, because these spikes are *predictable* — you know the kickoff time. The fact that the queue can back up without the site going down is the architecture working as designed.

### Q6: "How is a user timeline different from a home timeline?"

**Hint:** A **user timeline** is `WHERE user_id = X ORDER BY id DESC LIMIT 50` — one partition, one range scan, trivially cacheable. A **home timeline** requires merging tweets from the 400 accounts you follow — and that is the entire system. Candidates who conflate them end up designing a fanout architecture for a problem that a single index solves, or (worse) trying to solve the home timeline with a database query.

---

## Practice exercise

### Build the hybrid fanout, and prove the threshold matters

**~35 minutes. Produce a runnable Node script plus a short written answer.**

**Part 1 — Simulate (20 min).** Write `feed.js` with:
- An in-memory `follows` map: 10,000 synthetic users. Give them follower counts on a **power law** — most have 10–100 followers, a handful have 5,000, and **one has 9,000** (your "celebrity", following ~90% of the population).
- A `SnowflakeGenerator` (copy the one from §5).
- A `fanoutOnWrite(tweet)` that pushes the tweet ID into every follower's timeline array — and **counts total insertions**.
- A `getHomeTimeline(userId)` implementing the **4-step hybrid merge** from §3.
- A `CELEBRITY_THRESHOLD` constant.

Now post 1,000 tweets, distributed so the celebrity posts 5% of them. Log:
1. **Total timeline insertions with the threshold DISABLED** (pure fanout on write).
2. **Total timeline insertions with the threshold set to 1,000.**
3. The **average number of timelines touched per tweet** in each case.

**Part 2 — Answer in writing (15 min).**
- By what factor did the threshold reduce fanout work? Was the reduction proportional to the celebrity's share of *tweets*, or of *followers*?
- Sweep the threshold: 100, 1,000, 5,000. Plot (or just tabulate) total insertions vs. average read-merge cost. **Where's the crossover?**
- Your 9,000-follower celebrity now gains 90,000 followers overnight, crossing the threshold. What happens to the timelines that *already contain* their tweets? Is the transition safe? (Think about it: fanned-out tweets stay in the cache, and new ones get merged at read. **Is there a window where a tweet appears twice?** How would you dedupe?)

That last question is the real exercise. Write your answer down.

---

## Quick reference cheat sheet

- **Home timeline ≠ user timeline.** User timeline is one partition scan. Home timeline is the whole system. Never conflate them.
- **The read:write ratio (~6:1 at request level, far higher amplified) says: do the work at WRITE time.** Precompute; don't query at read.
- **Text is free; fanout and reads are not.** 275 TB over 5 years is trivial. 35,000 timeline assemblies/sec is not. Optimise the expensive thing.
- **Never fan out celebrity tweets.** 150M insertions = 41 hours serially, ~150s of your entire fanout fleet. Merge them at read time instead.
- **The hybrid read, in 4 steps:** (1) fetch precomputed timeline, (2) fetch tweets from the *few* celebrities you follow, (3) merge + sort by ID, (4) hydrate and return.
- **The cost inverts:** read-merge cost is proportional to how many celebrities *you follow* (≈5), not how many followers *they have* (150M).
- **The celebrity threshold is a TUNABLE KNOB**, not a constant. Lower it under load to shed fanout work.
- **Fan out IDs (8 bytes), never content (300 bytes).** 37× cheaper, and edits/deletes stay correct because the tweet lives in exactly one place.
- **Timeline cache = Redis ZSET of ~800 tweet IDs per user.** 150M × 800 × 8B ≈ 1 TB raw (2–3 TB with overhead) → a sharded Redis cluster.
- **The timeline cache is a CACHE, not a source of truth.** It can always be rebuilt from `follows` + `user_timeline`. Design for that.
- **Snowflake IDs** = 41-bit timestamp + 10-bit machine + 12-bit sequence. Coordination-free, 64-bit, and **time-sortable** — so the ID *is* the sort key and *is* the time index.
- **Trending is a RATE OF CHANGE, not a volume.** Score by z-score against the topic's own baseline, or `#love` trends forever. This is the insight interviewers want.
- **Count-Min Sketch** gives you top-K hashtags in fixed memory with counts that never underestimate. Approximate is the correct answer here.
- **Search = an inverted index fed by the same tweet stream**, with a two-tier (real-time + archival) structure so tweets are findable within seconds.
- **Shard tweets by tweet_id** (even distribution, no hot shards), and keep a **separate user_id-partitioned index** for user timelines. Denormalise per access pattern.
- **The write path is ~10ms and returns before fanout runs.** The queue absorbs every spike. That's why a backed-up queue is a delay, not an outage.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [100 — Design Instagram](./100-hld-instagram.md) — where you first met fanout-on-write vs fanout-on-read and the celebrity problem; Twitter is the extreme version of the same trade |
| **Next** | [128 — LLD: Social Media Feed](./128-lld-social-media-feed.md) — zoom in from boxes-and-arrows to the actual classes: Feed, Post, FanoutStrategy, TimelineCache |
| **Related** | [59 — Caching in Depth](./59-caching-in-depth.md) — the timeline cache, the tweet cache, hydration, thundering herds, and request coalescing all live here |
| **Related** | [79 — Search Systems](./79-search-systems.md) — the inverted index behind tweet search, and why near-real-time indexing is hard |
| **Related** | [64 — Database Sharding](./64-database-sharding.md) — why you shard tweets by tweet_id, store the follow graph twice, and how a celebrity creates a hot partition |
