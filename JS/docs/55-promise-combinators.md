# 55 — Promise Combinators

## What is this?

Promise combinators are **static methods on the `Promise` class that let you manage multiple Promises together** as a group, rather than handling them one by one. Instead of chaining Promises sequentially (wait for A, then B, then C), combinators let you fire multiple async operations simultaneously and react when some or all of them settle. Think of them like different strategies for waiting on a group of people to finish tasks: `Promise.all` waits for everyone and fails if anyone fails; `Promise.allSettled` waits for everyone and reports each individual outcome; `Promise.race` goes with whoever finishes first; `Promise.any` goes with whoever succeeds first.

## Why does it matter?

In real backend development you constantly need to fetch multiple resources in parallel — user data, permissions, settings, cached values — and do something when they're all ready. Using sequential `.then()` chains for independent operations wastes time (each waits for the previous). The combinators let you run all of them at once, cutting total wait time from the sum of all durations down to the maximum. The difference between knowing vs not knowing these methods can mean the difference between a 3-second API response and a 300ms one.

---

## The four combinators at a glance

```
┌──────────────────┬──────────────────────────────┬──────────────────────────────────┐
│ Method           │ Resolves when                │ Rejects when                     │
├──────────────────┼──────────────────────────────┼──────────────────────────────────┤
│ Promise.all      │ ALL promises fulfil          │ ANY promise rejects (fast-fail)  │
│ Promise.allSettled│ ALL promises settle (any way)│ Never rejects                    │
│ Promise.race     │ FIRST promise fulfils        │ FIRST promise rejects            │
│ Promise.any      │ FIRST promise fulfils        │ ALL promises reject (AggregateError)│
└──────────────────┴──────────────────────────────┴──────────────────────────────────┘
```

---

## Setup — reusable helpers used in all examples

```js
// Utility: create a Promise that resolves after ms with a value
function delay(ms, value) {
  return new Promise(resolve => setTimeout(() => resolve(value), ms));
}

// Utility: create a Promise that rejects after ms with a message
function delayReject(ms, message) {
  return new Promise((_, reject) => setTimeout(() => reject(new Error(message)), ms));
}
```

---

## `Promise.all` — all must succeed

`Promise.all(iterable)` takes an array (or any iterable) of Promises and:
- **Resolves** with an array of all their values (in original order) when **every** Promise fulfils
- **Rejects immediately** with the first rejection reason if **any** Promise rejects — other Promises keep running but their results are discarded

```js
// All succeed — resolves with array of results in original order
Promise.all([
  delay(300, "user data"),
  delay(100, "settings"),
  delay(200, "notifications"),
])
.then(([userData, settings, notifications]) => {
  // waits 300ms (slowest), not 300+100+200=600ms
  console.log(userData);      // "user data"
  console.log(settings);      // "settings"
  console.log(notifications); // "notifications"
});

// One fails — rejects immediately when the failing one rejects
Promise.all([
  delay(300, "user data"),
  delayReject(150, "settings service down"),
  delay(200, "notifications"),
])
.then(results => console.log(results))   // never runs
.catch(err => console.error("Failed:", err.message));
// After 150ms: "Failed: settings service down"
// The other two Promises (300ms, 200ms) keep running but results are ignored
```

### Real-world example: parallel dashboard data fetching

```js
async function loadDashboard(userId) {
  try {
    const [user, orders, notifications, settings] = await Promise.all([
      fetchUser(userId),              // 200ms
      fetchOrders(userId),            // 400ms
      fetchNotifications(userId),     // 150ms
      fetchUserSettings(userId),      // 100ms
    ]);
    // Total wait: 400ms (slowest) — not 200+400+150+100=850ms

    return { user, orders, notifications, settings };
  } catch (err) {
    // If ANY of the four fails, we land here
    throw new Error(`Dashboard load failed: ${err.message}`);
  }
}
```

### `Promise.all` with non-Promise values

```js
// Non-Promise values are treated as already-resolved Promises
Promise.all([
  Promise.resolve(1),
  2,                    // not a Promise — wrapped in Promise.resolve(2)
  delay(100, 3),
])
.then(values => console.log(values));  // [1, 2, 3]

// Empty array resolves immediately with []
Promise.all([]).then(v => console.log(v));  // []
```

