# 66 — Circuit breaker pattern

## What is this?

A circuit breaker is a wrapper around a risky operation (usually a network call to another service, database, or API) that watches for repeated failures and, once a threshold is crossed, **stops calling the failing operation entirely** for a while — instead failing fast or returning a fallback. It works exactly like the circuit breaker in your home's electrical panel: if a circuit keeps tripping because of a fault, the breaker flips OFF to protect the rest of the house from damage, instead of letting the fault keep drawing current forever. In Node.js backends, the `opossum` library is the standard tool for implementing this pattern around any async function.

## Why does it matter for backend development?

In a microservices or multi-dependency backend, one slow or dead downstream service (a payment gateway, a third-party API, a database under load) can silently take down your entire app. Without a circuit breaker, every incoming request keeps calling the broken service, piles up waiting on timeouts, exhausts your connection pool and event loop capacity, and the failure **cascades** — one broken dependency turns into a fully broken server. A circuit breaker detects the failing dependency quickly, "opens" the circuit so new requests fail instantly (or use a fallback) instead of waiting and queuing, and periodically checks if the dependency has recovered. Every backend developer working with external APIs, microservices, or unreliable third-party integrations needs this pattern to keep the rest of the system healthy while one part is down.

---

## Syntax / API

```js
// Install first: npm install opossum
const CircuitBreaker = require('opossum'); // opossum wraps any async function

// The function we want to protect — anything that can fail or hang
async function fetchUserProfile(userId) {
  // Imagine this calls a slow/unreliable downstream microservice
  const response = await fetch(`https://profile-service.internal/users/${userId}`);
  if (!response.ok) throw new Error(`Profile service returned ${response.status}`);
  return response.json();
}

// Options that configure the breaker's behavior
const options = {
  timeout: 3000,              // if fetchUserProfile takes longer than 3s, count it as a failure
  errorThresholdPercentage: 50, // open the circuit if 50% of requests in the window fail
  resetTimeout: 10000,         // after opening, wait 10s before trying again (half-open state)
  rollingCountTimeout: 10000,  // size of the rolling window used to calculate the failure %
  rollingCountBuckets: 10,     // number of buckets the rolling window is split into
};

// Wrap the risky function in a breaker
const profileBreaker = new CircuitBreaker(fetchUserProfile, options);

// Register a fallback — runs when the circuit is OPEN or the call fails
profileBreaker.fallback((userId) => ({
  userId,
  name: 'Unknown user',       // degraded response instead of a hard crash
  fromFallback: true,
}));

// Listen to breaker lifecycle events for logging/monitoring
profileBreaker.on('open', () => console.log('[breaker] OPEN — calls are being short-circuited'));
profileBreaker.on('halfOpen', () => console.log('[breaker] HALF-OPEN — testing if service recovered'));
profileBreaker.on('close', () => console.log('[breaker] CLOSED — service is healthy again'));
profileBreaker.on('fallback', (result) => console.log('[breaker] fallback used:', result));

// Call it exactly like the original function — breaker handles the rest
profileBreaker.fire('user_42')
  .then((profile) => console.log(profile))
  .catch((err) => console.error('Both call and fallback failed:', err));
```

---

## How it works — line by line

The circuit breaker keeps a rolling count of successes and failures for the wrapped function and moves between **three states**:

- **CLOSED** — normal operation. Calls go through to the real function. Failures are being counted.
- **OPEN** — too many failures happened. Calls are immediately rejected (or sent to the fallback) **without even trying** the real function. This is what stops the cascade.
- **HALF-OPEN** — after `resetTimeout` elapses, the breaker lets **one** test call through. If it succeeds, the breaker goes back to CLOSED. If it fails, it goes back to OPEN and waits again.

Walking through the syntax block:
- `new CircuitBreaker(fn, options)` wraps `fetchUserProfile` — it does not change the function, it creates a supervisor around it.
- `timeout` makes the breaker itself abort a call that hangs too long, treating a slow response the same as a failed one — this is critical because a hanging call is just as dangerous as an error.
- `errorThresholdPercentage` is the trip point — once this percentage of recent calls has failed, the breaker flips to OPEN.
- `resetTimeout` is the "cool-down" period — how long the breaker stays OPEN before trying a HALF-OPEN test call.
- `.fallback(fn)` registers what to return instead of throwing when the circuit is OPEN or the underlying call fails — this is what keeps the caller's code from crashing.
- `.on('open'/'halfOpen'/'close'/'fallback', ...)` are event hooks so you can log state changes or send alerts to your monitoring system.
- `.fire(...args)` is how you actually invoke the protected function — it returns a Promise just like calling `fetchUserProfile` directly would, so calling code doesn't need to know a breaker is involved.

---

## Example 1 — basic

```js
// File: src/breakers/paymentBreaker.js
const CircuitBreaker = require('opossum'); // the circuit breaker library

