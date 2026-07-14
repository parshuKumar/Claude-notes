# 67 — Message Queues and Async Processing
## Category: HLD Components

---

## What is this?

A **message queue** is a buffer that sits between two pieces of your system. One side (the **producer**) drops a note in — "please resize image 42" — and immediately walks away. The other side (the **consumer**) picks notes up when it has capacity and does the work.

Think of the **order spike at a coffee shop**. The barista doesn't make your latte while you stand at the register blocking the line. The cashier writes your name on a cup, puts the cup on the rail, and takes the next customer. The rail is the queue. The barista drains it at whatever speed they can manage. Nobody at the register is stuck waiting for milk to steam.

That's the whole idea: **stop making the user wait for work that doesn't need to happen right now.**

---

## Why does it matter?

Without a queue, every slow thing your system does happens **inside the HTTP request**, while the user's browser spinner turns.

A user uploads a profile photo. Your handler:
1. Saves the file to S3 — 400 ms
2. Generates 4 thumbnail sizes — 6,000 ms
3. Runs it through a nudity classifier — 1,500 ms
4. Sends a "photo updated" email — 300 ms
5. Invalidates the CDN cache — 200 ms

**Total: 8.4 seconds** of the user staring at a spinner. And if the email service is down, the whole upload fails — even though the photo is already safely in S3.

With a queue, the handler saves the file, enqueues a job, and returns in **420 ms**. Everything else happens in the background. If the email service is down, the email job retries in 30 seconds; the photo upload was never affected.

**In interviews:** Almost every HLD answer past a certain size involves a queue. Notifications, video encoding, feed fanout, payments, analytics ingestion, search indexing — all queue-shaped. If you draw a system with no async boundary anywhere, the interviewer will ask "what happens when the notification service is slow?" and you'll have no answer.

**At work:** Queues are how you survive traffic spikes, how you make retries safe, and how you stop one flaky downstream service from taking down your API. They're also how you accidentally lose data or process the same payment twice — so you need to understand the guarantees, not just the API.

---

## The core idea — explained simply

### The Restaurant Kitchen Analogy

A busy restaurant. The **waiter** takes your order, writes it on a ticket, and clips the ticket to a **rail** in the kitchen window. Then he goes back to the floor. The **chefs** pull tickets off the rail, cook, and ring a bell.

Now look at what this buys the restaurant:

- **Decoupling** — the waiter doesn't need to know how a risotto is made, or which chef is free. He only needs to know how to write a ticket.
- **Load levelling** — at 8pm, 30 tickets hit the rail in 5 minutes. The kitchen can only cook 10 per 5 minutes. The rail **absorbs the spike**. The kitchen doesn't explode; it just runs a deeper backlog for 20 minutes and then catches up.
- **Retries** — a chef burns a steak. He doesn't tell the customer "sorry, your order is gone forever." The ticket is still on the rail. He cooks it again.
- **Getting slow work off the critical path** — you don't stand at the front door waiting for your food to be cooked before you're allowed to sit down.

And the failure modes map too:
- **The rail keeps growing all night** — the kitchen is too slow for the demand. This is **backpressure**, and queue depth is the alarm.
- **One ticket says "cook a unicorn"** — no chef can ever complete it, and it keeps coming back around. That's a **poison message**, and it needs to go in a special "we can't do this" box: the **dead-letter queue**.

| Restaurant | Technical term | What it means |
|---|---|---|
| Waiter | **Producer** | The code that creates a message |
| Ticket | **Message / Job** | A small blob of data describing work to do |
| Rail | **Queue / Topic** | Durable buffer holding pending messages |
| The kitchen window itself | **Broker** | The server that stores and hands out messages (Kafka, RabbitMQ, SQS, Redis) |
| Chef | **Consumer / Worker** | The process that pulls messages and does the work |
| All chefs on the hot line | **Consumer group** | A set of workers sharing one queue, each message going to exactly one of them |
| Chef rings the bell | **Ack** | "Done, you can delete this message" |
| Chef drops the ticket back | **Nack / retry** | "I failed, give it to someone else" |
| Ticket rail length | **Queue depth / lag** | Your #1 health metric |
| "Cook a unicorn" ticket | **Poison message** | A message that fails forever |
| The "can't do this" box | **Dead-letter queue (DLQ)** | Where messages go after N failed attempts |

**The one-sentence version:** a queue turns a synchronous, fragile, slow call chain into an asynchronous, retryable, spike-absorbing pipeline — at the cost of no longer knowing, at request time, whether the work actually succeeded.

---

## Key concepts inside this topic

### 1. The vocabulary (learn these, they're every interview's first 3 minutes)

- **Producer** — publishes messages. Your API server.
- **Broker** — the queue server. Kafka, RabbitMQ, AWS SQS, Redis (via BullMQ), NATS.
- **Consumer** — reads messages and does work. Your worker process.
- **Queue** — a named buffer. Messages are removed once processed.
- **Topic** — a named *stream* of messages (Kafka's word). Messages are NOT removed on read; they expire on a retention policy.
- **Partition** — a topic is split into N partitions for parallelism. Each partition is an **ordered, append-only log**.
- **Offset** — a message's position within a partition (0, 1, 2, ...). The consumer remembers "I've read up to offset 5,132."
- **Consumer group** — N workers cooperating. Each partition is assigned to exactly one consumer in the group, so each message is handled once *per group*. Add a second group and it gets its own independent copy of the stream.
- **Ack** — "I processed this successfully, delete/advance."
- **Nack** — "I failed. Requeue it."
- **Visibility timeout** — when a consumer picks up a message, the broker hides it from other consumers for N seconds. If the consumer doesn't ack within N seconds (it crashed, it's slow), the message becomes visible again and someone else gets it. **This is why crashes don't lose work — and also why slow consumers cause duplicate processing.**
- **Dead-letter queue (DLQ)** — after `maxAttempts` failures, the message is moved here instead of retried forever. A human (or an alert) looks at it.