### `Promise.all` preserves ORDER, not completion time

```js
// Even though the 100ms Promise finishes first, the results are in original order
Promise.all([
  delay(500, "A"),   // slowest
  delay(100, "B"),   // fastest
  delay(300, "C"),
])
.then(([a, b, c]) => {
  console.log(a, b, c);  // "A" "B" "C" — always in original order
});
```

---

## `Promise.allSettled` — wait for all, never fail

`Promise.allSettled(iterable)` waits for **all** Promises to settle (either fulfil or reject) and resolves with an array of result objects — **it never rejects itself**.

Each result object is either:
- `{ status: "fulfilled", value: <the resolved value> }`
- `{ status: "rejected",  reason: <the rejection error> }`

```js
Promise.allSettled([
  delay(200, "user"),
  delayReject(100, "orders service down"),
  delay(150, "notifications"),
  delayReject(300, "settings timeout"),
])
.then(results => {
  results.forEach((result, index) => {
    if (result.status === "fulfilled") {
      console.log(`Task ${index}: ✓`, result.value);
    } else {
      console.log(`Task ${index}: ✗`, result.reason.message);
    }
  });
});

// Output (after 300ms — waits for ALL):
// Task 0: ✓ user
// Task 1: ✗ orders service down
// Task 2: ✓ notifications
// Task 3: ✗ settings timeout
```

### Real-world example: batch operations where partial failure is acceptable

```js
async function sendBulkNotifications(userIds) {
  const sendJobs = userIds.map(id =>
    sendNotification(id).then(() => ({ id, sent: true }))
  );

  const results = await Promise.allSettled(sendJobs);

  const succeeded = results
    .filter(r => r.status === "fulfilled")
    .map(r => r.value.id);

  const failed = results
    .filter(r => r.status === "rejected")
    .map((r, i) => ({ id: userIds[i], reason: r.reason.message }));

  console.log(`Sent: ${succeeded.length}, Failed: ${failed.length}`);
  if (failed.length > 0) {
    console.warn("Failed notifications:", failed);
  }

  return { succeeded, failed };
}
```

### Use `allSettled` when you need a complete picture

```js
// Use Promise.all when: ALL results are required — any failure = abort
// Use Promise.allSettled when: partial success is OK, or you need to report each outcome

// E.g.: deleting multiple S3 files — want to know which succeeded and which failed
async function deleteFiles(fileKeys) {
  const results = await Promise.allSettled(
    fileKeys.map(key => s3.deleteObject({ Bucket: "my-bucket", Key: key }).promise())
  );

  return {
    deleted: results.filter(r => r.status === "fulfilled").length,
    errors:  results.filter(r => r.status === "rejected").map(r => r.reason.message),
  };
}
```

---

## `Promise.race` — whoever finishes first wins

`Promise.race(iterable)` resolves or rejects with the first Promise that settles — whether that's a fulfil or a reject. The other Promises keep running but their outcomes are ignored.

```js
Promise.race([
  delay(300, "slow"),
  delay(100, "fast"),
  delay(200, "medium"),
])
.then(winner => console.log("Winner:", winner));
// "Winner: fast"  (after 100ms)
```

### Real-world use case #1: timeout any async operation

```js
function withTimeout(promise, ms, label = "Operation") {
  const timeout = new Promise((_, reject) =>
    setTimeout(() => reject(new Error(`${label} timed out after ${ms}ms`)), ms)
  );
  return Promise.race([promise, timeout]);
}

// Any fetch that takes more than 5 seconds is aborted (rejected)
withTimeout(fetch("/api/slow-endpoint"), 5000, "Fetch user")
  .then(res  => res.json())
  .then(data => console.log(data))
  .catch(err => console.error(err.message));
// If fetch takes > 5s: "Fetch user timed out after 5000ms"
```

### Real-world use case #2: fastest available server (CDN fallback)

