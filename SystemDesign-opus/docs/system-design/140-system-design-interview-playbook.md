# 140 — System Design Interview Playbook — The Complete Guide
## Category: Advanced

---

## What is this?

This is the capstone. It is the single document you re-read the morning of your interview, when your hands are a little cold and your brain is spinning up. Everything in the previous 139 topics was the *raw material* — the databases, the caches, the queues, the patterns, the case studies. This document is the **process** that turns that raw material into a passing interview.

Think of it like a pilot's pre-flight checklist. A pilot who has logged a thousand hours does not fly from memory and vibes. They run the checklist every single time, because the checklist is what keeps a good pilot from making a rookie mistake under pressure. You now have the flight hours. This is your checklist.

---

## Why does it matter?

Here is the thing almost every candidate gets wrong, and it is the most important sentence in this entire course:

> **The interviewer is not grading your answer. They are grading how you get to an answer.**

There is no single "correct" system design. Ask ten senior engineers to design Twitter and you will get ten different diagrams, all defensible. If a "right answer" existed, the interview would be a multiple-choice quiz, not a 45-minute conversation. It is a conversation *on purpose* — because the company is trying to answer one question: **"What will it be like to work with this person on a hard, ambiguous problem?"**

- **Interview angle:** You can know every fact in this course and still fail, if you jump straight to a solution, go silent for ten minutes, over-engineer a system for 100 users, or cannot say *why* you picked Cassandra over Postgres. Conversely, you can forget a detail and still pass, if you reason cleanly from fundamentals out loud. Process beats recall.
- **Real-work angle:** This is not interview theater. The exact skills being tested — scoping an ambiguous project, estimating load before you build, choosing tools deliberately, and explaining trade-offs to teammates — are *the actual job* of a mid-to-senior engineer. The interview is a compressed simulation of a Tuesday.

If you internalize nothing else: **process, communication, and trade-off reasoning are what get scored. The diagram is just the byproduct.**

---

## The core idea — explained simply

### The analogy: you are the tour guide, not the tourist

Imagine a friend visits your city for one day and says, "Show me around." A **bad guide** freezes — "Where do you want to go?" — and waits to be told. A **worse guide** silently marches them to the first landmark they thought of, no plan, no talking. A **great guide** takes charge warmly: "Okay, we have one day. Are you more into food, history, or nightlife? Great — food it is. Here's my plan: three neighborhoods, we'll start close and work outward, and I'll flex if you fall in love with somewhere. Sound good? Let's go." Then they narrate the whole way, read your reactions, and adjust.

The system design interview is *exactly* that. The interviewer is your tourist. You are the guide. The system is the city.

| In the analogy | In the interview |
|---|---|
| "Show me around" (vague) | "Design Instagram" (deliberately vague) |
| Asking food vs. history vs. nightlife | Gathering requirements & scoping |
| Proposing a plan for the day | The time-boxed framework |
| Narrating as you walk | Thinking out loud constantly |
| Reading their reactions, adjusting | Reading the interviewer's hints |
| "We could go here OR here, here's why I'd pick..." | Stating trade-offs |
| Not marching off silently | Not going quiet |
| Not planning a two-week trip for one day | Not over-engineering |

The whole rest of this document is just this analogy made concrete. A great guide has a **framework** (a plan for the day), **communicates** (narrates and reads the room), **avoids known blunders**, and has **prepared** (knows the city cold). Let's take each in turn.

---

## Key concepts inside this topic

### 1. What the interviewer is actually evaluating — the rubric

Interviewers do not score "did the design work." They fill out a rubric, usually with these six dimensions. Know them, and you know exactly what to *demonstrate*.

| Dimension | Junior signal (weak) | Senior signal (strong) |
|---|---|---|
| **Requirements & scoping** | Starts drawing boxes immediately | Asks clarifying questions, narrows scope, names what's *out* of scope |
| **Estimation** | Skips it or guesses wildly | Does back-of-envelope math, states assumptions, uses the numbers to drive design |
| **High-level design** | One giant box, or a random pile of services | Clean client → LB → service → data flow; sensible components |
| **Deep-dive depth** | Hand-waves the hard part | Picks the genuinely hard sub-problem and gets specific |
| **Trade-off reasoning** | "I'd use Cassandra." (no why) | "Cassandra *because* write-heavy + horizontal scale; the cost is no joins and eventual consistency, which is fine here *because*..." |
| **Communication** | Silent, waits to be prompted | Drives, narrates, collaborates, reads hints |

