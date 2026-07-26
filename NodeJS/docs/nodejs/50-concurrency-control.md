# 50 — Concurrency control

## What is this?

Concurrency control means limiting **how many asynchronous operations run at the same time**, instead of firing them all at once. Think of a bank with 3 tellers and 200 customers — you don't let all 200 customers shout at every teller simultaneously; you form a queue and let only 3 through at a time. `p-limit` and `p-queue` are the most popular npm packages for doing exactly this with promises — they let you say "run these 500 tasks, but only 5 at a time."

## Why does it matter for backend development?

Backend code constantly needs to call external things — a third-party API, a database, an email provider, S3. If you fire 10,000 `fetch()` calls at once with `Promise.all()`, you can crash your own process (too many open sockets), get rate-limited or banned by the external API, exhaust your database connection pool, or overwhelm a downstream service that then falls over and takes your whole system down with it. Concurrency control is the difference between "I processed 50,000 rows and my API partner blocked my IP" and "I processed 50,000 rows safely, 10 at a time, and nothing broke." Every backend developer who does bulk imports, batch jobs, web scraping, or third-party API integration needs this.

---

## Syntax / API

```js
// npm install p-limit
// NOTE: p-limit v4+ is ESM-only. For CommonJS (require), install v3: npm install p-limit@3
const pLimit = require('p-limit'); // import the p-limit factory function

// Create a limiter that allows a MAXIMUM of 3 concurrent promises at once
const limit = pLimit(3); // "3" is the concurrency ceiling — never more than 3 run together

// Wrap every task function with limit() — it queues extra tasks until a slot frees up
const tasks = [1, 2, 3, 4, 5].map((userId) =>
  limit(() => fetchUserFromApi(userId)) // limit() returns a promise, runs fn when a slot opens
);

// Promise.all still works the same way — it just waits for the limited tasks to finish
const results = await Promise.all(tasks); // resolves once all 5 fetches (3-at-a-time) complete

// ── p-queue: same idea, but with more control (pause, priority, event hooks) ──
// npm install p-queue    (also ESM-only from v7+, use p-queue@6 for CommonJS require)
const PQueue = require('p-queue').default; // p-queue exports a class as the default export

const queue = new PQueue({ concurrency: 3 }); // same concept — max 3 jobs running at once

queue.add(() => sendWelcomeEmail(userId)); // add() queues a job, runs it when a slot is free
await queue.onIdle(); // wait until every queued job has finished running
```

---

## How it works — line by line

A concurrency limiter does **not** change what your async functions do — it only controls **when** they are allowed to start. Internally, `p-limit` keeps a counter of how many wrapped functions are currently running and a queue (array) of functions waiting their turn.

1. You call `pLimit(3)` — this creates a limiter object with `activeCount = 0` and an empty waiting queue, remembering the ceiling of 3.
2. Every time you call `limit(fn)`, it checks: is `activeCount < 3`? If yes, it runs `fn` immediately and increments `activeCount`. If no, it pushes `fn` into the waiting queue and returns a promise that will resolve later.
3. When a running task finishes (its promise settles), the limiter decrements `activeCount` and immediately pulls the next waiting function off the queue to run, keeping the "3 at a time" rule true at every moment.
4. `Promise.all()` around the array of `limit()` calls behaves exactly as it always does — it just happens to be waiting on promises that are being started in controlled batches rather than all at once.

`p-queue` works on the same principle but is a full class instance with extra features: pausing (`queue.pause()`), per-task priority, size/pending inspection (`queue.size`, `queue.pending`), interval-based rate limiting (`intervalCap` + `interval`), and events (`queue.on('idle', ...)`).

---

## Example 1 — basic

```js
// File: src/scripts/concurrency-basic.js
// Simulates fetching 6 "users" from a slow API, but only 2 requests in flight at once.

const pLimit = require('p-limit'); // p-limit@3 for CommonJS

const limit = pLimit(2); // never run more than 2 fake API calls at the same time

// Fake async task — pretends to call an external API and takes 1 second
function fetchUserFromApi(userId) {
  console.log(`[start] fetching user ${userId}`); // log when the task actually begins
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log(`[done]  fetching user ${userId}`); // log when it finishes
      resolve({ userId, name: `User-${userId}` }); // fake response payload
    }, 1000); // simulate 1 second of network latency
  });
}

async function main() {
  const userIds = [1, 2, 3, 4, 5, 6]; // 6 users to fetch

  // Wrap each call with limit() — only 2 will actually be "in flight" at once
  const tasks = userIds.map((userId) => limit(() => fetchUserFromApi(userId)));

  const users = await Promise.all(tasks); // wait for all 6, but respecting the concurrency cap
  console.log('All users fetched:', users); // final combined result
}

main(); // run the demo
// Output shows users 1 & 2 start together, then 3 & 4 (after ~1s), then 5 & 6 (after ~2s)
// Total time ≈ 3 seconds, NOT 1 second (which is what Promise.all alone would give)
```

