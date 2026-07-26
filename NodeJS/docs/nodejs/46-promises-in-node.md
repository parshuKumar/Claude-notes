# 46 — Promises in Node context

## What is this?

A Promise is an object that represents a value which is not available yet but will be — either resolved successfully or rejected with an error. In Node.js, almost every asynchronous operation (reading a file, querying a database, calling an external API) is Promise-based today via `fs/promises`, database drivers, and `fetch`. Think of a Promise like a food delivery tracking number — you don't have the food yet, but you have a token that will eventually tell you "delivered" (resolved) or "order cancelled" (rejected), and you can plan what to do in either case ahead of time.

## Why does it matter for backend development?

A backend server rarely does just one async thing — it fetches a user from the database, checks a cache, calls a payment API, and writes an audit log, often at the same time. `Promise.all`, `Promise.allSettled`, `Promise.race`, and `Promise.any` are the tools that let you run these operations concurrently instead of one after another, which directly cuts down your API's response time. Knowing which combinator to use — and how to chain `.then()`/`.catch()` correctly — is the difference between a server that responds in 50ms and one that responds in 500ms for no good reason.

---

## Syntax / API

```js
// ── Creating a Promise manually ─────────────────────────────────────────────
function fetchUserFromDb(userId) {
  // The Promise constructor takes a function with (resolve, reject)
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      if (!userId) {
        reject(new Error('userId is required'));  // rejects — goes to .catch()
      } else {
        resolve({ userId, name: 'Asha Verma' });   // resolves — goes to .then()
      }
    }, 100);
  });
}

// ── Chaining with .then / .catch / .finally ─────────────────────────────────
fetchUserFromDb('user_1')
  .then((user) => console.log('Found:', user))     // runs on success
  .catch((err) => console.error('Failed:', err.message)) // runs on failure
  .finally(() => console.log('DB call finished'));  // always runs, success or fail

// ── Promise.all — wait for ALL, fail fast if ANY rejects ────────────────────
Promise.all([fetchUserFromDb('user_1'), fetchUserFromDb('user_2')])
  .then((users) => console.log('Both users:', users))
  .catch((err) => console.error('One failed, all rejected:', err.message));

// ── Promise.allSettled — wait for ALL, never short-circuits ─────────────────
Promise.allSettled([fetchUserFromDb('user_1'), fetchUserFromDb(null)])
  .then((results) => console.log(results));
  // → [{ status: 'fulfilled', value: {...} }, { status: 'rejected', reason: Error }]

// ── Promise.race — settles as soon as the FIRST one settles (win or lose) ───
Promise.race([fetchUserFromDb('user_1'), fetchUserFromDb('user_2')])
  .then((firstUser) => console.log('First to finish:', firstUser));

// ── Promise.any — settles as soon as the FIRST one FULFILLS (ignores rejections) ─
Promise.any([fetchUserFromDb(null), fetchUserFromDb('user_2')])
  .then((firstSuccess) => console.log('First success:', firstSuccess))
  .catch((aggErr) => console.error('All rejected:', aggErr.errors)); // AggregateError
```

---

## How it works — line by line

- `new Promise((resolve, reject) => {...})` — you create a Promise by giving it a function; call `resolve(value)` when the work succeeds, call `reject(error)` when it fails.
- `.then(onSuccess)` — attaches a callback that only runs if the Promise resolved; it receives the resolved value.
- `.catch(onError)` — attaches a callback that only runs if the Promise (or any `.then()` before it) rejected or threw.
- `.finally(cleanup)` — attaches a callback that runs no matter what happened, useful for closing connections or stopping loading spinners.
- `Promise.all([...])` — runs every Promise in the array at the same time; resolves with an array of all results **only if every single one succeeds**; if even one rejects, the whole thing rejects immediately with that one error, and the results of the others are discarded.
- `Promise.allSettled([...])` — also runs everything at the same time, but it **always** resolves (never rejects) with an array describing each outcome individually — either `{ status: 'fulfilled', value }` or `{ status: 'rejected', reason }`. Nothing is ever thrown away.
- `Promise.race([...])` — resolves or rejects as soon as the **first** Promise in the array settles, whichever that is — win or lose, first one to finish decides the outcome.
- `Promise.any([...])` — resolves as soon as the **first** Promise **succeeds**; it ignores rejections unless every single Promise rejects, in which case it rejects with an `AggregateError` containing all the individual errors.

