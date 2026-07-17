# 87 — Batch Processing vs Stream Processing
## Category: HLD Components

---

## What is this?

**Batch processing** takes a finite pile of data that has already been collected — "all of yesterday's orders" — and grinds through it on a schedule. **Stream processing** takes an endless firehose of events that never stops arriving — every click, every payment, every GPS ping — and reacts to each one as it lands.

Think of a **post office**. Batch is the sorting facility: mail piles up all day, and at 2am a machine sorts the entire day's sack in one go. Stream is the courier on a motorbike: a package appears, and they deliver it *now*, one at a time, forever.

Neither is "better." They answer different questions. Batch answers *"what happened yesterday, exactly?"* Stream answers *"what is happening right now, roughly?"*

---

## Why does it matter?

If you don't understand this split, you will reach for the wrong tool and pay for it:

- You'll build a **fraud detection system as a nightly batch job** — and discover the fraud sixteen hours after the money left. Useless.
- You'll build a **month-end financial report as a stream** — and produce numbers that quietly disagree with each other because late-arriving events kept mutating the totals after you'd already sent the report to the CFO.
- You'll build a real-time dashboard, aggregate by "when my server saw the event" instead of "when the event actually happened," and ship metrics that are subtly, unfixably wrong.

**Interview angle:** the moment you say "we'll run analytics on this," a good interviewer asks *"batch or stream?"* — and then immediately follows with *"event time or processing time?"* and *"what about late events?"* These three questions separate people who've read a blog post from people who've operated a pipeline.

**Real-work angle:** essentially every company above a certain size runs both. Batch feeds the warehouse, the ML training sets, and the billing runs. Stream feeds the fraud engine, the live dashboards, and the alerting. Your job is knowing which lane a new requirement belongs in.

---

## The core idea — explained simply

### The Bakery Analogy

You run a bakery. Two very different jobs happen every day.

**Job 1 — The nightly stock count (batch).** At midnight the shop is closed. Nothing is moving. You count every item on every shelf, reconcile it against every receipt from the day, and produce one exact number: today's revenue, today's waste, tomorrow's flour order. You have a **bounded** pile — the day is over, no new receipts can appear. You care about being *exactly right*, and it's fine if it takes two hours. Nobody needs the number at 12:01am.

**Job 2 — The oven alarm (stream).** While the shop is open, a temperature sensor on the oven emits a reading every second. If it crosses 260°C, the bread burns. You need to react within **seconds**, not tomorrow. You will never have "all" the readings — the sensor keeps emitting forever. There is no "end of the data." You care about being *fast*, and it's fine if you're occasionally approximate.

That's the whole distinction. One dataset has an end; the other doesn't.

| Bakery | Data engineering |
|---|---|
| The day's closed till, counted at midnight | **Batch job** over a bounded dataset |
| The oven sensor, ticking forever | **Stream** — an unbounded sequence of events |
| "It's fine if it takes 2 hours" | Optimizing for **throughput** and completeness |
| "React in 3 seconds or the bread burns" | Optimizing for **latency** |
| The day is over — no new receipts | **Bounded** input: you know when you're done |
| The sensor never stops | **Unbounded** input: you're *never* done |
| A receipt found under the counter next morning | A **late-arriving event** — the hardest problem in streaming |

### The comparison table you should memorize

| Dimension | Batch | Stream |
|---|---|---|
| **Input** | Bounded (a fixed, complete dataset) | Unbounded (never ends) |
| **Trigger** | A schedule (cron: nightly, hourly) | Event arrival (continuous) |
| **Latency** | Minutes to hours | Milliseconds to seconds |
| **Optimizes for** | Throughput, completeness, cost per GB | Latency, freshness |
| **Correctness** | Easy — all the data is sitting there | Hard — data is still arriving |
| **Reprocessing** | Trivial: just re-run the job on the same files | Hard: you must replay the log |
| **Failure recovery** | Re-run the job from scratch | Restore from a checkpoint |
| **State** | Usually stateless per run | Almost always **stateful** across events |
| **Typical tools** | Hadoop MapReduce, Spark, dbt, Airflow | Flink, Kafka Streams, Spark Structured Streaming |
| **Classic use** | Nightly billing, ML training sets, reports | Fraud alerts, live dashboards, recommendations |

