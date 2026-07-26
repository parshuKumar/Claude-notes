# 100 — Message queues — BullMQ

## What is this?

BullMQ is a Node.js library for building **job queues** backed by Redis. Instead of doing slow or unreliable work directly inside an HTTP request (sending an email, resizing an image, charging a card), you push a "job" onto a queue and a separate **worker** process picks it up and runs it later — with automatic retries if it fails. Think of it like a restaurant kitchen ticket rail: the waiter (your API) doesn't cook the food, they just clip the order ticket to the rail, and a cook (the worker) pulls tickets off one at a time and prepares them, at their own pace.

## Why does it matter for backend development?

Real backend systems constantly need to do work that shouldn't block the user's HTTP response — sending a welcome email, generating a PDF invoice, transcoding a video, calling a flaky third-party API. If you do that work inline, your request hangs, timeouts happen, and one slow dependency can take your whole API down. BullMQ lets you respond to the client instantly ("your order is confirmed") while the actual heavy lifting happens asynchronously in the background, with built-in retries, backoff, delays, concurrency control, and progress reporting. It is the de facto standard job queue in the Node.js ecosystem, sitting on top of Redis, and shows up constantly in production Express/Fastify backends.

---

## Syntax / API

```js
// Install first: npm install bullmq ioredis
const { Queue, Worker, QueueEvents } = require('bullmq');

// Connection config shared by every Queue/Worker — points at your Redis instance
const connection = { host: '127.0.0.1', port: 6379 };

// ── Producer side — create a queue and add jobs to it ───────────────────────
const emailQueue = new Queue('email-queue', { connection }); // name identifies the queue in Redis

// .add(jobName, data, options) — pushes a job onto the queue
await emailQueue.add(
  'send-welcome-email',                 // job name — used for routing/logging, not unique per job
  { userId: 'user_42', email: 'a@b.com' }, // job payload — must be JSON-serializable
  {
    attempts: 3,                        // retry up to 3 times total if the job throws
    backoff: { type: 'exponential', delay: 1000 }, // wait 1s, 2s, 4s... between retries
    delay: 5000,                        // wait 5 seconds before the job becomes available
    removeOnComplete: true,             // clean up Redis after success (avoid memory bloat)
    removeOnFail: 1000,                 // keep the last 1000 failed jobs for debugging
  }
);

// ── Consumer side — a worker that processes jobs from that queue ────────────
const emailWorker = new Worker(
  'email-queue',                        // must match the queue name exactly
  async (job) => {                      // processor function — runs for every job pulled off the queue
    await job.updateProgress(50);       // report progress (visible to anyone listening)
    // ... do the actual work here, e.g. call an email provider API ...
    return { sent: true };              // return value is stored as job.returnvalue
  },
  { connection, concurrency: 5 }        // process up to 5 jobs at the same time
);

// ── Listening for lifecycle events ───────────────────────────────────────────
emailWorker.on('completed', (job, result) => {
  console.log(`Job ${job.id} finished:`, result); // fires when a job succeeds
});

emailWorker.on('failed', (job, err) => {
  console.error(`Job ${job.id} failed:`, err.message); // fires after all retry attempts are exhausted
});
```

---

## How it works — line by line

A `Queue` object is just a client that writes job data into Redis under a specific queue name — it does not do any processing itself. Every job you `.add()` becomes an entry in a Redis list/sorted-set structure that BullMQ manages internally.

A `Worker` is a separate process (or the same process, running in the background) that continuously polls Redis for that same queue name, pulls jobs off one at a time (or several at once if `concurrency` is set), and runs your processor function against each one. If the processor function throws an error or its returned promise rejects, BullMQ automatically schedules a retry according to the `attempts` and `backoff` options you set on the job — you don't write any retry logic yourself.

Delayed jobs work by BullMQ storing the job in a "delayed" set with a timestamp, and a background scheduler moves it into the active queue only once that timestamp has passed. Progress tracking works because `job.updateProgress()` writes the progress value back into Redis, so any other process (like your API server) can call `job.progress` to read how far along it is — this is how you build "processing... 40%" UI without polling a database.

Because everything lives in Redis, the producer (your Express API) and the worker (a separate Node.js process, maybe even on a different machine) don't need to know about each other directly — Redis is the shared communication layer.

---

## Example 1 — basic

```js
// File: queues/notification.queue.js
const { Queue, Worker } = require('bullmq');

// Shared Redis connection settings for this queue
const connection = { host: '127.0.0.1', port: 6379 };

// Create the queue — this is what other files import to add jobs
const notificationQueue = new Queue('notification-queue', { connection });

// Add a single job with default options (no retries configured here)
async function scheduleNotification(userId, message) {
  const job = await notificationQueue.add('push-notification', {
    userId,   // who the notification is for
    message,  // the text to send
  });
  console.log(`Queued job ${job.id} for user ${userId}`);
  return job.id; // caller can use this to check status later
}

// A worker that actually sends the notification
const notificationWorker = new Worker(
  'notification-queue',                         // listens to the same queue name
  async (job) => {
    console.log(`Sending "${job.data.message}" to ${job.data.userId}`);
    // Simulate sending — in real life this calls a push notification service
    await new Promise((resolve) => setTimeout(resolve, 500));
    return { delivered: true }; // stored as the job's return value
  },
  { connection }
);

// Log success and failure so we can see what happened in the terminal
notificationWorker.on('completed', (job) => {
  console.log(`Job ${job.id} completed`); // job finished without error
});
notificationWorker.on('failed', (job, err) => {
  console.error(`Job ${job.id} failed: ${err.message}`); // ran out of retry attempts
});

module.exports = { scheduleNotification };
```

