# 51 — Retry and timeout patterns

## What is this?

A **timeout** is a rule that says "if this operation doesn't finish within X milliseconds, give up and treat it as failed" — like hanging up a phone call if nobody answers after 30 seconds instead of waiting forever. A **retry** is the decision to try that same operation again after a failure, usually waiting a bit longer between each attempt (called **backoff**) so you don't hammer a struggling service. `AbortController` is Node's built-in tool for actually cancelling an in-flight operation (like a `fetch` call) when a timeout fires, instead of just ignoring the result and hoping it goes away.

## Why does it matter for backend development?

Backend services constantly call other things that can be slow or flaky: databases, third-party APIs (payment gateways, email providers), internal microservices, and external DNS. Networks are unreliable — a request can hang for 60 seconds, a downstream service can return a transient 503, or a connection can drop mid-flight. Without a timeout, one slow dependency can freeze your entire request and exhaust your server's resources (open sockets, memory, event loop pressure). Without retries, a single blip in a normally-healthy service causes a hard failure the user sees. Every production backend — REST API, worker, cron job — wraps its outbound calls in timeout + retry logic, because "the network is reliable" is the first fallacy of distributed systems.

---

## Syntax / API

```js
// AbortController — the standard way to cancel an in-flight async operation
const controller = new AbortController(); // create a controller/signal pair
const { signal } = controller;            // signal is passed into cancellable APIs

// Cancel automatically after 5 seconds — this is how you implement a timeout
const timer = setTimeout(() => controller.abort(), 5000); // fires abort() at 5s

// fetch() (built into Node 18+) accepts a signal and rejects if aborted
fetch('https://api.example.com/users', { signal })
  .then((res) => res.json())
  .finally(() => clearTimeout(timer)); // always clear the timer once done — avoids leaks

// AbortController.timeout() — Node 17.3+ / modern shorthand, no manual setTimeout needed
const signalShortcut = AbortSignal.timeout(5000); // auto-aborts after 5000ms
fetch('https://api.example.com/orders', { signal: signalShortcut });

// Basic retry loop with exponential backoff (the core pattern)
async function retryWithBackoff(taskFn, maxRetries = 3, baseDelayMs = 200) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) { // attempt 0 = first try, then retries
    try {
      return await taskFn();                 // try the operation
    } catch (err) {
      if (attempt === maxRetries) throw err;  // out of retries — bubble the error up
      const delay = baseDelayMs * 2 ** attempt; // exponential: 200ms, 400ms, 800ms...
      await new Promise((resolve) => setTimeout(resolve, delay)); // wait before next try
    }
  }
}

module.exports = { retryWithBackoff };
```

---

## How it works — line by line

`AbortController` is a small object with one job: hold a `signal` that other code can watch, and a `.abort()` method that flips that signal to "aborted." Any API that supports cancellation (`fetch`, many database drivers, `fs` streams) accepts a `signal` and internally checks it — when the signal fires, the API stops what it's doing and rejects its promise with an `AbortError`.

To turn that into a **timeout**, you start a `setTimeout` at the same time you start the operation. If the operation finishes first, you clear the timer (so it never fires). If the timer fires first, it calls `controller.abort()`, which cancels the still-running operation. This is why a timeout is really "a race between the real work and a timer" — whichever finishes first decides the outcome.

`retryWithBackoff` works by wrapping the operation in a loop instead of calling it once. On failure, instead of giving up immediately, it waits a calculated delay and tries again. The delay grows **exponentially** (`baseDelay * 2^attempt`) so that if a service is overloaded, you back off harder each time instead of retrying at a constant rate that keeps adding load. Once `maxRetries` is exhausted, the last error is re-thrown so the caller still sees a failure — retries buy you resilience against *transient* problems, not a guarantee of success.

---

## Example 1 — basic

