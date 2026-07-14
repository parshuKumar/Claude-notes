# 51 — Producer-Consumer Pattern
## Category: LLD Patterns

---

## What is this?

The Producer-Consumer pattern **decouples the code that creates work from the code that does the work**, by putting a **shared bounded buffer** (a queue with a maximum size) between them. Producers drop items into the buffer. Consumers pull items out. Neither one ever calls the other directly, and neither one waits for the other to finish.

Think of a **busy restaurant**: waiters (producers) clip order tickets onto a rail above the pass. Cooks (consumers) grab tickets off the rail whenever they're free. A waiter never stands in the kitchen waiting for a steak to cook, and a cook never stands in the dining room waiting for someone to order. The **ticket rail is the buffer** — and crucially, it only holds so many tickets.

---

## Why does it matter?

Without this pattern, the thing that *creates* work and the thing that *does* work are welded together, and they must run at exactly the same speed. That is almost never true in a real system.

**What breaks without it:**
- **Bursty load kills you.** 5,000 signups arrive in one minute. If your HTTP handler sends the welcome email inline, all 5,000 requests block on a slow SMTP server and your API times out. With a buffer, you accept 5,000 items in milliseconds and drain them at whatever rate the email provider allows.
- **You can't scale the two sides independently.** Maybe you need 2 producers and 20 consumers. Without a buffer, they're the same code path — you can only scale them together.
- **You have no backpressure.** If work arrives faster than you can process it and you just keep accepting it into an unbounded array, you consume memory until the process dies with `JavaScript heap out of memory`. A **bounded** buffer gives you a place to *push back*.

**Interview angle:** This is the single most reused LLD pattern in system design. "Design a rate limiter", "design a web crawler", "design a logging pipeline", "design a job scheduler" — all of them have a producer-consumer core. Interviewers love asking "what happens when the queue is full?" because it separates people who've read about queues from people who've operated them.

**Real-work angle:** Every time you use Node streams, a `Bull`/`BullMQ` job queue, Kafka, RabbitMQ, SQS, or a `worker_threads` pool, you are using this pattern. Understanding the primitive means you can debug the framework.

---

## The core idea — explained simply

### The Restaurant Ticket Rail Analogy

Picture a restaurant at 8pm on a Saturday.

**Waiters** take orders. They write a ticket and clip it to a metal rail above the kitchen pass. Then they immediately walk away and take the next table's order. A waiter does not stand there watching the pasta boil.

**Cooks** work the line. When a cook finishes a dish, they turn around, pull the next ticket off the rail, and start cooking it. If the rail is empty, the cook wipes down their station and *waits* — they don't go hunting for waiters.

**The rail has a limit.** It physically fits about 20 tickets. When it's full, the head chef yells "STOP SEATING!" and the host stops letting new parties in. That yell is **backpressure** — the kitchen telling the front of house to slow down.

Now watch what this buys you:

- The restaurant absorbs a **burst** (a tour bus arrives) without the waiters freezing, because the rail holds the surge.
- You can **hire more cooks** without hiring more waiters. The rail doesn't care.
- If a cook calls in sick, tickets pile up on the rail — you *see* the problem, and the rail eventually fills and forces the host to slow seating. Nothing silently explodes.
- If a cook drops a plate, that one ticket fails. The other tickets are untouched.

### Mapping the analogy to the technical concept

| Restaurant | Producer-Consumer | In Node.js |
|---|---|---|
| Waiter writing a ticket | **Producer** — creates a unit of work | Your HTTP handler, a file reader, a Kafka poller |
| The ticket | **Item / job / message** | A JS object `{ type, payload }` |
| The ticket rail | **Bounded buffer / queue** | `BoundedQueue`, a stream's internal buffer, Redis list |
| Rail capacity (20 tickets) | **Buffer capacity** | `highWaterMark`, `maxSize` |
| Cook pulling a ticket | **Consumer** — processes one item | A worker function in a pool |
| Number of cooks on the line | **Consumer concurrency / pool size** | `N` workers draining the queue |
| Cook waiting at an empty station | **Consumer blocks on empty** | `await queue.take()` |
| Head chef yelling "stop seating" | **Backpressure** | `await queue.put()` blocks the producer |
| Host turning people away at the door | **Load shedding / dropping** | Return HTTP 503, drop oldest item |
| Rail so full tickets fall on the floor | **Unbounded queue → OOM crash** | The bug you're trying to prevent |

The whole pattern is one sentence: **put a bounded buffer between the two sides, and make both sides block when the buffer can't serve them.**

---

## Key concepts inside this topic

### 1. The problem it solves — start with the painful code

Here's a real-looking Node service with no buffer. It handles image uploads and generates thumbnails inline.

```javascript
// ❌ BAD — no buffer. Producer (HTTP) and consumer (thumbnailing) are welded together.
import express from 'express';
import sharp from 'sharp';

const app = express();

app.post('/upload', async (req, res) => {
  const buf = await readBody(req);

  // Thumbnailing is CPU-heavy: ~300ms per image.
  // The HTTP request cannot return until it's done.
  const thumb = await sharp(buf).resize(200, 200).toBuffer();
  await uploadToS3(thumb);

  res.json({ ok: true });
});
```

