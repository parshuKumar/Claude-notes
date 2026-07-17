# 100 — Design Instagram / Photo Sharing
## Category: HLD Case Study

---

## What is this?

Instagram is a **photo-sharing social network**. You upload a picture with a caption, you follow other people, and when you open the app you see a **news feed** — a stream of photos from everyone you follow, freshest first. You can like a photo and leave a comment.

That description takes ten seconds to say and hides one of the most beautiful problems in distributed systems: **how do you assemble a personalized feed for 500 million people, 5 billion times a day, in under 200 milliseconds — when every single one of those feeds is different?**

Think of it like a newspaper. A normal newspaper prints one edition for everybody. Instagram has to print **500 million unique editions every day**, and re-print each one every time somebody opens the app. The entire architecture of this system is an answer to that one question.

---

## Why does it matter?

This is the single most-asked "social" system design question, and it's asked for one reason: it forces you to confront the **fanout problem** — the choice between doing work at write time or at read time. Almost every social product on earth (Twitter, Facebook, LinkedIn, TikTok, Reddit) sits somewhere on that spectrum. If you understand fanout, you understand all of them.

**In the interview**, this question separates people who memorize architectures from people who reason about them. Anybody can say "we'll use a cache." The interviewer is waiting for the moment they say *"okay, now what happens when a user with 100 million followers posts?"* — and either you have the hybrid answer ready, or you don't.

**At work**, the same trade-off shows up constantly and rarely with the word "fanout" attached:
- Do you compute a dashboard on page load, or materialize it nightly?
- Do you denormalize a counter into the row, or COUNT(*) every time?
- Do you push notifications into per-user inboxes, or query a shared log?

Same decision. Precompute vs compute-on-demand. Instagram is just the loudest, most extreme version of it.

---

## The core idea — explained simply

### The Mailroom Analogy

Imagine a giant office building with 500 million employees. Everybody publishes a little newsletter now and then, and everybody subscribes to the newsletters of a few hundred colleagues.

There are exactly two ways to run the mailroom.

**Option A — the "go fetch it yourself" mailroom (fanout on read).**
Nobody delivers anything. Each newsletter just sits in the author's own filing cabinet. When you want to catch up, you walk the building, visit the filing cabinet of all 500 people you subscribe to, pull out their latest issues, carry the pile back to your desk, and sort it by date.

Nothing is wasted — you only fetch what you actually read. And a newsletter is readable the *instant* it's filed. But your morning walk takes forever, and **everybody does this walk ten times a day.** The building's hallways are permanently jammed.

**Option B — the "delivery" mailroom (fanout on write).**
The moment you publish, the mailroom makes a copy for every one of your subscribers and drops it in their personal inbox tray. Now reading is trivial: you glance at your tray. One motion. Done.

Publishing costs more — one publish becomes 500 deliveries — but you publish rarely and read constantly, so the trade is wildly in your favour.

**Until the CEO publishes.** The CEO has 100 million subscribers. The mailroom now has to make 100 million copies for one memo. The mail carts catch fire, the hallways flood, and the memo is still being delivered forty minutes later.

**Option C — what Instagram actually does (hybrid).**
Deliver normal people's newsletters into trays (Option B). But for the handful of celebrities in the building, **don't deliver at all** — leave their newsletters in their filing cabinets. When you check your tray, you also make one quick stop at the cabinets of the two or three celebrities you follow, and merge that into your pile.

You get O(1) reads for 99.9% of the content, and you never pay the 100-million-copy bill.

| Analogy | Technical concept |
|---|---|
| Employee | User |
| Newsletter | Photo / post |
| Filing cabinet | The `photos` table, sharded by user |
| Personal inbox tray | Precomputed feed list in Redis (`feed:{userId}`) |
| The mailroom copying to every tray | **Fanout on write** — the fanout worker |
| Walking the building yourself | **Fanout on read** — scatter-gather across followees |
| The CEO's 100M subscribers | **The celebrity problem** |
| Checking your tray + a few cabinets | **The hybrid feed** |
| Only keeping the last 500 memos in the tray | Feed cache size cap |
| Not delivering to people who quit years ago | Skipping inactive users |

---

## Key concepts inside this topic

We'll run the 5-step framework from [93 — The System Design Interview Framework](./93-hld-approach-framework.md): requirements → estimation → API → data model → architecture → deep dives.

---

### 1. Requirements

**Clarifying questions to ask the interviewer (ask these out loud — they earn points):**
- Are we designing the *whole* app or just the core photo + feed flow? (Scope it down. Say: "I'll focus on upload, follow, and feed. I'll skip stories, DMs, and reels unless you want them.")
- Is the feed strictly reverse-chronological, or ML-ranked? (Start chronological, mention ranking as an extension.)
- Do we support video? (Assume photos only; note video changes the media pipeline, not the feed logic.)
- Roughly how many users, and what's the follower distribution? (You're fishing for the celebrity problem — make the interviewer say it, or say it yourself.)

**Functional requirements (in scope):**
1. **Upload a photo** with an optional caption.
2. **Follow** another user (and unfollow).
3. **View a personalized news feed** — photos from people you follow, newest first (reverse-chronological; ranked is an extension).
4. **Like** a photo and **comment** on it.

**Out of scope (say this explicitly):** stories, direct messages, search, hashtags, explore page, ads.

**Non-functional requirements — this is where the design is actually decided:**

| Requirement | Target | Why it drives the design |
|---|---|---|
| **Extremely read-heavy** | **~100:1 read:write** | People scroll far more than they post. This single ratio is the reason we precompute feeds. |
| **Feed latency** | **< 200ms p99** | A slow feed is a dead product. Users abandon. This forces the feed into a cache, not a database join. |
| **High availability** | 99.9%+ | It's better to serve a slightly stale feed than no feed. Availability > consistency here. |
| **Eventual consistency is fine** | seconds of lag OK | **If your friend's photo shows up 5 seconds late, nobody dies.** |
| Durability of photos | Never lose a photo | Photos are precious; feed entries are disposable. Different storage guarantees. |

That fourth row deserves its own paragraph, because it is the sentence that **unlocks the entire design**:

> **Eventual consistency is perfectly acceptable for the feed.** Nobody notices, nobody complains, nobody is harmed if a post takes 5 seconds to appear. Compare that to a bank transfer — where a 5-second inconsistency means money exists in two places at once.

Because we accepted eventual consistency, we are *allowed* to do the post asynchronously, in the background, via a queue, into a cache that may lag reality. If the interviewer had said "strong consistency required," fanout-on-write would be illegal and this whole design collapses. Say this out loud in the interview. It shows you understand that non-functional requirements aren't a checkbox ritual — they're the constraints that generate the architecture.

---

### 2. Capacity estimation

Show the arithmetic. Round aggressively — nobody wants five significant figures.