The word "senior" is not about years. It is about *judgment made visible*. A junior candidate presents a solution. A senior candidate presents a **reasoned decision** — options, a pick, and the cost of the pick. Every time you say "I'd do X *because* Y, and the trade-off is Z," you are depositing a senior signal. Do it constantly.

The single highest-leverage habit: **make your thinking auditable.** The interviewer cannot give you credit for reasoning they cannot hear.

---

### 2. The universal time-boxed framework for a 45-minute round

This synthesizes the HLD framework from [93 — HLD Approach Framework](./93-hld-approach-framework.md) and the LLD framework from [111 — LLD Approach Framework](./111-lld-approach-framework.md) into one clock-driven plan. Memorize the phases and their budgets. When you sit down, mentally start a five-part timer.

```
  0────5────10───15───20───25───30───35───40───45  (minutes)
  │REQ │ EST │   HIGH-LEVEL   │   DEEP DIVE   │ WRAP│
  │5-8m│ ~5m │    10-15m      │    10-15m     │ ~5m │
```

| Phase | Budget | What you PRODUCE | The one-liner to say |
|---|---|---|---|
| **1. Requirements & scope** | 5–8 min | A short list of functional features + non-functional targets (scale, latency, consistency), plus what's out of scope | "Before I design, let me make sure I'm solving the right problem." |
| **2. Estimation** | ~5 min | QPS (read & write), storage over ~5 years, bandwidth — with visible math | "Let me size this so the design is grounded in real numbers." |
| **3. High-level design + API + data** | 10–15 min | The big architecture diagram, a few API signatures, the core data model + DB choice | "Here's the end-to-end flow; then I'll go deep where it's interesting." |
| **4. Deep dive** | 10–15 min | 1–2 hard sub-problems solved concretely (the fanout, the hot key, the index, the consistency boundary) | "The hard part here is X — let me focus there." |
| **5. Bottlenecks & wrap** | ~5 min | Failure modes, scaling limits, a crisp summary + "what I'd do next" | "Let me name where this breaks and how I'd evolve it." |

**Critical clock discipline:** the number one time sink is Phase 1. Requirements are important, but if you are still gathering them at minute 15, you will never reach the deep dive — which is where the *most* points live. Cap it. Say "I'll lock scope here and revisit if needed."

**HLD round vs. LLD / machine-coding round — how to tell, and how to adjust:**

- **Signals it's an HLD round:** "Design Instagram / a rate limiter / a chat system," questions about scale, QPS, sharding, CDNs, availability. → Use the framework above. Stay at the component level. Diagrams over code. Lean on [93](./93-hld-approach-framework.md).
- **Signals it's an LLD / machine-coding round:** "Design a parking lot / an elevator / a rate-limiter *class*," questions about classes, interfaces, extensibility, "how would you model...". → Shift to objects, entities, class diagrams, and real code. Lean on [111](./111-lld-approach-framework.md). The clock still applies, but Phase 3 becomes *class design* and Phase 4 becomes *working code + patterns*.
- **When unsure, ask:** "Are we designing this at the architecture level, or do you want me to code the core classes?" That single question shows maturity and saves you from solving the wrong problem.

---

### 3. Communication — the most under-practiced skill

Candidates spend weeks memorizing sharding strategies and zero minutes practicing *talking*. This is backwards. Communication is weighted as heavily as the design itself, and it is the cheapest thing to improve. Four rules:

**Rule 1 — Think out loud, constantly.** Silence is the single most damaging thing you can do, because the interviewer cannot distinguish "thinking deeply" from "totally stuck." They are the same from the outside. Narrate everything: "I'm weighing push vs. pull for the feed... push is fast to read but expensive for celebrities... let me think about the write amplification..." Even your dead ends are *signal* — they show reasoning. A whiteboard full of silence scores zero.

**Rule 2 — Drive the conversation.** Do not wait to be prompted. After each phase, *propose the next step yourself*: "Requirements look good — I'll move to estimation now." "That's the high-level flow; the interesting part is the feed fanout, so let me go deep there." The interviewer should feel like a passenger enjoying the ride, not a driving instructor grabbing the wheel. Driving is the clearest seniority signal there is.

