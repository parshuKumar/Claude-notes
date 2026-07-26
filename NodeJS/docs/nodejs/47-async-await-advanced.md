# 47 — async/await in Node — advanced

## What is this?

`async/await` is syntax sugar over Promises that lets you write asynchronous code that *reads* top-to-bottom like synchronous code. The "advanced" layer is everything that separates a developer who can write `await` from one who writes **correct, fast, crash-proof** async code: knowing when awaits should run one-after-another versus all-at-once, wrapping awaited code in error boundaries so one failure doesn't take down the whole process, and using async iterators (`for await...of`) to consume data that arrives over time instead of all at once. Think of it like a kitchen — a bad cook boils the pasta, *then* waits for it to finish before chopping the vegetables (sequential when it didn't need to be); a good cook starts the pasta and chops vegetables at the same time, and has a plan for what happens if either one goes wrong (parallel + error boundary).

---

## Why does it matter for backend development?

Every backend request path is a chain of async operations — hit the database, call a third-party API, read a file, check a cache. Writing every `await` one after another when the operations don't depend on each other is one of the most common performance bugs in Node backends — it silently doubles or triples response times under load. On top of that, a single unhandled rejection inside an async function can crash your entire Node process if there's no error boundary around it, taking down every other in-flight request with it. And modern APIs increasingly hand you data as async iterables — database cursors, paginated REST responses, readable streams — so knowing `for await...of` is required to consume them correctly instead of buffering everything into memory first. Mastering this topic is the difference between a backend that scales and one that falls over under real traffic.

---

## Syntax / API

```js
// ── Top-level await (Node 14.8+, ESM only — .mjs file or "type": "module" in package.json) ──
// In an ES Module, you can await directly at the top of the file, no wrapper needed:
// const dbConnection = await connectToDatabase();   // valid only in ESM

// ── In CommonJS (.js with require/module.exports), top-level await is NOT allowed ──
// You must wrap it in an async IIFE (Immediately Invoked Function Expression):
(async () => {
  const dbConnection = await connectToDatabase();     // now legal — inside an async function
  console.log('Connected:', dbConnection.status);
})();

// ── Sequential awaits — each one waits for the previous to finish ──────────────
async function getSequential(userId) {
  const user = await fetchUser(userId);               // waits ~100ms
  const orders = await fetchOrders(userId);           // THEN waits another ~100ms
  return { user, orders };                            // total: ~200ms
}

// ── Parallel awaits — start both requests at once, wait for both together ─────
async function getParallel(userId) {
  const [user, orders] = await Promise.all([          // both fire immediately
    fetchUser(userId),                                 // running concurrently
    fetchOrders(userId),                               // running concurrently
  ]);
  return { user, orders };                             // total: ~100ms (the slower of the two)
}

// ── Error boundary — one try/catch around the whole async operation ───────────
async function safeGetUser(userId) {
  try {
    const user = await fetchUser(userId);              // if this throws/rejects...
    return user;
  } catch (error) {
    console.error('fetchUser failed:', error.message);  // ...it's caught here, process stays alive
    throw new Error('Could not load user');             // re-throw a clean error for the caller
  }
}

// ── Async iterators — consuming data that arrives piece by piece ──────────────
async function* fetchPages(apiUrl) {                    // async generator function (note the *)
  let page = 1;
  while (true) {
    const response = await fetchPage(apiUrl, page);     // await each page as it's requested
    if (response.items.length === 0) return;            // stop when no more data
    yield response.items;                                // hand this page back to the consumer
    page++;
  }
}

// Consuming an async iterator with for await...of
async function processAllPages(apiUrl) {
  for await (const items of fetchPages(apiUrl)) {        // pulls one page at a time, awaiting each
    console.log(`Got ${items.length} items`);            // process this page before pulling the next
  }
}
```

---

## How it works — line by line

`async/await` does not create new concurrency — it is built entirely on top of Promises and the microtask queue you already learned in JavaScript. When you write `await somePromise`, the engine pauses that function at that exact line, registers a callback to resume it when the promise settles, and immediately gives control back to the event loop so other code can run in the meantime.

**Sequential vs parallel** comes down to *when* the promise-producing function is called:

- `await fetchUser(userId); await fetchOrders(userId);` — `fetchOrders` is not even *called* until the line above it finishes awaiting. The two requests happen back-to-back on the wire.
- `Promise.all([fetchUser(userId), fetchOrders(userId)])` — both `fetchUser(...)` and `fetchOrders(...)` are called on the same tick, so both HTTP/DB requests are already in flight before either promise is awaited. `await` here just waits for the array of promises to all settle.

**Error boundaries** work because `await` on a rejected promise throws a catchable exception at that exact `await` line — it behaves exactly like `throw` would in synchronous code. Wrapping a group of awaits in one `try/catch` means any failure anywhere in that block funnels into a single, predictable place to handle it, log it, and decide what to send back to the client.