// A flaky function that fails half the time — simulating a shaky payment API
async function chargeCard(apiKey, amountInCents) {
  // Randomly simulate the payment gateway being down (for demo purposes)
  const isDown = Math.random() < 0.5;
  if (isDown) {
    throw new Error('Payment gateway timeout'); // simulate a downstream failure
  }
  return { status: 'charged', amountInCents }; // simulate a successful charge
}

// Breaker configuration — trip after 50% failures within a 5-second window
const breakerOptions = {
  timeout: 2000,                 // treat calls slower than 2s as failed
  errorThresholdPercentage: 50,  // open circuit once half the recent calls fail
  resetTimeout: 5000,             // wait 5s before allowing a test call again
};

// Wrap chargeCard with the breaker
const chargeBreaker = new CircuitBreaker(chargeCard, breakerOptions);

// Fallback response when the circuit is open or the call fails
chargeBreaker.fallback(() => ({
  status: 'failed',
  reason: 'Payment service temporarily unavailable, please retry shortly',
}));

// Log every state transition so we can see the breaker working
chargeBreaker.on('open', () => console.log('Circuit OPEN — skipping real calls'));
chargeBreaker.on('close', () => console.log('Circuit CLOSED — calls flowing normally'));

// Fire several calls in a row to observe the breaker trip
async function runDemo() {
  for (let i = 0; i < 10; i++) {
    const result = await chargeBreaker.fire('sk_test_apiKey', 1999); // fire() always resolves
    console.log(`Call ${i + 1}:`, result);
  }
}

runDemo(); // start the demo
```

---

## Example 2 — real world backend use case

```js
// File: src/services/inventoryService.js
// A real backend scenario: our order API depends on an external Inventory
// microservice. If that service degrades, we don't want it to take down checkout.

const axios = require('axios');                 // HTTP client for the downstream call
const CircuitBreaker = require('opossum');       // circuit breaker library

// The risky downstream call — checking stock before allowing an order
async function checkStock(productId, requestedQty) {
  const response = await axios.get(
    `https://inventory-service.internal/stock/${productId}`,
    { timeout: 2500 } // axios-level timeout as a first line of defense
  );
  return response.data.availableQty >= requestedQty;
}

// Production-grade breaker configuration
const stockBreaker = new CircuitBreaker(checkStock, {
  timeout: 3000,                  // breaker-level timeout, slightly above axios's own
  errorThresholdPercentage: 40,   // trip if 40% of recent checks fail
  resetTimeout: 15000,             // give the inventory service 15s to recover
  rollingCountTimeout: 20000,      // measure failure rate over a 20s rolling window
  rollingCountBuckets: 20,         // split that window into 20 buckets for accuracy
  volumeThreshold: 5,              // need at least 5 calls before the breaker can trip
});

// Fallback: when inventory service is unreachable, assume stock is available
// but flag the order for manual review — safer than blocking every sale outright
stockBreaker.fallback((productId, requestedQty) => {
  console.warn(`[inventory-fallback] Could not verify stock for ${productId}, flagging for review`);
  return { available: true, needsManualReview: true };
});

// Emit metrics on every breaker state change — wire this into your monitoring stack
stockBreaker.on('open', () => {
  console.error('[inventory-breaker] OPEN — inventory service considered down');
  // In production: send an alert to Slack/PagerDuty here
});
stockBreaker.on('halfOpen', () => {
  console.log('[inventory-breaker] HALF-OPEN — testing inventory service recovery');
});
stockBreaker.on('close', () => {
  console.log('[inventory-breaker] CLOSED — inventory service recovered');
});