**Rule 3 — State every assumption out loud.** "I'll assume reads dominate writes roughly 100:1 for a social feed — reasonable?" This does three things: it lets the interviewer correct you *early* (cheap) instead of late (expensive), it shows you know these are choices, and it keeps you moving when information is missing. Never wait for perfect information — assume, state it, proceed.

**Rule 4 — Treat it as collaborative. Read the hints.** The interviewer is a teammate, not an examiner. When they ask "are you sure about that?" or "what happens if this node dies?" — **that is not a trap, it is a gift.** They are pointing a flashlight at the exact spot they want you to explore, either because you missed something or because it's where the points are. *Follow the flashlight.* Ignoring a hint is one of the fastest ways to fail; engaging it eagerly is one of the fastest ways to pass.

**Handling "I don't know" gracefully.** You will hit something you don't know. Every candidate does. What separates pass from fail is *how* you handle it. **Bluffing is fatal** — interviewers are domain experts and they can smell a confident fabrication instantly, and once they catch one bluff they distrust everything else you said. Instead, reason from fundamentals:

> "I haven't used Kafka's exactly-once semantics in depth, but let me reason about it. Exactly-once is hard because of the two-generals problem — the producer can't know if a message landed. So I'd expect it's built on idempotency plus transactional offsets... let me think about how that would work."

That answer — honest, then reasoning from first principles — often scores *higher* than a correct memorized fact, because it demonstrates the exact skill the job requires: making progress on things you haven't seen before.

---

### 4. The top mistakes that FAIL candidates

Learn these by name so you catch yourself mid-mistake. Each one is a documented, common failure.

1. **Jumping to a solution before requirements.** The single most common failure. You hear "design Twitter" and start drawing. You have now committed to solving a problem you don't understand. *Fix:* first sentence out of your mouth is a clarifying question.

2. **Over-engineering from minute one.** Microservices + Kafka + five databases + a service mesh — for a system with 100 users. This signals *poor judgment*, the opposite of seniority. Good engineers reach for the *simplest thing that meets the requirements* and scale only when the numbers demand it. Start with a monolith and one database if that's honestly enough; earn your complexity.

3. **Going silent.** Covered above, but it's on the fail list because it's so common. Ten seconds of silence is fine. Sixty is a red flag. Narrate.

4. **Never mentioning a single trade-off.** If you present your design as a series of facts with no alternatives considered, you read as junior — someone who knows *what* but not *why*. Every meaningful choice has a cost; name it.

5. **Not managing the clock.** Spending 25 minutes on requirements and then panic-drawing in the last 5. Or getting so deep in one rabbit hole you never present a coherent whole. Watch the clock; hit every phase.

6. **Unable to justify a technology choice.** "I'd use Cassandra." *Why?* "...it's fast." That is an automatic downgrade. Every tech you name, be ready to defend: what property of *this* problem makes *this* tool right, and what you're giving up. If you can't defend it, don't name it.

7. **Ignoring the interviewer's hints.** They said "hmm, what about a hot key?" three times and you kept talking about your LB. You just walked past three free points. Listen.

8. **Hand-waving the hard part.** Every problem has a genuinely difficult core — the feed fanout, the geospatial index, the exactly-once delivery, the consistency boundary. Weak candidates spend their time on the easy 80% (load balancers, "add a cache") and wave vaguely at the hard 20%. The hard 20% is where the points *are*. Run *toward* it.

---

### 5. The connective tissue: how the whole course fits one interview

You have covered a lot. Here is how the pieces map onto the phases of a single interview, so the course stops feeling like 139 separate facts and starts feeling like one motion:

- **Foundations** (HLD vs LLD, how to approach, capacity estimation, consistency, CAP) → Phases 1–2. This is your *scoping and sizing* toolkit.
- **LLD fundamentals + patterns** (SOLID, the GoF patterns) → the LLD round, and the "how would you structure this class" moments.
- **HLD components** (LB, cache, queue, CDN, DB types, sharding, replication) → Phase 3. These are the *nouns* you assemble into the architecture.
- **HLD & LLD case studies** (URL shortener, chat, Instagram, Uber, parking lot...) → your *pattern library*. Every new problem rhymes with one you've drilled.
- **Advanced** (idempotency, leader election, distributed locking, observability...) → Phase 4 deep-dive ammunition.

One interview is a *traversal* of the course, not a slice of it.

---

## Visual / Diagram description

The mental model to carry into the room — a funnel from ambiguity to concreteness, with a feedback loop back to the interviewer at every stage:

