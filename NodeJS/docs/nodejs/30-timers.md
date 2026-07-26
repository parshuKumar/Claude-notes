# 30 — timers in Node

## What is this?

Timers are functions that let you schedule code to run later instead of right now — `setTimeout` runs something once after a delay, `setInterval` runs something repeatedly on a fixed cadence, `setImmediate` runs something right after the current I/O cycle finishes, and `process.nextTick` runs something before Node does anything else at all. Think of them as different kinds of alarm clocks: one rings after N minutes, one rings every N minutes until you turn it off, one rings "as soon as I'm free," and one rings "before I even put my shoes on."

## Why does it matter for backend development?

Backend servers constantly need to defer work: retrying a failed database call after a delay, polling a job queue every few seconds, batching writes before flushing them to disk, or breaking up a huge loop so it doesn't block incoming HTTP requests. Get the wrong timer or misunderstand execution order, and you get bugs that are brutal to debug — a `setInterval` that never gets cleared and leaks memory for weeks, a `process.nextTick` recursion that starves the event loop and makes your API stop responding, or a retry delay that fires "immediately" instead of after the delay you expected. Every production Node service uses timers somewhere — rate limiters, cron-like polling, debounced logging, connection timeouts — so understanding exactly when each one fires is a core backend skill, not a nice-to-have.

---

## Syntax / API

```js
// ── setTimeout: run ONCE, after at least N milliseconds ─────────────────────
const timeoutId = setTimeout(() => {
  console.log('Ran once after 1000ms'); // callback body
}, 1000); // delay in ms — the MINIMUM wait, not a guarantee

clearTimeout(timeoutId); // cancels it — callback will never run if called before it fires

// ── setInterval: run REPEATEDLY, every N milliseconds, until cleared ────────
const intervalId = setInterval(() => {
  console.log('Ran again'); // fires over and over
}, 5000); // repeats every 5000ms

clearInterval(intervalId); // stops future repeats — MUST call this or it runs forever

// ── setImmediate: run once, right after the current I/O event phase ─────────
const immediateId = setImmediate(() => {
  console.log('Ran right after I/O callbacks'); // fires in the "check" phase
});

clearImmediate(immediateId); // cancels it, same idea as clearTimeout

// ── process.nextTick: run once, BEFORE the event loop continues at all ──────
process.nextTick(() => {
  console.log('Ran before any timer, I/O, or immediate'); // highest priority, not part of the loop phases
});

// ── Passing extra arguments to the callback ─────────────────────────────────
setTimeout((userId, action) => {
  console.log(`User ${userId} did ${action}`); // args after the delay are passed through
}, 1000, 'user_42', 'logout'); // 'user_42' and 'logout' become the callback's params
```

---

## How it works — line by line

- `setTimeout(callback, delay)` tells Node "wait at least `delay` milliseconds, then run this callback once." It returns a `Timeout` object (an ID) that you can pass to `clearTimeout()` to cancel it before it fires.
- The delay is a **minimum**, not exact — if the event loop is busy running other code when the timer expires, the callback waits until the loop is free. A `setTimeout(fn, 0)` does not run immediately; it still waits for the current call stack to finish and gets placed in the timer queue.
- `setInterval(callback, delay)` works like `setTimeout` but keeps firing every `delay` milliseconds forever, until you call `clearInterval()` with its returned ID. If you forget to clear it, it keeps the Node process alive and keeps consuming resources indefinitely.
- `setImmediate(callback)` schedules the callback to run in the "check" phase of the event loop, which happens right after the "poll" phase (where I/O callbacks like file reads and network responses are handled). This makes it useful for "run this right after any pending I/O, but don't block it."
- `process.nextTick(callback)` is not technically part of the event loop's phases at all — it is processed on a separate "microtask-like" queue that Node drains completely **before moving to the next phase of the event loop**, even before Promise `.then()` callbacks. This makes it the fastest way to defer code, but also the most dangerous: recursive `nextTick` calls can starve I/O forever.
- The general priority order for code scheduled during the *same* tick is: synchronous code first, then `process.nextTick` queue, then Promise microtasks, then the event loop phases (timers → pending callbacks → poll/I/O → check/`setImmediate` → close callbacks).
- `clearTimeout()` and `clearInterval()` are actually interchangeable internally in Node (both cancel either kind of timer), but you should still use the matching one for readability — `clearTimeout` for timeouts, `clearInterval` for intervals.