What's wrong:

1. **No burst absorption.** 200 uploads/sec × 300ms of CPU = 60 seconds of CPU work arriving every second. Requests queue up in the event loop and every single one times out.
2. **No independent scaling.** Thumbnailing needs CPU; HTTP handling needs almost none. You can't give them different resources.
3. **No backpressure.** Node happily accepts all 200 connections and holds all 200 image buffers in memory. At 4 MB each that's 800 MB — and climbing.
4. **Failure coupling.** S3 has a blip → the *HTTP layer* returns 500s.

Now the same thing with a buffer between the halves:

```javascript
// ✅ GOOD — bounded buffer decouples accept-work from do-work.
const queue = new BoundedQueue(100);          // holds at most 100 jobs

app.post('/upload', async (req, res) => {
  const buf = await readBody(req);

  // put() RESOLVES immediately if there's room, and BLOCKS (awaits) if full.
  // We don't want an HTTP request hanging forever, so we bound the wait:
  const accepted = await queue.tryPut({ buf }, { timeoutMs: 500 });
  if (!accepted) {
    // Backpressure surfaced to the client. This is a FEATURE, not a bug.
    return res.status(503).json({ error: 'busy, retry shortly' });
  }
  res.status(202).json({ ok: true, status: 'processing' });  // 202 Accepted
});

// Consumers run in the background, at their own pace.
startWorkerPool(queue, 4, async (job) => {
  const thumb = await sharp(job.buf).resize(200, 200).toBuffer();
  await uploadToS3(thumb);
});
```

The HTTP layer now returns in ~2ms. The thumbnailing runs at exactly 4-at-a-time forever, no matter how hard traffic spikes. And when we're genuinely overloaded, we say so honestly (503) instead of dying quietly.

### 2. The structure — who plays which role

| Participant | Role | Responsibility |
|---|---|---|
| **Producer** | Work creator | Calls `buffer.put(item)`. Must handle the case where `put` blocks or is rejected. Knows nothing about consumers. |
| **BoundedBuffer** | The rendezvous point | Holds at most `capacity` items. Blocks producers when full, blocks consumers when empty. The *only* shared state. |
| **Consumer** | Work processor | Loops: `item = await buffer.take()`, process, repeat. Knows nothing about producers. |
| **WorkerPool** | Consumer supervisor | Runs `N` consumers concurrently, handles per-item errors, supports graceful shutdown. |

The critical property: **producers and consumers have zero references to each other.** They only both reference the buffer. That's what "decoupled" means concretely — you could delete every consumer and the producers would still compile and run (they'd just eventually block).

### 3. Full JavaScript implementation — a blocking `BoundedQueue`

In a language with threads, you'd build this with a mutex and two condition variables. In JavaScript there are no threads on the main isolate — but there *is* `await`. So a "blocking" call is just **a function that returns a promise you don't resolve until the condition is met**.

The trick: keep two arrays of **waiters** — pending promise `resolve` functions.