// The function our Express route actually calls — breaker is transparent to it
async function isProductAvailable(productId, requestedQty) {
  const result = await stockBreaker.fire(productId, requestedQty);
  // fire() resolves with either the real result or the fallback's return value
  return result;
}

module.exports = { isProductAvailable };

// Usage inside an Express route handler:
// const { isProductAvailable } = require('../services/inventoryService');
//
// app.post('/orders', async (req, res) => {
//   const { productId, quantity, userId } = req.body;
//   const stockCheck = await isProductAvailable(productId, quantity);
//   if (stockCheck.needsManualReview) {
//     console.log(`Order from ${userId} flagged for manual stock review`);
//   }
//   // ... continue creating the order
//   res.status(201).json({ orderCreated: true, stockCheck });
// });
```

---

## Common mistakes

### Mistake 1 — Wrapping the entire request handler instead of just the risky call

```js
// ❌ WRONG — the whole route handler is wrapped, so DB writes, validation,
// and unrelated logic all get treated as "the risky operation"
const orderBreaker = new CircuitBreaker(async (req, res) => {
  validateOrder(req.body);              // has nothing to do with the failing dependency
  await dbConnection.query('INSERT ...'); // our own database — not what's flaky
  return callInventoryService(req.body); // this is the ONLY part that actually fails
}, options);

// ✅ CORRECT — wrap only the unreliable dependency call itself
const inventoryBreaker = new CircuitBreaker(callInventoryService, options);

app.post('/orders', async (req, res) => {
  validateOrder(req.body);                          // runs normally, not protected/needed
  const stock = await inventoryBreaker.fire(req.body); // only THIS call is circuit-protected
  await dbConnection.query('INSERT ...');            // our own DB call stays outside the breaker
  res.json({ stock });
});
```

### Mistake 2 — No fallback registered, so an open circuit just throws an unhandled error

```js
// ❌ WRONG — without a fallback, an OPEN circuit rejects fire() with an error,
// which crashes the request instead of degrading gracefully
const apiBreaker = new CircuitBreaker(callThirdPartyApi, options);
app.get('/report', async (req, res) => {
  const data = await apiBreaker.fire();   // throws EOPENBREAKER when open — unhandled!
  res.json(data);
});

// ✅ CORRECT — always register a fallback so calling code gets a usable response
const apiBreaker2 = new CircuitBreaker(callThirdPartyApi, options);
apiBreaker2.fallback(() => ({ cached: true, data: getCachedReportData() })); // graceful degrade

app.get('/report', async (req, res) => {
  const data = await apiBreaker2.fire();  // resolves with fallback value if circuit is open
  res.json(data);
});
```

### Mistake 3 — Setting resetTimeout too short (or too long) without thinking about the dependency

```js
// ❌ WRONG — resetTimeout of 200ms means the breaker hammers a struggling service
// with test calls almost immediately, never giving it room to recover
const flakyBreaker = new CircuitBreaker(callAuthService, {
  errorThresholdPercentage: 50,
  resetTimeout: 200,   // way too aggressive — the dependency never gets breathing room
});