---

## Key concepts inside this topic

### 1. Batch: MapReduce, properly explained

MapReduce is the idea that made large-scale batch processing possible. It has exactly three phases. Let's count words across 900 GB of text spread over 100 machines.

**Phase 1 — MAP.** Each worker reads *its own local chunk* and emits a key-value pair per word. It doesn't know or care what any other worker is doing. This is **embarrassingly parallel** — you can throw 10 machines or 10,000 at it with zero coordination.

```javascript
// Runs independently on every worker, on its local chunk of the data.
// Note: it emits (word, 1) — no counting, no cleverness. Just emit.
function map(lineOfText, emit) {
  for (const word of lineOfText.toLowerCase().split(/\W+/)) {
    if (word) emit(word, 1);   // ("the", 1), ("cat", 1), ("the", 1) ...
  }
}
```

**Phase 2 — SHUFFLE / SORT.** Every `(word, 1)` pair must now travel to the reducer responsible for that word — typically `reducer = hash(word) % numReducers`. All the `("the", 1)`s on all 100 machines must end up on the *same* reducer, because you can't sum a word's count if its occurrences are scattered.

**This is the expensive part.** It is a full, network-wide sort of the entire dataset. Every machine talks to every other machine. Everything hits disk. When a MapReduce job is slow, it is almost always the shuffle. Remember that — interviewers love it.

**Phase 3 — REDUCE.** Each reducer now holds all values for its keys, grouped. It sums them.

```javascript
// The framework guarantees: all values for one key arrive together, on one reducer.
function reduce(word, counts, emit) {
  emit(word, counts.reduce((a, b) => a + b, 0));   // ("the", 47281)
}
```

**The insight that made Hadoop work: move the computation to the data.** You cannot move a petabyte across the network to a compute cluster — at 10 Gbps that's over 9 days of pure transfer. So instead, Hadoop stores the data on the same machines that will process it (HDFS), and ships the *code* — a few kilobytes — to whichever machine already holds the relevant block. Data locality. Bytes stay put; the tiny function moves.

**Why Spark largely replaced MapReduce.** Classic MapReduce writes intermediate results **to disk after every single stage**. A 5-stage pipeline = 5 rounds of disk write + disk read. Spark instead:
- Keeps intermediate results **in memory** across stages (RDDs / DataFrames),
- Builds a **DAG** (directed acyclic graph — a plan of all your operations) *before* executing, so it can fuse steps, skip pointless materialization, and optimize the whole chain at once.

Result: commonly **10–100x faster** for multi-stage and iterative workloads (ML training, which loops over the same dataset dozens of times, was the killer case). Same mental model — map, shuffle, reduce — just without the disk tax between every step.

### 2. Stream: windowing (you can't aggregate infinity)

"Count all events" is meaningless on an unbounded stream — the answer never stops changing and never arrives. So you slice time into **windows** and aggregate within each slice.

**Tumbling window** — fixed size, no overlap, no gaps. Every event lands in exactly one window. *"Orders per 5 minutes."*

**Sliding (hopping) window** — fixed size, but they overlap because they advance by a smaller step. Every event lands in *several* windows. *"Orders in the last 5 minutes, recomputed every 1 minute."* This is what powers most "rolling average" dashboards.

**Session window** — no fixed size at all. Events are grouped by *activity*: a window stays open as long as events keep arriving, and closes after a **gap timeout** of inactivity. *"One user's browsing session"* — perfect, because sessions are naturally variable-length. User clicks 12 times over 4 minutes, goes quiet for 30 minutes → that's one session. Comes back → new session.

```javascript
const WINDOW_MS = 5 * 60 * 1000;

// Tumbling: integer-divide the timestamp. Every event maps to exactly one bucket.
const tumblingKey = (eventTs) => Math.floor(eventTs / WINDOW_MS) * WINDOW_MS;

// Sliding (5-min window, 1-min hop): one event belongs to 5 overlapping windows.
const SLIDE_MS = 60 * 1000;
function slidingKeys(eventTs) {
  const keys = [];
  const first = Math.floor(eventTs / SLIDE_MS) * SLIDE_MS;
  for (let start = first; start > eventTs - WINDOW_MS; start -= SLIDE_MS) keys.push(start);
  return keys;
}
```