```javascript
// bounded-queue.js
// A bounded, async-blocking FIFO queue.
//   put(item)  -> resolves when the item is in the buffer (blocks while full)
//   take()     -> resolves with an item (blocks while empty)

export class BoundedQueue {
  #items = [];              // the actual buffer (FIFO)
  #capacity;
  #takeWaiters = [];        // consumers parked on an empty queue: [{ resolve, reject }]
  #putWaiters = [];         // producers parked on a full queue:    [{ item, resolve, reject }]
  #closed = false;

  constructor(capacity = 16) {
    if (capacity < 1) throw new RangeError('capacity must be >= 1');
    this.#capacity = capacity;
  }

  get size() { return this.#items.length; }
  get capacity() { return this.#capacity; }
  get isFull() { return this.#items.length >= this.#capacity; }
  get isEmpty() { return this.#items.length === 0; }
  /** How many producers are currently blocked. This is your backpressure gauge. */
  get waitingProducers() { return this.#putWaiters.length; }

  /**
   * Add an item. Returns a promise that resolves once the item is accepted.
   * If the queue is full, the promise stays pending — that IS the backpressure.
   */
  put(item) {
    if (this.#closed) return Promise.reject(new Error('queue closed'));

    // Fast path 1: a consumer is already parked waiting. Hand the item straight
    // to it and skip the buffer entirely — no reason to store then immediately load.
    const waiter = this.#takeWaiters.shift();
    if (waiter) {
      waiter.resolve(item);
      return Promise.resolve();
    }

    // Fast path 2: there's room in the buffer.
    if (!this.isFull) {
      this.#items.push(item);
      return Promise.resolve();
    }

    // Slow path: FULL. Park the producer. We do NOT push the item — we hold it
    // in the waiter record and only admit it once a slot frees up. This keeps
    // memory bounded at capacity + (number of blocked producers).
    return new Promise((resolve, reject) => {
      this.#putWaiters.push({ item, resolve, reject });
    });
  }

  /**
   * Like put(), but gives up after timeoutMs. Returns true if accepted, false if not.
   * Use this on request paths where you must not hang a client forever.
   */
  async tryPut(item, { timeoutMs = 0 } = {}) {
    if (timeoutMs <= 0) {
      // Non-blocking variant: accept only if there's room right now.
      if (this.#closed || (this.isFull && this.#takeWaiters.length === 0)) return false;
      await this.put(item);
      return true;
    }

    let timer;
    const record = { settled: false };
    const timeout = new Promise((resolve) => {
      timer = setTimeout(() => { record.settled = true; resolve('timeout'); }, timeoutMs);
    });

    const accepted = this.put(item).then(() => 'ok', () => 'error');
    const outcome = await Promise.race([accepted, timeout]);
    clearTimeout(timer);

    if (outcome === 'timeout') {
      // Remove our parked waiter so we don't leak it (and so the item isn't
      // silently enqueued 3 seconds later, long after the client gave up).
      const i = this.#putWaiters.findIndex((w) => w.item === item);
      if (i !== -1) this.#putWaiters.splice(i, 1);
      return false;
    }
    return outcome === 'ok';
  }

  /**
   * Remove and return the oldest item. Blocks (awaits) while the queue is empty.
   * Rejects with a CLOSED sentinel once the queue is closed and drained.
   */
  take() {
    // Fast path: buffer has an item.
    if (!this.isEmpty) {
      const item = this.#items.shift();
      // A slot just opened — admit exactly one blocked producer, in FIFO order.
      this.#admitOneProducer();
      return Promise.resolve(item);
    }

    // Buffer empty, but a producer may be parked (only possible when capacity
    // is momentarily 0-length AND producers queued — defensive, keeps invariants).
    const parked = this.#putWaiters.shift();
    if (parked) {
      parked.resolve();
      return Promise.resolve(parked.item);
    }

    if (this.#closed) return Promise.reject(new QueueClosedError());

    // Slow path: EMPTY. Park the consumer.
    return new Promise((resolve, reject) => {
      this.#takeWaiters.push({ resolve, reject });
    });
  }

  #admitOneProducer() {
    if (this.isFull) return;
    const parked = this.#putWaiters.shift();
    if (!parked) return;
    this.#items.push(parked.item);
    parked.resolve();
  }

  /** Stop accepting new items. Consumers still drain what's left, then get CLOSED. */
  close() {
    this.#closed = true;
    // Wake every parked consumer so they can exit instead of hanging forever.
    for (const w of this.#takeWaiters) w.reject(new QueueClosedError());
    this.#takeWaiters = [];
    for (const p of this.#putWaiters) p.reject(new Error('queue closed'));
    this.#putWaiters = [];
  }
}

export class QueueClosedError extends Error {
  constructor() { super('QUEUE_CLOSED'); this.name = 'QueueClosedError'; }
}
```

**The one idea to internalize:** `take()` on an empty queue returns a promise that *nobody resolves yet*. It sits in `#takeWaiters`. Later, a `put()` finds it and calls `waiter.resolve(item)` — and the consumer's `await` wakes up. That is a blocking queue, built out of nothing but promises.

### 4. The worker pool — N consumers draining one queue

One consumer processes items one at a time. A **pool** runs `N` independent consumer loops against the *same* queue. Because the queue hands each item to exactly one `take()`, no two workers ever get the same item — you get work-stealing for free.

```javascript
// worker-pool.js
import { QueueClosedError } from './bounded-queue.js';

/**
 * Runs `size` concurrent consumer loops against `queue`.
 * Returns a handle with a promise that resolves when all workers have exited.
 */
export function startWorkerPool(queue, size, handler, { onError } = {}) {
  const stats = { processed: 0, failed: 0, active: 0 };

  async function worker(id) {
    for (;;) {
      let job;
      try {
        job = await queue.take();       // blocks here when there's nothing to do
      } catch (err) {
        if (err instanceof QueueClosedError) return;   // clean shutdown
        throw err;
      }

      stats.active++;
      try {
        await handler(job, id);
        stats.processed++;
      } catch (err) {
        stats.failed++;
        // A failing JOB must never kill the WORKER. Isolate the blast radius.
        (onError ?? ((e, j) => console.error('[worker]', id, e.message, j)))(err, job);
      } finally {
        stats.active--;
      }
    }
  }

  const workers = Array.from({ length: size }, (_, i) => worker(i + 1));
  return { stats, done: Promise.all(workers) };
}
```

And the whole thing running end to end:

```javascript
// main.js — a fast producer, a slow consumer pool, and a small buffer.
import { BoundedQueue } from './bounded-queue.js';
import { startWorkerPool } from './worker-pool.js';

const sleep = (ms) => new Promise((r) => setTimeout(r, ms));
const queue = new BoundedQueue(5);        // deliberately tiny, so we SEE backpressure

const pool = startWorkerPool(queue, 3, async (job, workerId) => {
  await sleep(120);                        // pretend this is a thumbnail / an email
  console.log(`  worker ${workerId} finished job ${job.id} (queue depth ${queue.size})`);
});

async function produce() {
  for (let id = 1; id <= 20; id++) {
    const t0 = Date.now();
    await queue.put({ id });               // <-- will BLOCK once we outrun the workers
    const blockedFor = Date.now() - t0;
    console.log(
      `produced ${id}` +
      (blockedFor > 5 ? `  (backpressure: producer blocked ${blockedFor}ms)` : '')
    );
  }
  queue.close();                           // no more work; let workers drain and exit
}

await produce();
await pool.done;
console.log('done:', pool.stats);          // { processed: 20, failed: 0, active: 0 }
```