---

## Example 1 — basic

```js
// File: src/examples/timers-basic.js

console.log('1: synchronous code runs first'); // always runs before any timer

// Schedule a one-time callback after 1 second
setTimeout(() => {
  console.log('4: setTimeout fired after ~1000ms'); // runs later, after the loop cycles
}, 1000);

// Schedule a callback to run right after I/O / current poll phase
setImmediate(() => {
  console.log('3: setImmediate fired'); // runs before the 1000ms timeout, after nextTick
});

// Schedule a callback that jumps the entire queue
process.nextTick(() => {
  console.log('2: process.nextTick fired'); // runs before setImmediate and setTimeout
});

console.log('1b: still synchronous code'); // runs immediately, same as line 1

// Expected order:
// 1: synchronous code runs first
// 1b: still synchronous code
// 2: process.nextTick fired
// 3: setImmediate fired
// 4: setTimeout fired after ~1000ms

// ── clearTimeout example ─────────────────────────────────────────────────────
const reminderId = setTimeout(() => {
  console.log('This will never print'); // cancelled before it can fire
}, 2000);

clearTimeout(reminderId); // cancel the scheduled callback
console.log('Reminder was cancelled'); // confirms cancellation happened synchronously
```

---

## Example 2 — real world backend use case

```js
// File: src/services/retryWithBackoff.js
// Retries a flaky external API call (e.g. a payment gateway) using setTimeout
// with exponential backoff — a pattern used in almost every production backend.

function callPaymentGateway(orderId) {
  // Simulates an unreliable network call — fails 60% of the time
  return new Promise((resolve, reject) => {
    const success = Math.random() > 0.6; // fake success/failure
    setTimeout(() => {
      if (success) {
        resolve({ orderId, status: 'CHARGED' }); // pretend the payment succeeded
      } else {
        reject(new Error(`Payment gateway timeout for order ${orderId}`)); // simulate failure
      }
    }, 100); // pretend network latency of 100ms
  });
}

// Retries the call up to `maxAttempts` times, waiting longer between each attempt
function retryWithBackoff(orderId, attempt = 1, maxAttempts = 4) {
  return callPaymentGateway(orderId) // try the actual call
    .catch((error) => {
      if (attempt >= maxAttempts) {
        throw error; // out of retries — bubble the failure up to the caller
      }

      const delayMs = attempt * 500; // linear backoff: 500ms, 1000ms, 1500ms...
      console.log(`Attempt ${attempt} failed for order ${orderId}. Retrying in ${delayMs}ms`);

      // Wrap the retry in a Promise so setTimeout works with async/await
      return new Promise((resolve, reject) => {
        setTimeout(() => {
          retryWithBackoff(orderId, attempt + 1, maxAttempts) // recursive retry
            .then(resolve) // pass success up the chain
            .catch(reject); // pass final failure up the chain
        }, delayMs);
      });
    });
}

// Usage in an Express-style route handler:
async function chargeOrderHandler(orderId) {
  try {
    const result = await retryWithBackoff(orderId); // waits through all retries
    console.log('Payment succeeded:', result);
  } catch (error) {
    console.error('Payment permanently failed:', error.message); // all attempts exhausted
  }
}

chargeOrderHandler('order_9981'); // run the example
module.exports = { retryWithBackoff }; // export for use in real routes/tests

// ── Bonus: a job-queue poller using setInterval ─────────────────────────────
// Common pattern for lightweight background workers without a full queue library
function startJobPoller(dbConnection, intervalMs = 5000) {
  const pollerId = setInterval(async () => {
    const pendingJob = await dbConnection.fetchNextPendingJob(); // check for work
    if (pendingJob) {
      console.log(`Processing job ${pendingJob.id}`); // handle it
      await dbConnection.markJobComplete(pendingJob.id); // mark done
    }
  }, intervalMs); // repeats every intervalMs

  return () => clearInterval(pollerId); // return a "stop" function for graceful shutdown
}

// const stopPolling = startJobPoller(dbConnection);
// process.on('SIGTERM', stopPolling); // stop polling cleanly on shutdown (Topic 64)
```