```js
const mirrors = [
  "https://cdn1.example.com/data.json",
  "https://cdn2.example.com/data.json",
  "https://cdn3.example.com/data.json",
];

// Use the fastest responding CDN mirror
Promise.race(mirrors.map(url => fetch(url).then(r => r.json())))
  .then(data => console.log("Got data from fastest mirror:", data))
  .catch(err  => console.error("All mirrors failed:", err));
```

### Caution: `Promise.race` can reject on first rejection

```js
Promise.race([
  delayReject(50, "fast failure"),
  delay(200, "slow success"),
])
.then(v => console.log("Won:", v))
.catch(e => console.error("Lost:", e.message));
// "Lost: fast failure" — the rejection came first
```

---

## `Promise.any` — first SUCCESS wins

`Promise.any(iterable)` is like `Promise.race` but **only cares about fulfillments, not rejections**. It:
- Resolves with the **first fulfilled** value
- Rejects only when **ALL** Promises have rejected — with an `AggregateError` containing all reasons

```js
Promise.any([
  delayReject(100, "server A down"),
  delay(200, "server B — OK"),
  delay(300, "server C — OK"),
])
.then(winner => console.log("First success:", winner))
// "First success: server B — OK"  (ignores the rejection from A)

// All fail → AggregateError
Promise.any([
  delayReject(100, "A failed"),
  delayReject(200, "B failed"),
  delayReject(300, "C failed"),
])
.catch(err => {
  console.log(err instanceof AggregateError);  // true
  console.log(err.message);     // "All promises were rejected"
  console.log(err.errors);      // [Error: A failed, Error: B failed, Error: C failed]
});
```

### Real-world use case: try multiple fallback sources

```js
async function loadUserAvatar(userId) {
  // Try three sources — use whichever responds successfully first
  return Promise.any([
    fetch(`https://primary-cdn.com/avatars/${userId}.jpg`).then(r => {
      if (!r.ok) throw new Error("Primary CDN failed");
      return r.blob();
    }),
    fetch(`https://fallback-cdn.com/avatars/${userId}.jpg`).then(r => {
      if (!r.ok) throw new Error("Fallback CDN failed");
      return r.blob();
    }),
    fetch(`https://api.example.com/users/${userId}/avatar`).then(r => {
      if (!r.ok) throw new Error("API failed");
      return r.blob();
    }),
  ]).catch(aggregateError => {
    console.warn("All avatar sources failed:", aggregateError.errors.map(e => e.message));
    return getDefaultAvatarBlob();  // return placeholder
  });
}
```

---

## Head-to-head comparison — all four on the same input

```js
const promises = [
  delay(100, "A"),
  delayReject(200, "B failed"),
  delay(300, "C"),
];

// Promise.all — fails fast
Promise.all(promises)
  .then(v => console.log("all:", v))
  .catch(e => console.log("all error:", e.message));
// "all error: B failed"  (after 200ms)

// Promise.allSettled — waits for all, reports each
Promise.allSettled(promises)
  .then(results => console.log("allSettled:", results.map(r => r.status)));
// "allSettled: ["fulfilled", "rejected", "fulfilled"]"  (after 300ms)

// Promise.race — first to settle wins
Promise.race(promises)
  .then(v => console.log("race:", v))
  .catch(e => console.log("race error:", e.message));
// "race: A"  (after 100ms — A resolves first)

// Promise.any — first FULFILLMENT wins
Promise.any(promises)
  .then(v => console.log("any:", v))
  .catch(e => console.log("any error:", e.message));
// "any: A"  (after 100ms — same as race here since A resolves before B rejects)
```

Now change the order to make B reject fastest:

```js
const promises2 = [
  delayReject(50, "A failed"),   // rejects at 50ms — fastest
  delay(200, "B"),
  delay(300, "C"),
];

Promise.race(promises2).then(v => console.log("race:", v)).catch(e => console.log("race err:", e.message));
// "race err: A failed"  — race includes rejections