**Async iterators** are objects that implement `Symbol.asyncIterator` — a method that returns a promise resolving to `{ value, done }` each time it's called. `for await...of` is a loop built specifically to call that method repeatedly, awaiting each result before moving to the next iteration. An `async function*` (async generator) is the easiest way to create one — every `yield` pauses the generator and hands a value to the consumer, and every resumption can `await` more async work before yielding again.

---

## Example 1 — basic

```js
// File: src/examples/sequential-vs-parallel.js

// Simulates an async operation that takes a fixed amount of time
function delay(label, ms) {
  return new Promise((resolve) => {
    setTimeout(() => {
      console.log(`${label} finished after ${ms}ms`);
      resolve(label);
    }, ms);
  });
}

// ── Sequential version — each await blocks the next line ──────────────────────
async function runSequential() {
  console.time('sequential');                     // start a timer labeled 'sequential'
  await delay('Task A', 500);                      // waits 500ms before moving on
  await delay('Task B', 500);                      // THEN waits another 500ms
  console.timeEnd('sequential');                    // → sequential: ~1000ms total
}

// ── Parallel version — both start at the same instant ──────────────────────────
async function runParallel() {
  console.time('parallel');                        // start a timer labeled 'parallel'
  await Promise.all([                               // both delay() calls fire immediately
    delay('Task C', 500),
    delay('Task D', 500),
  ]);
  console.timeEnd('parallel');                       // → parallel: ~500ms total (not 1000ms)
}

// Run both to see the difference
(async () => {
  await runSequential();                             // wait for sequential demo to finish first
  await runParallel();                               // then run parallel demo
})();
```

---

## Example 2 — real world backend use case

```js
// File: src/services/dashboardService.js
// A typical "user dashboard" endpoint: needs data from three independent sources.
// Independent calls are parallelized; failures are caught in one error boundary;
// a paginated audit log is streamed in using an async iterator instead of loading it all at once.

const db = require('../db');                          // hypothetical DB client
const apiClient = require('../lib/apiClient');         // hypothetical HTTP client

// Fetches the three independent pieces of dashboard data at once
async function loadDashboard(userId) {
  try {
    // These three calls do NOT depend on each other — run them in parallel
    const [profile, recentOrders, notifications] = await Promise.all([
      db.users.findById(userId),                       // query 1 — independent
      db.orders.findRecentByUser(userId, 5),            // query 2 — independent
      apiClient.get(`/notifications/${userId}`),         // query 3 — independent, external API
    ]);

    return { profile, recentOrders, notifications };     // all three resolved successfully
  } catch (error) {
    // ── Error boundary — ANY of the three failing lands here, once ──────────────
    console.error(`[dashboardService] Failed to load dashboard for ${userId}:`, error.message);
    throw new Error('DASHBOARD_LOAD_FAILED');             // caller (route handler) decides the HTTP status
  }
}

// Async generator that pulls a user's audit log one page at a time from a paginated API
async function* streamAuditLog(userId, apiUrl) {
  let page = 1;
  while (true) {
    const response = await apiClient.get(`${apiUrl}?userId=${userId}&page=${page}`);
    if (!response.data || response.data.length === 0) {  // no more pages left
      return;                                              // ends the async iterator cleanly
    }
    yield response.data;                                   // hand this page back to the caller
    page++;                                                 // advance for the next loop iteration
  }
}

// Consuming the audit log without ever holding the whole log in memory
async function countTotalAuditEvents(userId) {
  let total = 0;
  try {
    for await (const page of streamAuditLog(userId, '/audit-log')) {  // pulls + awaits page by page
      total += page.length;                              // accumulate count as pages arrive
    }
  } catch (error) {
    console.error('[dashboardService] Audit log stream failed:', error.message);
    throw new Error('AUDIT_LOG_STREAM_FAILED');
  }
  return total;
}

module.exports = { loadDashboard, streamAuditLog, countTotalAuditEvents };
```

---

## Common mistakes

### Mistake 1 — Awaiting independent operations sequentially

```js
// ❌ WRONG — fetchUser and fetchSettings don't depend on each other,
// but writing them like this forces them to run one after another
async function getProfile(userId) {
  const user = await fetchUser(userId);          // waits ~150ms
  const settings = await fetchSettings(userId);  // THEN waits another ~150ms — wasted time
  return { user, settings };                     // total: ~300ms
}

// ✅ CORRECT — fire both immediately, await them together
async function getProfile(userId) {
  const [user, settings] = await Promise.all([   // both start on the same tick
    fetchUser(userId),
    fetchSettings(userId),
  ]);
  return { user, settings };                     // total: ~150ms — the slower of the two
}
```

### Mistake 2 — No error boundary, letting a rejection crash the process

```js
// ❌ WRONG — no try/catch; if fetchUser rejects, this becomes an unhandled
// rejection and can crash the entire Node process (topic 07 covers why)
async function handleRequest(userId) {
  const user = await fetchUser(userId);          // if the DB is down, this throws
  return user;                                    // this line never runs, and nothing caught it
}

// ✅ CORRECT — wrap the awaited work in a try/catch error boundary
async function handleRequest(userId) {
  try {
    const user = await fetchUser(userId);
    return user;
  } catch (error) {
    console.error('handleRequest failed:', error.message);  // logged, process stays alive
    throw new Error('USER_FETCH_FAILED');          // convert to a clean, expected error
  }
}
```