---

## Example 2 — real world backend use case

```js
// File: queues/order.queue.js
// A real backend pattern: when an order is placed, queue background work
// (send confirmation email + generate invoice PDF) instead of blocking the request.

const { Queue, Worker } = require('bullmq');
const connection = { host: process.env.REDIS_HOST, port: Number(process.env.REDIS_PORT) };

// One queue can carry multiple related job types via the job name
const orderQueue = new Queue('order-processing', { connection });

// Called from your Express route handler after saving the order to the DB
async function enqueueOrderJobs(orderId, userId, email) {
  // Job 1 — send confirmation email, retry on transient SMTP failures
  await orderQueue.add(
    'send-order-email',
    { orderId, userId, email },
    { attempts: 5, backoff: { type: 'exponential', delay: 2000 } } // 2s, 4s, 8s, 16s, 32s
  );

  // Job 2 — generate invoice PDF, delayed 10s so the order record is fully committed
  await orderQueue.add(
    'generate-invoice',
    { orderId, userId },
    { delay: 10_000, attempts: 3 }
  );
}

// Worker routes on job.name to reuse one process for multiple job types
const orderWorker = new Worker(
  'order-processing',
  async (job) => {
    if (job.name === 'send-order-email') {
      await job.updateProgress(10);              // report we've started
      await sendConfirmationEmail(job.data.email, job.data.orderId); // pretend email service call
      await job.updateProgress(100);              // report completion
      return { emailSent: true };
    }

    if (job.name === 'generate-invoice') {
      await job.updateProgress(20);
      const pdfPath = await buildInvoicePdf(job.data.orderId); // pretend PDF generation
      await job.updateProgress(100);
      return { pdfPath };
    }

    throw new Error(`Unknown job name: ${job.name}`); // defensive — should never happen
  },
  { connection, concurrency: 10 } // handle up to 10 orders' jobs in parallel
);

// If invoice generation keeps failing after all retries, alert engineering
orderWorker.on('failed', (job, err) => {
  if (job.name === 'generate-invoice' && job.attemptsMade >= job.opts.attempts) {
    console.error(`ALERT: invoice generation permanently failed for order ${job.data.orderId}: ${err.message}`);
    // In production: send this to Sentry/Slack/PagerDuty
  }
});

// Stub implementations — real ones would call an email API / PDF library
async function sendConfirmationEmail(email, orderId) {
  console.log(`Emailing ${email} about order ${orderId}`);
}
async function buildInvoicePdf(orderId) {
  console.log(`Generating invoice for order ${orderId}`);
  return `/invoices/${orderId}.pdf`;
}

module.exports = { enqueueOrderJobs };
```

---

## Common mistakes

### Mistake 1 — Doing slow work inline instead of queuing it

```js
// ❌ WRONG — the HTTP request stays open until the email actually sends,
// so a slow email provider makes your API feel broken or time out
app.post('/signup', async (req, res) => {
  const user = await createUser(req.body);
  await sendWelcomeEmail(user.email); // blocks the response for seconds
  res.json({ success: true, userId: user.id });
});

// ✅ CORRECT — respond immediately, let a worker send the email in the background
app.post('/signup', async (req, res) => {
  const user = await createUser(req.body);
  await welcomeQueue.add('welcome-email', { email: user.email }); // fire-and-forget, instant
  res.json({ success: true, userId: user.id });
});
```

### Mistake 2 — Not setting `removeOnComplete` / `removeOnFail`, letting Redis fill up

```js
// ❌ WRONG — every completed job stays in Redis forever by default,
// eventually exhausting Redis memory in a high-traffic system
await orderQueue.add('generate-invoice', { orderId });

// ✅ CORRECT — clean up completed jobs, keep a bounded history of failures for debugging
await orderQueue.add(
  'generate-invoice',
  { orderId },
  {
    removeOnComplete: true,   // delete immediately once successful
    removeOnFail: 500,        // keep only the last 500 failures for investigation
  }
);
```

### Mistake 3 — Assuming a job that returns without error means the whole flow succeeded

```js
// ❌ WRONG — swallowing the real error means BullMQ thinks the job succeeded,
// so it will NOT retry even though the invoice was never actually created
const worker = new Worker('order-processing', async (job) => {
  try {
    await buildInvoicePdf(job.data.orderId);
  } catch (err) {
    console.log('something went wrong'); // error is hidden, job "completes" anyway
  }
}, { connection });

// ✅ CORRECT — let the error propagate so BullMQ marks the job failed and retries it
const worker = new Worker('order-processing', async (job) => {
  try {
    await buildInvoicePdf(job.data.orderId);
  } catch (err) {
    console.error(`Invoice generation failed for order ${job.data.orderId}:`, err.message);
    throw err; // re-throw — BullMQ needs this to trigger retry/backoff logic
  }
}, { connection });
```