Promise.any(promises2).then(v => console.log("any:", v)).catch(e => console.log("any err:", e));
// "any: B"  — any skips the rejection, waits for first fulfillment
```

---

## Combining combinators — real-world patterns

### Pattern 1: Parallel fetch with sequential processing

```js
async function getReportData(reportId) {
  // Phase 1: fetch everything in parallel
  const [reportMeta, rawData, userPrefs] = await Promise.all([
    fetchReportMeta(reportId),
    fetchRawData(reportId),
    fetchUserPreferences(),
  ]);

  // Phase 2: process sequentially (each step depends on previous)
  const filtered  = await filterData(rawData, userPrefs.filters);
  const formatted = await formatData(filtered, reportMeta.schema);
  const rendered  = await renderReport(formatted, reportMeta.template);

  return rendered;
}
```

### Pattern 2: Race with cleanup

```js
async function fetchWithFallback(primaryUrl, fallbackUrl, timeoutMs) {
  const controller = new AbortController();

  const primary  = fetch(primaryUrl, { signal: controller.signal }).then(r => r.json());
  const fallback = delay(timeoutMs).then(() => fetch(fallbackUrl).then(r => r.json()));

  try {
    const result = await Promise.race([primary, fallback]);
    controller.abort();  // cancel the primary if fallback won (or vice versa)
    return result;
  } catch (err) {
    controller.abort();
    throw err;
  }
}
```

### Pattern 3: Controlled concurrency using `Promise.all` in chunks

```js
// Process 100 items but only 5 at a time (avoid overwhelming the server)
async function processInBatches(items, batchSize, processor) {
  const results = [];

  for (let i = 0; i < items.length; i += batchSize) {
    const batch = items.slice(i, i + batchSize);
    // Run this batch of 5 in parallel
    const batchResults = await Promise.all(batch.map(processor));
    results.push(...batchResults);
    // Loop iteration: only start next batch after this one completes
  }

  return results;
}

// Usage:
const userIds = Array.from({ length: 50 }, (_, i) => i + 1);
const allProfiles = await processInBatches(userIds, 5, fetchUserProfile);
```

### Pattern 4: `allSettled` for audit logging

```js
async function syncToMultipleServices(data) {
  const services = [
    { name: "Elasticsearch", fn: () => indexInElastic(data) },
    { name: "Redis",         fn: () => cacheInRedis(data)   },
    { name: "Analytics",     fn: () => trackInAnalytics(data)},
  ];

  const results = await Promise.allSettled(services.map(s => s.fn()));

  results.forEach((result, i) => {
    const { name } = services[i];
    if (result.status === "rejected") {
      logger.warn(`Sync to ${name} failed:`, result.reason.message);
      metrics.increment(`sync.${name.toLowerCase()}.failure`);
    } else {
      metrics.increment(`sync.${name.toLowerCase()}.success`);
    }
  });

  // Primary operation succeeds even if some syncs failed
  return { synced: results.filter(r => r.status === "fulfilled").length };
}
```

---

## `AggregateError` — understanding `Promise.any` rejections

```js
async function demonstrateAggregateError() {
  try {
    await Promise.any([
      Promise.reject(new TypeError("type error")),
      Promise.reject(new RangeError("range error")),
      Promise.reject(new Error("generic error")),
    ]);
  } catch (err) {
    console.log(err instanceof AggregateError);  // true
    console.log(err.message);  // "All promises were rejected"
    console.log(err.errors.length);  // 3
    err.errors.forEach((e, i) => {
      console.log(`Error ${i}:`, e.constructor.name, e.message);
    });
    // Error 0: TypeError type error
    // Error 1: RangeError range error
    // Error 2: Error generic error
  }
}
```

---

## Tricky things

### 1. `Promise.all` with an already-rejected Promise

```js
// The rejected Promise was created BEFORE Promise.all — rejection is already queued
const p1 = Promise.resolve("ok");
const p2 = Promise.reject(new Error("already rejected"));  // rejected immediately
const p3 = delay(1000, "slow");  // hasn't resolved yet

Promise.all([p1, p2, p3])
  .catch(err => console.error("Caught:", err.message));
// "Caught: already rejected" — fires as microtask, doesn't wait for p3
```

### 2. `Promise.race` with an empty array hangs forever

```js
Promise.race([])  // never resolves or rejects — pending forever
  .then(() => console.log("never"));