### 3. Event time vs processing time — the concept that matters most

Give this one real attention. It is *the* interview question.

- **Event time** = when the thing actually happened in the real world. Stamped onto the event at the source: `{ type: 'purchase', eventTime: 14:58:12 }`.
- **Processing time** = when your system got around to seeing it. `Date.now()` inside your consumer.

They are never identical, and sometimes they diverge wildly.

**The story.** Priya is on a train. At **14:58** she taps "Buy" on a $90 pair of shoes. The purchase succeeds locally on her phone. Two seconds later the train enters a tunnel. Her phone loses signal. The app queues the event.

At **15:38** — forty minutes later — the train exits the tunnel, her phone reconnects, and the buffered event finally reaches your Kafka topic.

Now: **which 15-minute revenue window does that $90 belong to?**

- By **event time**: the 14:45–15:00 window. That is when the sale *happened*. That is the truth. It's the answer that matches the accounting ledger, the answer that reconciles with the nightly batch job, the answer that is *the same every time you compute it*.
- By **processing time**: the 15:30–15:45 window. Which is a lie. Your 14:45 window is now permanently $90 short, your 15:30 window is $90 too rich, and — the killer — if you replay the same Kafka topic tomorrow, the event will land in a *completely different* window, because processing time depends on when you happened to run.

That last point is the one to say out loud in an interview: **processing-time results are not reproducible.** Same input, different answer, depending on network weather. Event-time results are **deterministic** — replay the log a hundred times, get the identical answer a hundred times. That property is what makes stream output trustworthy enough to sit next to batch output.

The catch, and it's a real one: event time forces you to confront the question *"how long do I wait for stragglers?"* Processing time never asks that — which is exactly why it's easy and wrong.

### 4. Late events and watermarks

If you aggregate by event time, you have a problem: when is the 14:45–15:00 window *finished*? You can't wait forever. You can't close it at exactly 15:00 either — Priya's event is still in the tunnel.

A **watermark** is the system's formal assertion:

> "I believe I have now seen all events with event-time ≤ **T**."

When the watermark passes 15:00, the 14:45–15:00 window closes and emits its result. The watermark is usually computed as `maxEventTimeSeenSoFar - allowedLateness`. With a 2-minute allowed lateness, once you observe an event stamped 15:02, you declare the watermark at 15:00 and fire the window.

**The trade-off is stark, and you should state it exactly like this:**

| Allowed lateness | Result |
|---|---|
| **Long** (e.g. 1 hour) | More correct — you catch Priya's straggler. But every window's answer is delayed by an hour. Your "real-time" dashboard is an hour behind. |
| **Short** (e.g. 5 seconds) | Fast, fresh answers. But stragglers arrive after the window is already closed and are dropped or mis-bucketed. |

There is no free lunch here. **Latency and completeness are the same dial.** Turning one up turns the other down.

**What do you do with an event that arrives after its window closed?** Three legitimate answers — know all three:

1. **Drop it.** Increment a `lateEventsDropped` metric and move on. Fine for a click-count dashboard where 0.01% error is invisible. Never fine for money.
2. **Side-output it.** Route late events to a separate "dead letter" stream. Nothing is lost; a batch job reconciles them later. This is the pragmatic default.
3. **Retract and recompute.** Keep the window's state around, re-emit a *corrected* result, and require the downstream sink to handle updates (an upsert keyed by window, not an append). Most correct, most expensive — your downstream must tolerate a number changing after it was published.

### 5. Stateful streaming and checkpointing

Most useful stream jobs are **stateful**: "count per user per hour" means you're holding a partial count in memory for every active user. That state can be gigabytes.

Now the job crashes 55 minutes into a 1-hour window. If that state was only in RAM, it's gone — and you cannot recover it, because you can't un-consume events from an hour ago without replaying.

So streaming engines **checkpoint**: periodically (every few seconds) they snapshot all operator state *plus* the exact offsets they'd consumed from the source, and write both, atomically, to durable storage (S3, HDFS). On restart, the job restores the state snapshot **and** rewinds the source to the matching offsets. It resumes as if the crash never happened.