Run it and you'll see jobs 1–5 produced instantly (they fill the buffer), then `produced 6 (backpressure: producer blocked 118ms)` — the producer is now **paced by the consumers**. That pacing, emerging automatically from a bounded buffer, is the whole point of the pattern.

### 5. Backpressure — what happens when producers outrun consumers

This is the question interviewers actually care about. If work arrives at 1,000/sec and you can process 600/sec, you are permanently 400/sec in debt. Something must give. You have exactly **four** options.

| Strategy | Mechanism | You keep | You lose | Use when |
|---|---|---|---|---|
| **Block** (the default above) | `await queue.put()` doesn't resolve until there's room | Every item. Perfect durability of intent. | Producer throughput and latency. If the producer is an HTTP handler, clients hang. | The producer is a batch job / file reader / internal pipeline where slowing down is fine. **This is the correct default.** |
| **Drop (load shed)** | `tryPut()` returns `false`; return 503, or evict oldest item | Bounded latency and memory. System stays healthy. | Data. Some work is never done. | Items are low-value or replaceable: metrics, analytics events, live location pings, log lines. |
| **Spill to disk / external queue** | On full, write the item to Redis / SQS / Kafka / a local file | Both data and producer throughput. | Complexity, an ops dependency, and ordering guarantees get murky. Disk fills too — it's a bigger bucket, not an infinite one. | Items are valuable and bursts are temporary: orders, payments, emails. |
| **Scale consumers** | Autoscale pool size / add worker processes | Everything — if you can actually go faster. | Money. And it doesn't help if the bottleneck is downstream (a rate-limited API, a single DB). | The backlog is a resource problem, not a structural one. |

The **anti-strategy** — and the one you'll find in real codebases — is the unbounded array:

```javascript
// ❌ THE BUG. An unbounded queue is not "no backpressure" — it's DEFERRED backpressure,
// paid later, all at once, as an OOM crash.
const jobs = [];
app.post('/x', (req, res) => { jobs.push(req.body); res.send('ok'); });
```

This "works" in staging and dies in production. Memory grows, GC pressure rises, the event loop slows, which slows the consumers, which grows memory faster. That feedback loop is called the **death spiral**. A bounded buffer makes the failure happen early, visibly, and recoverably.

**Consequences, stated plainly:**
- Block → *latency* degrades gracefully. Throughput is capped at the consumer's rate.
- Drop → *correctness* degrades gracefully. Latency stays flat. You must be able to tolerate loss.
- Spill → *cost and complexity* increase. Latency for spilled items is unbounded.
- Unbounded → *availability* fails catastrophically. Never do this.

You should also **monitor queue depth**. Queue depth is the single best early-warning signal in a producer-consumer system: a depth that trends up over minutes means your consumers are structurally too slow, and you have a while-to-fail window to react.

### 6. The Node-native versions

You rarely hand-roll `BoundedQueue` in production, because Node ships two implementations of this exact pattern.

**a) Streams — `highWaterMark` *is* the buffer capacity**

A Node stream is a producer-consumer pipeline where `highWaterMark` is `capacity`, `.write()` is `put()`, and the `'drain'` event is "a slot opened up".

```javascript
import { Readable, Writable, pipeline } from 'node:stream';
import { promisify } from 'node:util';

const source = Readable.from(async function* () {
  for (let i = 1; i <= 100_000; i++) yield `row ${i}\n`;   // a very fast producer
}());

const slowSink = new Writable({
  highWaterMark: 16,                 // <-- the bounded buffer. 16 chunks, then backpressure.
  async write(chunk, _enc, cb) {
    await new Promise((r) => setTimeout(r, 5));            // a slow consumer (a DB insert)
    cb();                                                  // "I've taken it — send the next"
  },
});

// pipeline() wires backpressure for you: when slowSink's buffer hits highWaterMark,
// write() returns false, and pipeline PAUSES the source until 'drain' fires.
await promisify(pipeline)(source, slowSink);
```

This is why `pipeline()` can stream a 50 GB file through a slow transform in constant memory, while this cannot:

```javascript
// ❌ Reads the entire 50 GB into RAM. No buffer, no backpressure, no bound.
const all = await fs.promises.readFile('huge.csv');
```

If you ever call `.write()` in a loop and ignore its `false` return value, you have written an unbounded queue. That's the #1 Node memory-leak bug.

**b) `worker_threads` — a pool of consumers with *real* parallelism**

The `BoundedQueue` pool above gives you **concurrency** (interleaved I/O waits) but not **parallelism** — all 3 workers still share one CPU core and one event loop. For CPU-bound work (hashing, image resizing, compression), you need `worker_threads`, where each consumer is a real OS thread.