### 2. The before/after that justifies the whole topic

**Before — everything on the request path:**

```javascript
// BAD: the user waits for all of this
app.post('/photos', async (req, res) => {
  const url = await s3.upload(req.file);           //  400 ms
  await generateThumbnails(url);                   // 6000 ms  <-- CPU-bound, blocking
  await moderationApi.classify(url);               // 1500 ms  <-- 3rd party, can be down
  await email.send(req.user, 'photo_updated');     //  300 ms  <-- can be down
  await cdn.purge(url);                            //  200 ms
  res.json({ url });                               // total ~8400 ms
});
```

p99 latency: **8.4 s**. Availability: the product of every downstream's availability. If the email provider is at 99.5%, your upload endpoint can never be better than 99.5%.

**After — only the essential work is synchronous:**

```javascript
// GOOD: the user waits for the upload and nothing else
app.post('/photos', async (req, res) => {
  const url = await s3.upload(req.file);           // 400 ms — genuinely required
  const photo = await db.photos.insert({ url, userId: req.user.id, status: 'PROCESSING' });

  // Enqueue is ~2 ms. It's a network write to Redis/Kafka, not real work.
  await photoQueue.add('process-photo', { photoId: photo.id, url });

  res.status(202).json({ photoId: photo.id, status: 'PROCESSING' });  // ~420 ms
});
```

p99 latency: **~420 ms** — a **20x improvement**. Availability: only depends on S3 + your DB + the broker. The email provider being down no longer breaks uploads.

Note the `202 Accepted` and the `status: 'PROCESSING'`. That's the honest contract: **"I have accepted your work, I have not finished it."** The client polls, or you push a WebSocket event when it's done.

### 3. Delivery semantics — done properly

This is the part people get wrong in interviews. There are three theoretical guarantees:

**At-most-once** — ack the message *before* doing the work.
```
receive → ack → process
                  ↑ crash here = message is gone forever
```
Fast, never duplicates, **can lose data**. Fine for metrics/telemetry where dropping 0.01% of samples doesn't matter. Never for payments.

**At-least-once** — ack *after* the work succeeds.
```
receive → process → ack
                     ↑ crash here = the visibility timeout expires,
                       the message is redelivered, you process it TWICE
```
Never loses data, **can duplicate**. This is the default in Kafka, SQS, RabbitMQ, BullMQ — essentially everywhere.

**Exactly-once** — every message processed once and only once. Everyone wants it. Almost nobody has it.

**Why exactly-once is essentially impossible end-to-end:**

The consumer does two separate things: (a) the side effect (charge the card, write the row, send the email) and (b) the ack to the broker. These live in **two different systems**. To be exactly-once you'd need them to commit atomically. They can't, because there's no shared transaction across "Stripe's servers" and "the Kafka broker."

So there's always a gap:

```
process (charge card ✅) ────X crash────  ack never sent
                                          → redelivered → card charged AGAIN
```

Reorder it and you get the other failure:

```
ack sent ✅ ────X crash──── process never ran
                            → message gone, card never charged
```

You must pick which failure you prefer. That's the two-generals problem wearing a different hat, and no amount of broker configuration removes it.

> "But Kafka has exactly-once semantics!" — Kafka has exactly-once *within Kafka*: read from topic A, transform, write to topic B, commit the offset, all in one Kafka transaction. That works because both sides are Kafka. The moment your consumer calls Stripe or writes to Postgres, that guarantee stops at the boundary.

**The real-world answer, and the one you say in the interview:**

> **At-least-once delivery + idempotent consumers = effectively-exactly-once.**

You accept duplicates at the transport layer and make them *harmless* at the application layer. The consumer records "I already handled message `evt_abc123`" and short-circuits on the second delivery.

```javascript
// Idempotent consumer: safe to run the same message 5 times
async function handleCharge(msg) {
  // A unique constraint on processed_messages.message_id does the heavy lifting.
  const claimed = await db.query(
    `INSERT INTO processed_messages (message_id, handled_at)
     VALUES ($1, NOW()) ON CONFLICT (message_id) DO NOTHING RETURNING message_id`,
    [msg.id]
  );
  if (claimed.rowCount === 0) return;  // already handled — this is a duplicate, drop it

  await stripe.charges.create({ amount: msg.amount, idempotencyKey: msg.id });
}
```

We'll go deep on this in [85 — Idempotency](./85-idempotency.md). For now: **remember the phrase, and remember why.**

### 4. Queue (RabbitMQ) vs Log (Kafka) — the key mental model

These two get lumped together as "message queues" and they are fundamentally different data structures.