---

## Example 1 — basic

```js
// Simulates three async operations with different delays
function delayedValue(value, ms) {
  // Returns a Promise that resolves with `value` after `ms` milliseconds
  return new Promise((resolve) => setTimeout(() => resolve(value), ms));
}

function delayedFailure(message, ms) {
  // Returns a Promise that rejects with an Error after `ms` milliseconds
  return new Promise((_, reject) => setTimeout(() => reject(new Error(message)), ms));
}

async function runAllDemos() {
  // Promise.all — fails fast because one of them rejects
  try {
    const results = await Promise.all([
      delayedValue('cache-ok', 50),
      delayedFailure('cache-miss', 30),   // this one rejects first
    ]);
    console.log('all() results:', results);       // never reached
  } catch (err) {
    console.error('all() rejected because:', err.message); // 'cache-miss'
  }

  // Promise.allSettled — never rejects, reports each outcome
  const settled = await Promise.allSettled([
    delayedValue('emailSent', 20),
    delayedFailure('smsFailed', 10),
  ]);
  console.log('allSettled() results:', settled);
  // → [{ status: 'fulfilled', value: 'emailSent' }, { status: 'rejected', reason: Error }]

  // Promise.race — whichever settles first wins (the faster failure wins here)
  try {
    const winner = await Promise.race([
      delayedValue('slowSuccess', 100),
      delayedFailure('fastFailure', 10),
    ]);
    console.log('race() winner:', winner);          // never reached
  } catch (err) {
    console.error('race() lost to:', err.message);   // 'fastFailure'
  }

  // Promise.any — ignores the rejection, returns the first success
  const firstSuccess = await Promise.any([
    delayedFailure('firstAttemptFailed', 10),
    delayedValue('secondAttemptOk', 40),
  ]);
  console.log('any() first success:', firstSuccess); // 'secondAttemptOk'
}

runAllDemos();
```

---

## Example 2 — real world backend use case

```js
// File: src/services/dashboardService.js
// A dashboard endpoint that needs data from three independent sources:
// user profile, order history, and notification count — none depend on each other.

const dbConnection = require('../db/connection');      // fake db client
const cachePromise  = require('../cache/redisClient');   // fake redis client

async function getUserProfile(userId) {
  return dbConnection.query('SELECT * FROM users WHERE id = $1', [userId]);
}

async function getRecentOrders(userId) {
  return dbConnection.query('SELECT * FROM orders WHERE user_id = $1 LIMIT 5', [userId]);
}

async function getNotificationCount(userId) {
  return cachePromise.get(`notif_count:${userId}`);
}

// Build the whole dashboard in one round trip using Promise.allSettled
// so that ONE failing source (e.g. cache down) doesn't break the whole page.
async function buildDashboard(userId) {
  const [profileResult, ordersResult, notifResult] = await Promise.allSettled([
    getUserProfile(userId),
    getRecentOrders(userId),
    getNotificationCount(userId),
  ]);

  // Pull out values where fulfilled, fall back to safe defaults where rejected
  const profile = profileResult.status === 'fulfilled' ? profileResult.value : null;
  const orders  = ordersResult.status === 'fulfilled' ? ordersResult.value : [];
  const notifs  = notifResult.status === 'fulfilled' ? notifResult.value : 0;

  // Log any failures for observability without failing the whole request
  [profileResult, ordersResult, notifResult].forEach((result, index) => {
    if (result.status === 'rejected') {
      const sourceName = ['profile', 'orders', 'notifications'][index];
      console.error(`[dashboard] ${sourceName} failed:`, result.reason.message);
    }
  });

  return { profile, orders, notificationCount: notifs };
}

// Express route handler that uses buildDashboard
async function dashboardHandler(req, res) {
  const userId = req.params.userId;         // route param, e.g. /dashboard/:userId

  try {
    const dashboard = await buildDashboard(userId);
    res.status(200).json(dashboard);        // 200 even if some sources degraded
  } catch (err) {
    // Only reached if buildDashboard() itself throws synchronously — very rare
    res.status(500).json({ error: 'Failed to build dashboard' });
  }
}

module.exports = { buildDashboard, dashboardHandler };

// ── A second pattern: race an API call against a timeout ───────────────────
function withTimeout(promise, ms, timeoutMessage) {
  // A Promise that rejects after `ms` if the real one hasn't settled yet
  const timeoutPromise = new Promise((_, reject) =>
    setTimeout(() => reject(new Error(timeoutMessage)), ms)
  );
  // Whichever settles first — the real call or the timeout — wins the race
  return Promise.race([promise, timeoutPromise]);
}

async function callPaymentGateway(orderId, apiKey) {
  const paymentCall = dbConnection.callExternalApi('/charge', { orderId, apiKey });
  // If the gateway hasn't responded in 3 seconds, fail fast instead of hanging
  return withTimeout(paymentCall, 3000, 'Payment gateway timed out');
}
```