**Users and writes**
```
DAU                       = 500,000,000
Photos uploaded / day     = 2,000,000
Seconds in a day          ≈ 86,400  (call it 100,000 for easy math)

Upload QPS = 2,000,000 / 86,400 ≈ 23  →  call it ~25 uploads/sec
Peak (3x)  ≈ 75 uploads/sec
```
**~25 writes per second.** That is *nothing*. A single Postgres box handles that half asleep.

**Reads**
```
Feed refreshes per user per day  = 10
Feed reads / day = 500,000,000 × 10 = 5,000,000,000  (5 billion)

Feed read QPS = 5,000,000,000 / 86,400 ≈ 57,870  →  ~58,000 reads/sec
Peak (3x)     ≈ 175,000 reads/sec
```

**Now say the loud part out loud:**
```
   Reads : Writes
   58,000 : 25
   ≈ 2,300 : 1  at the feed layer
```
Even using the conservative "100:1" product-level ratio (which counts likes, profile views, etc.), reads dominate by **orders of magnitude**.

> **This asymmetry IS the architecture.** When one side of a system is thousands of times hotter than the other, you move all the work to the cold side. Writes are cheap and rare — so we happily make each write do 500× more work, if it buys us an O(1) read. Every major decision below is a consequence of this one number.

**Storage**
```
Original photo             ≈ 2 MB
Resized variants (thumb, feed, full)  ≈ 1 MB total
Per photo (all variants)   ≈ 3 MB

Per day    = 2,000,000 × 3 MB   = 6,000,000 MB = 6 TB/day
Per year   = 6 TB × 365         ≈ 2.2 PB/year
Over 5 yrs ≈ 11 PB  (before replication; ×3 replicas ≈ 33 PB)
```
That is **way** past a single disk, a single machine, or a single database. Photos go in **blob storage** (S3/GCS) — see [78 — Blob Storage](./78-blob-storage.md). Never, ever in the database.

**Metadata storage** (the cheap part)
```
photos row ≈ 500 bytes (id, user_id, s3_key, caption, timestamps)
2M/day × 500B = 1 GB/day  ≈ 365 GB/year
```
Metadata for 5 years fits on **one laptop**. The photos need a datacentre. Different problems, different storage.

**Bandwidth (serving)**
```
Feed load = 10 photos, each ~200 KB (feed-sized variant) = 2 MB per feed load
5B feed loads/day × 2 MB = 10 PB/day egress
10 PB/day ÷ 86,400s ≈ 115 GB/s  ≈ 900+ Gbps sustained
```
No origin server farm serves ~1 Tbps of images. This number is why the **CDN is not optional** — see [60 — CDN](./60-cdn.md). The CDN absorbs ~95%+ of that; your origin sees the rest.

**Feed cache memory**
```
Store last 500 post-IDs per user, 8 bytes each = 4 KB/user
Only cache ACTIVE users (say 100M of the 500M DAU are "hot")
100,000,000 × 4 KB = 400 GB
```
400 GB of Redis. That's ~7 machines with 64 GB each, or one beefy cluster. **Completely affordable.** Notice we cache *post IDs*, not post content — that's what keeps it small.

---

### 3. Photo upload and storage

The naive design — client POSTs a 4 MB file to your Node API server, which streams it to S3 — is wrong, and it's wrong for a reason worth internalizing.

**Why not through the API server?** Your API servers are small, stateless, CPU-light boxes. Streaming 4 MB uploads through them ties up a connection and a chunk of memory for the whole duration of a slow mobile upload (which could be 30 seconds on bad 4G). At 75 peak uploads/sec, you'd have thousands of connections parked in your app tier doing nothing but babysitting bytes. You'd be scaling API servers to act as a very expensive, very fragile file proxy.

**The right pipeline — pre-signed URLs (recall [78 — Blob Storage](./78-blob-storage.md)):**

```
 ┌────────┐  1. POST /photos/upload-url          ┌──────────────┐
 │ Client │ ───────────────────────────────────▶ │  API Server  │
 │ (phone)│ ◀─────────────────────────────────── │              │
 └───┬────┘  2. { uploadUrl, photoId, s3Key }    └──────┬───────┘
     │                                                   │ (writes a
     │ 3. PUT the raw bytes DIRECTLY                     │  "PENDING"
     │    (never touches your servers)                   │  metadata row)
     ▼                                                   ▼
 ┌──────────────────┐   4. ObjectCreated event   ┌──────────────┐
 │  Blob Storage    │ ─────────────────────────▶ │ Media Queue  │
 │  (S3 / GCS)      │                            │  (SQS/Kafka) │
 │  raw/{photoId}   │                            └──────┬───────┘
 └──────────────────┘                                   │ 5.
     ▲                                            ┌─────▼────────┐
     │ 6. write variants                          │ Media Worker │
     │    thumb / feed / full                     │ (sharp/libvips)
     └────────────────────────────────────────────┤              │
                                                  └─────┬────────┘
 ┌──────────────────┐   7. mark photo READY            │
 │  Metadata DB     │ ◀─────────────────────────────────┘
 │  (photos table)  │        8. enqueue FANOUT job
 └──────────────────┘ ───────────────────────────▶ [ Fanout Queue ]

 Serving:  Client ──▶ CDN edge ──(miss)──▶ Blob Storage
                       (99% hit)
```

**Step by step:**
1. Client asks the API for permission to upload. The API does auth, rate-limiting, and generates a **pre-signed URL** — a time-limited, single-purpose URL that grants write access to exactly one S3 key.
2. API inserts a `photos` row with `status = 'PENDING'` and returns the URL.
3. Client PUTs the bytes **straight to S3**. Your servers see zero image bytes. Your upload capacity is now S3's capacity, which is effectively infinite.
4. S3 emits an `ObjectCreated` event → lands on a queue.
5. An async **media worker** picks it up.
6. The worker generates **resized variants**. This is not a nicety — it's a bandwidth decision. Sending a 4 MB original to a phone rendering a 400px thumbnail wastes ~95% of the bytes. At 10 PB/day of egress, a 4× reduction in image size saves you *millions of dollars a year*. Typical variants:

   | Variant | Longest edge | Approx size | Used for |
   |---|---|---|---|
   | `thumb` | 320 px | ~30 KB | Profile grid, comments |
   | `feed` | 1080 px | ~200 KB | The news feed (the hot one) |
   | `full` | 2048 px | ~800 KB | Tap-to-zoom / detail view |