```javascript
// pool.js — main thread: the producer + the buffer
import { Worker } from 'node:worker_threads';
import os from 'node:os';
import { BoundedQueue } from './bounded-queue.js';

const queue = new BoundedQueue(64);
const size = os.cpus().length;                 // CPU-bound → one worker per core

for (let i = 0; i < size; i++) {
  const worker = new Worker(new URL('./worker.js', import.meta.url));

  // Each thread runs its own consumer loop against the SHARED queue.
  (async function consumeLoop() {
    for (;;) {
      const job = await queue.take();          // blocks when empty — same as before
      const result = await new Promise((resolve, reject) => {
        worker.once('message', resolve);
        worker.once('error', reject);
        worker.postMessage(job);               // hand the job to the real thread
      });
      console.log('hashed', job.id, '→', result.hash.slice(0, 12));
    }
  })();
}

for (let id = 1; id <= 200; id++) {
  await queue.put({ id, data: `payload-${id}` });   // backpressure still applies
}
```

```javascript
// worker.js — the thread that actually burns CPU
import { parentPort } from 'node:worker_threads';
import crypto from 'node:crypto';

parentPort.on('message', (job) => {
  // scrypt is deliberately slow — this would freeze the event loop on the main thread.
  const hash = crypto.scryptSync(job.data, 'salt', 64).toString('hex');
  parentPort.postMessage({ id: job.id, hash });
});
```

Same pattern, same buffer, same backpressure — the only thing that changed is *where the consumer runs*. That's the sign of a good abstraction. (We go much deeper on threads, pools, and the event loop in [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md).)

### 7. This pattern IS a message queue

Take the `BoundedQueue` above and change three things:

1. Move the buffer **out of process** — into Redis, or a Kafka broker, or SQS.
2. Make it **durable** — write items to disk so a crash doesn't lose them.
3. Add **acknowledgements** — an item isn't removed on `take()`, it's removed when the consumer says "I finished it", so a crashed consumer's item gets redelivered.

You now have RabbitMQ. Add partitions and a retained log instead of a delete-on-ack buffer, and you have Kafka.

| In-memory `BoundedQueue` | Message queue (Kafka / RabbitMQ / SQS) |
|---|---|
| `put(item)` | `producer.send(topic, message)` |
| `take()` | `consumer.poll()` / a message handler callback |
| `capacity` | Broker disk / retention / `max.queue.size` |
| Blocked producer | Producer-side throttling / `queue.full` error |
| Worker pool of N | Consumer group with N consumers |
| Item lost if process dies | Item persisted; redelivered on failure |
| Single process | Any number of machines |

So when [67 — Message Queues](./67-message-queues.md) talks about producers, consumers, consumer groups, backpressure, and at-least-once delivery — it's this doc, scaled out across a network. **Learn the pattern here and the infrastructure there is just plumbing.**

### 8. When NOT to use it / how it's abused