### Mistake 3 — Using `await` inside `Array.forEach()`

```js
// ❌ WRONG — forEach does NOT wait for async callbacks; every iteration fires
// its promise and moves on immediately, so orders are NOT processed one-by-one
// and the function returns before any of them finish
async function processOrders(orderIds) {
  orderIds.forEach(async (orderId) => {
    await processOrder(orderId);                 // forEach ignores this returned promise
  });
  console.log('All orders processed');            // ❌ runs immediately, before any order finishes
}

// ✅ CORRECT — use a for...of loop (sequential) or Promise.all (parallel),
// both of which the outer async function actually awaits
async function processOrders(orderIds) {
  for (const orderId of orderIds) {
    await processOrder(orderId);                  // waits for each order in turn
  }
  console.log('All orders processed');            // ✅ only logs after every order is done

  // OR, if orders can run in parallel:
  // await Promise.all(orderIds.map((orderId) => processOrder(orderId)));
}
```

---

## Practice exercises

### Exercise 1 — easy

Write two async functions that each simulate a network call using `setTimeout` wrapped in a `Promise` (e.g. `fetchInventoryCount(productId)` taking 300ms and `fetchProductPrice(productId)` taking 400ms, each resolving with a fake value). Write a third async function `getProductSummary(productId)` that calls both and returns a combined object `{ inventoryCount, price }`. Use `console.time`/`console.timeEnd` to prove your version runs in parallel (~400ms total), not sequentially (~700ms total).

```js
// Write your code here
```

---

### Exercise 2 — medium

Write an async function `getUserAccountSnapshot(userId)` that fetches three pieces of data in parallel: `fetchUser(userId)`, `fetchBillingInfo(userId)`, and `fetchActivityLog(userId)` (simulate all three with `setTimeout`-based promises, and make `fetchBillingInfo` randomly reject about 30% of the time to simulate a flaky service). Wrap the whole operation in a single error boundary: if any of the three fails, catch it, log a clear message naming which piece of data is unavailable, and return a partial snapshot object with `null` for whichever field failed instead of throwing and losing the other two successful results. (Hint: `Promise.allSettled` is the tool that fits this requirement better than `Promise.all`.)

```js
// Write your code here
```

---

### Exercise 3 — hard

Build an async generator function `paginateResults(fetchPageFn, pageSize)` that takes a function `fetchPageFn(pageNumber, pageSize)` (which simulates a paginated API and returns a promise resolving to an array — return an empty array to signal "no more pages") and yields one page of results at a time using `for await...of` compatible syntax. Then write a consumer function `findFirstMatch(fetchPageFn, pageSize, predicate)` that uses `for await...of` over `paginateResults` to search page by page for the first item matching a `predicate` function, stopping and returning that item as soon as it's found (without fetching any further pages). Test it against a fake dataset of at least 50 items split across pages of 10, searching for an item somewhere in the middle, and log how many pages were actually fetched to prove it stopped early.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
SEQUENTIAL vs PARALLEL
  await a(); await b();              → sequential — b() only starts after a() resolves
  await Promise.all([a(), b()])      → parallel — both start on the same tick
  Rule: if b() does NOT depend on the RESULT of a(), run them in parallel

TOP-LEVEL AWAIT
  ESM (.mjs or "type":"module")      → await works directly at the top of the file
  CommonJS (.js, require/exports)    → NOT allowed — wrap in (async () => { ... })();

ERROR BOUNDARIES
  try { await x(); await y(); }      → ONE catch block guards the whole group
  catch (error) { ... }              → catches rejections from ANY awaited call inside try
  Never leave an async function's await un-caught — unhandled rejection risk

ASYNC ITERATORS
  async function* gen() { yield x; } → creates an async generator (note the *)
  for await (const val of gen())     → pulls + awaits one value at a time
  Use when: data arrives in chunks/pages and you don't want it all in memory at once

COMMON GOTCHAS
  array.forEach(async (x) => await f(x))   → does NOT wait — forEach ignores promises
  for (const x of arr) { await f(x); }     → correct sequential loop, actually waits
  arr.map((x) => f(x)) + Promise.all(...)  → correct parallel pattern

WHEN TO USE WHICH Promise COMBINATOR (see topic 46 for full detail)
  Promise.all        → need ALL to succeed; fails fast on first rejection
  Promise.allSettled  → need results from ALL regardless of individual failures
  Promise.race        → need whichever settles FIRST (success or failure)
  Promise.any          → need the first SUCCESS, ignoring failures until all fail
```

---

## Connected topics

- **46 — Promises in Node context** — `Promise.all`, `allSettled`, `race`, `any` are the exact building blocks `await` sits on top of; this topic assumes that one.
- **49 — Streams as async iterables** — extends the `for await...of` pattern from this topic directly onto Node's `Readable` streams for file and network data.
- **51 — Retry and timeout patterns** — builds on error boundaries here to add automatic retries with backoff and `AbortController`-based timeouts around awaited calls.