The key idea is that the state and the offset are snapshotted **together**. If they could drift apart, you'd either double-count (restored old state, new offset) or lose data (new state, old offset).

### 6. Exactly-once semantics

"Exactly-once" is the promise that each event affects the result **exactly one time**, even across failures. It's built from two pieces:

1. **Checkpointed state + replayable source** (above) — internal state is consistent with the offsets.
2. **Transactional / idempotent sink** — the output write is committed in the same transaction as the checkpoint, or is keyed so that rewriting it is harmless.

Be precise in interviews: the *messages* are not delivered exactly once — on recovery, the source genuinely re-reads events it already read. What's guaranteed is that their **effect on the state and the output** appears exactly once. The honest name is **"effectively once."** This is the same idea as **idempotency** in APIs: the operation may physically happen twice, but the observable outcome is as if it happened once. If your sink is a plain `INSERT` with no dedupe key, no framework can save you — you *will* get duplicates.

### 7. Lambda architecture vs Kappa architecture

**Lambda** (Nathan Marz, ~2011): you want batch's correctness *and* stream's speed, so you run **both**.

- **Batch layer** — reprocesses the full history nightly; slow but exactly right.
- **Speed layer** — a streaming job producing approximate, low-latency results for the recent window only.
- **Serving layer** — merges them at query time: "batch view for everything up to last night + speed view for today so far."

It works. And it has one **damning flaw**: you implement the *same business logic twice*, in two different systems, with two different programming models, and then you must keep them in agreement **forever**. Every rule change ("sessions now time out after 25 minutes, not 30") is two PRs in two codebases, and if they drift, the two layers silently disagree and your merged number is wrong in a way nobody can debug.

**Kappa** (Jay Kreps, ~2014): delete the batch layer. Treat *everything* as a stream. One codebase, one engine.

"But how do I reprocess history when I fix a bug?" **You replay the log.** Recall from [67 — Message Queues](./67-message-queues.md) that Kafka is not a queue that deletes on read — it is a **durable, ordered, replayable log** with a retention period. History is *already sitting there*. So you:

1. Start a *second* instance of the fixed job, reading the topic from **offset 0**.
2. Let it chew through the history and write to a *new* output table.
3. When it catches up to the live head, atomically switch readers to the new table.
4. Delete the old one.

That reprocessing run is, functionally, a batch job — but it uses the **same code**, the **same engine**, and the **same semantics** as the live stream. One implementation. That's why Kappa won mindshare: it eliminated the duplicated-logic problem rather than managing it.