**RabbitMQ is a queue.** A message arrives, the broker **pushes** it to a consumer, the consumer acks, and the broker **deletes it**. The message is gone. The broker tracks per-message state (delivered? acked?). The broker also does clever **routing**: exchanges can fan a message out by topic pattern (`orders.*.eu`), so one publish can land in six different queues.

```
                          ┌──────────────┐
producer ──publish──▶ │   EXCHANGE   │──routing key──┬──▶ [queue A] ──push──▶ consumer 1
                          └──────────────┘               ├──▶ [queue B] ──push──▶ consumer 2
                                                          └──▶ [queue C] ──push──▶ consumer 3
                              Message is DELETED from a queue once acked.
```

**Kafka is a log.** A message is **appended** to a partition file and stays there for the retention period (7 days, 30 days, forever — your choice), whether or not anyone read it. Consumers **pull**, and each consumer group stores its own **offset** — a bookmark. Kafka's broker is dumb and fast: it doesn't track per-message delivery, it just serves byte ranges from a file.

```
Topic "orders", partition 0 — an append-only log:

  offset:   0     1     2     3     4     5     6     7     8
          ┌────┬────┬────┬────┬────┬────┬────┬────┬────┐
          │ m0 │ m1 │ m2 │ m3 │ m4 │ m5 │ m6 │ m7 │ m8 │ ◀── producers append here
          └────┴────┴────┴────┴────┴────┴────┴────┴────┘
                            ▲                   ▲
                            │                   │
              group "billing" is here    group "analytics" is here
                    (offset = 3)              (offset = 7)

  Nothing is deleted on read. Rewind billing to offset 0 and it replays everything.
```

**That "rewind" is the superpower.** You shipped a bug in your billing consumer that mis-computed tax for 6 hours. With RabbitMQ, those messages are gone — acked and deleted. With Kafka, you fix the code, reset the consumer group offset to where the bug started, and **replay**. Same for adding a brand-new consumer six months later: it can read the entire history from offset 0.

| Dimension | RabbitMQ (queue) | Kafka (log) |
|---|---|---|
| **Data structure** | Queue — messages deleted on ack | Append-only log — messages retained by policy |
| **Delivery model** | Broker **pushes** to consumers | Consumers **pull** at their own pace |
| **Who tracks progress** | The broker (per-message state) | The consumer (an offset per partition) |
| **Replay / rewind** | ❌ No — it's gone once acked | ✅ Yes — reset the offset, reprocess history |
| **Ordering** | Per-queue, but lost with competing consumers | Strict **within a partition** |
| **Throughput** | ~tens of thousands msg/s per node | ~millions msg/s per cluster (sequential disk I/O, zero-copy) |
| **Routing flexibility** | ✅ Excellent — exchanges, topic/header/fanout routing | ❌ Basic — you subscribe to a topic, filtering is the consumer's job |
| **Fanout to N independent readers** | Bind N queues to one exchange | Just add N consumer groups — free, no data duplication |
| **Per-message TTL, priority, delay** | ✅ Built in | ❌ Not natively (needs workarounds) |
| **Typical use case** | Task/job queues, RPC, complex routing, "do this one thing" | Event streaming, analytics pipelines, event sourcing, cross-service event bus |
| **The one-liner** | "A smart broker with dumb consumers" | "A dumb broker with smart consumers" |

**When to choose which:**
- **Background jobs** (send email, resize image, generate PDF): RabbitMQ, SQS, or BullMQ. You want per-job retries, delays, priorities. You don't need to replay a thumbnail from 3 weeks ago.
- **Event streams** (user activity, order events, anything 5 teams want to consume differently): Kafka. Multiple independent consumer groups, replay, high throughput.
- Many companies run **both**, and that's a perfectly good interview answer.

### 5. Ordering — and why partition keys decide your fate

Kafka guarantees ordering **only within a single partition**. Across partitions, all bets are off — they're separate files being read by separate consumers at different speeds.

So if you publish `AccountCreated` to partition 3 and `AccountDeleted` to partition 7, a consumer can legitimately see the delete before the create.

The fix: **the partition key**. Kafka hashes the key to pick the partition, so **the same key always lands on the same partition**, and therefore stays in order.

```javascript
// All events for one account go to the same partition → strictly ordered for that account.
await producer.send({
  topic: 'account-events',
  messages: [{ key: String(accountId), value: JSON.stringify(event) }],
  //           ^^^^^^^^^^^^^^^^^^^^^^ this is the whole ballgame
});
```

**Choosing the key is a real design decision:**

| Key choice | Ordering you get | Risk |
|---|---|---|
| `accountId` | All events for one account, in order | A whale account = a **hot partition** (one consumer melts) |
| `orderId` | All events for one order, in order | Usually great — high cardinality, even spread |
| `region` | All events in a region, in order | Terrible — 4 regions = only 4 usable partitions |
| `null` (random) | **No ordering at all** | Perfect spread, zero guarantees |

**Rule:** pick the smallest entity whose events must be ordered relative to each other. Usually that's the aggregate ID (order, user, account) — not the region, not the event type.

And note the ceiling this creates: **max useful consumers in a group = number of partitions.** 12 partitions means the 13th consumer sits idle. Over-provision partitions a little at creation time; adding them later reshuffles the key→partition mapping and breaks your ordering guarantee for in-flight keys.