---

## Common mistakes

### Mistake 1 — Forgetting to clear a setInterval, causing a memory/process leak

```js
// ❌ WRONG — interval never stops, keeps running even after the work is irrelevant,
// and keeps the Node process alive forever (never exits on its own)
function pollSessionExpiry(sessionId) {
  setInterval(() => {
    console.log(`Checking session ${sessionId}`); // runs forever, no way to stop it
  }, 10000);
}

// ✅ CORRECT — store the ID and provide a way to stop it
function pollSessionExpiry(sessionId) {
  const intervalId = setInterval(() => {
    console.log(`Checking session ${sessionId}`);
  }, 10000);

  return () => clearInterval(intervalId); // caller can stop polling when session ends
}

const stopPolling = pollSessionExpiry('sess_123');
// later, when the session ends:
stopPolling(); // frees the interval — no leak
```

### Mistake 2 — Assuming setTimeout(fn, 0) runs "immediately"

```js
// ❌ WRONG — expecting synchronous-like behavior from setTimeout(fn, 0)
console.log('Start');
setTimeout(() => {
  console.log('Timeout callback'); // this does NOT run before "End"
}, 0);
console.log('End');
// Actual output: Start, End, Timeout callback — NOT Start, Timeout callback, End

// ✅ CORRECT — understand that setTimeout ALWAYS defers to after the current
// synchronous code finishes, no matter how small the delay is
console.log('Start');
setTimeout(() => {
  console.log('Timeout callback — runs after ALL sync code, even with delay 0');
}, 0);
console.log('End');
// If you truly need "run after this stack finishes" ordering, use setImmediate
// or process.nextTick, and pick the one whose exact priority you actually need
```

### Mistake 3 — Recursive process.nextTick starving the event loop

```js
// ❌ WRONG — process.nextTick queue is fully drained before I/O ever gets a turn;
// recursing on it forever means incoming HTTP requests NEVER get processed
function unsafeRecursiveTick(count) {
  process.nextTick(() => {
    console.log(count);
    unsafeRecursiveTick(count + 1); // queues ANOTHER nextTick before I/O can run
  });
}
// unsafeRecursiveTick(0); // this would freeze the server — I/O starves forever

// ✅ CORRECT — use setImmediate or setTimeout for recursive/looping work so the
// event loop gets a chance to process I/O (like incoming requests) between iterations
function safeRecursiveLoop(count) {
  setImmediate(() => {
    console.log(count);
    if (count < 1000) {
      safeRecursiveLoop(count + 1); // yields to the I/O/poll phase between each step
    }
  });
}
safeRecursiveLoop(0); // server stays responsive throughout
```

---

## Practice exercises

### Exercise 1 — easy