```js
// File: examples/basic-timeout-retry.js
// Demonstrates a timeout wrapper and a retry wrapper independently, then combined.

const sleep = (ms) => new Promise((resolve) => setTimeout(resolve, ms)); // helper delay

// A fake "unreliable" async task — fails the first 2 times, succeeds on the 3rd
let attemptCount = 0;
async function unreliableTask() {
  attemptCount++;                                        // track how many times we tried
  await sleep(50);                                        // simulate some work happening
  if (attemptCount < 3) {
    throw new Error(`Attempt ${attemptCount} failed`);     // simulate a transient failure
  }
  return `Success on attempt ${attemptCount}`;             // eventually succeeds
}

// withTimeout — races a promise against a timer; rejects if the timer wins
function withTimeout(promise, ms) {
  return new Promise((resolve, reject) => {
    const timer = setTimeout(() => {
      reject(new Error(`Operation timed out after ${ms}ms`)); // reject on timeout
    }, ms);

    promise
      .then((result) => {
        clearTimeout(timer);   // work finished first — cancel the timer
        resolve(result);       // pass the real result through
      })
      .catch((err) => {
        clearTimeout(timer);   // work failed first — still cancel the timer
        reject(err);           // pass the real error through
      });
  });
}

// retryWithBackoff — retries a failing async function with exponential delay
async function retryWithBackoff(taskFn, maxRetries = 3, baseDelayMs = 100) {
  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    try {
      return await taskFn();                         // attempt the task
    } catch (err) {
      if (attempt === maxRetries) throw err;          // no attempts left — fail for real
      const delay = baseDelayMs * 2 ** attempt;       // 100ms, 200ms, 400ms...
      console.log(`Retrying after ${delay}ms (attempt ${attempt + 1})`);
      await sleep(delay);                             // wait before trying again
    }
  }
}

async function main() {
  const result = await retryWithBackoff(unreliableTask, 3, 100); // will succeed on 3rd try
  console.log(result); // → "Success on attempt 3"

  // Combine both: retry a task that must also individually respect a timeout
  attemptCount = 0; // reset for the next demo
  const safeResult = await retryWithBackoff(
    () => withTimeout(unreliableTask(), 2000), // each attempt gets its own 2s timeout
    3,
    100
  );
  console.log(safeResult); // → "Success on attempt 3"
}

main();
```

---

## Example 2 — real world backend use case

```js
// File: src/services/paymentGatewayClient.js
// A real pattern: calling an external payment API with a per-request timeout,
// exponential backoff + jitter, and a distinction between retryable vs fatal errors.

const PAYMENT_API_URL = 'https://api.payment-provider.com/v1/charges';

// Jitter avoids the "thundering herd" — if 1000 requests fail at once, we don't
// want all 1000 retrying at the exact same millisecond and re-overloading the service.
function backoffWithJitter(baseDelayMs, attempt) {
  const exponential = baseDelayMs * 2 ** attempt;          // pure exponential growth
  const jitter = Math.random() * exponential * 0.3;        // add up to 30% randomness
  return Math.min(exponential + jitter, 10_000);            // cap so it never waits forever
}

// Only some errors are worth retrying — a 400 (bad request) will never succeed
// on retry, but a 503 (service unavailable) or a network drop probably will.
function isRetryableError(err, response) {
  if (err && err.name === 'AbortError') return true;       // our own timeout — retry
  if (err && err.code === 'ECONNRESET') return true;        // dropped connection — retry
  if (response && response.status >= 500) return true;      // server-side failure — retry
  if (response && response.status === 429) return true;      // rate limited — retry
  return false;                                              // everything else (4xx) — do not retry
}

/**
 * Charges a card through the payment gateway with a timeout + retry-with-backoff.
 * @param {object} requestBody - { userId, amountCents, currency, idempotencyKey }
 * @param {string} apiKey - secret key for the payment provider (never logged)
 */
async function chargeCard(requestBody, apiKey) {
  const maxRetries = 4;      // total attempts = 1 initial + 4 retries
  const timeoutMs = 8000;    // give the payment provider 8s per attempt
  const baseDelayMs = 300;   // starting backoff delay

  let lastError;

  for (let attempt = 0; attempt <= maxRetries; attempt++) {
    // A fresh AbortController per attempt — a controller can only abort once
    const controller = new AbortController();
    const timer = setTimeout(() => controller.abort(), timeoutMs);

    try {
      const response = await fetch(PAYMENT_API_URL, {
        method: 'POST',
        signal: controller.signal,                 // ties this fetch to our timeout
        headers: {
          'Content-Type': 'application/json',
          'Authorization': `Bearer ${apiKey}`,
          // Idempotency key ensures the provider treats retries of the SAME
          // logical charge as one operation — critical for non-idempotent POSTs
          'Idempotency-Key': requestBody.idempotencyKey,
        },
        body: JSON.stringify(requestBody),
      });

      clearTimeout(timer); // request completed (success or HTTP error) — stop the timer

      if (!response.ok && isRetryableError(null, response)) {
        lastError = new Error(`Payment API returned ${response.status}`);
        throw lastError; // caught below, triggers backoff + retry
      }

      if (!response.ok) {
        // Non-retryable (e.g. 402 card declined) — fail immediately, no point retrying
        const errorBody = await response.json();
        throw new Error(`Payment declined: ${errorBody.message}`);
      }

      return await response.json(); // success — return the charge confirmation

    } catch (err) {
      clearTimeout(timer); // clean up the timer on any failure path too

      const shouldRetry = attempt < maxRetries && isRetryableError(err, null);
      if (!shouldRetry) throw err; // fatal error, or out of retries — stop here

      const delay = backoffWithJitter(baseDelayMs, attempt);
      console.warn(
        `[paymentGatewayClient] attempt ${attempt + 1} failed (${err.message}), ` +
        `retrying in ${Math.round(delay)}ms`
      );
      await new Promise((resolve) => setTimeout(resolve, delay)); // wait, then loop again
    }
  }
}

module.exports = { chargeCard, isRetryableError, backoffWithJitter };

// Usage:
// const result = await chargeCard(
//   { userId: 'user_42', amountCents: 1999, currency: 'usd', idempotencyKey: sessionId },
//   process.env.PAYMENT_API_KEY
// );
```