---

## Example 2 — real world backend use case

```js
// File: src/jobs/syncOrdersToWarehouse.js
// A nightly job that syncs thousands of orders to a slow warehouse API.
// Without a limit, this would open thousands of sockets at once and get the
// service account rate-limited or banned. We cap it at 5 concurrent requests.

const pLimit = require('p-limit');
const axios = require('axios'); // assume axios is installed for HTTP calls

const WAREHOUSE_API_URL = 'https://warehouse.example.com/api/orders/sync';
const MAX_CONCURRENT_REQUESTS = 5; // safe ceiling agreed with the warehouse team

const limit = pLimit(MAX_CONCURRENT_REQUESTS); // shared limiter for this job run

// Sends one order to the warehouse API and returns a normalized result object
async function syncSingleOrder(order) {
  try {
    const response = await axios.post(
      WAREHOUSE_API_URL,
      { orderId: order.orderId, sku: order.sku, quantity: order.quantity },
      { headers: { Authorization: `Bearer ${process.env.WAREHOUSE_API_KEY}` } }
    );
    return { orderId: order.orderId, success: true, status: response.status }; // success record
  } catch (error) {
    // Never let one failed order crash the whole batch — capture and continue
    return { orderId: order.orderId, success: false, error: error.message };
  }
}

async function syncAllOrders(pendingOrders, dbConnection) {
  console.log(`Syncing ${pendingOrders.length} orders, ${MAX_CONCURRENT_REQUESTS} at a time...`);

  // Every order is wrapped in limit() — only 5 axios requests run simultaneously
  const syncTasks = pendingOrders.map((order) => limit(() => syncSingleOrder(order)));

  const results = await Promise.all(syncTasks); // wait for the whole controlled batch

  const succeeded = results.filter((result) => result.success); // orders that synced fine
  const failed = results.filter((result) => !result.success); // orders that need a retry

  // Persist the outcome so a retry job can pick up only the failures later
  await dbConnection.query(
    'UPDATE orders SET synced_at = NOW() WHERE order_id = ANY($1)',
    [succeeded.map((result) => result.orderId)]
  );

  console.log(`Synced: ${succeeded.length}, Failed: ${failed.length}`);
  return { succeeded, failed }; // caller can log or alert on failed.length
}

module.exports = { syncAllOrders };

// Usage:
// const dbConnection = require('../db');
// const { syncAllOrders } = require('./syncOrdersToWarehouse');
// const pendingOrders = await dbConnection.query('SELECT * FROM orders WHERE synced_at IS NULL');
// await syncAllOrders(pendingOrders.rows, dbConnection);
```

---

## Common mistakes

### Mistake 1 — Using `Promise.all()` with no limit at all

```js
// ❌ WRONG — fires all 10,000 requests at once, likely gets rate-limited or crashes
const userIds = await getAllUserIds(); // returns 10,000 IDs
const results = await Promise.all(
  userIds.map((userId) => fetchUserFromApi(userId)) // all 10,000 start immediately
);
// The remote API sees 10,000 simultaneous connections from one IP → 429 Too Many Requests

// ✅ CORRECT — cap concurrency with p-limit so the API never sees more than 10 at once
const pLimit = require('p-limit');
const limit = pLimit(10); // safe, agreed-upon ceiling

const results2 = await Promise.all(
  userIds.map((userId) => limit(() => fetchUserFromApi(userId))) // max 10 in flight
);
```

### Mistake 2 — Awaiting inside a loop, thinking it "controls concurrency" (it kills it entirely)

```js
// ❌ WRONG — awaiting one at a time is not "concurrency control", it's zero concurrency
// This is 10x slower than necessary — nothing runs in parallel at all
const results = [];
for (const userId of userIds) {
  const user = await fetchUserFromApi(userId); // waits fully before starting the next one
  results.push(user);
}

// ✅ CORRECT — use a limiter so several run in parallel, up to a safe cap
const pLimit = require('p-limit');
const limit = pLimit(5); // 5 at a time — fast, but still bounded

const results2 = await Promise.all(
  userIds.map((userId) => limit(() => fetchUserFromApi(userId)))
);
```

### Mistake 3 — Installing the wrong major version for a CommonJS project