---

## Common mistakes

### Mistake 1 — Using Promise.all when one failure should not sink everything

```js
// ❌ WRONG — one slow/failing notification service breaks the entire signup flow
async function sendWelcomeEmails(userId) {
  const [emailResult, smsResult, pushResult] = await Promise.all([
    sendWelcomeEmail(userId),
    sendWelcomeSms(userId),     // if this rejects, email + push results are lost
    sendPushNotification(userId),
  ]);
  return { emailResult, smsResult, pushResult };
}

// ✅ CORRECT — use allSettled so one channel failing doesn't hide the others
async function sendWelcomeEmails(userId) {
  const results = await Promise.allSettled([
    sendWelcomeEmail(userId),
    sendWelcomeSms(userId),
    sendPushNotification(userId),
  ]);
  const failed = results.filter((r) => r.status === 'rejected');
  if (failed.length) {
    console.warn(`${failed.length} of 3 welcome notifications failed`);
  }
  return results;
}
```

### Mistake 2 — Awaiting Promises one by one when they don't depend on each other

```js
// ❌ WRONG — sequential awaits add up latency for no reason (100ms + 100ms + 100ms)
async function loadUserPageData(userId) {
  const profile = await getUserProfile(userId);     // waits 100ms
  const orders  = await getRecentOrders(userId);    // then waits another 100ms
  const notifs  = await getNotificationCount(userId); // then waits another 100ms
  return { profile, orders, notifs };               // total: ~300ms
}

// ✅ CORRECT — start them all at once, they run concurrently
async function loadUserPageData(userId) {
  const [profile, orders, notifs] = await Promise.all([
    getUserProfile(userId),      // all three start immediately
    getRecentOrders(userId),     // and run at the same time
    getNotificationCount(userId),
  ]);
  return { profile, orders, notifs };                // total: ~100ms (slowest one)
}
```

### Mistake 3 — Forgetting to return a Promise inside .then(), breaking the chain

```js
// ❌ WRONG — the inner Promise is not returned, so .then() resolves BEFORE the
// database write finishes — the caller thinks the save succeeded too early
function saveOrderThenNotify(order) {
  return dbConnection.saveOrder(order).then((savedOrder) => {
    notifyWarehouse(savedOrder);   // forgot `return` — chain doesn't wait for this
  });
}

// ✅ CORRECT — return the inner Promise so the chain waits for it to settle
function saveOrderThenNotify(order) {
  return dbConnection.saveOrder(order).then((savedOrder) => {
    return notifyWarehouse(savedOrder);  // chain now waits for the notification too
  });
}
```

---

## Practice exercises

### Exercise 1 — easy