### 6. Retries, backoff, jitter, and the DLQ

A failed job should not be retried immediately, and it should not be retried forever.

**Exponential backoff:** wait 1s, 2s, 4s, 8s, 16s. If a downstream is overloaded, hammering it every 100 ms makes it worse. Backing off gives it room to recover.

**Jitter:** if 5,000 jobs all fail at the same moment (the database went down), and they all back off by exactly 4 s, then in exactly 4 s they all retry **at the same instant** — a thundering herd that knocks the database back over. Add randomness.

```javascript
// Full jitter: pick a random delay in [0, cap]. Spreads the herd across the window.
function backoffMs(attempt) {
  const base = 1000;             // 1 s
  const cap = 60_000;            // never wait more than a minute
  const exp = Math.min(cap, base * 2 ** attempt);   // 1s, 2s, 4s, 8s, 16s, 32s, 60s...
  return Math.floor(Math.random() * exp);           // full jitter
}
```

**Retry only what's retryable.** A 503 from the payment gateway is transient — retry. A 400 "invalid card number" will fail identically forever — retrying it 20 times just wastes the DLQ's time. Classify errors.

**Poison messages** are messages that can *never* succeed: malformed JSON, a reference to a deleted row, a bug that throws on a specific edge case. Without a retry cap, a poison message loops forever, burning CPU and — if it's at the head of an ordered partition — **blocking every message behind it**. This is the classic Kafka outage: one bad message, one consumer stuck in a crash-restart loop, and lag climbing to millions.

**The DLQ is the escape hatch:** after `maxAttempts`, stop retrying and move the message to a separate queue. The pipeline keeps flowing. A human inspects the DLQ, fixes the bug or the data, and replays it.

**Alert on DLQ depth > 0.** A DLQ that nobody watches is a silent data-loss bucket.

### 7. Backpressure and queue depth — your #1 metric

The queue absorbs spikes. But it can only absorb them if the *average* consumer rate exceeds the *average* producer rate.

- Producers: 1,000 msg/s sustained
- Consumers: 800 msg/s sustained
- Queue grows by **200 msg/s forever** → 720,000 messages backed up after an hour → your "real-time" notification arrives tomorrow → eventually the broker runs out of disk.

A queue does **not** fix an under-provisioned consumer. It only fixes *bursty* traffic. If your steady-state rate exceeds your consumer's capacity, the queue just changes a fast failure into a slow, invisible one.

**So the metric that matters is queue depth (Kafka calls it consumer lag): the number of messages waiting.**

- **Depth flat and low** → healthy.
- **Depth spiking then draining** → healthy; the queue is doing its job.
- **Depth monotonically increasing** → you are underwater. Scale consumers, or shed load.

Derived metric, and the one to put on the dashboard: **time-to-drain = depth / consumption rate.** "We have 400,000 messages and drain 2,000/s → 200 seconds of backlog." That number is what your on-call actually cares about.

Also **cap producers.** If the broker is full or the queue depth exceeds a threshold, the producer should start rejecting or shedding — otherwise you push the failure into the broker's disk and lose everything at once.

### 8. A complete Node.js worker: producer, consumer, ack, retry, DLQ

Using **BullMQ** (Redis-backed — the standard Node job queue). The concepts map 1:1 to SQS and RabbitMQ.

```javascript
// ─────────────────────────────────────────────────────────────
// queue.js — shared setup
// ─────────────────────────────────────────────────────────────
import { Queue, Worker, QueueEvents } from 'bullmq';

const connection = { host: '127.0.0.1', port: 6379 };

export const photoQueue = new Queue('photo-processing', { connection });
export const photoDLQ   = new Queue('photo-processing-dlq', { connection });
```

```javascript
// ─────────────────────────────────────────────────────────────
// producer.js — the API server. Fast, and does NOT do the work.
// ─────────────────────────────────────────────────────────────
import express from 'express';
import { photoQueue } from './queue.js';

const app = express();

app.post('/photos', async (req, res) => {
  const url = await s3.upload(req.file);
  const photo = await db.photos.insert({ url, userId: req.user.id, status: 'PROCESSING' });

  await photoQueue.add(
    'process-photo',
    { photoId: photo.id, url, userId: req.user.id },
    {
      // jobId makes the ENQUEUE itself idempotent: a client retrying POST /photos
      // twice with the same photo won't create two jobs.
      jobId: `photo:${photo.id}`,
      attempts: 5,                                  // 1 try + 4 retries, then it's dead
      backoff: { type: 'exponential', delay: 1000 },// 1s, 2s, 4s, 8s (+ BullMQ jitter)
      removeOnComplete: 1000,                       // keep last 1k for debugging
      removeOnFail: false,                          // keep failures so we can inspect them
    }
  );

  // 202 = "accepted, not finished". The honest status code for async work.
  res.status(202).json({ photoId: photo.id, status: 'PROCESSING' });
});

app.listen(3000);
```