```js
// ❌ WRONG — p-limit v4+ and p-queue v7+ are ESM-only (pure "type":"module" packages)
// require('p-limit') throws: "Error [ERR_REQUIRE_ESM]: require() of ES Module not supported"
const pLimit = require('p-limit'); // crashes in a CommonJS ("type":"commonjs") project

// ✅ CORRECT — pin the last CommonJS-compatible major version for require() projects
// npm install p-limit@3   (and p-queue@6 for p-queue)
const pLimit = require('p-limit'); // works fine on v3.x
const limit = pLimit(5);

// Alternative: keep the latest version but load it with a dynamic import in an async context
// const { default: pLimit } = await import('p-limit'); // works from CJS too, just async
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that has an array of 8 numbers `[1, 2, 3, 4, 5, 6, 7, 8]`. Write an async function `squareAfterDelay(number)` that waits 500ms (using `setTimeout` wrapped in a promise) and then returns `number * number`. Using `p-limit` with a concurrency of `2`, run `squareAfterDelay` for every number in the array and log the final array of squared results. Also log a timestamp before and after the whole run to confirm it takes roughly 4 rounds of 500ms (not all-at-once, not fully sequential).

```js
// Write your code here
```

---

### Exercise 2 — medium

You have an array of 20 `filePath` strings representing image files that need to be uploaded to a (simulated) cloud storage API. Write:
1. An async function `uploadImage(filePath)` that simulates an upload — it should randomly succeed after 300–800ms, or throw an error about 20% of the time (use `Math.random()`).
2. A function `uploadAllImages(filePaths, concurrency)` that uses `p-limit` to upload all files with the given concurrency, catches individual failures so one failed upload does not stop the others, and returns an object `{ uploaded: [...], failed: [...] }`.
3. Call `uploadAllImages` with `concurrency = 4` and log how many succeeded vs failed.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a reusable `RateLimitedQueue` class (using `p-queue` under the hood, or your own hand-rolled limiter if you don't want the dependency) that supports:
1. A constructor that accepts `{ concurrency, requestsPerSecond }` — `concurrency` caps how many tasks run at once, `requestsPerSecond` caps the total throughput over time (e.g. never more than 10 tasks *started* per second, even if concurrency would allow more).
2. An `enqueue(taskFn)` method that adds a task and returns a promise resolving with that task's result (or rejecting with its error).
3. A `getStats()` method returning `{ pending, running, completed, failed }` counts.
4. Test it by enqueueing 50 simulated `apiKey`-authenticated requests to a mock third-party API (a function that resolves after a random delay and randomly rejects ~10% of the time), with `concurrency: 5` and `requestsPerSecond: 8`, then logging `getStats()` after everything settles.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
THE PROBLEM
  Promise.all(bigArray.map(fn))  → fires EVERY call at once → crashes, rate limits, exhausts pools
  for...of with await            → fires ONE call at a time  → correct but painfully slow

THE FIX — bound the number of in-flight promises

p-limit
  const limit = pLimit(N);                 // N = max concurrent tasks
  limit(() => someAsyncFn(arg));           // wrap each call, returns a promise
  Promise.all(items.map(i => limit(() => fn(i))));  // standard usage pattern

p-queue (more features: pause, priority, interval rate limiting)
  const queue = new PQueue({ concurrency: N });
  queue.add(() => someAsyncFn(arg));       // add() queues + runs when a slot is free
  queue.add(fn, { priority: 1 });          // higher priority runs first
  await queue.onIdle();                    // wait for the whole queue to finish
  queue.pause();  queue.start();           // pause/resume processing
  new PQueue({ concurrency: N, interval: 1000, intervalCap: 10 }); // max 10 starts/sec

VERSIONING GOTCHA
  p-limit v4+  and p-queue v7+  → ESM-only, require() throws ERR_REQUIRE_ESM
  For CommonJS projects: npm install p-limit@3 p-queue@6
  Or use dynamic import(): const { default: pLimit } = await import('p-limit');

CHOOSING A NUMBER
  Start conservative (3–10), check the third-party API's documented rate limit
  Database calls → cap near your connection pool size, not above it
  CPU-bound work  → cap near os.cpus().length (see worker_threads / cluster topics)

ALWAYS catch per-task errors
  One rejected promise inside Promise.all() rejects the WHOLE batch
  Wrap each task in try/catch (or .catch()) so failures don't abort the others
```

---

## Connected topics

- **46 — Promises in Node context** — `Promise.all`, `Promise.allSettled` are the foundation concurrency limiters are built on top of.
- **51 — Retry and timeout patterns** — pairs naturally with concurrency control: limit how many run at once, then retry the ones that fail.
- **70 — Rate limiting** — concurrency control is the client-side mirror of server-side rate limiting; here you protect a downstream API, there you protect your own.
