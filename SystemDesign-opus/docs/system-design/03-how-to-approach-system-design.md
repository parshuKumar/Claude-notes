# 03 — How to Approach a System Design Problem
## Category: HLD Fundamentals

---

## What is this?

This is your **recipe**. When someone says "Design Instagram" and stares at you, a blank whiteboard is terrifying — unless you have a fixed sequence of steps you always follow. That sequence is what this doc teaches.

Think of a doctor in an emergency room. They don't panic and start randomly treating things. They run a protocol: check airway, check breathing, check circulation. Same order, every patient, every time. The protocol is what turns chaos into competence. System design has exactly the same kind of protocol, and once you internalize it, "Design Instagram" stops being scary and becomes just *step 1, step 2, step 3...*

---

## Why does it matter?

**Without a framework, here's what happens in a real interview:** The interviewer says "Design Twitter." You immediately start drawing boxes. Ten minutes in, you're deep in the weeds of how to shard the database — but you never asked whether the system needs 1,000 users or 500 million. You never asked whether direct messages are in scope. You built the wrong system, beautifully.

The interviewer is not grading your final diagram. They're grading **how you got there.** A structured approach signals: *this person will not flail when handed an ambiguous project at work.*

**The interview angle:** System design interviews are deliberately vague. The vagueness is the test. A candidate who asks "how many daily active users?" before drawing anything has already scored points. A candidate who silently draws for 15 minutes has already lost.

**The real-work angle:** Your manager says "we need a notification service." That's the same ambiguous prompt. The same framework applies — you write a design doc that goes: requirements → constraints → estimates → API → data model → architecture → risks. The only difference is you have days instead of 45 minutes.

---

## The core idea — explained simply

### The Architect Meeting a Client Analogy

Imagine you're an architect. A client walks in and says: **"Build me a building."**

A bad architect immediately starts sketching a skyscraper.

A good architect asks questions first:

| The architect asks... | The system designer asks... |
|----------------------|----------------------------|
| "What's the building FOR? Homes? Offices? A hospital?" | "What features must the system support? (Functional requirements)" |
| "How many people will use it?" | "How many daily active users?" |
| "What's your budget? What's the deadline?" | "What are the constraints? Cost? Latency budget?" |
| "Is it in an earthquake zone? Will it flood?" | "What happens when a server dies? A whole region?" |
| "How many floors, how many rooms, how big?" | "Let me estimate: QPS, storage, bandwidth." |
| "Here's the floor plan." | "Here's the high-level architecture diagram." |
| "The elevators are the hard part — let's discuss." | "Let's deep-dive the feed generation service." |
| "Concrete is cheaper but slower to build than steel." | "SQL gives consistency; NoSQL gives horizontal scale." |

Notice: the architect **spends the first chunk of time NOT designing.** They spend it understanding. Then they design once, correctly, instead of designing three times wrongly.

That's the entire lesson. **Understand → estimate → design → refine.** Never skip forward.

---

## Key concepts inside this topic

### 1. Step 1 — Clarify Requirements (the questions to actually ask)

Never start drawing. Start asking. Spend **3–5 minutes** here.

You are gathering two things (covered in depth in [04 — Functional vs Non-Functional Requirements](./04-functional-vs-non-functional-requirements.md)):
- **Functional:** what the system DOES
- **Non-functional:** how WELL it does it

**The exact questions to ask — memorize these:**

**Scope questions (functional):**
```
"Who are the users? (consumers, businesses, internal teams?)"
"What are the 3 most important features? Can I focus on those?"
"Is <feature X> in scope? (e.g. for Twitter: are DMs in scope? Search? Ads?)"
"Do we need to support mobile, web, or both?"
"Is there a follow/friend relationship, or is everything public?"
```