- **The work is trivially fast.** If a "job" takes 0.2ms, the queue's own overhead and the loss of a direct return value cost more than they save. Just call the function.
- **The caller needs the result *now*.** Producer-consumer is fundamentally **fire-and-forget**. If your HTTP handler must return the computed value, you're not producing — you're calling. (You *can* bolt a correlation-ID/promise-map on top, but at that point you've built RPC with extra steps.)
- **Abuse: the unbounded queue.** Covered above. If someone says "just use an array", ask what happens at 10× traffic.
- **Abuse: queueing to hide a slow consumer.** A queue converts a throughput problem into a latency problem — it does not fix it. If consumers are permanently slower than producers, the queue only decides *how* you fail. Fix the consumer.
- **Abuse: one giant queue for everything.** Mixing a 5ms job type and a 30s job type in one queue means the fast jobs sit behind the slow ones (**head-of-line blocking**). Use separate queues per job class, or a priority queue.
- **Abuse: assuming ordering.** With N consumers, items *start* in order but *finish* out of order. If order matters, use one consumer per ordering key (this is exactly why Kafka partitions by key).

### 9. Related patterns and how they differ

This is the #1 interview follow-up. Know these cold.

| Pattern | How it differs from Producer-Consumer |
|---|---|
| **Observer** ([37](./37-pattern-observer.md)) | Observer is **push, synchronous, fan-out to all**: every subscriber gets every event, and `emit()` runs them inline on the caller's stack. Producer-Consumer is **pull, asynchronous, one-of-N**: exactly one consumer gets each item, and the producer doesn't run it. Node's `EventEmitter` is Observer, *not* a queue — it has no buffer and no backpressure. |
| **Mediator** ([48](./48-pattern-mediator.md)) | The buffer is arguably a degenerate mediator, but a mediator *routes and coordinates* (it knows about the participants). A buffer is dumb — it holds items and knows nothing. |
| **Thread Pool** | A thread pool is the **consumer half** of this pattern. Every thread pool has a queue in front of it; you just don't always see it. |
| **Pipeline / Chain of Responsibility** ([47](./47-pattern-chain-of-responsibility.md)) | CoR passes *one* request through *many* handlers until one handles it. Producer-Consumer passes *many* items to *one of many* interchangeable handlers. Chain a series of producer-consumer stages and you get a **pipeline** (which is exactly what a Node stream `pipeline()` is). |
| **Pub/Sub** | Pub/Sub = producer-consumer with **fan-out**: each message goes to *every* subscriber group, not one consumer. Kafka does both (partitions within a group = queue; multiple groups = pub/sub). |
| **Reactor / Event Loop** | The event loop *is* a producer-consumer: the OS produces I/O completion events into a queue, and the loop is the single consumer. See [52](./52-concurrency-fundamentals.md). |

---

## Visual / Diagram description

### Diagram 1: The bounded buffer, with both sides blocked

```
                          BOUNDED BUFFER (capacity = 8)
                       ┌───────────────────────────────────┐
                       │  head ──▶                  ◀── tail│
   PRODUCERS           │ ┌───┬───┬───┬───┬───┬───┬───┬───┐ │        CONSUMERS
                       │ │ J3│ J4│ J5│ J6│ J7│ J8│ J9│J10│ │        (worker pool)
 ┌──────────┐  put()   │ └─▲─┴───┴───┴───┴───┴───┴───┴─▲─┘ │ take() ┌──────────┐
 │Producer 1│─────────▶│   │      size = 8 = FULL      │   │───────▶│ Worker 1 │─▶ J1
 └──────────┘          │   │                           │   │        └──────────┘
 ┌──────────┐  put()   │  oldest out                newest in       ┌──────────┐
 │Producer 2│─────────▶│   (FIFO)                                   │ Worker 2 │─▶ J2
 └────┬─────┘          └───────────────────────────────────┘        └──────────┘
      │                            ▲                                ┌──────────┐
      │  BLOCKED                   │                                │ Worker 3 │─▶ (idle,
      │  (queue full —             │  a slot frees ⇒ admit exactly  │          │   parked in
      ▼  parked in #putWaiters)    │  ONE parked producer           └──────────┘   #takeWaiters)
 ┌────────────────────┐            │
 │  #putWaiters       │────────────┘
 │  [{item:J11, ...}] │   ◀── THIS IS BACKPRESSURE.
 │  [{item:J12, ...}] │       The producers are now paced by the consumers.
 └────────────────────┘
```

**How to read it:** items flow strictly left → right. The buffer is a FIFO ring: `put` appends at the tail, `take` removes from the head. Two things can go wrong, and the buffer handles each symmetrically:

- **Buffer FULL** → producers park in `#putWaiters`. Every `take()` frees one slot and wakes **exactly one** parked producer (FIFO, so no producer starves).
- **Buffer EMPTY** → consumers park in `#takeWaiters`. Every `put()` wakes **exactly one** parked consumer — and hands the item to it directly, bypassing the buffer.

The buffer is the *only* shared state. Draw this on a whiteboard, label the two waiter lists, and you've explained the pattern.

### Diagram 2: The four backpressure strategies

```
   Producers: 1000 items/sec                Consumers: 600 items/sec
                │                                       ▲
                ▼                                       │
        ┌───────────────┐                               │
        │ BUFFER (full) │───────────────────────────────┘
        └───────┬───────┘
                │  400 items/sec have nowhere to go. Pick one:
                │
   ┌────────────┼────────────┬────────────────┬──────────────────┐
   ▼            ▼            ▼                ▼                  ▼
┌───────┐  ┌─────────┐  ┌─────────┐    ┌────────────┐     ┌────────────┐
│ BLOCK │  │  DROP   │  │  SPILL  │    │   SCALE    │     │ UNBOUNDED  │
│       │  │         │  │         │    │ CONSUMERS  │     │  (never)   │
│await  │  │503 /    │  │write to │    │pool: 3→5   │     │ arr.push() │
│put()  │  │evict    │  │Redis,   │    │            │     │            │
│       │  │oldest   │  │SQS,disk │    │            │     │            │
├───────┤  ├─────────┤  ├─────────┤    ├────────────┤     ├────────────┤
│lose:  │  │lose:    │  │lose:    │    │lose:       │     │lose:       │
│LATENCY│  │DATA     │  │SIMPLICITY│   │MONEY       │     │THE PROCESS │
│       │  │         │  │+ ordering│   │(and only   │     │(OOM, at    │
│keep:  │  │keep:    │  │keep:    │    │ if you can │     │ 3am, with  │
│all    │  │health   │  │data +   │    │ actually   │     │ no warning)│
│data   │  │+latency │  │throughput│   │ go faster) │     │            │
└───────┘  └─────────┘  └─────────┘    └────────────┘     └────────────┘
 batch      metrics,     orders,        elastic            ← THE BUG
 jobs,      logs,        payments,      workloads
 ETL        telemetry    emails
```

---

## Real world examples

### 1. Node.js streams — backpressure built into the language

Every Node `Readable`/`Writable` pair is a producer-consumer system, and `highWaterMark` (default 16 KB for byte streams, 16 objects for object-mode streams) is the buffer capacity. When a `Writable`'s internal buffer exceeds `highWaterMark`, `.write()` returns `false`. `pipe()` and `pipeline()` react to that `false` by calling `.pause()` on the source, and resume on the `'drain'` event. This is why `fs.createReadStream(huge).pipe(res)` serves a 50 GB file in ~64 KB of RAM. The whole design exists because the Node core team decided the *default* behavior should be "block", not "buffer unboundedly".

### 2. Kafka — the pattern as durable infrastructure

Kafka is producer-consumer with the buffer moved to a distributed, disk-backed commit log. Producers append to a **partition**; a **consumer group** assigns each partition to exactly one consumer in the group, which is precisely the "one item goes to exactly one worker" property of a worker pool — enforced across machines. Kafka's backpressure knobs are the direct analogue of ours: producers have a bounded local send buffer (`buffer.memory`) and either block (`max.block.ms`) or throw when it's full. **Consumer lag** — how far behind the head of the log a group is — is the distributed version of "queue depth", and it's the metric every Kafka team alerts on, for exactly the reason described above: a lag that trends up means consumers are structurally too slow.

### 3. Cloudflare / CDN edge log pipelines (representative architecture)

Edge servers generate an enormous, extremely bursty firehose of request logs. The producer (the request-serving hot path) must **never** block — a 10ms stall to log a request would be catastrophic. So log pipelines of this shape use a bounded in-memory ring buffer per node with an explicit **drop** policy: if the buffer is full, the oldest log lines are discarded and a `dropped_count` counter is incremented. They deliberately choose "lose data" over "block the request path" or "OOM the edge node" — and they make the loss *observable* by counting it. This is the textbook case for the drop strategy: telemetry is valuable in aggregate but any individual line is expendable.

---

## Trade-offs

| Dimension | With a bounded producer-consumer buffer | Without one (direct call) |
|---|---|---|
| **Burst tolerance** | High — the buffer absorbs spikes up to `capacity` | None — a spike directly becomes latency and timeouts |
| **Independent scaling** | Yes — tune producers and consumers separately | No — one code path, one scaling knob |
| **Memory safety** | Bounded by construction | Bounded only by luck |
| **Latency of a single item** | *Worse* — the item may sit in the buffer waiting | Best possible — processed immediately |
| **Result delivery** | Hard — the producer has already moved on (fire-and-forget) | Trivial — it's a return value |
| **Ordering** | Only guaranteed with 1 consumer; N consumers finish out of order | Naturally in-order |
| **Debuggability** | Harder — the stack trace doesn't span the buffer | Easy — one continuous stack trace |
| **Failure isolation** | Good — a failed job doesn't kill the producer | Poor — a downstream failure surfaces as a caller failure |

| Buffer size | Benefit | Cost |
|---|---|---|
| **Small (8–64)** | Backpressure kicks in fast; low memory; problems become visible immediately | Producers block often; small bursts still cause blocking |
| **Large (10k+)** | Absorbs big bursts; producers rarely block | Large memory footprint; long tail latency (an item can wait a long time); *hides* a slow consumer until it's a crisis |
| **Unbounded** | (none) | OOM |

**Rule of thumb:** Size the buffer at roughly **1–2 × (peak burst you must absorb without shedding)**, not "as big as possible". Then always answer these three questions before you ship: **(1) What happens when it's full? (2) What happens when a consumer dies mid-item? (3) What's my alert on queue depth?** If you can't answer all three, you don't have a design — you have an array.

---

## Common interview questions on this topic

### Q1: "Implement a blocking bounded queue in JavaScript. There are no threads — how?"
**Hint:** A "blocking" call is just a promise you don't resolve yet. Keep two waiter lists: `#takeWaiters` (consumers parked on empty) and `#putWaiters` (producers parked on full). `take()` on an empty queue pushes `{resolve}` into `#takeWaiters` and returns the pending promise. `put()` first checks `#takeWaiters` and hands the item straight to a parked consumer (fast path), else pushes to the buffer, else parks itself. Every `take()` that frees a slot must wake **exactly one** parked producer, FIFO, or you get starvation.

### Q2: "Your consumers are permanently slower than your producers. What do you do?"
**Hint:** Name the four options and their costs — block (lose latency), drop (lose data), spill to durable storage (lose simplicity, gain an ops dependency), scale consumers (lose money, and only works if the bottleneck isn't downstream). Then say the key line: **a queue converts a throughput problem into a latency problem; it does not solve it.** If the deficit is permanent, no buffer size saves you — you must make consumers faster or accept loss. Call out that the unbounded-array "solution" is just deferred OOM.

### Q3: "How do you guarantee a job is processed exactly once?"
**Hint:** You don't — not in general. The honest answer: at-most-once (ack before processing → lost on crash) and at-least-once (ack after processing → duplicated on crash) are the real choices. Pick **at-least-once + idempotent consumers**: give every job an ID, and make the handler safe to run twice (e.g. `INSERT ... ON CONFLICT DO NOTHING`, or check a `processed_jobs` set). "Exactly-once" as marketed by Kafka is really at-least-once delivery plus transactional/idempotent writes — same trick.

### Q4: "How do you size the worker pool?"
**Hint:** Depends on what the work *is*. **CPU-bound** (hashing, image resize, compression): pool size ≈ number of cores, and it must be `worker_threads`, not async functions — 100 concurrent `await`s on the main thread give you zero extra CPU. **I/O-bound** (HTTP calls, DB queries): go much higher — the limit is set by the *downstream* system's capacity (connection pool size, API rate limit), not your cores. A common mistake is a pool of 200 hammering a DB with a 20-connection pool: you've just moved the queue, not removed it.

### Q5: "How is this different from an EventEmitter?"
**Hint:** `EventEmitter` is the Observer pattern: **push**, **synchronous**, **fan-out to all listeners**, **no buffer, no backpressure**. `emitter.emit()` runs every listener on the caller's stack before returning — so a slow listener *does* block the producer, and a fast producer *can't* be slowed down. A queue is **pull**, **asynchronous**, **one-of-N**, and **bounded**. If you're using an `EventEmitter` to hand work to a slow async handler, you have an unbounded queue and you don't know it.

---

## Practice exercise

### Build a Rate-Limited Web Crawler

Build a crawler with a producer-consumer core. Roughly 30–40 minutes.

**Requirements:**
1. A `BoundedQueue` of URLs with **capacity 50**. Use the promise-waiter implementation (write it yourself — don't copy it, type it).
2. A worker pool of **5 consumers**. Each consumer: `take()` a URL, "fetch" it (simulate with `await sleep(200 + Math.random() * 300)`), extract 0–3 fake child URLs, and `put()` them back into the same queue. Yes — the consumers are also producers. This is where it gets interesting.
3. A `Set` of visited URLs so you never enqueue the same URL twice.
4. Stop after 200 URLs have been crawled, then `close()` the queue and let workers exit cleanly.

**Then answer these, in code:**
- **Deadlock check:** with a shared queue where consumers are also producers, a full queue can deadlock — all 5 workers blocked in `put()`, and nobody left to `take()`. Reproduce it (make the queue capacity 5 and have every page yield 3 children). Then fix it. *Hint: the fix is not "make the buffer bigger". Think about what a worker should do when `put()` would block.*
- **Instrument it:** log `queue.size` and `queue.waitingProducers` every 500ms. Watch backpressure appear.
- **Add drop:** switch `put()` to `tryPut(url, { timeoutMs: 100 })` and count how many URLs you shed. Was the crawl still useful?

**Produce:** `bounded-queue.js`, `crawler.js`, and a 5-line comment at the top of `crawler.js` explaining *why* the deadlock happened and how your fix removed it. That comment is the actual deliverable — the deadlock is the lesson.

---

## Quick reference cheat sheet

- **Producer-Consumer** = decouple *making* work from *doing* work with a **bounded buffer** between them.
- **The buffer is the only shared state.** Producers and consumers must have zero references to each other.
- **"Blocking" in JS** = returning a promise you resolve later. Park `resolve` functions in a waiter list.
- **Two waiter lists:** `#takeWaiters` (consumers parked on empty), `#putWaiters` (producers parked on full). Wake exactly one, FIFO.
- **Bounded, always.** An unbounded queue is not "no backpressure" — it's deferred backpressure, paid as an OOM crash at 3am.
- **Four responses to a full buffer:** block (lose latency), drop (lose data), spill to disk (lose simplicity), scale consumers (lose money). Choose consciously.
- **A queue does not fix a slow consumer.** It converts a throughput problem into a latency problem.
- **Queue depth is your best alert.** A depth trending up over minutes = consumers are structurally too slow.
- **Pool sizing:** CPU-bound ≈ #cores (and must be `worker_threads`); I/O-bound ≈ bounded by the *downstream* system's capacity.
- **Node streams already do this:** `highWaterMark` = capacity, `.write()` returning `false` = "full", `'drain'` = "slot free". Ignoring that `false` is the classic Node memory leak.
- **`worker_threads`** gives real parallelism for CPU-bound consumers; async workers give concurrency only.
- **N consumers ⇒ no ordering guarantee.** Need order? One consumer per key (this is why Kafka partitions by key).
- **Ordering + at-least-once ⇒ make consumers idempotent.** Give every job an ID.
- **This pattern IS a message queue.** Move the buffer out of process + add durability + add acks = RabbitMQ/Kafka.
- **Not Observer.** `EventEmitter` is push/sync/fan-out-to-all/unbounded. A queue is pull/async/one-of-N/bounded.

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [50 — Visitor Pattern](./50-pattern-visitor.md) — the last of the classic GoF behavioural patterns |
| **Next** | [52 — Concurrency Fundamentals](./52-concurrency-fundamentals.md) — thread pools, mutexes, deadlock, and the Node event loop that makes all of this run |
| **Related** | [67 — Message Queues and Async Processing](./67-message-queues.md) — this exact pattern, moved out of process and made durable |
| **Related** | [47 — Chain of Responsibility](./47-pattern-chain-of-responsibility.md) — chain producer-consumer stages together and you get a pipeline |
| **Related** | [68 — Event-Driven Architecture](./68-event-driven-architecture.md) — what you build once producers and consumers live on different machines |