```
        "Design Instagram"  (deliberately vague prompt)
                 │
                 ▼
    ┌─────────────────────────────┐   ◄── clarify, narrow
    │ 1. REQUIREMENTS & SCOPE 5-8m │──────────────┐
    │  functional + non-functional │              │
    │  + what's OUT of scope       │              │
    └──────────────┬──────────────┘              │
                   ▼                              │
    ┌─────────────────────────────┐              │
    │ 2. ESTIMATION      ~5m       │              │
    │  DAU → QPS → storage → BW    │              │  ◄─── T
    └──────────────┬──────────────┘              │       H
                   ▼                              │       I
    ┌─────────────────────────────┐              │       N
    │ 3. HIGH-LEVEL + API + DATA   │              │       K
    │    10-15m                    │              │
    │  client→LB→svc→cache→db      │              │       O
    └──────────────┬──────────────┘              │       U
                   ▼                              │       T
    ┌─────────────────────────────┐              │
    │ 4. DEEP DIVE       10-15m    │◄─────────────┤       L
    │  the ONE hard sub-problem    │  hints /     │       O
    └──────────────┬──────────────┘  push-back    │       U
                   ▼                              │       D
    ┌─────────────────────────────┐              │
    │ 5. BOTTLENECKS & WRAP  ~5m   │──────────────┘
    │  failures, scale, summary,   │   ◄── read the room
    │  "what I'd do next"          │       the whole time
    └─────────────────────────────┘
```

Read it top to bottom: the prompt is intentionally wide, and your job is to *funnel* it down to something concrete, tightening at each stage. The vertical bar on the right — **THINK OUT LOUD** — runs the entire height because narration never stops. The feedback arrows on the left show the loop: at every phase you check back with the interviewer, absorb their hints, and adjust. The design flows down; the collaboration flows sideways, constantly.

---

## Real world examples

### Google / Meta / Amazon — the rubric is written down

At large tech companies the interviewer is not improvising a gut feeling. After your round they fill out a structured scorecard (representative: dimensions like problem-solving, technical depth, communication, and a hire/no-hire with justification). This is *why* process beats answer — a "great design, but I couldn't follow their reasoning and they ignored my questions" writeup is a no-hire, while "didn't finish, but clear thinker who scoped well and reasoned honestly" is often a hire. The written rubric is the reason the six dimensions in Concept 1 are not guesswork.

### Engineering blogs — where real designs are documented

The systems you'll be asked to design are documented by the companies that built them. These are, conceptually, the "answer keys" — though remember, they show *one* team's choices under *their* constraints, not the universal truth:

- **The High Scalability blog** — decades of real architecture write-ups.
- **Company engineering blogs** — Netflix, Uber, Meta, Airbnb, Dropbox, Discord, Stripe, and Cloudflare all publish detailed posts on how their real systems work (feed ranking, geospatial matching, message storage, rate limiting).
- **"Designing Data-Intensive Applications"** by Martin Kleppmann — the canonical deep reference for the data-layer fundamentals this whole course is built on.

Reading these trains your intuition for what a *credible* answer sounds like.

### Mock-interview platforms — deliberate practice at scale

Platforms where engineers interview each other (peer mock interviews) exist precisely because the skill is *performed*, not *known*. Candidates who do a dozen timed mocks reliably outperform candidates who read twice as much, because the bottleneck is almost never knowledge — it's doing it out loud, on a clock, with someone watching.

---

## Trade-offs

Even the meta-skill of interviewing has trade-offs to balance in the room:

| Do this | ...but watch the cost |
|---|---|
| Gather thorough requirements | Don't spend 25 min; cap at ~8 and lock scope |
| Show detailed estimation math | Don't get lost in arithmetic; round aggressively, the goal is *order of magnitude* |
| Go deep on the hard problem | Don't go so deep you never present a whole system |
| Consider multiple options | Don't list 5 databases and never decide — *pick one* and justify |
| Think out loud | Don't narrate so much you never make a decision |
| Drive the conversation | Don't steamroll — leave room, and follow their hints |

| Candidate style | Pros | Cons |
|---|---|---|
| **The Encyclopedia** (knows every fact) | Deep knowledge | Fails if silent, can't scope, or can't choose |
| **The Talker** (great communicator, thin depth) | Pleasant, drives well | Fails on the deep dive when it gets real |
| **The Balanced Driver** | Scopes, sizes, decides, narrates, goes deep on the hard part | This is the hire |