**Scale questions (non-functional):**
```
"How many daily active users (DAU) should I design for?"
"Is this read-heavy or write-heavy? What's the rough read:write ratio?"
"What's an acceptable latency for the main operation? (e.g. <200ms)"
"How available must this be? Can we tolerate a few seconds of staleness?"
"How long do we retain data? Forever, or 30 days?"
```

**The scoping-down move (this is the pro move):**

> "Twitter is huge. In 45 minutes I can't design ads, search, trends, and DMs. I'd like to focus on **posting a tweet** and **generating the home timeline**, since those carry the hardest scale problems. Does that work for you?"

This does three things: shows judgment, gives you a manageable scope, and gets the interviewer to commit so they can't ding you later for "not covering search."

### 2. Step 2 — Define Scope and Constraints Explicitly

Write your assumptions **on the board.** Out loud. This is the contract for the rest of the interview.

```
IN SCOPE:
  - Post a tweet (text, max 280 chars)
  - View home timeline (tweets from people you follow)
  - Follow / unfollow a user

OUT OF SCOPE (stated, not forgotten):
  - Search, trending topics, ads, DMs, video

ASSUMPTIONS:
  - 200M DAU
  - Read-heavy: ~100 reads per 1 write
  - Timeline can be eventually consistent (a few seconds of staleness is fine)
  - Tweets are immutable once posted (no edit)
```

Writing "OUT OF SCOPE" explicitly is a senior signal. It says: *I know these exist, I'm choosing not to spend time on them.* That reads very differently from forgetting them.

### 3. Step 3 — Capacity Estimation (do the math out loud)

This is where you convert "200M users" into concrete numbers that **justify your architecture**. Full treatment is in [05 — Capacity Estimation](./05-capacity-estimation.md); here's the shape of it.

```
DAU = 200M
Each user posts 2 tweets/day     → 400M writes/day
Each user reads 100 tweets/day   → 20B  reads/day

Write QPS = 400M / 86,400s   ≈ 4,600 writes/sec
Read  QPS = 20B  / 86,400s   ≈ 231,000 reads/sec
Peak = 2x average            → ~9k writes/sec, ~460k reads/sec

Storage: 400M tweets/day × 300 bytes ≈ 120 GB/day
         → ~44 TB/year → ~220 TB over 5 years
```

**Why this matters:** these numbers make your later decisions *defensible*.

- 231k read QPS → you **must** have a cache. Not "a cache would be nice." You must.
- 100:1 read:write → you should **precompute** the timeline on write (fanout-on-write), because reads dominate.
- 220 TB → a single database machine will not hold this. You **must** shard.

That's the point of estimation. It's not arithmetic for its own sake — it's the evidence that forces your design.

### 4. Step 4 — API Design

Define the contract between client and server before you design internals. Keep it small — 3 to 5 endpoints.

```javascript
// Keep endpoints few and obvious. Show auth, pagination, and idempotency awareness.

// POST /v1/tweets
// Auth: Bearer token → the server derives userId from the token, NOT from the body.
// (If userId came from the body, anyone could tweet as anyone.)
{
  text: "Hello world",           // string, max 280 chars
  idempotencyKey: "uuid-1234"    // client retries safely; server dedupes
}
// → 201 { tweetId, createdAt }

// GET /v1/timeline?cursor=<opaque>&limit=20
// Cursor-based, NOT offset-based. Offset pagination breaks when new tweets are
// inserted at the head — you'd see duplicates. A cursor is stable.
// → 200 { tweets: [...], nextCursor: "..." }

// POST /v1/users/:userId/follow
// → 204 No Content
```

Two details that consistently impress interviewers:

1. **`idempotencyKey`** — the client's phone loses signal mid-request and retries. Without this, the user posts the same tweet twice. The server stores the key and returns the original result on a retry.
2. **Cursor pagination over offset pagination** — for any feed that changes at the head.

### 5. Step 5 — Data Model

Decide *what* you store and *where*. Say the DB choice **and the reason**.