---

## Practice exercises

### Exercise 1 — easy

Create a queue named `'image-resize-queue'` and a worker that processes it. The worker's job data will contain `{ filePath, width, height }`. Inside the processor, just `console.log` a message like `Resizing ${filePath} to ${width}x${height}`, wait 300ms (simulate work with a Promise + setTimeout), and return `{ resized: true }`. Add one job to the queue with sample data, and log `'completed'` and `'failed'` events from the worker to the console.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a `videoQueue` ('video-transcode-queue') where jobs carry `{ videoId, sourcePath }`. Configure jobs added to this queue with: 3 retry attempts, exponential backoff starting at 2000ms, and `removeOnComplete: true`. In the worker's processor function, call `job.updateProgress()` at least three times (e.g. 25, 60, 100) with a short delay between each to simulate a multi-step transcode. Add a `'progress'` event listener on the worker that logs `Job ${job.id} is at ${data}% for video ${job.data.videoId}`. Then write a function `simulateFlakyTranscode(videoId)` that throws an error roughly half the time (use `Math.random()`) so you can observe the retry/backoff behavior in your terminal logs.

```js
// Write your code here
```

---

### Exercise 3 — hard

Design a `reportQueue` ('report-generation-queue') system for a backend that generates large CSV reports on demand:
1. An `enqueueReport(userId, reportType)` function adds a job with `{ userId, reportType, requestedAt: Date.now() }`, `attempts: 4`, and exponential backoff.
2. The worker processor simulates generating a report in 4 stages (fetching data, transforming, writing CSV, uploading) — call `job.updateProgress()` after each stage with an increasing percentage, and add a short delay between stages.
3. If `reportType` is not one of `'sales'`, `'users'`, or `'inventory'`, throw an error immediately (invalid input should NOT be retried — research and use a way to signal "don't retry this" in BullMQ, or explain in a comment why you chose your approach).
4. Add a `getReportStatus(jobId)` function that fetches the job by ID from the queue and returns an object `{ state, progress, result }` describing its current status (hint: look at `Queue.getJob()` and the methods/properties available on a returned job, such as `getState()`).
5. Wire up `'completed'` and `'failed'` listeners on the worker that log enough detail to debug a stuck report in production (job id, userId, reportType, error message if failed).

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE PIECES
  Queue        → producer-side handle, used to .add() jobs to Redis
  Worker       → consumer-side process, pulls jobs and runs your processor function
  QueueEvents  → listen to job lifecycle events from a process that isn't the worker itself

ADDING A JOB
  queue.add(jobName, data, options)
    jobName   → string label for the job type (not unique, used for routing/logging)
    data      → JSON-serializable payload
    options   → { attempts, backoff, delay, removeOnComplete, removeOnFail, priority }

RETRY OPTIONS
  attempts: 3                                  → try up to 3 times total
  backoff: { type: 'exponential', delay: 1000 } → wait 1s, 2s, 4s... between attempts
  backoff: { type: 'fixed', delay: 1000 }       → wait exactly 1s every retry

DELAYED JOBS
  { delay: 5000 }   → job won't be picked up until 5 seconds have passed

PROGRESS TRACKING
  await job.updateProgress(50)         → inside the processor, report progress
  worker.on('progress', (job, data))   → listen for progress updates elsewhere
  job.progress                          → read current progress from a fetched job

WORKER OPTIONS
  concurrency: 5   → process up to 5 jobs at once in this worker

LIFECYCLE EVENTS (on Worker or QueueEvents)
  'completed' → job succeeded, gives (job, returnValue)
  'failed'    → all retry attempts exhausted, gives (job, error)
  'progress'  → job called updateProgress()
  'active'    → job started processing

CLEANUP (avoid unbounded Redis growth)
  removeOnComplete: true   → delete immediately on success
  removeOnFail: 1000       → keep only last N failed jobs

REQUIREMENTS
  Redis server must be running — BullMQ is Redis-backed, not in-memory
  npm install bullmq ioredis

GOTCHAS
  Worker and Queue must use the SAME queue name string, or jobs never get picked up
  Job data must be JSON-serializable — no functions, no class instances
  A processor that swallows errors instead of re-throwing breaks retry logic
  Long-running Node process needed for the Worker — it isn't request/response like Express
```

---

## Connected topics

- **92 — Connecting to Redis** — BullMQ is entirely Redis-backed; understanding `ioredis` connections is the foundation this topic sits on top of
- **101 — Pub/Sub with Redis** — a related but different real-time pattern (fire-and-forget broadcast) versus BullMQ's durable, retryable job queue
- **51 — Retry and timeout patterns** — BullMQ's `attempts`/`backoff` options are a production-grade implementation of the retry-with-backoff pattern covered generically there