Write three functions, `fetchInventory()`, `fetchPricing()`, and `fetchShippingEstimate()`, each returning a Promise that resolves after a random delay (`Math.random() * 500` ms) with a hardcoded object of your choice (e.g. `{ inStock: true }`). Use `Promise.all` to run all three concurrently and log the combined result once every one of them resolves. Then log how long the whole thing took using `Date.now()` before and after.

```js
// Write your code here
```

---

### Exercise 2 — medium

You have an array of 5 `userId` values. Write a function `fetchAllUserStatuses(userIds)` that calls an (imaginary) `checkUserStatus(userId)` function for every id concurrently — make `checkUserStatus` randomly reject for some ids (simulate a flaky service) and resolve for others. Use `Promise.allSettled` so the function never throws. Return an object shaped like `{ successful: [...], failed: [...] }` where `successful` contains the resolved values and `failed` contains `{ userId, reason }` pairs for the rejected ones.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a function `fetchFromFastestMirror(mirrorUrls, timeoutMs)` that:
1. Accepts an array of mirror URLs (imagine each one is a different server holding the same data) and a timeout in milliseconds.
2. Simulates calling each mirror with a function `callMirror(url)` that returns a Promise resolving after a random delay with `{ url, data: 'payload' }`, and occasionally rejects to simulate a dead mirror.
3. Races all the mirror calls against each other using `Promise.any` so the function returns as soon as the FIRST mirror successfully responds, ignoring any that fail.
4. Wraps the whole `Promise.any` call with a timeout (using the `Promise.race` + timeout-Promise pattern from Example 2) so that if ALL mirrors are too slow or all reject, the function rejects with a clear timeout or "all mirrors failed" error instead of hanging forever.
5. Test it by calling `fetchFromFastestMirror(['mirror1', 'mirror2', 'mirror3'], 2000)` several times and confirming it behaves correctly whether mirrors succeed, fail, or all time out.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
PROMISE BASICS
  new Promise((resolve, reject) => {...})   → create manually
  .then(onSuccess)                          → runs when resolved
  .catch(onError)                           → runs when rejected
  .finally(cleanup)                         → always runs

COMBINATORS — WHEN TO USE WHICH
  Promise.all(arr)
    Waits for ALL to resolve
    Rejects immediately if ANY one rejects (fail-fast)
    Use when: every result is REQUIRED, none is optional

  Promise.allSettled(arr)
    Waits for ALL to settle (resolve or reject)
    NEVER rejects — returns [{status, value|reason}, ...]
    Use when: partial success is OK, you want every outcome reported

  Promise.race(arr)
    Settles as soon as the FIRST one settles (win OR lose)
    Use when: implementing timeouts, "whichever responds first" logic

  Promise.any(arr)
    Settles as soon as the FIRST one FULFILLS
    Rejects only if ALL reject → AggregateError with .errors array
    Use when: you have redundant sources and need just ONE success

CHAINING RULES
  Always `return` a Promise inside .then() to keep the chain awaited
  A thrown error inside .then() is caught by the nearest .catch()
  .catch() at the end catches errors from ANY step above it

PERFORMANCE RULE OF THUMB
  Independent async calls → Promise.all / allSettled (concurrent)
  Dependent async calls (B needs A's result) → sequential await

GOTCHAS
  Promise.all short-circuits — other results are silently discarded on 1 failure
  Promise.any needs at least 1 resolve, or you get AggregateError, not a normal Error
  Promise.race resolves/rejects on losers too — they keep running in the background,
    they just no longer affect the outcome (they are not cancelled)
```

---

## Connected topics

- **45 — Callback pattern deep dive** — Promises were built to solve the callback-hell problems this topic covers; understanding callbacks first makes Promises click.
- **47 — async/await in Node (advanced)** — `await` is syntax sugar over `.then()`; this topic builds directly on the combinators covered here for sequential vs parallel control flow.
- **51 — Retry and timeout patterns** — The `Promise.race` + timeout-Promise pattern from Example 2 is the foundation for building proper retry-with-timeout logic.