```javascript
// ─────────────────────────────────────────────────────────────
// consumer.js — the worker. Slow work lives here.
// ─────────────────────────────────────────────────────────────
import { Worker, UnrecoverableError } from 'bullmq';
import { photoDLQ } from './queue.js';

const connection = { host: '127.0.0.1', port: 6379 };

const worker = new Worker(
  'photo-processing',
  async (job) => {
    const { photoId, url, userId } = job.data;

    // ── Idempotency guard ──────────────────────────────────────
    // At-least-once delivery means this handler WILL sometimes run twice for the
    // same job (worker crashed after the work but before the ack). Make that safe.
    const already = await db.photos.findOne({ id: photoId, status: 'READY' });
    if (already) {
      console.log(`[skip] photo ${photoId} already processed — duplicate delivery`);
      return;
    }

    // ── Non-retryable errors: fail FAST, don't burn 5 attempts ──
    const photo = await db.photos.findOne({ id: photoId });
    if (!photo) {
      // The row was deleted. No number of retries will bring it back.
      throw new UnrecoverableError(`photo ${photoId} no longer exists`);
    }

    // ── The actual slow work ───────────────────────────────────
    await generateThumbnails(url, ['128', '512', '1024']);  // ~6 s
    await job.updateProgress(50);                            // heartbeat: extends the lock

    const verdict = await moderationApi.classify(url);       // ~1.5 s, 3rd party
    if (verdict.blocked) {
      await db.photos.update(photoId, { status: 'BLOCKED' });
      return;                                                // success — a decision was made
    }

    await db.photos.update(photoId, { status: 'READY' });
    await cdn.purge(url);
    await notifications.push(userId, 'Your photo is ready');
    // Returning normally = ACK. BullMQ removes the job.
    // Throwing = NACK. BullMQ requeues it with backoff, or fails it after `attempts`.
  },
  {
    connection,
    concurrency: 5,          // 5 jobs in flight per worker process
    lockDuration: 60_000,    // the visibility timeout: if we don't heartbeat within
                             // 60 s, another worker assumes we died and takes the job
  }
);

// ── The dead-letter queue ────────────────────────────────────
// Fires when a job has exhausted every attempt (or hit an UnrecoverableError).
worker.on('failed', async (job, err) => {
  if (!job) return;
  const exhausted = job.attemptsMade >= (job.opts.attempts ?? 1);
  if (!exhausted) {
    console.warn(`[retry ${job.attemptsMade}] ${job.id}: ${err.message}`);
    return;   // BullMQ will retry it with backoff — nothing to do
  }

  console.error(`[DLQ] ${job.id} died after ${job.attemptsMade} attempts: ${err.message}`);
  await photoDLQ.add('dead', {
    originalJobId: job.id,
    payload: job.data,
    error: err.message,
    stack: err.stack,
    attempts: job.attemptsMade,
    diedAt: new Date().toISOString(),
  });
  await db.photos.update(job.data.photoId, { status: 'FAILED' });
  metrics.increment('photo.dlq');   // ← alert on this
});

// ── Backpressure monitoring ──────────────────────────────────
setInterval(async () => {
  const counts = await photoQueue.getJobCounts('waiting', 'active', 'failed');
  metrics.gauge('photo.queue.depth', counts.waiting);
  if (counts.waiting > 10_000) {
    console.error(`BACKPRESSURE: ${counts.waiting} jobs waiting — scale up workers`);
  }
}, 10_000);

// Graceful shutdown: finish in-flight jobs before dying, so nothing is
// abandoned mid-work and forced to wait out the full lock duration.
process.on('SIGTERM', async () => { await worker.close(); process.exit(0); });
```

**Draining the DLQ once you've shipped the fix:**

```javascript
// replay-dlq.js — run manually after fixing the bug
import { photoQueue, photoDLQ } from './queue.js';

const dead = await photoDLQ.getJobs(['waiting'], 0, 1000);
for (const job of dead) {
  await photoQueue.add('process-photo', job.data.payload, {
    jobId: `${job.data.originalJobId}:replay`,
    attempts: 3,
    backoff: { type: 'exponential', delay: 2000 },
  });
  await job.remove();
}
console.log(`replayed ${dead.length} messages`);
```

For **RabbitMQ** with `amqplib`, the shape is identical — the ack is just explicit:

```javascript
import amqp from 'amqplib';

const conn = await amqp.connect('amqp://localhost');
const ch = await conn.createChannel();

await ch.assertExchange('photos', 'topic', { durable: true });
await ch.assertQueue('photo.process', {
  durable: true,
  deadLetterExchange: 'photos.dlx',   // where nacked/expired messages go
});
await ch.bindQueue('photo.process', 'photos', 'photo.uploaded.*');

await ch.prefetch(5);   // BACKPRESSURE: never hold more than 5 unacked messages.
                        // Without this, RabbitMQ shoves the whole queue at one consumer.

ch.consume('photo.process', async (msg) => {
  try {
    await handle(JSON.parse(msg.content.toString()));
    ch.ack(msg);                       // done — delete it
  } catch (err) {
    const attempts = (msg.properties.headers['x-attempts'] ?? 0) + 1;
    if (attempts >= 5) {
      ch.nack(msg, false, /* requeue */ false);   // requeue=false → routed to the DLX
    } else {
      ch.nack(msg, false, /* requeue */ true);    // back on the queue for another go
    }
  }
});
```

The one line to notice is `ch.prefetch(5)`. Without it, a single consumer will happily accept 200,000 unacked messages into memory and fall over. **Prefetch is backpressure.**

---

## Visual / Diagram description

### Diagram 1: The basic async pipeline