---

## Common mistakes

### Mistake 1 — Retrying immediately with no backoff (thundering herd)

```js
// ❌ WRONG — retries instantly, hammering an already-struggling service harder
async function fetchUser(userId) {
  for (let i = 0; i < 5; i++) {
    try {
      return await fetch(`/api/users/${userId}`);
    } catch (err) {
      // no delay at all — if the API is down, this fires 5 requests in milliseconds
      continue;
    }
  }
  throw new Error('Failed after 5 attempts');
}

// ✅ CORRECT — exponential backoff gives the failing service room to recover
async function fetchUser(userId) {
  const baseDelayMs = 200;
  for (let attempt = 0; attempt < 5; attempt++) {
    try {
      return await fetch(`/api/users/${userId}`);
    } catch (err) {
      if (attempt === 4) throw err;
      const delay = baseDelayMs * 2 ** attempt; // 200ms, 400ms, 800ms, 1600ms
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
}
```

### Mistake 2 — Forgetting to clear the timeout timer

```js
// ❌ WRONG — the setTimeout keeps running even after the request finishes,
// leaking a timer handle and, if this runs often, piling up active timers
async function fetchWithTimeout(url, ms) {
  const controller = new AbortController();
  setTimeout(() => controller.abort(), ms); // never cleared
  return fetch(url, { signal: controller.signal });
}

// ✅ CORRECT — always clearTimeout once the operation settles, success or failure
async function fetchWithTimeout(url, ms) {
  const controller = new AbortController();
  const timer = setTimeout(() => controller.abort(), ms);
  try {
    return await fetch(url, { signal: controller.signal });
  } finally {
    clearTimeout(timer); // runs whether fetch resolves OR rejects
  }
}
```

### Mistake 3 — Blindly retrying non-idempotent operations

```js
// ❌ WRONG — retrying a "create charge" call on timeout may double-charge the
// customer if the first request actually succeeded but the response was lost
async function chargeCustomer(requestBody, apiKey) {
  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      return await fetch(PAYMENT_API_URL, {
        method: 'POST',
        headers: { Authorization: `Bearer ${apiKey}` },
        body: JSON.stringify(requestBody), // no idempotency key — retry = new charge
      });
    } catch (err) {
      await new Promise((r) => setTimeout(r, 300 * 2 ** attempt));
    }
  }
}

// ✅ CORRECT — attach a stable idempotency key so the server can recognize a
// retried request as "the same operation" instead of creating a duplicate
async function chargeCustomer(requestBody, apiKey) {
  const idempotencyKey = requestBody.orderId; // stable across retries of the same order
  for (let attempt = 0; attempt < 3; attempt++) {
    try {
      return await fetch(PAYMENT_API_URL, {
        method: 'POST',
        headers: {
          Authorization: `Bearer ${apiKey}`,
          'Idempotency-Key': idempotencyKey, // server dedupes on this
        },
        body: JSON.stringify(requestBody),
      });
    } catch (err) {
      await new Promise((r) => setTimeout(r, 300 * 2 ** attempt));
    }
  }
}
```

---

## Practice exercises

### Exercise 1 — easy