**The sweet spot:** be the Balanced Driver. Enough depth to survive the deep dive, enough communication to make that depth *visible*, and enough judgment to spend your 45 minutes where the points actually are. Knowledge gets you in the room; process gets you the offer.

---

## Common interview questions on this topic

### Q1: "There's no single right answer here — so what makes a good one?"

**Hint:** A good answer is *internally consistent* and *justified*: the design meets the stated requirements, the tech choices follow from the estimated scale, every non-trivial decision names its trade-off, and the hard part is engaged rather than waved at. "Good" means "defensible under questioning," not "matches a secret ideal diagram."

### Q2: "You have 45 minutes. Walk me through how you'd spend them."

**Hint:** Requirements & scope (5–8m) → estimation (~5m) → high-level + API + data (10–15m) → deep dive on the hardest sub-problem (10–15m) → bottlenecks + summary + next steps (~5m). Emphasize that you'd narrate throughout, cap requirements so you reach the deep dive, and adjust if it turns out to be an LLD round.

### Q3: "You don't know how technology X works and I just asked about it. What do you do?"

**Hint:** Never bluff — say honestly you haven't used it in depth, then *reason from fundamentals* out loud toward a plausible answer. Interviewers reward honest first-principles reasoning over confident fabrication, because that's the actual job skill. Bluffing, once caught, poisons trust in everything else you said.

### Q4: "I keep pushing you on the same point. Why might I be doing that?"

**Hint:** It's a hint, not harassment. The interviewer is pointing a flashlight at either a gap in your design or the highest-value part of the problem. The correct response is to *stop*, engage that specific point directly, and explore it — not to keep defending your original path.

### Q5: "A candidate designed a flawless microservices architecture with Kafka and five databases for an app with 100 users. Hire?"

**Hint:** No — this is over-engineering, a *judgment* failure that reads as junior. The senior move is the simplest design that meets requirements (likely a monolith + one database at that scale), with a clear story for how it would evolve *when the numbers demand it*. Complexity must be earned by requirements, not added for show.

---

## Practice exercise

### The 4-Week Interview Readiness Plan

Do not just read this document. Build the skill by *doing*, on a clock, out loud. Here is a concrete four-week plan. Each week has drills and a mock-interview milestone. Adjust the pace to your timeline, but keep the *structure*: fundamentals → components → case studies → full-timed-mocks.

**Week 1 — Fundamentals & the framework (build the skeleton).**
- Re-read and internalize [93 — HLD Approach Framework](./93-hld-approach-framework.md) and [111 — LLD Approach Framework](./111-lld-approach-framework.md) until you can recite the five phases from memory.
- Drill [05 — Capacity Estimation](./05-capacity-estimation.md) until the core numbers (below in the cheat sheet) are *automatic*. Do five estimation problems on paper: DAU → QPS → storage over 5 years.
- Skim [133 — System Design Trade-off Matrix](./133-system-design-trade-off-matrix.md) and [03 — How to Approach System Design](./03-how-to-approach-system-design.md).
- **Milestone:** whiteboard the *phases* of a generic design (no specific system) out loud in under 3 minutes, on a timer.

**Week 2 — Components & the toolkit (build the vocabulary).**
- Drill the building blocks: load balancers, caching, queues, CDNs, SQL vs. NoSQL, sharding, replication. For each, be able to say in one sentence *when you'd reach for it and what it costs*.
- Do two *untimed* full designs, focusing on getting the components right, not speed: a **URL shortener** and a **rate limiter** (the gentlest case studies).
- **Milestone:** one 45-minute *self-mock*, recorded (phone camera or screen recording), of the URL shortener. Watch it back. Note every silence and every un-justified choice.

**Week 3 — Case studies & pattern library (build pattern-matching).**
- Do one full timed design per day, out loud, on a 45-min clock: **Pastebin, Notification system, Chat system, Instagram, Twitter, YouTube, Uber.**
- After each, spend 10 minutes writing what the *hard part* was and how you handled it. Build your mental "when I see X, reach for Y" table.
- If prepping for LLD too: drill **parking lot, elevator, and a rate-limiter class** with real code.
- **Milestone:** two *peer mock interviews* — you interview a friend, they interview you. Being the interviewer is the fastest way to see what good looks like.