// No output — the Promise is permanently pending

// Promise.all([]) resolves immediately with []
// Promise.any([]) rejects immediately with AggregateError (no values)
// Promise.allSettled([]) resolves immediately with []
```

### 3. Mutations inside `Promise.all` callbacks aren't isolated

```js
// Be careful with shared mutable state inside Promise.all tasks
const shared = { count: 0 };

Promise.all([
  delay(100).then(() => { shared.count++; return shared.count; }),
  delay(100).then(() => { shared.count++; return shared.count; }),
  delay(100).then(() => { shared.count++; return shared.count; }),
])
.then(results => console.log(results));
// Could be [1,2,3] or [2,3,3] or other orderings depending on microtask scheduling
// Each .then() might see different intermediate values of shared.count
// Prefer returning computed values rather than mutating shared state
```

### 4. `Promise.allSettled` result order vs settlement order

```js
// Results are in INPUT order, not completion order
Promise.allSettled([
  delay(300, "slow"),    // index 0
  delay(100, "fast"),    // index 1
])
.then(([slow, fast]) => {
  // 'slow' is still at index 0, 'fast' at index 1 — despite fast settling first
  console.log(slow.value);  // "slow"
  console.log(fast.value);  // "fast"
});
```

### 5. `Promise.any` vs `Promise.race` — the key difference is what triggers rejection

```js
// Promise.race rejects on FIRST rejection (regardless of other fulfillments pending)
// Promise.any only rejects when ALL reject

// Scenario: one fast reject, one slow fulfil
const fastReject  = delayReject(100, "fast reject");
const slowFulfil  = delay(500, "slow value");

Promise.race([fastReject, slowFulfil])
  .catch(e => console.log("race caught:", e.message));
// "race caught: fast reject"  — race loses on first settled (even rejection)

Promise.any([fastReject, slowFulfil])
  .then(v => console.log("any won:", v));
// "any won: slow value"  — any skips the rejection, waits for fulfillment
```

---

## Common mistakes

### Mistake 1 — Using `Promise.all` when partial failure should be tolerated

```js
// WRONG — if one notification fails, ALL are reported as failed
async function notifyAllUsers(userIds) {
  await Promise.all(userIds.map(id => sendNotification(id)));
  console.log("All notified");   // never runs if even one fails
}

// RIGHT — use allSettled when partial success is fine
async function notifyAllUsers(userIds) {
  const results = await Promise.allSettled(userIds.map(id => sendNotification(id)));
  const failed  = results.filter(r => r.status === "rejected").length;
  console.log(`Notified ${userIds.length - failed} users. ${failed} failed.`);
}
```

### Mistake 2 — `Promise.race` for timeout but ignoring the ongoing Promise

```js
// The original fetch still runs even after the timeout rejects
// In browsers this wastes bandwidth; in Node.js it wastes memory

// WRONG — no cleanup
function withTimeout(promise, ms) {
  return Promise.race([
    promise,
    new Promise((_, r) => setTimeout(() => r(new Error("timeout")), ms))
  ]);
}

// RIGHT — use AbortController to cancel the original request
function withTimeout(url, ms) {
  const controller = new AbortController();
  const timeoutId  = setTimeout(() => controller.abort(), ms);

  return fetch(url, { signal: controller.signal })
    .finally(() => clearTimeout(timeoutId));
}
```

### Mistake 3 — Sequential `await` when operations are independent

```js
// WRONG — sequential: takes 300 + 400 + 200 = 900ms
async function loadPage(userId) {
  const user     = await fetchUser(userId);     // 300ms
  const posts    = await fetchPosts(userId);    // 400ms
  const comments = await fetchComments(userId); // 200ms
  return { user, posts, comments };
}

// RIGHT — parallel: takes max(300, 400, 200) = 400ms
async function loadPage(userId) {
  const [user, posts, comments] = await Promise.all([
    fetchUser(userId),
    fetchPosts(userId),
    fetchComments(userId),
  ]);
  return { user, posts, comments };
}
```

### Mistake 4 — Not handling `AggregateError` from `Promise.any`

```js
// WRONG — catching as a regular Error loses the list of individual errors
Promise.any([p1, p2, p3])
  .catch(err => console.error(err.message));  // "All promises were rejected" — no detail