Write a function `withTimeout(promise, ms)` that:
1. Takes an existing Promise and a millisecond limit
2. Returns a new Promise that resolves/rejects with whatever `promise` produces **if** it settles within `ms`
3. Rejects with an `Error` whose message is `Timed out after {ms}ms` if the time limit is reached first
4. Properly cleans up any timer so it doesn't keep the process alive after the promise settles

Test it with a `sleep(ms)` helper: one call that finishes before the timeout (should resolve normally) and one that finishes after (should reject with the timeout error).

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a function `retryWithBackoff(taskFn, options)` where `options` is `{ maxRetries, baseDelayMs, isRetryable }`:
1. Calls `taskFn()` (an async function with no arguments) and returns its result on success
2. On failure, checks `options.isRetryable(err)` — if it returns `false`, throw immediately without retrying
3. If retryable, waits `baseDelayMs * 2^attempt` milliseconds, then tries again
4. Stops after `maxRetries` retries and throws the last error encountered
5. Logs each retry attempt with the attempt number and the delay used

Test it against a fake `taskFn` that throws a "retryable" error twice and a "fatal" error type that should NOT be retried, verifying both behaviors.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `resilientFetch(url, options)` function that combines everything in this topic:
1. Accepts `options: { timeoutMs, maxRetries, baseDelayMs, requestInit }`
2. Wraps every attempt in an `AbortController` tied to `timeoutMs` (a fresh controller per attempt — controllers can't be reused after aborting)
3. Treats network errors, `AbortError`, HTTP `429`, and any HTTP `5xx` as retryable; treats all other `4xx` responses as fatal (throw immediately, no retry)
4. Uses exponential backoff **with jitter** (add a random 0–30% extra delay) between attempts
5. Returns the parsed JSON body on success
6. Logs, for every attempt, whether it succeeded, was retried, or failed fatally — including the attempt number and elapsed time for that attempt

Test it against a local mock function that simulates: a slow response that should trigger the timeout, a `503` that should retry and then succeed, and a `404` that should fail immediately without retrying.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
ABORTCONTROLLER
  const controller = new AbortController();
  controller.signal          → pass this into fetch()/cancellable APIs
  controller.abort()         → cancels the operation, rejects with AbortError
  AbortSignal.timeout(ms)    → shorthand: auto-aborting signal, no manual setTimeout

TIMEOUT PATTERN (race work against a timer)
  const timer = setTimeout(() => controller.abort(), ms);
  try   { await fetch(url, { signal: controller.signal }); }
  finally { clearTimeout(timer); }   // ALWAYS clear — avoids leaked timers

RETRY WITH EXPONENTIAL BACKOFF
  delay = baseDelayMs * 2 ** attempt        // 200, 400, 800, 1600 ms...
  delay = delay + random(0, delay * 0.3)    // add jitter → avoids thundering herd
  cap the delay (e.g. Math.min(delay, 10000)) so it never grows unbounded

WHAT TO RETRY (transient)              WHAT NOT TO RETRY (fatal)
  AbortError (your own timeout)          400 Bad Request
  ECONNRESET / network drop              401 Unauthorized
  HTTP 429 Too Many Requests             403 Forbidden
  HTTP 5xx server errors                 404 Not Found
                                          422 Validation error

IDEMPOTENCY
  Non-idempotent ops (POST /charge, POST /orders) MUST carry a stable
  Idempotency-Key so a retry after a lost response doesn't duplicate the action.
  GET, PUT, DELETE are naturally safer to retry as-is.

NEVER DO
  Retry instantly with no delay             → hammers a struggling service
  Forget clearTimeout on the success path   → leaked timer handles
  Retry a POST payment call with no idempotency key → risk of double charge
  Reuse one AbortController across attempts → it can only abort() once

NODE VERSION NOTES
  fetch() is global and built-in from Node 18+ (stable in Node 20/22)
  AbortSignal.timeout() available from Node 17.3+ / 18+
```

---

## Connected topics

- **50 — Concurrency control** — retry and timeout are usually applied *per call* while `p-limit`/`p-queue` control how many of those calls run at once; the two combine in most real HTTP clients.
- **66 — Circuit breaker pattern** — the next step up from retries: after enough consecutive failures, a circuit breaker stops retrying altogether for a cooldown period instead of continuing to hammer a dead dependency.
- **47 — async/await in Node — advanced** — the retry loop and timeout race in this topic are built entirely on `async/await` and `Promise` composition, so a solid grasp of that topic is a prerequisite here.