```
Table: users            (PostgreSQL — relational, low volume, needs consistency)
  user_id     BIGINT   PK
  handle      VARCHAR  UNIQUE
  created_at  TIMESTAMP

Table: follows          (PostgreSQL — a graph edge; heavy reads, small rows)
  follower_id  BIGINT   ┐ composite PK
  followee_id  BIGINT   ┘
  created_at   TIMESTAMP
  INDEX on (followee_id)   -- "who follows me?" for fanout

Table: tweets           (Cassandra — write-heavy, append-only, time-ordered)
  tweet_id    BIGINT   PK   (Snowflake ID: time-sortable, globally unique)
  user_id     BIGINT
  text        TEXT
  created_at  TIMESTAMP
  PARTITION KEY: user_id, CLUSTERING KEY: created_at DESC

Cache: timeline:<userId>  (Redis LIST — precomputed home timeline, ~800 tweet IDs)
```

**Say the WHY out loud.** "Tweets go in Cassandra because it's an append-heavy, immutable, time-ordered workload with no joins — exactly what a wide-column store is built for. Users and follows go in Postgres because the volume is small and I want transactional integrity."

Choosing a database without a reason is the single most common way to lose points here.

### 6. Step 6 — High-Level Architecture

Now — finally — you draw. Start with the simplest correct picture: client → load balancer → service → database. Then add components **only when your estimates demand them.**

Talk through **one write path** and **one read path** end to end. Don't just point at boxes; trace a request.

> "A user posts a tweet. It hits the load balancer, then the API gateway which authenticates and rate-limits. The Tweet Service writes to Cassandra, then publishes a `tweet.created` event to Kafka. The Fanout Worker consumes that event, looks up the author's followers, and pushes the tweet ID into each follower's Redis timeline list. Now, when a follower opens the app, the Timeline Service just reads a precomputed Redis list — that's a single cache hit, sub-millisecond, no database join at read time."

That paragraph is worth more than a beautiful diagram you never explain.

### 7. Step 7 — Deep Dives (where senior candidates win)

The interviewer will now pick the hard part and probe. **Anticipate it.** Every system has one or two genuinely hard sub-problems:

| System | The hard part |
|--------|--------------|
| Twitter | The celebrity problem — fanning out to 100M followers |
| Uber | Geospatial indexing — find nearby drivers fast |
| Ticketmaster | Concurrency — two people booking the last seat |
| YouTube | Video transcoding pipeline and adaptive bitrate |
| Chat | Delivering to a user who is offline right now |

A strong move is to **volunteer the deep dive yourself**:

> "The interesting failure in fanout-on-write is the celebrity problem. If Taylor Swift tweets to 100M followers, we'd do 100M Redis writes for one tweet. So I'd use a **hybrid**: normal users get fanout-on-write, but users above ~100k followers are marked `isCelebrity` and skipped at fanout time. At read time, the Timeline Service merges the precomputed list with a live fetch of the celebrities you follow. You pay a small read cost for a huge write saving."

### 8. Step 8 — Bottlenecks, Failure Modes, and Trade-offs

Close the loop. Ask yourself, aloud: **"What breaks first as we grow 10x?"**

```
Bottleneck            → Mitigation
─────────────────────────────────────────────────────────────
Redis memory limit    → Shard Redis by userId; cap timelines at 800 entries;
                        only cache active users (logged in last 30 days)
Fanout worker lag     → Scale Kafka partitions and worker count; monitor lag
Cassandra hot shard   → A celebrity's partition gets hammered; add a shard suffix
Single region down    → Multi-region with async replication; accept staleness
Thundering herd       → A hot key expires and 10k requests hit the DB at once;
                        use a lock so only one request repopulates the cache
```

Then name your trade-offs honestly: *"I chose eventual consistency for the timeline. The cost is that a follower might not see your tweet for 2–3 seconds. For a social feed that's completely fine. For a bank balance it would not be."*

---

## Visual / Diagram description

### Diagram 1: The 45-Minute Interview Flow