// ✅ CORRECT — pick resetTimeout based on how long the dependency realistically
// needs to recover (e.g. a DB reconnect, an autoscaled service spinning up new pods)
const authBreaker = new CircuitBreaker(callAuthService, {
  errorThresholdPercentage: 50,
  resetTimeout: 15000,          // 15s gives the auth service real time to stabilize
  volumeThreshold: 10,           // also require enough call volume before tripping
});
```

---

## Practice exercises

### Exercise 1 — easy

Write a function `unreliableApi()` that returns a resolved promise 70% of the time and a rejected promise 30% of the time (use `Math.random()`). Wrap it with `opossum` using a `timeout` of 1000ms and an `errorThresholdPercentage` of 50. Register a fallback that logs `"Fallback triggered"` and returns `{ degraded: true }`. Call `.fire()` 15 times in a loop (with a short delay between calls) and log each result along with the breaker's current state (`breaker.opened` is a boolean you can check).

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a small module `authServiceClient.js` that wraps a function `verifyAuthToken(authToken)` (simulate it with random failures/latency) in a circuit breaker. Requirements:
1. Configure `timeout`, `errorThresholdPercentage`, and `resetTimeout` sensibly for an auth check (should fail fast, recover reasonably quickly).
2. Register a fallback that returns `{ valid: false, reason: 'auth-service-unavailable' }` — never let the app crash.
3. Attach listeners for `open`, `halfOpen`, and `close` that `console.log` a timestamped message for each transition.
4. Export a function `checkAuth(authToken)` that calls the breaker and returns its result.
5. Write a small test script that calls `checkAuth()` in a loop and forces enough failures to observe the circuit opening, then wait long enough to see it attempt `halfOpen` again.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `ServiceBreakerRegistry` class that manages circuit breakers for **multiple** downstream services (e.g. `'inventory'`, `'payments'`, `'shipping'`) so a real backend doesn't have to hand-wire a separate breaker for each dependency. Requirements:
1. Constructor takes no arguments and initializes an empty internal map of breakers.
2. Method `register(serviceName, asyncFn, options)` creates a new `CircuitBreaker` for `asyncFn`, stores it under `serviceName`, wires up `open`/`halfOpen`/`close` logging that includes the service name, and attaches a default fallback if none is provided in `options.fallback`.
3. Method `call(serviceName, ...args)` looks up the breaker by name and calls `.fire(...args)`, throwing a clear error if no breaker was registered under that name.
4. Method `getStatus()` returns an object like `{ inventory: 'closed', payments: 'open', shipping: 'closed' }` reflecting each breaker's current state (hint: `breaker.opened` and `breaker.halfOpen` booleans exist on an opossum breaker instance).
5. Demonstrate it by registering three fake services with different failure rates, firing calls against all three in a loop, and logging `getStatus()` every few iterations to show breakers independently opening and closing.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
INSTALL
  npm install opossum

CORE API
  new CircuitBreaker(fn, options)   → wraps an async function
  breaker.fire(...args)             → calls fn (or fallback) through the breaker
  breaker.fallback(fn)               → registered fallback when open/failing
  breaker.opened                     → boolean, true if circuit is OPEN
  breaker.halfOpen                   → boolean, true if circuit is HALF-OPEN
  breaker.close()                    → manually force circuit closed
  breaker.open()                     → manually force circuit open

KEY OPTIONS
  timeout                    → ms before a call is treated as failed (hung request)
  errorThresholdPercentage   → % of failures in the window that trips the breaker
  resetTimeout               → ms to wait in OPEN before trying HALF-OPEN
  rollingCountTimeout        → size (ms) of the rolling stats window
  rollingCountBuckets        → number of buckets the window is divided into
  volumeThreshold            → min calls needed before % failure can trip it

STATES
  CLOSED     → normal, calls go through, failures are counted
  OPEN       → calls short-circuited immediately, fallback used
  HALF-OPEN  → one test call allowed through to check recovery

EVENTS TO LISTEN FOR
  'open'      → circuit just tripped open — alert/log this
  'halfOpen'  → about to test if the dependency recovered
  'close'     → circuit closed again — dependency is healthy
  'fallback'  → fallback function was invoked
  'success'   → underlying call succeeded
  'failure'   → underlying call failed
  'timeout'   → underlying call exceeded the timeout option

NEVER DO
  Wrap unrelated logic (validation, your own DB writes) inside the breaker
  Skip .fallback() — an open circuit without one throws an unhandled error
  Set resetTimeout unrealistically short — gives the dependency no recovery time
  Use one breaker instance for many unrelated dependencies — isolate per-dependency

WHY IT MATTERS
  Stops one failing dependency from exhausting connections/threads/memory
  and taking the whole backend down with it — fails fast instead of hanging.
```

---

## Connected topics

- **51 — Retry and timeout patterns** — circuit breakers are usually combined with retry-with-backoff and `AbortController` timeouts on the same call.
- **62 — Centralized error handling** — the fallback path from a tripped breaker should flow into the same `AppError`/error-response format the rest of the app uses.
- **65 — Health checks and readiness probes** — a breaker's `open` state is a strong signal to report as "not ready" or "degraded" in a `/health` endpoint.