**Week 4 — Full timed mocks & polish (build performance).**
- Three to five full, timed, out-loud mock interviews — ideally with peers or on a mock platform — on systems you haven't drilled recently, so you're pattern-matching cold.
- Redesign a system you *use every day* (your company's product, WhatsApp, Spotify) from scratch — this proves the skill transfers beyond memorized answers.
- Re-read this document's cheat sheet daily.
- **Milestone:** a mock where you hit all five phases, narrate throughout, name at least three trade-offs, and engage the hard part — with time to spare. When you can do that repeatably, you're ready.

**What to produce:** a filled-in tracker with, for each mock, the date, system, whether you hit all five phases, number of silences over 15 seconds, number of trade-offs stated, and one thing to fix next time. That tracker *is* deliberate practice — it turns vague "I studied" into measurable improvement.

---

## Quick reference cheat sheet

The morning-of revision sheet. This is the most comprehensive one in the course by design — it compresses the whole thing into a page. Skim it, breathe, walk in.

**THE PROCESS (never skip a phase):**
- 🕐 **The clock:** Requirements 5–8m → Estimation 5m → High-level+API+data 10–15m → Deep dive 10–15m → Wrap 5m.
- 🗣️ **Golden rules:** Think out loud. Drive. State assumptions. Read hints. Never bluff — reason from fundamentals.
- ❌ **Never:** jump to a solution, over-engineer for tiny scale, go silent, skip trade-offs, blow the clock, name tech you can't justify, ignore hints, or hand-wave the hard part.

**ESTIMATION NUMBERS TO MEMORIZE (recall [05](./05-capacity-estimation.md)):**
- **Powers of 2 → data:** 2^10 ≈ 1 Thousand (KB), 2^20 ≈ 1 Million (MB), 2^30 ≈ 1 Billion (GB), 2^40 ≈ 1 Trillion (TB).
- **Seconds:** 1 day ≈ 86,400 s ≈ **~10^5 s**. (Handy: DAU / 10^5 ≈ average QPS.)
- **QPS shortcut:** 1M requests/day ≈ **~12 QPS** average. Peak ≈ 2–3× average.
- **Latency ladder:** memory read ~100 ns; SSD ~100 µs; **network round-trip within a datacenter ~0.5 ms**; disk seek ~10 ms; cross-continent round trip ~150 ms.
- **Sizes:** a tweet/short row ~1 KB; a compressed photo ~200 KB–2 MB; a minute of video ~a few MB–tens of MB.
- **Availability:** "three nines" (99.9%) ≈ 8.7 hrs down/yr; "four nines" ≈ 52 min/yr; "five nines" ≈ 5 min/yr.

**THE SCALING TOOLKIT (when the numbers hurt, reach for these):**
- **Cache** — repeated reads of the same data (Redis/Memcached, CDN). Cost: staleness + invalidation.
- **Shard** — data too big for one node; split by key. Cost: cross-shard queries, hot keys, rebalancing.
- **Replicate** — read-heavy or need HA; copy data to more nodes. Cost: replication lag, consistency.
- **Queue** — smooth spikes / decouple / async work (Kafka, SQS). Cost: eventual processing, more moving parts.
- **CDN** — static/media assets served near the user. Cost: invalidation, cost per GB.
- **Denormalize** — reads too slow from joins; precompute/duplicate data. Cost: write amplification, update anomalies.

**WHICH DATABASE? (the heuristic):**
- **Relational (Postgres/MySQL):** you need ACID transactions, joins, strong consistency (payments, orders, anything with money). *Default here unless you have a reason to leave.*
- **Key-value (Redis/DynamoDB):** simple lookups by key, ultra-low latency, sessions, caches, counters.
- **Wide-column (Cassandra):** massive write throughput, horizontal scale, time-series/feeds; no joins, eventual consistency.
- **Document (MongoDB):** flexible/nested schema, per-object reads, rapid iteration.
- **Graph (Neo4j):** the *relationships* are the query (social graph, recommendations, fraud rings).
- **Search (Elasticsearch):** full-text search, filtering, ranking.
- **Blob store (S3/GCS):** large files/media — store the file in blob, the *metadata* in a database.

**THE FANOUT DECISION (feeds/notifications):**
- **Fanout-on-write (push):** precompute each user's feed at post time. Fast reads, expensive writes. Great when the average user has few followers.
- **Fanout-on-read (pull):** build the feed at read time. Cheap writes, expensive reads. Great for celebrities with millions of followers.
- **Hybrid (the real answer):** push for normal users, pull for celebrities. Almost always what you should propose.

**THE CONSISTENCY DECISION (recall CAP):**
- **Strong consistency:** everyone sees the latest write immediately. Needed for money, inventory, unique usernames. Costs latency & availability.
- **Eventual consistency:** replicas converge "soon." Fine for likes, view counts, feeds, timelines. Buys scale & availability.
- **Rule:** *Money → strong. Social/analytics/counts → eventual.* Always name which you chose and why.

**THE STANDARD BUILDING BLOCKS (the nouns of almost every diagram):**
- **Load Balancer** — spreads traffic, health-checks, single entry point.
- **App / Service tier** — stateless (so you can scale horizontally); state lives in data stores.
- **Cache** — in front of the DB and/or the app.
- **Message Queue** — decouple + absorb spikes + async jobs.
- **CDN** — edge delivery of static & media.
- **Blob Store** — files/media; DB holds metadata + the blob URL.
- **Database(s)** — the source of truth, chosen per the heuristic above.

**MAJOR TRADE-OFF AXES (recall [133](./133-system-design-trade-off-matrix.md)):**
- Consistency ↔ Availability (CAP under partition) • Latency ↔ Throughput • Read-optimized ↔ Write-optimized • Normalization ↔ Denormalization • Push ↔ Pull • Simplicity ↔ Flexibility • Cost ↔ Performance • SQL ↔ NoSQL • Synchronous ↔ Asynchronous.
- **The meta-rule:** there is no free lunch. Every choice buys one thing by spending another. *Naming the spend* is what makes you sound senior.

**OPENING LINE (memorize it):** *"Before I design anything, let me ask a few clarifying questions to make sure I'm solving the right problem, and I'll state my assumptions as I go."*

**CLOSING LINE (memorize it):** *"To summarize: [design in two sentences]. The key trade-offs I made were [X and Y]. With more time, I'd dig into [Z]."*

---

## Connected topics

| Direction | Topic | Why |
|---|---|---|
| **Previous** | [93 — HLD Approach Framework](./93-hld-approach-framework.md) | The detailed HLD-round framework this playbook synthesizes into the 45-minute clock. |
| **Next** | [111 — LLD Approach Framework](./111-lld-approach-framework.md) | The parallel framework for LLD / machine-coding rounds — pair it with this playbook for either interview type. |
| **Related** | [05 — Capacity Estimation](./05-capacity-estimation.md) | The estimation numbers and math behind Phase 2; drill this until it's automatic. |
| **Related** | [133 — System Design Trade-off Matrix](./133-system-design-trade-off-matrix.md) | The catalog of trade-off axes you'll name in the room — the source of the "always state the cost" habit. |
| **Related** | [03 — How to Approach System Design](./03-how-to-approach-system-design.md) | The very first framing of the mindset; a fitting bookend to revisit now that the picture is complete. |

---

## A closing note — you made it

Take a breath. You started this course at zero — not knowing HLD from LLD, unsure what a load balancer even did. You now hold a complete foundation: the **foundations** of estimation, consistency, and the CAP theorem; the **LLD** craft of SOLID and clean objects; the **design patterns** that show up in every codebase; the **HLD components** — caches, queues, shards, CDNs — that every large system is assembled from; the **case studies** where you watched real systems come together piece by piece; and the **advanced** topics — idempotency, leader election, observability — that separate a working system from a resilient one. That is a genuinely large body of knowledge, and you built it.

But here is the truth this final document exists to tell you: **system design is a skill built by doing, not by reading.** You do not get better at it by knowing more facts. You get better by standing at a whiteboard, on a clock, talking out loud, making a decision, defending it, hitting a wall, and reasoning your way past it. The framework in this playbook and the fundamentals now in your head are not the finish line — they are the *equipment*. The framework tells you where to spend your 45 minutes. The fundamentals give you something true to say in each phase. Together, they mean you will never again stare at "Design Instagram" with a blank mind. You have a plan. You have the vocabulary. You have the reasoning.

So the last instruction of this entire course is not "read one more topic." It is: **go do the practice exercises.** Start the 4-week plan above. Design something out loud today, on a timer, badly. Then do it again tomorrow, slightly less badly. That loop — do, reflect, adjust, repeat — is the whole thing. Every senior engineer you'll ever interview with went through exactly this, and none of them were born knowing it.

You are ready to begin the part that actually makes you good. Go build. Good luck — you've got this.