```
   ┌──────────┐
   │  Client   │
   └─────┬────┘
         │ POST /photos
         ▼
   ┌─────────────────────┐        ┌──────────────────────────────────┐
   │    API Server        │ enqueue│            BROKER                 │
   │   (PRODUCER)         │───────▶│   Queue: "photo-processing"       │
   │                      │  ~2 ms │  ┌──┬──┬──┬──┬──┬──┬──┬──┐        │
   │  1. upload to S3     │        │  │j1│j2│j3│j4│j5│j6│j7│j8│ ...    │
   │  2. insert DB row    │        │  └──┴──┴──┴──┴──┴──┴──┴──┘        │
   │  3. enqueue job      │        │        DEPTH = 8  ◀── ALERT ON ME  │
   │  4. return 202 ✅     │        └──────┬─────────────┬──────────────┘
   └─────┬────────────────┘               │ pull        │ pull
         │ 202 Accepted                   ▼             ▼
         │ { photoId, PROCESSING }   ┌─────────┐  ┌─────────┐
         ▼                           │Worker 1 │  │Worker 2 │  (CONSUMER GROUP)
   ┌──────────┐                      │         │  │         │
   │  Client   │◀─── WebSocket ──────│ resize  │  │ resize  │
   │           │     "photo ready"   │ moderate│  │ moderate│
   └──────────┘                      └────┬────┘  └────┬────┘
                                          │ fail x5    │ ack ✅
                                          ▼            ▼
                                    ┌──────────┐   (job deleted)
                                    │   DLQ     │
                                    │ ┌──┬──┐   │  ← humans look here
                                    │ │j3│j9│   │  ← alert if depth > 0
                                    │ └──┴──┘   │
                                    └──────────┘
```

The critical path is only the top-left box: upload, insert, enqueue, return. Everything below the broker happens on someone else's clock. The client learns about completion by polling `GET /photos/:id` or over a WebSocket.

### Diagram 2: Queue vs Log — draw this one on the whiteboard

```
 ═══════════════ RABBITMQ: A QUEUE (messages are CONSUMED) ═══════════════

  producer ──▶ ┌──────────┐  routing key    ┌───────────┐
               │ EXCHANGE  │─── "order.eu" ─▶│ queue: eu │─push─▶ consumer A ─ack─▶ ✂ DELETED
               │ (topic)   │─── "order.us" ─▶│ queue: us │─push─▶ consumer B ─ack─▶ ✂ DELETED
               └──────────┘                 └───────────┘
  • Broker pushes.  • Broker tracks each message.  • Once acked, it is GONE.
  • Rich routing.   • No replay. Ever.


 ═══════════════ KAFKA: A LOG (messages are RETAINED) ════════════════════

  Topic "orders", 3 partitions. key = orderId → hash → partition.

  P0: ┌───┬───┬───┬───┬───┬───┬───┐          append ▶
      │ 0 │ 1 │ 2 │ 3 │ 4 │ 5 │ 6 │
      └───┴───┴─▲─┴───┴───┴─▲─┴───┘
                │           │
      group "billing"   group "analytics"
        offset=2          offset=5

  P1: ┌───┬───┬───┬───┬───┐              P2: ┌───┬───┬───┬───┐
      │ 0 │ 1 │ 2 │ 3 │ 4 │                  │ 0 │ 1 │ 2 │ 3 │
      └───┴───┴───┴─▲─┴───┘                  └───┴─▲─┴───┴───┘
                    │                              │
              billing offset=3               billing offset=1

  • Consumers pull.  • Consumers own their offsets.  • Nothing deleted on read.
  • Rewind billing to offset 0 → REPLAY the entire history.
  • Ordering is guaranteed WITHIN a partition only. Never across partitions.
  • Add a new consumer group tomorrow → it reads all of history for free.
```

The single most important visual difference: in RabbitMQ the arrow between broker and consumer means "the message moved and is now gone." In Kafka it means "the consumer read a byte range and moved its own bookmark." That's why only one of them can rewind.

### Diagram 3: The retry ladder

```
   attempt 1 ──✗ 503 ──▶ wait rand(0..1s)  ──┐
                                              │
   attempt 2 ──✗ 503 ──▶ wait rand(0..2s)  ──┤   exponential backoff
                                              │   + full jitter
   attempt 3 ──✗ 503 ──▶ wait rand(0..4s)  ──┤   (jitter prevents 5,000 jobs
                                              │    from all retrying at once)
   attempt 4 ──✗ 503 ──▶ wait rand(0..8s)  ──┤
                                              │
   attempt 5 ──✗ 503 ─────────────────────────┘
        │
        ▼
   ┌─────────────────────┐
   │  DEAD-LETTER QUEUE   │  ← the pipeline keeps flowing.
   │  payload + error +   │  ← a human/alert investigates.
   │  stack + attempts    │  ← replay after the fix ships.
   └─────────────────────┘

   Shortcut: a 400 "invalid card" is NOT retryable →
             UnrecoverableError → straight to the DLQ, skip attempts 2–5.
```

---

## Real world examples

### 1. Uber — Kafka as the company-wide nervous system

Uber runs one of the largest Kafka deployments in the world — trillions of messages a day. Every trip event (`requested`, `driver_assigned`, `started`, `completed`) is published to Kafka once, and then consumed independently by many teams: dynamic pricing, fraud detection, driver payouts, the data warehouse, and the analytics pipeline all read the *same* stream through separate consumer groups.