```
   0 min                                                            45 min
   │                                                                  │
   ▼                                                                  ▼
┌──────────┬──────────┬──────────┬───────────────────┬──────────┬────────┐
│  CLARIFY │ ESTIMATE │   API +  │   HIGH-LEVEL      │   DEEP   │ WRAP   │
│   REQS   │          │   DATA   │   ARCHITECTURE    │   DIVE   │  UP    │
│          │          │  MODEL   │                   │          │        │
│  5 min   │  5 min   │  5 min   │      10 min       │  15 min  │  5 min │
└──────────┴──────────┴──────────┴───────────────────┴──────────┴────────┘
     │           │          │              │                │          │
     │           │          │              │                │          └─▶ Bottlenecks,
     │           │          │              │                │              trade-offs,
     │           │          │              │                │              "what I'd do next"
     │           │          │              │                └─▶ The 1-2 HARD sub-problems
     │           │          │              └─▶ Boxes + arrows + trace ONE write, ONE read
     │           │          └─▶ 3-5 endpoints; tables + DB choice WITH REASONS
     │           └─▶ DAU → QPS → storage → bandwidth. Numbers justify the design.
     └─▶ Ask questions. Write scope on the board. Get agreement.
```

**How to read it:** the first 15 minutes contain **zero architecture.** That feels wrong the first time you do it and it is exactly right. If you're 20 minutes in and still haven't drawn a box, you are on schedule. If you drew boxes at minute 2, you are already lost.

### Diagram 2: The Architecture You Build (and the order you build it in)

```
                                 ┌─────────────┐
   STEP A: start here            │   Client    │
   (always draw this first)      │ (web/mobile)│
                                 └──────┬──────┘
                                        │ HTTPS
                                        ▼
                                 ┌─────────────┐
   STEP B: add when you have     │    Load     │
   more than one server          │  Balancer   │
                                 └──────┬──────┘
                                        ▼
                                 ┌─────────────────────────┐
   STEP C: auth + rate limiting  │      API Gateway        │
   belong in ONE place           │ (authN, rate limit,     │
                                 │  routing)               │
                                 └───┬─────────────────┬───┘
                                     │                 │
                    ┌────────────────▼───┐      ┌──────▼──────────┐
   STEP D: split    │   Write Service    │      │  Read Service   │
   read from write  │  (Tweet Service)   │      │(Timeline Service)│
   when the ratio   └────┬──────────┬────┘      └────────┬────────┘
   is lopsided           │          │                    │
                         │          │ publish event      │ cache hit (>95%)
                         ▼          ▼                    ▼
                  ┌───────────┐ ┌────────┐        ┌────────────┐
   STEP E: the    │ Cassandra │ │ Kafka  │        │   Redis    │
   storage tier   │  (tweets) │ │ (queue)│        │(timelines) │
                  └───────────┘ └───┬────┘        └─────▲──────┘
                                    │                   │
                                    ▼                   │
                             ┌──────────────┐           │
   STEP F: async work        │Fanout Worker │───────────┘
   goes behind a queue       │(push tweetId │  writes into each
                             │ to followers)│  follower's list
                             └──────────────┘
```

**What the diagram shows:** the read path (right side) never touches Cassandra in the happy case — it's a pure Redis read. The write path (left side) is fast because the expensive work (fanout to millions of followers) is pushed *behind a queue* and done asynchronously. That single decision — **move slow work off the request path** — is the most reusable idea in all of HLD.

---

## Real world examples

### 1. Amazon's "Working Backwards" Process

Before writing a line of code, Amazon teams write a **press release and an FAQ** for a product that doesn't exist yet — describing it as if it already shipped. Only after that document is approved does design begin.

This is Step 1 of our framework, institutionalized. The press release forces you to state *who the user is and what problem is solved* (functional requirements), and the FAQ forces you to confront *scale, cost, and hard questions* (non-functional requirements) before any architecture exists.