// RIGHT — check for AggregateError to get all failure reasons
Promise.any([p1, p2, p3])
  .catch(err => {
    if (err instanceof AggregateError) {
      console.error("All failed:");
      err.errors.forEach(e => console.error(" -", e.message));
    } else {
      console.error("Unexpected error:", err);
    }
  });
```

---

## Practice exercises

### Exercise 1 — easy

Write a function `fetchAllUserData(userId)` that uses `Promise.all` to fetch three things simultaneously (simulate with `delay`):
- `fetchProfile(userId)` — resolves after 200ms with `{ id, name, bio }`
- `fetchStats(userId)` — resolves after 350ms with `{ posts: 42, followers: 1500 }`
- `fetchBadges(userId)` — resolves after 150ms with `["early adopter", "top contributor"]`

Log the combined result and prove it takes ~350ms (not 200+350+150=700ms) by logging timestamps.

```js
// Write your code here
```

### Exercise 2 — medium

Build a `resilientFetch(urls, timeoutMs)` function that:
- Takes an array of URLs (fallback mirrors) and a timeout
- Tries all URLs simultaneously using `Promise.any` so the fastest success wins
- Wraps each fetch in a `timeoutMs` timeout (using `Promise.race` per URL)
- If all URLs fail or time out, throws a descriptive error listing all failures

Then build a `batchFetch(urls, concurrency)` function that fetches an array of URLs but limits to `concurrency` simultaneous requests at a time, collecting all results.

```js
// Write your code here
```

### Exercise 3 — hard

Implement all four combinators from scratch without using the built-ins:

1. `myAll(promises)` — resolves when all fulfil (fast-fail on any rejection)
2. `myAllSettled(promises)` — always resolves with `[{ status, value/reason }]`
3. `myRace(promises)` — resolves/rejects with first to settle
4. `myAny(promises)` — resolves with first fulfillment; rejects with `AggregateError` if all reject

Each must:
- Handle empty arrays correctly (match the spec)
- Handle non-Promise values in the array
- Preserve original order in results
- Never call the outer resolve/reject more than once

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Method | Input | Resolves | Rejects | Result shape |
|---|---|---|---|---|
| `Promise.all` | Array of Promises | All fulfil | Any rejects (fast-fail) | `[val1, val2, ...]` in order |
| `Promise.allSettled` | Array of Promises | All settle (never rejects) | — | `[{status, value/reason}]` |
| `Promise.race` | Array of Promises | First to fulfil | First to reject | Single value/error |
| `Promise.any` | Array of Promises | First to fulfil | All reject → AggregateError | Single value / AggregateError |
| `Promise.all([])` | Empty array | Immediately with `[]` | — | `[]` |
| `Promise.allSettled([])` | Empty array | Immediately with `[]` | — | `[]` |
| `Promise.race([])` | Empty array | Never | — | Pending forever |
| `Promise.any([])` | Empty array | — | Immediately | AggregateError (no errors) |

### When to use which

| Scenario | Use |
|---|---|
| Need ALL results, abort on any failure | `Promise.all` |
| Need to know outcome of EACH, partial OK | `Promise.allSettled` |
| Timeout wrapper, fastest server | `Promise.race` |
| Multiple fallbacks, use first success | `Promise.any` |
| N items but only K at a time | `Promise.all` in batches |
| Fire-and-forget multiple, log failures | `Promise.allSettled` |

---

## Connected topics

- **54 — Promises** — All combinators return Promises and take Promises as input; the core Promise mechanics are the prerequisite.
- **56 — async/await** — You can `await Promise.all([...])` — this is one of the most common patterns in async backend code.
- **57 — Error handling** — `Promise.all` fast-fail errors, `AggregateError` from `Promise.any`, and how `try/catch` interacts with these.
- **58 — fetch API** — Real HTTP requests return Promises; combining `fetch` calls with `Promise.all` is the primary use case for parallel data fetching.