This is the fanout property in action. If the payments team wants to change how they compute a fare, they don't ask the trip service to send them anything new — they just add a consumer group and replay. Uber also publishes an open-source **uReplicator** for cross-datacenter Kafka mirroring, because that stream is too important to live in one region.

### 2. Slack — job queues in front of everything user-facing

Slack's message-send path historically pushed a large amount of work into an asynchronous job queue: push notifications, unfurling links, search indexing, billing counters, webhook delivery. The user's message appears instantly in the channel; the notification to your phone and the link preview arrive a moment later.

They famously had to migrate this from a Redis-based job queue to a **Kafka-fronted** one, because during incidents the Redis queue could fill up faster than it drained and there was no durable place to spill to. The lesson is exactly the backpressure point above: a queue that runs out of room turns a slow system into a data-loss event.

### 3. Netflix — event-driven encoding

When a studio delivers a new title, Netflix doesn't encode it in a request handler. The upload triggers an event, and a pipeline of workers fans out to encode hundreds of variants (resolutions, bitrates, codecs, audio tracks, subtitle burn-ins) in parallel across a large worker fleet. Each encode is a job with retries and a DLQ; a single failed 4K HDR variant doesn't ruin the whole title.

This is the load-levelling property: a studio drop is extremely bursty. The queue lets Netflix run a fixed-size (or autoscaled) encoding fleet at high utilisation instead of provisioning for the peak.

---

## Trade-offs

| What you gain with a queue | What it costs you |
|---|---|
| Fast API responses (8.4 s → 420 ms) | The client no longer knows if the work *succeeded* — you must add status polling or push |
| The API survives downstream outages | **Eventual consistency**: the DB says PROCESSING while the user swears they uploaded it |
| Spike absorption (load levelling) | Doesn't help if your *steady-state* rate exceeds consumer capacity — it just hides it |
| Free retries with backoff | Duplicates. At-least-once means **you must write idempotent consumers** |
| Add new consumers without touching the producer | Harder debugging — no single stack trace spans producer and consumer |
| Independent scaling of API vs workers | A whole new stateful system to run, monitor, upgrade, and page someone about |

| Broker choice | Pick it when | Give up |
|---|---|---|
| **Redis / BullMQ** | Node shop, job queue, < ~50k jobs/s, you already run Redis | Durability guarantees are weaker than Kafka; not an event log |
| **RabbitMQ** | You need complex routing, priorities, per-message TTL, delayed jobs | Replay, and throughput past ~100k msg/s |
| **AWS SQS** | You want zero ops and are already on AWS | Ordering (unless FIFO queues, which cap at ~3k msg/s), replay |
| **Kafka** | Event streaming, multiple consumers of one stream, replay, huge volume | Operational complexity, no per-message priority/TTL/delay |

**Rule of thumb:** if the work is *"do this one thing for this one user"* → job queue (BullMQ/SQS/RabbitMQ). If the work is *"this happened, and several teams care"* → log (Kafka). And whatever you pick: **assume at-least-once, and make every consumer idempotent.** That single habit prevents more production incidents than any broker feature.

---

## Common interview questions on this topic

### Q1: "What's the difference between Kafka and RabbitMQ?"
**Hint:** Don't list features — give the mental model. RabbitMQ is a **queue**: the broker pushes, tracks per-message state, and *deletes on ack*. Kafka is an **append-only log**: consumers pull, own their offsets, and messages are retained by a time/size policy, so you can **replay**. Then name the consequences: Kafka gives you replay, huge throughput, and free fanout to N consumer groups; RabbitMQ gives you rich routing, priorities, TTLs, and delayed messages. Job queue → RabbitMQ. Event stream → Kafka.

### Q2: "Explain at-most-once, at-least-once, and exactly-once. Which one should I use?"
**Hint:** It's decided by *when you ack*. Ack-before-process = at-most-once (can lose). Ack-after-process = at-least-once (can duplicate). Exactly-once end-to-end is essentially impossible because the side effect (charging a card) and the ack (to the broker) live in two different systems with no shared transaction — a crash between them always leaves a gap. Kafka's "exactly-once" only holds *inside Kafka* (read-transform-write). The production answer: **at-least-once delivery + idempotent consumers**, using a dedupe key with a unique constraint.

### Q3: "Your Kafka consumer lag is growing steadily. Walk me through what you'd do."
**Hint:** First establish the shape: growing steadily means consumption rate < production rate — this is not a spike, it's under-provisioning or a stall. Check (1) is a consumer stuck in a crash loop on a **poison message**? (2) did a downstream dependency get slow, so each message now takes 300 ms instead of 5 ms? (3) is the load skewed onto one **hot partition** because of a bad partition key? Fixes: scale consumers — but only up to the partition count, that's the ceiling; increase concurrency per consumer; move slow I/O off the hot path; add partitions (and accept the rebalance). Short-term: shed load or pause non-critical producers.

### Q4: "How do you guarantee message ordering?"
**Hint:** You don't get global ordering in any distributed queue — accept that up front. Kafka gives you ordering **within a partition**, so you get ordering by *choosing the partition key*: same key → same partition → strict order. Pick the smallest entity that needs ordering (orderId, accountId), not something coarse like `region` (which collapses your parallelism) . Mention the two costs: a hot key becomes a hot partition, and your max parallelism equals your partition count. In RabbitMQ, ordering holds only with a single consumer per queue — the moment you add competing consumers, ordering is gone.