### 2. Google's Design Doc Culture

At Google, a significant system starts with a design doc that is circulated and reviewed. The standard structure is essentially the framework in this doc: context and goals, **non-goals** (explicit out-of-scope), the proposed design, alternatives considered, and the trade-offs of each.

The **"non-goals" section** is the real-world version of writing OUT OF SCOPE on the whiteboard. Senior engineers there will tell you that clearly stating what you are *not* building prevents more wasted quarters than any other section of the doc.

### 3. Twitter's Fanout Decision

Twitter's timeline architecture is the textbook case of estimation driving design. Their read:write ratio is enormously read-heavy. Computing a timeline on demand — "fetch everyone I follow, merge their recent tweets, sort" — is far too slow at read time when reads dominate that heavily.

So they **precompute**: when you tweet, the tweet is pushed into your followers' timelines ahead of time. Reads become a simple lookup. And for accounts with enormous follower counts, a hybrid approach avoids the pathological fanout cost. The estimation came first; the architecture fell out of it.

---

## Trade-offs

| Choice | Pro | Con |
|--------|-----|-----|
| **Spend 10 min on requirements** | You design the *right* system; interviewer sees judgment | Less time to draw; you must be fast later |
| **Spend 2 min on requirements** | More whiteboard time | High risk of designing the wrong thing entirely |
| **Detailed capacity estimation** | Every architecture decision is justified by a number | Can rabbit-hole into arithmetic; interviewer gets bored |
| **Skip estimation** | Straight to the fun part | Your "we need sharding" claim is unsupported hand-waving |
| **Draw a simple design first, then scale it** | Easy to follow; shows evolution of thinking | Feels slow if the interviewer wanted scale immediately |
| **Draw the final complex design immediately** | Looks impressive | Interviewer can't follow; you can't justify each piece |

| Interview behaviour | Effect |
|--------------------|--------|
| Silent thinking for 60 seconds | Interviewer has no idea if you're stuck or brilliant. **Avoid.** |
| Thinking out loud constantly | Interviewer can course-correct you early. **Do this.** |
| Waiting to be asked the next question | Reads as passive/junior |
| Driving the conversation yourself | Reads as senior. "Next I'd like to cover the data model — sound good?" |

**Rule of thumb:** Spend the first third of the interview NOT drawing architecture. Requirements + estimation are the cheapest insurance you will ever buy against designing the wrong system. And **narrate everything** — an unspoken thought scores zero.

---

## Common interview questions on this topic

### Q1: "Design Instagram." (and the interviewer says nothing else)
**Hint:** Do not draw. Say: "Let me clarify scope first." Ask about DAU, which features (photo upload + feed + follow, probably not stories/reels/DMs), read vs write ratio, and latency expectations. Then propose a scope-down: "I'll focus on upload and feed generation — that's where the interesting scale problems are. OK?" Get agreement, then estimate, then design.

### Q2: "How do you decide when to add a cache / queue / shard?"
**Hint:** Let the numbers decide, never a reflex. Cache when read QPS is high and data is read far more than written (say >10:1). Queue when work on the request path is slow or spiky and the caller doesn't need the result immediately (fanout, transcoding, emails). Shard when data exceeds what one machine can hold or write QPS exceeds what one primary can absorb. Always say *which estimate* triggered it.

### Q3: "You have 45 minutes. Where do you spend the time?"
**Hint:** ~5 requirements, ~5 estimation, ~5 API + data model, ~10 high-level architecture, ~15 deep dive, ~5 bottlenecks and trade-offs. The deep dive is the biggest block because that's where seniority is actually measured.

### Q4: "What if you don't know how to solve part of the design?"
**Hint:** Say so, then reason from first principles instead of freezing. "I haven't built a geospatial index before. But the core need is 'find drivers within 5km fast.' A naive scan is O(n). I'd want to partition space into cells so I only scan nearby cells — I believe QuadTree or Geohash solve exactly this. Let me reason through the cell-based approach." Honest reasoning beats a confidently wrong answer every single time.