Kappa's limits, honestly: it needs retention long enough to hold the history you'd ever want to replay (expensive for years of data — that's what the lake in [91 — Data Lakes and Warehouses](./91-data-lakes-and-warehouses.md) is for), and a full replay of a huge topic can be slow. Many real companies land in the middle: Kappa-style stream jobs for serving, plus a lake for long-horizon reprocessing and ad-hoc analysis.

### 8. The tools, named and placed

| Tool | Lane | What it's actually for |
|---|---|---|
| **Hadoop MapReduce** | Batch | The original. Disk-heavy, largely superseded, but the mental model everything else inherits. |
| **Apache Spark** | Batch | The modern batch workhorse. In-memory, DAG-optimized. ETL, ML training sets, warehouse loads. |
| **Spark Structured Streaming** | Stream | Streaming bolted onto Spark's engine via micro-batches (tiny batches every ~100ms–seconds). Great if you already run Spark; latency floor is higher than true streaming. |
| **Kafka Streams** | Stream | A *library*, not a cluster — it's just a JAR inside your app. Ideal when your data already lives in Kafka and you don't want to operate another system. |
| **Apache Flink** | Stream | The gold standard for **event-time correctness**: true per-event processing, first-class watermarks, sophisticated state backends, exactly-once. Reach for this when correctness under out-of-order data actually matters. |

Flink's unifying philosophy — worth quoting in an interview — is that **"batch is just a bounded stream."** A batch job is a stream that happens to have an end; when the source runs dry, you emit a final watermark of `+∞`, every window closes, and you're done. One engine, one API, both lanes. It's the intellectual endpoint of this entire topic.

---

## Visual / Diagram description

### Diagram 1: MapReduce word count

```
   INPUT (bounded: 3 files, already on disk)
   ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
   │ "the cat sat"│  │ "the cat ran"│  │ "the dog sat"│
   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
          │                 │                 │
   ┌──────▼───────┐  ┌──────▼───────┐  ┌──────▼───────┐
   │   MAPPER 1   │  │   MAPPER 2   │  │   MAPPER 3   │   ◀── code moves TO the data
   │ emit(w, 1)   │  │ emit(w, 1)   │  │ emit(w, 1)   │       (embarrassingly parallel)
   └──────┬───────┘  └──────┬───────┘  └──────┬───────┘
          │                 │                 │
   (the,1)(cat,1)    (the,1)(cat,1)    (the,1)(dog,1)
   (sat,1)           (ran,1)           (sat,1)
          │                 │                 │
          └────────┬────────┴────────┬────────┘
                   │                 │
        ╔══════════▼═════════════════▼══════════╗
        ║   SHUFFLE / SORT  (hash(key) % N)     ║   ◀── THE EXPENSIVE PART:
        ║   all values for one key → one node   ║       full network-wide sort,
        ╚══════════╤═════════════════╤══════════╝       every node talks to every node
                   │                 │
      ┌────────────▼───────┐  ┌──────▼─────────────┐
      │     REDUCER A      │  │     REDUCER B      │
      │ the → [1,1,1]      │  │ sat → [1,1]        │
      │ cat → [1,1]        │  │ ran → [1]          │
      │                    │  │ dog → [1]          │
      │  sum each key      │  │  sum each key      │
      └────────────┬───────┘  └──────┬─────────────┘
                   │                 │
              the=3, cat=2       sat=2, ran=1, dog=1
```

Read it top to bottom: mappers work alone on local data and emit unaggregated pairs; the shuffle drags every pair across the network so that identical keys are co-located; reducers then only have to sum a list. Mappers and reducers are cheap and scale linearly — the shuffle is where your job dies.

### Diagram 2: The three window types on one timeline

```
Events:      a   b     c        d   e        f       g    h
Time ──────────────────────────────────────────────────────────▶
          0    1    2    3    4    5    6    7    8    9   10  (minutes)

TUMBLING (5 min, no overlap) — every event in exactly ONE window
          ├──────── W1 ────────┤├──────── W2 ────────┤
          [0................5) [5...............10)
              a,b,c,d              e,f,g,h

SLIDING (5 min window, 1 min hop) — windows OVERLAP, event in MANY
          ├──────── W1 ────────┤            (covers 0-5)
             ├──────── W2 ────────┤         (covers 1-6)
                ├──────── W3 ────────┤      (covers 2-7)
                   ├──────── W4 ────────┤   (covers 3-8)
          ▲
          event 'd' at t=4 appears in W1, W2, W3, W4 — counted 4 times, once per window

SESSION (gap timeout = 2 min) — boundaries defined by INACTIVITY
          ├── S1 ──┤              ├───── S2 ─────┤
           a  b  c   ← 2min gap →  d  e  f  g  h
          (window closes only after 2 min of silence, then reopens on next event)
```

### Diagram 3: Lambda vs Kappa

```
LAMBDA — two paths, two codebases, one merge

                    ┌──────────────────────────┐
                    │  BATCH LAYER (Spark)     │
              ┌────▶│  full history, nightly   │────┐
              │     │  slow + exactly right    │    │
              │     └──────────────────────────┘    │   ┌──────────────┐
  ┌────────┐  │                                     ├──▶│   SERVING    │──▶ query
  │ EVENTS │──┤                                     │   │  merge both  │
  └────────┘  │     ┌──────────────────────────┐    │   │    views     │
              │     │  SPEED LAYER (Flink)     │    │   └──────────────┘
              └────▶│  recent only, seconds    │────┘
                    │  fast + approximate      │
                    └──────────────────────────┘
                       ▲
                       └── SAME BUSINESS LOGIC, WRITTEN TWICE.  ◀── the fatal flaw
                           Two languages, two engines, must agree forever.


KAPPA — one path, one codebase; history = replaying the log

  ┌────────────────────────────────────────────┐
  │  KAFKA — a DURABLE, REPLAYABLE LOG         │
  │  offset: 0 ......................... head  │
  └───────┬─────────────────────────┬──────────┘
          │ read from head          │ read from offset 0
          │ (live)                  │ (reprocessing after a bug fix)
   ┌──────▼────────┐         ┌──────▼──────────┐
   │  STREAM JOB   │         │  STREAM JOB v2  │   ◀── SAME CODE, same engine
   │     v1        │         │  (fixed logic)  │
   └──────┬────────┘         └──────┬──────────┘
          │                         │
   ┌──────▼────────┐         ┌──────▼──────────┐
   │  output_v1    │         │   output_v2     │
   └───────────────┘         └─────────────────┘
                                     │
                        when v2 catches up to head:
                        atomically switch readers to v2, drop v1
```

---

## Real world examples

### Netflix

Netflix runs both lanes deliberately. Its **batch** side (historically Hadoop, now heavily Spark on S3) builds the training datasets for recommendation models and the daily viewing analytics — jobs where being right matters more than being instant. Its **streaming** side (Flink, fed by Kafka) powers near-real-time signals: what you're watching *now*, playback quality telemetry, and operational alerting when a device type starts failing. The batch layer decides *what kind of shows you like*; the stream layer notices *that you just started a thriller at 11pm*.

### Uber

Uber's surge pricing is a textbook stream problem: you need supply-and-demand aggregates per geographic cell, updated within seconds, or the price is meaningless. Uber built heavily on **Flink over Kafka** for exactly this class of workload, and has been public about moving *away* from a Lambda-style dual-implementation toward unified streaming, precisely because maintaining two copies of the same logic was a persistent source of divergence bugs. Meanwhile driver earnings statements — which must reconcile to the cent — remain batch.

### A payments company (representative architecture)

Fraud scoring must be **stream**: the decision has to happen inside the ~200ms of the authorization call, using features like "cards used at this merchant in the last 5 minutes" (a sliding window) and "transactions in this user's current session" (a session window). But the **chargeback reconciliation and monthly merchant settlement** are batch: they run once a day over a bounded, closed set of transactions, and they are allowed to take hours because they must tie out exactly against the bank's ledger. Same events, same company, two lanes — chosen by *how stale the answer is allowed to be*.

---

## Trade-offs

| | Batch | Stream |
|---|---|---|
| **Pro** | Simple mental model — the data sits still while you work | Answers are fresh; you can act while it still matters |
| **Pro** | Trivially reprocessable: re-run on the same input files | Smooth, constant resource usage (no 3am spike) |
| **Pro** | Easy correctness — no late data, no watermarks, no windows | Enables entire product categories (fraud, live ops, alerting) |
| **Pro** | Cheap per GB; can use spot/preemptible instances | Failures are localized to a few events, not a whole run |
| **Con** | Latency is baked in — a nightly job is up to 24h stale | Genuinely harder: windows, watermarks, state, checkpoints |
| **Con** | Bursty load: cluster idle 22h, on fire for 2h | Correctness is subtle — out-of-order data will bite you |
| **Con** | A failed job at 3am can mean a missing report at 9am | Must be always-on; ops burden is higher |
| **Con** | Can't answer "what's happening right now" at all | Reprocessing history means a full replay (slow, costly) |

**The decision rule — ask exactly one question:**

> **"How stale can this answer be before it is worthless?"**

| Staleness tolerance | Lane |
|---|---|
| Hours or a day is fine | **Batch.** Don't pay the streaming complexity tax. |
| Minutes is fine | **Micro-batch** (Spark Structured Streaming) — the middle ground. |
| Seconds or less, or a decision blocks on it | **Stream.** You have no choice. |

**Rule of thumb:** default to **batch** — it is dramatically simpler, and simplicity is the thing you'll be grateful for at 3am. Reach for **stream** only when the *value of the answer decays fast*: fraud (worthless in an hour), live dashboards, alerting, real-time recommendations. Nightly billing, financial reports, and ML training sets are batch, and no amount of architectural fashion changes that.

---

## Common interview questions on this topic

### Q1: "What's the difference between event time and processing time, and why should I care?"
**Hint:** Event time = when it happened (stamped at source). Processing time = when your system saw it. They diverge when networks are slow, devices go offline, or consumers lag. Aggregating by processing time puts a delayed purchase into the wrong window, so your numbers are wrong *and* **non-reproducible** — replay the same log tomorrow and you'd get a different answer. Event time is deterministic. Always aggregate by event time; the price you pay is having to reason about watermarks and late data.

### Q2: "What is a watermark and what does it trade off?"
**Hint:** A watermark is the system's assertion "I've now seen everything with event-time ≤ T," which lets it close a window and emit. It's usually `maxEventTimeSeen - allowedLateness`. The trade-off is a single dial: **wait longer = more correct, higher latency; wait less = faster, but stragglers get dropped or mis-bucketed.** Name your three options for a late event: drop (with a metric), side-output for later reconciliation, or retract-and-recompute (needs an upsert-capable sink).

### Q3: "Lambda or Kappa? Defend it."
**Hint:** Lambda gives you correctness plus speed but forces you to implement the **same logic twice** in two systems and keep them in agreement forever — that's where the bugs live. Kappa drops the batch layer: everything is a stream, and reprocessing history is just replaying the Kafka log from offset 0 with the *same code*, writing to a new table, then cutting over. Kappa is the default today because it kills the duplicated-logic problem. Caveat honestly: Kappa needs long retention, and a full replay of a very large topic can be slow.

### Q4: "Is exactly-once processing real?"
**Hint:** It's **"effectively once."** Messages genuinely *are* re-read from the source after a crash. What's guaranteed is that the effect on state and output happens once — achieved by checkpointing state and source offsets **atomically together**, plus a transactional or idempotent sink. If your sink is a naive append-only `INSERT`, no framework can rescue you. Tie it back to idempotency keys in APIs — same idea, same reason.

### Q5: "Design a real-time trending-topics feature. Batch or stream?"
**Hint:** Stream — a trend that's 6 hours old isn't a trend. Use a **sliding window** (e.g. last 60 min, hopped every 1 min) keyed by hashtag, on event time. State per hashtag is a count, checkpointed. Emit the top-K per window. Then volunteer the trade-off: exact top-K over millions of keys is expensive, so use an approximate sketch (Count-Min Sketch) and accept small error — freshness matters more than the fourth decimal place here. Mention that a nightly **batch** job can recompute exact historical trends for the analytics team.

---

## Practice exercise

### Build a windowed aggregator over a live-ish event stream

Write a Node.js script (~30 minutes) that simulates a stream of purchase events and aggregates revenue by **event time**, not processing time.

**Requirements:**

1. Generate events shaped `{ userId, amountCents, eventTime }`, emitted every ~50ms.
2. **Deliberately inject out-of-order events**: with 10% probability, stamp `eventTime` 2–8 minutes *in the past* (simulating Priya in the tunnel).
3. Aggregate revenue into **1-minute tumbling windows keyed by event time**.
4. Implement a **watermark** = `maxEventTimeSeen - 30_000` (30s allowed lateness). When the watermark passes a window's end, close it, print the total, and free its state.
5. Any event arriving for an already-closed window is **late**: print it to a side-output list and increment a `lateDropped` counter.

**Then answer, in comments at the bottom of the file:**
- How many events were dropped as late? Now change allowed lateness to 10 minutes and re-run. How does the drop count change, and how much later does each window's result get emitted? *That number is the latency/completeness trade-off, in your own data.*
- If you'd keyed windows by `Date.now()` instead of `eventTime`, would re-running the exact same event list give you the exact same output? Why not?

Here's a skeleton to build on:

```javascript
const WINDOW_MS = 60_000;
const ALLOWED_LATENESS_MS = 30_000;

class TumblingWindowAggregator {
  constructor(onWindowClose) {
    this.windows = new Map();      // windowStart -> { totalCents, count }
    this.maxEventTimeSeen = 0;
    this.lateDropped = 0;
    this.onWindowClose = onWindowClose;
  }

  // Every event carries the ONLY timestamp we trust: when it actually happened.
  ingest({ amountCents, eventTime }) {
    const windowStart = Math.floor(eventTime / WINDOW_MS) * WINDOW_MS;

    // Has this window already been closed and emitted? Then we're too late.
    if (windowStart + WINDOW_MS <= this.watermark()) {
      this.lateDropped++;
      return { late: true, windowStart };
    }

    const w = this.windows.get(windowStart) ?? { totalCents: 0, count: 0 };
    w.totalCents += amountCents;
    w.count++;
    this.windows.set(windowStart, w);

    this.maxEventTimeSeen = Math.max(this.maxEventTimeSeen, eventTime);
    this.fireExpiredWindows();
    return { late: false, windowStart };
  }

  // "I believe I've now seen everything up to this event time."
  watermark() {
    return this.maxEventTimeSeen - ALLOWED_LATENESS_MS;
  }

  // A window can only be emitted once the watermark has passed its END.
  fireExpiredWindows() {
    const wm = this.watermark();
    for (const [start, agg] of this.windows) {
      if (start + WINDOW_MS <= wm) {
        this.onWindowClose(start, agg);
        this.windows.delete(start);   // free the state — otherwise you leak forever
      }
    }
  }
}

// --- demo ---
const agg = new TumblingWindowAggregator((start, { totalCents, count }) => {
  const label = new Date(start).toISOString().slice(11, 16);
  console.log(`window ${label} closed → $${(totalCents / 100).toFixed(2)} (${count} orders)`);
});

// TODO: emit events every 50ms, 10% of them stamped 2-8 minutes in the past.
// TODO: count how many come back { late: true } and print the total at the end.
```

**What to produce:** the running script, the two drop-counts (30s vs 10min lateness), and a two-sentence written answer to the reproducibility question.

---

## Quick reference cheat sheet

- **Batch** = bounded input, scheduled, optimizes **throughput** and completeness, latency in minutes-to-hours.
- **Stream** = unbounded input, continuous, optimizes **latency**, measured in ms-to-seconds. Never "done."
- **MapReduce** = map (parallel, emit pairs) → **shuffle** (network-wide sort, the expensive part) → reduce (aggregate per key).
- **Move the computation to the data** — shipping petabytes across a network is impossible; shipping a function is free.
- **Spark beat MapReduce** by keeping intermediates **in memory** across a **DAG** of stages instead of hitting disk between every step. 10–100x on multi-stage jobs.
- **Windows**: tumbling (fixed, no overlap) / sliding (overlapping, hops) / session (grouped by activity, closed by a gap timeout).
- **Event time** = when it happened. **Processing time** = when you saw it. Always aggregate by **event time** — it's the only one that's deterministic and reproducible.
- **Watermark** = "I've seen everything up to T," which lets a window close. Usually `maxEventTime - allowedLateness`.
- **The one dial:** wait longer = more correct, slower. Wait less = faster, drop stragglers. There is no third option.
- **Late event options:** drop it (with a metric), side-output it, or retract-and-recompute (needs an upsert sink).
- **Checkpointing** snapshots operator state **and** source offsets **atomically** — that's what makes crash recovery correct.
- **Exactly-once is "effectively once"** — messages get reprocessed; their *effect* happens once. Needs a transactional/idempotent sink.
- **Lambda** = batch + speed layers merged at query time. Fatal flaw: **the same logic, written twice, forever.**
- **Kappa** = stream only; reprocess history by **replaying the log** with the same code. Won because it kills the duplication.
- **Flink** is the event-time gold standard; its philosophy — **"batch is just a bounded stream"** — is the punchline of this whole topic.
- **The decision rule:** *how stale can this answer be before it's worthless?* Hours → batch. Seconds → stream.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [67 — Message Queues](./67-message-queues.md) — the durable, replayable log that makes Kappa's "just replay history" possible |
| **Next** | [91 — Data Lakes and Warehouses](./91-data-lakes-and-warehouses.md) — where batch jobs read from and write to, and where long-horizon history lives |
| **Related** | [68 — Event-Driven Architecture](./68-event-driven-architecture.md) — the events your stream processor consumes have to come from somewhere |
| **Related** | [76 — Eventual Consistency](./76-eventual-consistency.md) — a stream's aggregate is eventually consistent by construction; late events are exactly why |
| **Related** | [79 — Search Systems](./79-search-systems.md) — a search index is typically built by batch and kept fresh by a stream: the two lanes in one system |