7. Worker flips the row to `status = 'READY'`.
8. **Only now** does it enqueue the fanout job. (Fanning out before the variants exist would put a broken image in people's feeds.)

**Serving.** The `photos` row stores an `s3_key`, and the API returns a CDN URL like `https://cdn.example.com/feed/{photoId}.jpg`. First request for that URL misses the edge and pulls from S3; every subsequent request in that region is served from the edge in ~20ms.

> **Images are immutable.** A photo, once uploaded, never changes its bytes. Editing a caption doesn't touch the image. This makes CDN caching *trivially safe*: set `Cache-Control: public, max-age=31536000, immutable` (one year) and never think about invalidation again. Compare this to caching an HTML page, where you're constantly fighting staleness. **Immutable content is the easiest thing in the world to cache** — recall [59 — Caching in Depth](./59-caching-in-depth.md). If a photo must be replaced, you give it a new key, not new bytes at the old key.

---

### 4. NEWS FEED GENERATION — the heart of the system

Everything above was warm-up. This is the question.

**The problem:** user `alice` opens the app. She follows 500 people. Produce, in under 200ms, the 20 most recent photos posted by any of those 500 people.

There are exactly two families of answers, and the whole interview lives in the space between them.

---

#### 4a. Fanout on READ (pull model)

Do nothing at write time. A post is just a row in `photos`. When Alice opens the app, **go find her feed on demand**:

```sql
SELECT * FROM photos
WHERE user_id IN (SELECT followee_id FROM follows WHERE follower_id = 'alice')
ORDER BY created_at DESC
LIMIT 20;
```

In a sharded world that innocent SQL becomes a **scatter-gather**: your 500 followees live on, say, 40 different shards. You must query all 40, get back their recent posts, merge them in the app tier, sort by timestamp, take the top 20.

```
        Alice opens the app  (58,000 times/sec across all users)
                     │
          ┌──────────┴───────────┐
          │   Feed Service       │
          └──┬───┬───┬───┬───┬───┘
             │   │   │   │   │       500-way scatter
       ┌─────▼┐ ┌▼──┐ ▼  ┌▼──┐ ┌─────▼┐
       │shard1│ │s2 │ …  │s39│ │shard40│   ← query every shard
       └─────┬┘ └┬──┘    └┬──┘ └─────┬┘      holding a followee
             └───┴────┬───┴──────────┘
                      ▼
             merge + sort by time      ← k-way merge in memory
                      ▼
                  top 20 posts
```

**Pros:**
- **Dead simple.** No workers, no queues, no precomputed state, nothing to keep in sync.
- **Zero wasted work.** If a user never opens the app, you never compute their feed. Fanout-on-write would have done that work for nothing.
- **Instantly consistent.** A post is visible the microsecond it's committed. No lag.
- **Celebrity-proof.** A 100M-follower post is *one row insert*. Fanout-on-read literally does not have a celebrity problem.

**Cons — and this one is fatal:**
- **The read is expensive, and reads are 2,300× more common than writes.** Every single feed load pays for a 500-way scatter-gather and a merge sort. Multiply that by 58,000/sec (175,000/sec at peak). You have taken your **highest-volume operation** and made it your **most expensive operation**. That is precisely backwards.
- p99 latency is hostage to your slowest shard. One slow shard out of 40 and Alice waits.
- Impossible to hold a sub-200ms budget reliably at that fanout width.

> **The lesson: optimize the common case.** Reads are the common case here by three orders of magnitude. Fanout-on-read optimizes writes — which nobody was complaining about.

---

#### 4b. Fanout on WRITE (push model)

Invert it. Pay at write time, when almost nobody is watching.

When Bob posts a photo, a background worker looks up **everyone who follows Bob** and pushes the new `photoId` onto a **precomputed feed list** for each of them — a Redis list keyed `feed:{userId}`.

Reading a feed now costs one command:

```
LRANGE feed:alice 0 19     →  20 post IDs, in ~1ms
```

Then a single batched `MGET` to hydrate those 20 IDs from a post cache. **That's the whole read path.** No joins. No scatter. No sort — the list is *already* in order, because you pushed to the head.

```
        Bob posts a photo  (25 times/sec across all users)
                     │
                     ▼
              ┌─────────────┐
              │ Post Service│ ── insert into photos, then enqueue ──▶ [ Fanout Queue ]
              └─────────────┘                                              │
                                                                           ▼
                                                            ┌──────────────────────┐
                                                            │   Fanout Worker      │
                                                            │  followers(bob) = 500│
                                                            └───────┬──────────────┘
                                       ┌───────────────┬────────────┼──────────────┐
                                       ▼               ▼            ▼              ▼
                                 LPUSH feed:alice  feed:carol   feed:dave  …  feed:user500
                                 LTRIM  0 499      (cap at 500 entries each)

        ── later ──
        Alice opens the app  →  LRANGE feed:alice 0 19  →  DONE.  ~5ms total.
```

**Pros:**
- **Reads are O(1) and blazingly fast.** A single Redis LRANGE. This comfortably fits the <200ms budget with room to spare — you'll typically land in 10-30ms.
- Read cost is **independent of how many people you follow.** Follow 5,000 people? Still one LRANGE.
- Predictable, flat latency. The p99 is the p50.

**Cons:**
- Writes are amplified: one post → N list insertions. At 500 followers average and 25 posts/sec, that's `25 × 500 = 12,500` Redis LPUSHes/sec. Redis does millions/sec. **Fine.**
- Wasted work for users who never open the app.
- Feed is eventually consistent (a second or two of lag). **We already declared that acceptable.** This is why that requirement mattered.
- **And then there's the celebrity.**

---

#### 4c. The celebrity problem (the killer objection)

This is the reason the question gets asked. Do the arithmetic on a whiteboard:

```
A celebrity with 100,000,000 followers posts ONE photo.

Fanout-on-write must perform:
    100,000,000 LPUSH + LTRIM operations

Suppose Redis (pipelined, across a cluster) sustains 500,000 ops/sec for this job:
    100,000,000 / 500,000 = 200 seconds  ≈ 3.3 minutes  for ONE post

Meanwhile:
  • That job monopolizes your fanout workers and Redis cluster.
  • Ordinary users' posts sit behind it in the queue — their feeds lag by minutes.
  • The celebrity's own followers see the post trickle in over 3 minutes:
      follower #1 sees it instantly, follower #99,000,000 sees it 3 minutes later.
  • If 5 celebrities post at once, the whole feed pipeline is down.
```

**This is a thundering herd of writes, triggered by a single HTTP request.** One user's action generates 100 million writes. No amount of horizontal scaling makes that *cheap* — it only makes it *possible*, expensively.

And here's the insult: most of those 100 million people **won't open the app today.** You just did tens of millions of writes that will be evicted from cache before anybody reads them.

---

#### 4d. The HYBRID solution (the answer they want)

The insight is a distribution observation, not a technical trick:

> **There are very few celebrities, but each user follows only a handful of them.**

So: **push for normal users, pull for celebrities.**

- Define a threshold — say **`followerCount > 100,000`** → this account is a "celebrity."
- **Normal user posts** → fan out on write, exactly as in 4b. Cheap (a few hundred to a few thousand LPUSHes).
- **Celebrity posts** → **do not fan out at all.** Just write the row and go home. One insert. Done in 5ms.
- **At read time**, the Feed Service does *both*:
  1. `LRANGE feed:alice 0 199` — the precomputed feed (everything from her ~495 normal followees).
  2. Look up which celebrities Alice follows (a small set, cached — typically 2-10 accounts). Fetch their recent posts from a small, extremely hot **per-celebrity recent-posts cache** (`celeb_posts:{celebId}` — a Redis list of their last ~50 post IDs, always warm because *thousands* of users are pulling it every second).
  3. **Merge** the two lists by timestamp, take the top 20.

```
   ┌────────────────────────── WRITE PATH ──────────────────────────┐
   │                                                                │
   │   Bob posts (500 followers)          Ronaldo posts (100M)      │
   │        │                                    │                  │
   │        ▼                                    ▼                  │
   │   is celebrity? NO                    is celebrity? YES        │
   │        │                                    │                  │
   │        ▼                                    ▼                  │
   │   FAN OUT: 500 LPUSHes           DO NOTHING (just the row +    │
   │   into 500 feed lists            LPUSH celeb_posts:ronaldo)    │
   │   ~50ms of work                  ~5ms of work                  │
   └────────────────────────────────────────────────────────────────┘

   ┌────────────────────────── READ PATH ───────────────────────────┐
   │                   Alice opens the app                          │
   │                          │                                     │
   │            ┌─────────────┴──────────────┐                      │
   │            ▼                            ▼                      │
   │   LRANGE feed:alice 0 199     celebs Alice follows = [ronaldo, │
   │   (precomputed, from her                     taylor]  (2 only) │
   │    495 normal followees)      LRANGE celeb_posts:ronaldo 0 20  │
   │            │                  LRANGE celeb_posts:taylor  0 20  │
   │            │                            │                      │
   │            └────────────┬───────────────┘                      │
   │                         ▼                                      │
   │              MERGE by created_at, take top 20                  │
   │                         ▼                                      │
   │              hydrate post bodies (one MGET)                    │
   │                         ▼                                      │
   │                   ~15ms.  Done.                                │
   └────────────────────────────────────────────────────────────────┘
```

**Why does this work so well?**
- The read-time merge is **tiny and bounded**: 1 LRANGE + *k* small LRANGEs where k = number of celebrities you follow ≈ 2-10. That's a 3-11-way merge, not a 500-way one. Fanout-on-read's problem was the *width* of the scatter — the hybrid keeps the width in single digits.
- The celebrity caches are the hottest keys in the system, so they're always in memory with a ~100% hit rate. Reading them is ~1ms.
- You've eliminated the 100-million-write storm entirely, at the cost of a few extra milliseconds on the read.
- **Population math:** if 0.01% of accounts are celebrities but they hold most of the follower mass, you've removed the vast majority of your total fanout volume by special-casing a vanishingly small number of accounts.

**The threshold is a tunable knob**, not a law of physics. Too low → too many pull-path merges → reads get wide again. Too high → you're still doing multi-million-follower fanouts. Tune it against your actual follower distribution; 10k-1M is the usual range. Interview tip: say "I'd make it a config value and tune on real data," not "it's exactly 100,000."

---

#### 4e. The other wrinkles (mention these — they're free points)

**Inactive users.** Of 500M registered accounts, maybe 100M open the app on a given day. Fanning out to an account that hasn't logged in for 6 months is pure waste — memory in Redis, CPU in workers, all for a feed nobody reads.

*Fix:* keep a `last_active_at` timestamp per user. The fanout worker **skips users inactive for > 30 days**. When such a user does come back, their `feed:{userId}` key is missing → the Feed Service detects the miss and **lazily rebuilds** it via fanout-on-read (the scatter-gather from 4a), then caches the result. One slow load for a returning user; enormous savings the rest of the time. This is just **cache-miss-and-backfill** — you've seen it in [59 — Caching in Depth](./59-caching-in-depth.md).

**Feed size caps.** Nobody scrolls 10,000 posts deep. Store only the most recent **~500 entries** per feed list — `LTRIM feed:{userId} 0 499` after every push. This bounds memory at ~4 KB/user (500 × 8-byte IDs) and makes the total cache cost predictable. If a user *does* scroll past entry 500 (rare), fall back to the pull path for the older pages. Deep pagination is not the hot path; don't optimize it.

**IDs, not bodies.** Store only post IDs in the feed list. Post bodies live in a separate `post:{id}` cache, shared by every feed containing them. If you stored the full post in each of 500 feed lists, you'd store the caption and metadata 500 times. **Store the pointer, hydrate on read.**

---

### 5. Data model

**`users`** — Postgres/MySQL. Small, relational, needs transactions for signup.
```
id            BIGINT PK
username      VARCHAR(30) UNIQUE
email         VARCHAR
display_name  VARCHAR
avatar_key    VARCHAR
follower_count  INT        -- denormalized; drives the celebrity check
following_count INT
last_active_at  TIMESTAMP  -- drives the inactive-user skip
created_at      TIMESTAMP
```

**`photos`** — the metadata for each post. Append-only, never updated (except caption).
```
id          BIGINT PK        -- time-sortable ID (Snowflake) so ORDER BY id == ORDER BY time
user_id     BIGINT           -- INDEX (user_id, id DESC)  ← "give me this user's recent posts"
s3_key      VARCHAR          -- e.g. "photos/2026/07/abc123"
caption     TEXT
width,height INT
status      ENUM('PENDING','READY')
created_at  TIMESTAMP
```
Sharded by `user_id` (see deep dive C). **NoSQL (Cassandra/DynamoDB) is also a defensible choice** — this table is append-heavy, never joined, and always queried by a known key. But relational is fine too and easier to reason about; the metadata volume (~365 GB/year) is modest.

**`follows`** — **the hot table, and the tricky one.**
```
follower_id BIGINT   -- the person doing the following
followee_id BIGINT   -- the person being followed
created_at  TIMESTAMP
PRIMARY KEY (follower_id, followee_id)
```
You need **indexes in BOTH directions**, because you have two completely different queries:

| Query | Index needed | Used by |
|---|---|---|
| "Who does Alice follow?" | `(follower_id, followee_id)` | Read path — pulling celebrity posts, rebuilding a cold feed |
| **"Who follows Bob?"** | `(followee_id, follower_id)` | **Fanout** — this is the query that drives the entire write path |

The second one is the whole reason fanout works, and it's the one people forget. Without it, "get Bob's 500 followers" is a full table scan of a multi-billion-row table.

**Why the follow graph is hard:** it's a *graph*, and graphs resist sharding. Shard by `follower_id` and "who follows Bob?" scatters across every shard. Shard by `followee_id` and "who does Alice follow?" scatters. **The standard answer: store it twice.** Maintain two physical tables (or two Cassandra column families):
- `following:{follower_id} → [followee_ids]` sharded by follower
- `followers:{followee_id} → [follower_ids]` sharded by followee

Both are written on every follow (in a transaction, or via an outbox/CDC pipeline for eventual consistency). **You trade storage and write complexity for O(1) reads in both directions** — the same trade you made with fanout. Denormalization is a theme, not an accident.

**`likes`** and **`comments`**
```
likes:    (photo_id, user_id, created_at)   PK (photo_id, user_id)  -- idempotent by construction
comments: (id, photo_id, user_id, text, created_at)  INDEX (photo_id, id DESC)
```
Like counts are **not** a `COUNT(*)` at read time and **not** a naive `UPDATE ... SET likes = likes + 1` — see deep dive D.

**DB choice summary:**

| Data | Store | Why |
|---|---|---|
| Photo bytes | **S3 / blob storage** | Petabytes, immutable, cheap, CDN-friendly |
| Users, photos metadata | **Postgres (sharded)** | Small, structured, needs indexes |
| Follow graph | **Cassandra / wide-column** (or sharded SQL, stored twice) | Huge, write-heavy, needs both directions |
| Feed lists | **Redis** | The whole point is O(1) reads |
| Like counts | **Redis counters + async aggregation** | Contention (deep dive D) |

---

### 6. High-level architecture

```
                          ┌──────────────┐
                          │    Client    │
                          │  (iOS/web)   │
                          └───┬──────┬───┘
              image bytes     │      │  API calls (JSON)
              ┌───────────────┘      └────────────────┐
              ▼                                       ▼
      ┌───────────────┐                        ┌─────────────┐
      │      CDN      │                        │Load Balancer│
      │  (edge cache, │                        └──────┬──────┘
      │   ~95% hit)   │                               │
      └───────┬───────┘                        ┌──────▼───────┐
              │ miss                           │ API Gateway  │  auth, rate limit
              ▼                                └──────┬───────┘
      ┌───────────────┐                               │
      │ Blob Storage  │◀──── pre-signed PUT ──────┐   │
      │   (S3/GCS)    │      (direct from client) │   │
      └───────┬───────┘                           │   │
              │ ObjectCreated event               │   │
              ▼                       ┌───────────┴───┴────────────────┐
      ┌───────────────┐               │                                │
      │  Media Queue  │        ┌──────▼─────┐  ┌──────────┐  ┌─────────▼──────┐
      └───────┬───────┘        │   Post     │  │   Feed   │  │  User / Graph  │
              ▼                │  Service   │  │ Service  │  │    Service     │
      ┌───────────────┐        └──────┬─────┘  └────┬─────┘  └────────┬───────┘
      │ Media Worker  │               │             │                 │
      │ (resize:      │               │  enqueue    │ LRANGE          │
      │  thumb/feed/  │               │  fanout     │                 │
      │  full)        │               ▼             ▼                 ▼
      └───────┬───────┘        ┌────────────┐ ┌───────────┐  ┌────────────────┐
              │                │  Fanout    │ │   Redis   │  │  Redis: user +  │
              │ status=READY   │   Queue    │ │   Feed    │  │  graph cache    │
              ▼                │  (Kafka)   │ │   Cache   │  └────────┬───────┘
      ┌────────────────┐       └──────┬─────┘ └───────────┘           │
      │  Metadata DB   │              │             ▲                 │
      │  (photos,      │◀─────────────┼─────────────┘                 │
      │   sharded)     │              ▼         LPUSH/LTRIM           ▼
      └────────────────┘       ┌──────────────┐              ┌────────────────┐
              ▲                │Fanout Worker │─────────────▶│  Follow Graph  │
              └────────────────┤  (pool of N) │  "who follows│  DB (Cassandra)│
                               └──────────────┘   this user?"└────────────────┘
```

**Reading the diagram:**
- **Two paths leave the client**, and this is the most important structural fact on the page. Image bytes go to the CDN (reads) and directly to blob storage (writes). JSON API calls go to the load balancer. **Your API servers never touch an image byte.**
- **The CDN sits in front of everything visual.** ~95% of your terabits of image egress terminate at the edge.
- **Services split by responsibility:** Post (create/read photos), Feed (assemble the feed), User/Graph (follows, profiles), Media (async resizing).
- **The Fanout Queue decouples posting from feed delivery.** The user's POST returns as soon as the row is committed and the job is enqueued — typically <100ms. The fan-out happens *behind* them. This is what "eventual consistency is acceptable" bought us: the ability to make the expensive part asynchronous.
- **Redis holds the feed cache** — the single most important box in the diagram for hitting <200ms.
- **Fanout Workers are horizontally scalable.** More celebrities/traffic? Add workers, add queue partitions.

---

### 7. Deep dives

#### (a) Feed ranking — chronological vs ML-ranked

Everything so far assumed **reverse-chronological**: sort by time, done. That's what the feed list gives you for free, because you `LPUSH` to the head.

Real Instagram (and Facebook, and Twitter) uses a **ranked feed**: an ML model scores each candidate post for *this specific user* — predicted likelihood of like, comment, dwell time — and sorts by score.

**How ranking changes the design** (this is the part interviewers probe):

1. **Your feed list stops being the answer and becomes the *candidate set*.** You fetch ~500 candidates instead of 20, then rank them down to 20.
2. **You now need feature data at read time.** To score `(alice, photo_123)` you need: Alice's affinity with the author, her recent engagement history, the post's early engagement velocity, its age, its media type. That means a **feature store** (a low-latency key-value store of precomputed user and post features) sitting in the read path.
3. **The latency budget gets tight.** Fetch candidates (5ms) → fetch features (10ms) → score 500 posts through a model (20-50ms) → sort → hydrate (10ms). You've gone from ~15ms to ~80ms. Still under 200ms, but the headroom is gone.
4. **You need an inference tier** — either a lightweight model running in-process, or a separate model-serving service.

```
  feed:alice (500 candidate IDs)
        │
        ▼
  ┌──────────────┐   ┌───────────────┐
  │ Feature Store│──▶│ Ranking Model │──▶ scores ──▶ sort ──▶ top 20 ──▶ hydrate
  │ (user + post │   │  (inference)  │
  │  features)   │   └───────────────┘
  └──────────────┘
```

**The trade-off:** ranking measurably increases engagement, but it costs you a feature store, a model-serving tier, ~5× the read latency, and complete loss of "why am I seeing this?" explainability. **Start chronological in the interview; offer ranking as the natural v2.** Saying "we'd add a ranking layer that treats the precomputed feed as a candidate generator" is exactly the right level of answer.

#### (b) Caching strategy

The feed cache is the most important cache in the system, but it isn't the only one.

| Cache | Key | Value | Hit rate | Notes |
|---|---|---|---|---|
| **Feed cache** | `feed:{userId}` | list of ~500 post IDs | ~95% (active users) | **The critical one.** Miss = expensive rebuild. |
| **Post cache** | `post:{photoId}` | post metadata JSON | ~99% | Hydration. One `MGET` per feed load. Small. |
| **Celebrity posts** | `celeb_posts:{id}` | last 50 post IDs | ~100% | Hottest keys in the system. |
| **User cache** | `user:{userId}` | username, avatar, follower_count | ~99% | Every post in the feed needs its author's name+avatar. Without this, one feed load = 20 user lookups. |
| **Graph cache** | `following:{userId}` | set of followee IDs (or just the celebrity subset) | ~95% | Read path needs "which celebrities does Alice follow?" |

**Key insight: a feed load should be 3-4 cache round trips, not 40 DB queries.** LRANGE the feed → MGET the posts → MGET the authors → done. Everything else is a bug.

**Eviction:** LRU with a long TTL. Feed lists for inactive users age out naturally and get lazily rebuilt (see 4e). **Invalidation:** posts are immutable, so `post:{id}` basically never needs invalidating — another gift from immutability.

#### (c) Sharding the photos table and the follow graph

**Photos:** shard by `user_id` (hash it). Why not by `photo_id`? Because your dominant query is "give me this user's recent posts" (needed by the pull path and by every profile page view). Sharding by `user_id` makes that a **single-shard query**. Sharding by `photo_id` would scatter one user's posts across every shard.

The cost: a **hot shard** if a mega-celebrity lands on it. Mitigations: a separate "celebrity tier" of beefier shards; heavy caching in front (celebrity posts are the hottest cached objects anyway, so the DB barely sees the traffic).

Use **time-sortable IDs (Snowflake)** for `photo_id` so that `ORDER BY id DESC` == `ORDER BY created_at DESC` — no separate timestamp index needed, and merging results from multiple shards is a clean k-way merge on the ID.

**Follow graph:** as covered in the data model — **store it twice**, sharded two different ways, because you need both directions and no single shard key gives you both. This is the canonical example of "denormalize for read patterns."

#### (d) Counting likes at scale

Here's the trap. A viral photo gets 50,000 likes/second. The naive implementation:

```sql
-- DON'T DO THIS
UPDATE photos SET like_count = like_count + 1 WHERE id = 123;
```

**Why it breaks:** every one of those 50,000 concurrent transactions wants a **write lock on the same single row.** They queue up. Lock contention goes vertical, transactions time out, and this one hot row **stalls the entire shard** — including everybody else's unrelated writes. One viral photo takes down a database. This is **row contention**, and it's a genuinely common production incident.

**Fix 1 — Redis counters (simplest, usually enough):**
```js
await redis.incr(`likes:${photoId}`);   // ~1M ops/sec, no locks, atomic
```
Reads take the count straight from Redis. An async job flushes it to the DB every ~30s for durability. Redis is single-threaded and atomic — no contention. Downside: a Redis failure loses the un-flushed delta (you'd recount from the `likes` table).

**Fix 2 — sharded counters (the classic answer):** split the counter into N sub-counters so the writes spread across N rows instead of hammering one.
```js
// Write: pick a random shard, contention drops by N×
const shard = Math.floor(Math.random() * 20);
await db.query(
  'UPDATE like_counters SET count = count + 1 WHERE photo_id = $1 AND shard = $2',
  [photoId, shard]
);

// Read: sum the shards (20 cheap reads, easily cached)
const { rows } = await db.query(
  'SELECT SUM(count) AS total FROM like_counters WHERE photo_id = $1', [photoId]
);
```
20 shards → 20× less contention per row. 100 shards → 100×.

**Fix 3 — async aggregation pipeline (what the giants do):** don't update any counter synchronously. Write the like event to a stream (Kafka), and let a stream processor aggregate counts in windows and periodically write the total.
```
like event ──▶ Kafka ──▶ aggregator (count in 5s windows) ──▶ Redis / DB
```
Writes become append-only and infinitely scalable. The count is a few seconds stale — **and nobody cares.** "1.2M likes" vs "1,200,047 likes" is a distinction without a difference to a human.

> **The general lesson: exact counts at high write volume are expensive and rarely worth it.** Approximate, aggregate asynchronously, and round the display. Notice that this is *the same trade you made for the feed*: give up strict consistency you don't need, buy scale you do.

---

### 8. Bottlenecks, failure modes, and how to scale further

| Bottleneck / failure | Symptom | Mitigation |
|---|---|---|
| **Celebrity fanout storm** | Fanout queue backs up minutes deep; everyone's feed lags | Hybrid model (4d). Plus: a **separate queue/worker pool for large-fanout jobs** so they can't starve normal posts. |
| **Redis feed cache down** | Every feed load misses → falls back to 500-way scatter-gather → DB melts | Replicated Redis cluster; **circuit breaker** on the fallback path (serve a degraded/stale feed rather than DDoS your own DB); progressive rebuild, not a thundering-herd rebuild. |
| **Hot shard** (celebrity's posts) | One DB shard at 100% CPU, others idle | Cache aggressively (celeb posts are ~100% cache-hit anyway); dedicated celebrity shards; read replicas. |
| **Like-count row contention** | Viral post stalls its whole shard | Sharded/approximate counters, async aggregation (deep dive D). |
| **CDN cold or misconfigured** | Origin egress spikes 20× → S3 bill and latency explode | Long immutable TTLs; multi-CDN; pre-warm edges for celebrity posts (they're predictably hot). |
| **Media worker backlog** | Photos stuck in `PENDING`, never appear | Autoscale workers on queue depth; DLQ for poison images; the `PENDING` status means a half-processed photo never leaks into a feed. |
| **Fanout worker crashes mid-job** | Some followers got the post, others didn't | **Idempotent fanout** — LPUSH is not idempotent, so dedupe by checking/using a set, or checkpoint follower-batch offsets so a retry resumes rather than restarts. At-least-once + idempotency, always. |
| **Thundering herd on cold feed** | Returning user + missing key = 500-way scatter at peak | Rate-limit lazy rebuilds; single-flight (only one rebuild per user in flight); serve chronological-lite while rebuilding. |

**How to scale further:**
- **Geo-distribute.** Put feed caches and API tiers in multiple regions, close to users. Fanout crosses regions asynchronously.
- **Pagination via cursors,** never `OFFSET` — `OFFSET 10000` makes the DB scan and discard 10,000 rows. Use `WHERE id < :lastSeenId`.
- **Push vs pull for delivery:** WebSockets to push new posts into an open app; polling otherwise.
- **Precompute more aggressively for power users**, less for casual ones. The system doesn't have to treat everyone the same — and the hybrid model is exactly the recognition that it shouldn't.

---

## Visual / Diagram description

The three diagrams you must be able to redraw from memory:

1. **The upload pipeline** (section 3) — client → pre-signed URL → direct-to-blob → event → media worker → variants → metadata row → CDN. The single most important arrow is the one going *around* your API servers.

2. **Fanout on read vs fanout on write, side by side** (sections 4a/4b) — the 500-way scatter fan on one side, the write-time fan on the other. Draw them next to each other; the shape of the picture *is* the argument. One fans out on a path that runs 58,000 times/sec; the other on a path that runs 25 times/sec.

3. **The hybrid feed** (4d) — the write path branching on `isCelebrity`, and the read path merging the precomputed list with a handful of celebrity lists.

Below, the **complete request flow** for the two operations that matter:

```
POST A PHOTO
────────────
 Client ──1──▶ API: request pre-signed URL
        ◀─2──  { uploadUrl, photoId }
        ──3──▶ S3 (direct PUT, ~4MB)
                 └─4─▶ event ─▶ Media Worker ─▶ resize into 3 variants
                                     │
                                     5. UPDATE photos SET status='READY'
                                     6. enqueue { photoId, authorId } → Fanout Queue
                                            │
                              ┌─────────────▼─────────────┐
                              │      Fanout Worker        │
                              │  followerCount > 100k ?   │
                              └───────┬───────────┬───────┘
                                  NO  │           │  YES
                                      ▼           ▼
                          for each follower:   LPUSH celeb_posts:{id}
                            LPUSH feed:{f}     (and stop — no fanout)
                            LTRIM feed:{f} 0 499


READ THE FEED   (the 58,000/sec path — every millisecond here costs money)
─────────────
 Client ──▶ GET /feed
              │
              ├─ 1. LRANGE feed:{me} 0 199          ~1ms   (precomputed)
              ├─ 2. SMEMBERS celebs_i_follow:{me}   ~1ms   (2-10 IDs)
              ├─ 3. LRANGE celeb_posts:{c} 0 20     ~1ms   (per celeb, pipelined)
              ├─ 4. merge by timestamp, take 20     ~0ms   (in-process)
              ├─ 5. MGET post:{id} × 20             ~2ms   (batched hydration)
              ├─ 6. MGET user:{authorId} × 20       ~2ms   (batched)
              └─ 7. build CDN URLs, return JSON
                                                   ─────
                                                   ~10-20ms  ✓ well under 200ms
```

Notice the read path has **no database in it at all** on the happy path. That's not an accident — that's the entire point of the last 400 lines.

---

## Real world examples

### Instagram (Meta)

Instagram's engineering blog has long described a **Django + Postgres + Cassandra + Memcached** stack: Postgres for user and photo metadata (heavily sharded), Cassandra for feed-and-graph-shaped data, Memcached in front of everything, and photos in blob storage behind a CDN. Their public engineering writing has repeatedly emphasized the same two themes this doc hammers: **denormalize for read patterns**, and **push work off the read path**. The feed today is ML-ranked, which means the precomputed list acts as a candidate generator rather than the final answer (deep dive A).

### Twitter / X

The canonical public account of the hybrid model. Twitter's timeline architecture is widely described as **fanout-on-write into per-user timeline caches**, with a **read-time merge for high-follower accounts** — precisely the hybrid in 4d, arrived at because pure fanout-on-write for a celebrity tweet is untenable. Same problem, same solution, different noun. See [103 — Design Twitter](./103-hld-twitter.md) for the same architecture told from the text-post angle (and note how *little* changes when the payload is 280 characters instead of 2 MB — the feed problem is identical; the media pipeline is what differs).

### Facebook News Feed

Facebook's feed is famously **pull-heavy and ranking-dominated** — the ordering problem (which of the thousands of eligible stories do you show?) outweighs the assembly problem. This is the natural end state when your candidate set is huge and engagement is worth more than recency. It's a useful contrast: **the more your feed is ranked rather than chronological, the less a precomputed ordered list buys you**, and the more the read path shifts toward "generate candidates, then score them."

---

## Trade-offs

**Fanout on write vs fanout on read**

| | Fanout on WRITE (push) | Fanout on READ (pull) |
|---|---|---|
| Read cost | **O(1)** — one LRANGE | **O(followees)** — scatter-gather + merge |
| Write cost | **O(followers)** — can be catastrophic | **O(1)** — one insert |
| Feed latency | ~10ms | 100ms-1s+, hostage to slowest shard |
| Post visibility | Seconds of lag | Instant |
| Celebrity behaviour | **Breaks badly** | Perfectly fine |
| Wasted work | Yes (inactive users) | None |
| Memory cost | High (a list per user) | ~Zero |
| Complexity | Workers, queues, cache invalidation | Simple |
| **Right when** | **Reads ≫ writes (the normal case)** | Writes ≫ reads, or tiny user base |

**Other trade-offs made in this design**

| Decision | You gain | You give up |
|---|---|---|
| Eventual consistency on the feed | Async fanout, queues, huge scale | Instant visibility (nobody notices) |
| Direct-to-blob upload | API servers stay tiny and stateless | Pre-signed URL complexity, client-side retry logic |
| Multiple resized variants | ~4× bandwidth savings (millions of dollars) | Storage +50%, async processing latency |
| Store follow graph twice | O(1) reads in both directions | 2× storage, dual-write consistency problem |
| Approximate like counts | No row contention; a viral post can't stall a shard | Exact counts (nobody cares) |
| Cap feeds at 500 entries | Bounded, predictable memory | Deep scroll falls back to the slow path |
| Hybrid fanout | Both fast reads AND survivable celebrity posts | Two code paths, a threshold to tune |

**The sweet spot:** **Fanout-on-write by default, fanout-on-read for celebrities, lazy rebuild for inactive users.** That one sentence is the answer to this entire interview question.

**Rule of thumb:** Look at your read:write ratio *before* you design anything. If reads dominate by 100:1 or more, **precompute the read.** Move every joule of work you can from the read path to the write path — and then find the one pathological case where that inverts (here: the celebrity), and special-case it.

---

## Common interview questions on this topic

### Q1: "Would you use fanout on write or fanout on read? Why?"

**Hint:** Never answer with just one. Say: "Fanout on write, **because reads outnumber writes by roughly 2,000:1 at the feed layer** — 58,000 feed reads/sec vs 25 posts/sec. It's worth making the rare operation expensive to make the common one O(1)." Then *immediately volunteer the celebrity problem yourself* and present the hybrid. Leading them there beats being led.

### Q2: "A user with 100 million followers posts. Walk me through what happens."

**Hint:** Do the arithmetic on the board: 100M LPUSHes at ~500k ops/sec = **~200 seconds for one post**, monopolizing the fanout tier and delaying everyone else's feeds. Then the fix: **don't fan out for them at all.** Store the post, LPUSH it to a per-celebrity recent-posts list, and merge it in at read time. Cost: one extra small LRANGE per feed load. Close with: "This works because there are very few celebrities, and each user follows only a handful."

### Q3: "How do you make a feed load in under 200ms?"

**Hint:** Enumerate the budget. LRANGE the precomputed feed (~1ms) + a few celebrity LRANGEs (~1ms) + in-memory merge (~0ms) + one batched MGET to hydrate posts (~2ms) + one batched MGET for authors (~2ms) = **~10-20ms, no database on the happy path.** The key architectural claim: the feed is *already assembled* before you ask for it. Then mention the CDN — the *images* are the slow part for the user, and those come from the edge, not from you.

### Q4: "A photo goes viral and gets 50,000 likes per second. What breaks?"

**Hint:** `UPDATE photos SET like_count = like_count + 1 WHERE id = 123` — 50,000 transactions contending for a **write lock on one row**. Lock queue explodes, timeouts cascade, **the hot row stalls the whole shard**. Fixes, in increasing sophistication: Redis `INCR` (atomic, lock-free, ~1M ops/sec, flush to DB periodically); **sharded counters** (N sub-rows, sum on read, N× less contention); async aggregation via Kafka. Land the punchline: **exact counts aren't worth a shard outage — "1.2M likes" is the same to a human as "1,200,047".**

### Q5: "How do you handle a user who hasn't opened the app in a year, then comes back?"

**Hint:** You **skipped them during fanout** (`last_active_at` check), so `feed:{userId}` doesn't exist. On the cache miss, the Feed Service **lazily rebuilds** via fanout-on-read — the scatter-gather across their followees — then caches it. One slow load for them; you saved millions of pointless LPUSHes for months. Add: single-flight the rebuild so 10 concurrent requests don't trigger 10 scatter-gathers.

---

## Practice exercise

### Build the fanout worker and the feed reader

Write a working Node.js prototype of the hybrid feed. Use real Redis (`docker run -p 6379:6379 redis`) and an in-memory object as your "follow graph DB."

**Part 1 — the fanout worker.** Implement:

```js
// fanout-worker.js
import Redis from 'ioredis';
const redis = new Redis();

const CELEBRITY_THRESHOLD = 100_000;
const FEED_MAX = 500;
const INACTIVE_DAYS = 30;
const BATCH = 1000; // never load 500k followers into memory at once

export async function handleNewPost({ photoId, authorId, createdAt }) {
  const author = await db.getUser(authorId);

  // ── Celebrity path: DO NOT fan out. One write, done. ──
  if (author.followerCount > CELEBRITY_THRESHOLD) {
    await redis.multi()
      .lpush(`celeb_posts:${authorId}`, photoId)
      .ltrim(`celeb_posts:${authorId}`, 0, 49)
      .exec();
    return { strategy: 'pull', fannedOutTo: 0 };
  }

  // ── Normal path: push into every follower's precomputed feed. ──
  let cursor = null, count = 0;
  do {
    const { followers, next } = await db.getFollowers(authorId, cursor, BATCH);
    const active = followers.filter(f => isActive(f.lastActiveAt, INACTIVE_DAYS));

    const pipe = redis.pipeline();               // batch → 1 round trip, not 1000
    for (const f of active) {
      pipe.lpush(`feed:${f.id}`, photoId);
      pipe.ltrim(`feed:${f.id}`, 0, FEED_MAX - 1); // bound memory: ~4KB/user
    }
    await pipe.exec();

    count += active.length;
    cursor = next;
  } while (cursor);

  return { strategy: 'push', fannedOutTo: count };
}

function isActive(lastActiveAt, days) {
  return Date.now() - new Date(lastActiveAt).getTime() < days * 86_400_000;
}
```

**Part 2 — the feed reader.** Implement `getFeed(userId, limit = 20)` so that it:
1. `LRANGE feed:{userId} 0 199` for the precomputed portion.
2. Fetches the celebrities this user follows and pipelines an `LRANGE celeb_posts:{c} 0 20` for each.
3. Merges both sources by `createdAt` and takes the top `limit`.
4. Hydrates with **one batched `MGET`** for posts and **one** for authors. (If you write a loop of 20 individual gets, you have failed the exercise — that's an N+1 in the hottest path in the system.)
5. On a **cache miss** (`feed:{userId}` empty), falls back to fanout-on-read to rebuild, then caches the result.

**Part 3 — prove it.** Seed 10,000 normal users (avg 200 followers) and 1 celebrity with 1,000,000 followers. Then:
- Time `handleNewPost` for a normal user. (Expect: tens of milliseconds.)
- Time it for the celebrity **with the celebrity check disabled** — measure the real cost of the naive fanout. Extrapolate to 100M followers.
- Time it for the celebrity **with the check enabled.** (Expect: single-digit milliseconds.)
- Time `getFeed` for a user who follows 500 people including the celebrity.

**Deliverable:** the two functions, plus a table of your four measured timings and one paragraph explaining *why the hybrid wins* using your own numbers. If you can't articulate the win from your own measurements, you can't defend it in an interview.

---

## Quick reference cheat sheet

- **Read:write ratio is the design.** ~58,000 feed reads/sec vs ~25 posts/sec. When reads dominate by orders of magnitude, **precompute the read**.
- **"Eventual consistency is acceptable"** is the sentence that unlocks everything — it's what makes async fanout legal. Say it out loud.
- **Fanout on write (push):** precompute each user's feed into a Redis list at post time. Read = one `LRANGE`. **The right default.**
- **Fanout on read (pull):** scatter-gather across followees at read time. Simple, instant, but O(followees) on your hottest path. Wrong default.
- **The celebrity problem:** 100M followers × 1 post = 100M LPUSHes ≈ minutes of work, and it starves everyone else's fanout.
- **The hybrid (the answer):** push for normal users, **pull for celebrities**, merge at read time. Few celebrities exist; each user follows only a handful.
- **Inactive users:** don't fan out to them. Lazily rebuild their feed on return.
- **Cap the feed** at ~500 post IDs (`LTRIM`). Store **IDs, not bodies** — hydrate in one batched `MGET`.
- **Never upload through your API server.** Pre-signed URL → client uploads **directly** to blob storage.
- **Always resize into variants.** Sending a 4 MB original for a 400px thumbnail wastes ~95% of ~10 PB/day of egress.
- **Images are immutable** → CDN caching is trivially safe → one-year TTL, no invalidation. Free win.
- **The follow graph needs indexes BOTH ways:** "who I follow" (reads) and **"who follows me" (fanout)**. Store it twice if you must.
- **Never `UPDATE ... SET count = count + 1`** on a viral row — that's row contention, and one hot photo can stall a shard. Use Redis `INCR`, sharded counters, or async aggregation.
- **A feed load should touch zero databases.** 3-4 cache round trips, ~15ms. If there's a DB in your happy path, redesign.

---

## Connected topics

| Direction | Topic |
|---|---|
| **Previous** | [93 — The System Design Interview Framework](./93-hld-approach-framework.md) — the 5-step method this doc executes end to end |
| **Next** | [103 — Design Twitter](./103-hld-twitter.md) — the same fanout problem with a text payload; compare the two and notice how little the feed logic changes |
| **Related** | [78 — Blob Storage](./78-blob-storage.md) — pre-signed URLs and why photo bytes never touch your API servers |
| **Related** | [60 — CDN](./60-cdn.md) — how ~1 Tbps of image egress is served from the edge instead of from you |
| **Related** | [59 — Caching in Depth](./59-caching-in-depth.md) — the feed cache, the post cache, and lazy rebuild on miss |
| **Related** | [128 — LLD: Social Media Feed](./128-lld-social-media-feed.md) — the class-level design of the feed service you just architected |