### Q5: "How is this framework different at work vs in an interview?"
**Hint:** The steps are identical; only the clock and the artifact change. At work, requirements-gathering takes days (talk to PM, users, on-call engineers), estimation uses real telemetry instead of guesses, and the output is a written design doc with an "alternatives considered" section. The discipline — understand before you design — is the same.

---

## Practice exercise

### The Dry Run: Design a Pastebin in 45 Minutes

Set a **45-minute timer.** Get paper or a whiteboard. Design a Pastebin (users paste text, get a short link, others open the link and read the text). Force yourself through every step and **stop at each time box, even if unfinished** — the discipline is the point.

Produce these seven artifacts:

1. **(5 min) Requirements** — Write down at least **6 clarifying questions** you would ask. Then answer them yourself with reasonable assumptions. Explicitly list what is OUT of scope.
2. **(5 min) Estimation** — Assume 10M DAU, each user creates 1 paste/day and reads 10 pastes/day, average paste = 10 KB. Calculate: write QPS, read QPS, peak QPS, storage/day, storage over 5 years, and read bandwidth in MB/s.
3. **(3 min) API** — Write 3 endpoints with request and response bodies.
4. **(5 min) Data model** — Define the table(s). Pick a database. **Write one sentence justifying the choice.**
5. **(10 min) Architecture** — Draw the diagram. Then write a short paragraph tracing the CREATE path and another tracing the READ path.
6. **(12 min) Deep dive** — Answer: how do you generate a unique short key without two servers colliding? Give two approaches and pick one, with a reason.
7. **(5 min) Bottlenecks** — Name three things that break at 10x traffic and how you'd fix each.

**Self-check:** Did your storage estimate justify your database choice? Did your read:write ratio justify a cache? If any component in your diagram isn't traceable back to a number from step 2, you added it out of habit rather than reason — mark it and be ready to defend it.

---

## Quick reference cheat sheet

- **Never draw first.** Ask questions for 3–5 minutes before touching the whiteboard.
- **Write scope AND out-of-scope on the board.** Explicitly naming what you're skipping is a senior signal.
- **Scope down proactively:** "I'll focus on the two hardest features — does that work?"
- **Estimate before you architect.** Numbers are the evidence that justifies every box you draw.
- **The four numbers that matter:** DAU, QPS (avg + peak), storage over 5 years, read:write ratio.
- **API before internals.** 3–5 endpoints. Mention idempotency keys and cursor pagination.
- **Name the DB and the reason.** "Cassandra, because writes are append-only, time-ordered, and there are no joins."
- **Start simple, then scale.** Client → LB → service → DB. Add a cache/queue/shard only when a number demands it.
- **Trace one write and one read** end to end, out loud. Diagrams don't explain themselves.
- **Move slow work off the request path** — that's what queues are for. The most reusable idea in HLD.
- **Anticipate the deep dive.** Every system has 1–2 genuinely hard parts. Volunteer to go there.
- **Always end with bottlenecks and trade-offs.** "What breaks at 10x?" and "here's what I gave up."
- **Think out loud, always.** A silent 60 seconds is a wasted 60 seconds. Drive the conversation; don't wait to be asked.
- **Say "I don't know" and then reason from first principles.** Far stronger than confident nonsense.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [02 — HLD vs LLD](./02-hld-vs-lld.md) — the two zoom levels this framework operates at |
| **Next** | [04 — Functional vs Non-Functional Requirements](./04-functional-vs-non-functional-requirements.md) — a deep dive into Step 1 of this framework |
| **Related** | [05 — Capacity Estimation](./05-capacity-estimation.md) — a deep dive into Step 3, with all the arithmetic |
| **Related** | [93 — HLD Approach Framework](./93-hld-approach-framework.md) — the same method, applied specifically to case-study interviews |