Write a script that logs `'Server starting...'` immediately, then after a 2-second delay logs `'Server ready on port 3000'`. Store the timeout's ID in a variable. Immediately after scheduling it (before the 2 seconds pass), also schedule a `setTimeout` for 500ms that calls `clearTimeout` on the first timer and logs `'Startup aborted'` instead. Run it and confirm only the abort message prints, never the "ready" message.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a `debounceLogger(message, delayMs)` function that behaves like a debounce: every time it is called, it cancels any previously scheduled log for that same function and schedules a new one `delayMs` milliseconds later. Use `setTimeout` and `clearTimeout` internally (keep the timer ID in a variable outside the function, e.g. via a closure). Test it by calling `debounceLogger('save requested', 300)` five times in a tight loop with `setTimeout` spacing them 50ms apart — verify that only the LAST call's message actually gets logged, roughly 300ms after the last call.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `HeartbeatMonitor` class for a backend service that:
1. Takes a `serviceName` string and a `timeoutMs` number in its constructor.
2. Has a `beat()` method that should be called every time the monitored service reports it is alive — each call resets an internal timer.
3. If `beat()` is not called again within `timeoutMs` milliseconds, the monitor should log `` `${serviceName} is UNRESPONSIVE` `` exactly once (use `setTimeout`, and clear/reset it on every `beat()` call).
4. Has a `stop()` method that permanently stops monitoring (clears any pending timer, using `clearTimeout`).
5. Internally, also use `process.nextTick` to log `` `${serviceName} monitor initialized` `` right after construction, and explain in a comment why `nextTick` fires before any `setTimeout`, even one with a 0ms delay.

Test it:
```js
const monitor = new HeartbeatMonitor('payment-service', 1000);
monitor.beat(); // resets the 1000ms countdown
setTimeout(() => monitor.beat(), 500);  // beats again before timeout — should NOT log unresponsive
setTimeout(() => monitor.stop(), 2000); // stops monitoring after everything settles
```

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
FOUR TIMER TYPES
  setTimeout(fn, ms)     → run fn ONCE after at least ms milliseconds
  setInterval(fn, ms)    → run fn REPEATEDLY every ms milliseconds, forever, until cleared
  setImmediate(fn)       → run fn once, right after the current poll/I/O phase
  process.nextTick(fn)   → run fn before the event loop continues at all — highest priority

CANCELLING
  clearTimeout(id)   → cancels a setTimeout OR setInterval by its returned ID
  clearInterval(id)  → cancels an interval (same mechanism as clearTimeout internally)
  clearImmediate(id) → cancels a pending setImmediate

EXECUTION PRIORITY (same tick, roughly highest → lowest)
  1. Synchronous code (current call stack)
  2. process.nextTick queue (fully drained before anything else)
  3. Promise microtask queue (.then/.catch/.finally)
  4. Event loop phases: timers → pending callbacks → poll (I/O) → check (setImmediate) → close

setTimeout vs setImmediate (inside an I/O callback, order is guaranteed)
  Inside fs.readFile callback:
    setImmediate always fires BEFORE setTimeout(fn, 0) — because you're already
    past the poll phase, so "check" (immediate) comes next in that same loop tick

GOTCHAS
  setTimeout(fn, 0) does NOT mean "run now" — it still waits for the current
    stack AND the event loop to reach the timers phase
  Forgetting clearInterval() → the process never exits on its own (leak)
  Recursive process.nextTick() → starves I/O, server stops responding to requests
  Timer delay is a MINIMUM, not exact — a busy event loop delays firing further
  timer.unref() lets a timer NOT keep the process alive if it's the only thing pending
  timer.ref() re-attaches a timer so it DOES keep the process alive (default state)

WHEN TO USE WHICH
  setTimeout    → one-off delay: retries, request timeouts, debouncing
  setInterval   → recurring polling: heartbeat checks, job queue polling
  setImmediate  → "run after this I/O cycle, don't block current I/O"
  process.nextTick → "run before literally anything else" — use sparingly, rarely needed
    in application code, mostly used internally by libraries
```

---

## Connected topics

- **03 — The event loop in depth** — timers only make sense once you understand the call stack, libuv phases, and microtask/macrotask queues that decide when each timer callback actually runs.
- **04 — process.nextTick vs setImmediate vs setTimeout(0)** — a focused deep dive into the exact ordering rules briefly summarized here, with more edge cases and diagrams.
- **51 — Retry and timeout patterns** — builds directly on `setTimeout`-based backoff shown in Example 2, plus `AbortController` for cancelling in-flight requests.