### Q5: "What's a dead-letter queue and why do you need one?"
**Hint:** A separate queue that messages are moved to after N failed attempts. Without it, a **poison message** — one that can never succeed — retries forever, and in an ordered partition it *blocks everything behind it*, so lag climbs to infinity from a single bad record. The DLQ keeps the pipeline flowing while preserving the failed message with its error and payload for a human. Then say the thing that shows you've operated one: **alert when DLQ depth > 0** — an unwatched DLQ is just a silent data-loss bucket.

---

## Practice exercise

### Move the Notification Path Off the Request

You're given this endpoint. It's a real one — every part of it is on the critical path.

```javascript
app.post('/orders', async (req, res) => {
  const order = await db.orders.insert(req.body);           //   30 ms
  await inventory.reserve(order.items);                     //  120 ms
  await paymentGateway.charge(order.total, req.body.card);  // 2500 ms
  await email.sendConfirmation(order);                      //  800 ms
  await sms.send(order.phone, 'Order placed!');             //  600 ms
  await analytics.track('order_placed', order);             //  200 ms
  await warehouse.notify(order);                            // 1500 ms
  res.json(order);
});
```

**Produce these five things:**

1. **The split.** For each of the 7 steps, decide: does it stay **synchronous** (the user genuinely cannot get a correct answer without it) or move to a **queue**? Write one sentence of justification per step. (Hint: the payment charge is the interesting one. Argue it either way — but argue it.)

2. **The new latency.** Compute p50 latency before and after. Show the arithmetic.

3. **The rewritten handler**, in Node, returning the right status code with the right body. Include the `jobId` that makes enqueueing idempotent.

4. **One consumer** — pick the email one — written with BullMQ, including: an idempotency guard, exponential backoff with jitter, a max attempt count, non-retryable error handling (what does a `422 invalid email address` from the provider mean here?), and the DLQ hook.

5. **The failure story.** Answer in prose: the SMS provider is down for 20 minutes. Exactly what does the user see? What does the queue do? What does your monitoring say? And when the provider comes back — what happens to the 12,000 SMS jobs that piled up, and how do you keep the recovery from becoming a thundering herd?

Aim for ~30 minutes. Part 5 is the one interviewers actually care about.

---

## Quick reference cheat sheet

- **Why queues exist:** decoupling, load levelling (absorb spikes), free retries, and getting slow work OFF the request path.
- **The pitch in one number:** 8.4 s synchronous → 420 ms with a queue. Return `202 Accepted`, not `200 OK`.
- **The vocabulary:** producer → broker → consumer; queue/topic, partition, offset, consumer group, ack/nack, visibility timeout, DLQ.
- **Visibility timeout** is what makes crashes safe: unacked after N seconds → redelivered to someone else. It's also why duplicates exist.
- **Delivery semantics are decided by WHEN you ack.** Before = at-most-once (lossy). After = at-least-once (duplicates).
- **Exactly-once end-to-end is a myth** — the side effect and the ack are in two different systems. Kafka's version only holds inside Kafka.
- **The real answer: at-least-once + idempotent consumers.** Memorise this sentence. See [85 — Idempotency](./85-idempotency.md).
- **RabbitMQ = queue** (broker pushes, deletes on ack, rich routing, no replay). **Kafka = log** (consumers pull, own offsets, retained, replayable, massive throughput).
- **Ordering exists only within a Kafka partition.** The partition key *is* your ordering guarantee — choose the aggregate ID, and beware hot keys.
- **Max consumers in a group = number of partitions.** The 13th consumer on 12 partitions does nothing.
- **Retry with exponential backoff + jitter.** No jitter = a thundering herd retrying in lockstep.
- **Poison message:** a message that can never succeed. Unbounded retries + ordering = it blocks the entire partition behind it.
- **DLQ after N attempts**, and **alert on DLQ depth > 0**. An unwatched DLQ is silent data loss.
- **Queue depth / consumer lag is your #1 metric.** Flat = healthy. Spiky-then-draining = healthy. Monotonically rising = you are underwater.
- **A queue does not fix an under-provisioned consumer.** It only absorbs bursts. Sustained producer rate > consumer rate = infinite backlog.
- **`prefetch` / `concurrency` is backpressure.** Without it, one consumer will pull the whole queue into memory and die.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [66 — Database Transactions and Isolation Levels](./66-database-transactions-and-isolation.md) — the atomicity guarantees that stop at your database's edge, which is exactly why exactly-once delivery is impossible |
| **Next** | [68 — Event-Driven Architecture](./68-event-driven-architecture.md) — what you build once you have a queue: events, event sourcing, CQRS, and the outbox pattern |
| **Related** | [85 — Idempotency](./85-idempotency.md) — the other half of "at-least-once + idempotent consumers"; without this, your queue will double-charge someone |
| **Related** | [87 — Batch vs Stream Processing](./87-batch-vs-stream-processing.md) — what you do with the Kafka log once it exists |
| **Related** | [80 — Monitoring and Observability](./80-monitoring-and-observability.md) — how you actually watch queue depth, lag, and DLQ size before your users do |
