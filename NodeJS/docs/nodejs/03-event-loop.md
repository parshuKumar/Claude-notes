# 03 — The Node.js Event Loop in Depth

## What is this?

The event loop is the mechanism that allows Node.js to perform non-blocking I/O despite running on a single thread. It is a continuous loop that checks: "Is there any work to do? If so, what order should I do it in?" It manages multiple queues — each with different priority levels — and drains them in a strict, deterministic order on every "tick". Think of it like a hospital triage system: different queues are different urgency levels, and the loop always processes the highest-priority queue completely before moving to the next.

## Why does it matter for backend development?

Understanding the event loop lets you predict *exactly* when your code runs. This matters when you are:
- Debugging why a callback fires later than expected
- Deciding between `process.nextTick`, `setImmediate`, and `setTimeout(fn, 0)` for deferring work
- Understanding why a Promise callback runs before a `setTimeout` callback
- Preventing accidental starvation of the I/O queue by overusing `process.nextTick`
- Writing correct async code in servers where timing affects correctness

---

## Syntax / API

```js
// The four main ways to schedule deferred work — each goes to a different queue

// 1. process.nextTick — runs before ANYTHING else async (microtask, highest priority)
process.nextTick(() => {
  console.log('nextTick callback');   // runs before Promises, before I/O callbacks
});

// 2. Promise.then (microtask queue) — runs after nextTick, before I/O/timers
Promise.resolve().then(() => {
  console.log('Promise microtask');
});

// 3. setTimeout(fn, 0) — macrotask — runs after all microtasks are drained
setTimeout(() => {
  console.log('setTimeout 0ms');
}, 0);

// 4. setImmediate — runs in the "check" phase, after I/O callbacks
setImmediate(() => {
  console.log('setImmediate');
});

console.log('synchronous');

// OUTPUT ORDER (always):
// synchronous
// nextTick callback
// Promise microtask
// setTimeout 0ms    (or setImmediate — these two can swap OUTSIDE I/O callbacks)
// setImmediate
```

---

## How it works — line by line

### The event loop has 6 phases (libuv)

```
   ┌───────────────────────────────┐
   │           timers              │  ← executes setTimeout and setInterval callbacks
   └───────────────┬───────────────┘
                   │
   ┌───────────────▼───────────────┐
   │       pending callbacks       │  ← I/O errors from the previous loop iteration
   └───────────────┬───────────────┘
                   │
   ┌───────────────▼───────────────┐
   │         idle, prepare         │  ← internal use only (you won't use this)
   └───────────────┬───────────────┘
                   │
   ┌───────────────▼───────────────┐
   │            poll               │  ← retrieve new I/O events; execute I/O callbacks
   └───────────────┬───────────────┘      (fs.readFile, net connections, etc.)
                   │
   ┌───────────────▼───────────────┐
   │            check              │  ← setImmediate callbacks execute here
   └───────────────┬───────────────┘
                   │
   ┌───────────────▼───────────────┐
   │       close callbacks         │  ← e.g. socket.on('close', ...) handlers
   └───────────────┬───────────────┘
                   │
        (back to timers phase)
```

### The two microtask queues (run between EVERY phase transition)

Before the loop moves from one phase to the next, it completely drains **both** microtask queues:

```
  1. process.nextTick queue   ← drained first, completely
  2. Promise microtask queue  ← drained second, completely
```

This means microtasks run **between phases**, not during a phase. If a microtask schedules another microtask, that new one also runs before moving to the next phase — this is how `process.nextTick` can starve the event loop if called recursively.

### Complete priority order (top = highest priority)

```
Priority 1 — Synchronous code (call stack)
Priority 2 — process.nextTick queue (drained completely)
Priority 3 — Promise microtask queue (drained completely)
Priority 4 — timers phase (setTimeout / setInterval whose time has expired)
  → nextTick + Promise queues drained
Priority 5 — pending callbacks phase
  → nextTick + Promise queues drained
Priority 6 — poll phase (I/O callbacks — fs, net, etc.)
  → nextTick + Promise queues drained
Priority 7 — check phase (setImmediate)
  → nextTick + Promise queues drained
Priority 8 — close callbacks phase
```

---

## Example 1 — basic

```js
// File: event-loop-order.js
// Demonstrates the exact execution order of all scheduling mechanisms

console.log('--- START ---');       // 1. synchronous

process.nextTick(() => {
  console.log('nextTick 1');        // 3. nextTick queue (before promises)
});

Promise.resolve().then(() => {
  console.log('Promise 1');         // 4. microtask queue
});

setTimeout(() => {
  console.log('setTimeout 0');      // 6. timers phase
  process.nextTick(() => {
    console.log('nextTick inside setTimeout'); // 7. nextTick drains before check phase
  });
}, 0);

setImmediate(() => {
  console.log('setImmediate');      // 8. check phase (after nextTick above drains)
});

process.nextTick(() => {
  console.log('nextTick 2');        // 3b. still in nextTick queue drain
});

Promise.resolve().then(() => {
  console.log('Promise 2');         // 4b. still in microtask queue drain
});

console.log('--- END ---');         // 2. synchronous

// OUTPUT:
// --- START ---
// --- END ---
// nextTick 1
// nextTick 2
// Promise 1
// Promise 2
// setTimeout 0
// nextTick inside setTimeout
// setImmediate
```

---

## Example 2 — real world backend use case

```js
// File: request-lifecycle.js
// Shows when different types of work execute during an HTTP request lifecycle.
// Understanding this prevents subtle ordering bugs in middleware and handlers.

const http = require('http');
const fs = require('fs');

const server = http.createServer((req, res) => {
  if (req.url !== '/data') {
    res.end('Not found');
    return;
  }

  console.log('[1] Request handler entered — synchronous');

  // Schedule some deferred work at different priority levels
  process.nextTick(() => {
    console.log('[3] nextTick — runs before any I/O callbacks this tick');
    // Good use case: cleanup or error propagation that must happen
    // before any other async work in the current tick
  });

  Promise.resolve().then(() => {
    console.log('[4] Promise microtask — runs after nextTick drains');
    // Good use case: async validation, token verification
  });

  // I/O operation — goes to libuv poll phase
  fs.readFile('./package.json', 'utf8', (err, configData) => {
    console.log('[5] fs.readFile callback — poll phase');
    // This is where you build the response from real data
    res.writeHead(200, { 'Content-Type': 'application/json' });
    res.end(JSON.stringify({ status: 'ok', configSize: configData.length }));
  });

  setImmediate(() => {
    console.log('[6] setImmediate — check phase, after I/O callbacks');
    // Good use case: logging/metrics that should fire after the I/O completes
  });

  console.log('[2] Request handler exiting — synchronous');
});

server.listen(3000, () => {
  console.log('Server on http://localhost:3000/data');
  // Run: curl http://localhost:3000/data
  // Then watch the numbered order in the console
});
```

---

## Common mistakes

### Mistake 1 — Abusing process.nextTick and starving I/O

```js
// ❌ WRONG — recursive nextTick starves the event loop
// The poll phase (I/O callbacks) NEVER gets to run
function recursiveNextTick() {
  process.nextTick(recursiveNextTick);   // keeps scheduling itself
}
recursiveNextTick();
// fs.readFile callbacks, HTTP responses — all frozen indefinitely

// ✅ CORRECT — if you need to defer work recursively, use setImmediate
// setImmediate runs in the check phase — it won't block I/O
function recursiveImmediate() {
  setImmediate(recursiveImmediate);   // I/O callbacks can still run between iterations
}
```

### Mistake 2 — Assuming setTimeout(fn, 0) and setImmediate have a guaranteed order

```js
// ❌ WRONG assumption — this order is NOT guaranteed outside an I/O callback
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
// Output could be EITHER order depending on OS timing and system load

// ✅ CORRECT — inside an I/O callback, setImmediate ALWAYS comes before setTimeout
const fs = require('fs');
fs.readFile('./package.json', () => {
  setTimeout(() => console.log('timeout'), 0);    // always second
  setImmediate(() => console.log('immediate'));   // always first
  // Inside an I/O callback we are already in the poll phase —
  // check phase (setImmediate) comes before the next timers phase
});
```

### Mistake 3 — Mixing up process.nextTick and Promise resolution order

```js
// ❌ WRONG assumption — thinking Promises run before nextTick
Promise.resolve().then(() => console.log('promise'));
process.nextTick(() => console.log('nextTick'));

// ✅ CORRECT understanding:
// process.nextTick ALWAYS runs before Promise.then callbacks
// Output:
// nextTick    ← nextTick queue drains first
// promise     ← then Promise microtask queue drains

// WHY: Node.js has TWO microtask queues.
// nextTick queue has higher priority than the Promise queue.
// Both drain completely before any macrotask (setTimeout, setImmediate, I/O).
```

---

## Practice exercises

### Exercise 1 — easy

Without running the code, predict the exact output order of this script. Then run it and verify:

```js
console.log('A');

setTimeout(() => console.log('B'), 0);

Promise.resolve().then(() => console.log('C'));

process.nextTick(() => console.log('D'));

console.log('E');
```

Write your prediction first as a comment, then run it. If you were wrong, explain why in a comment.

```js
// Write your prediction here as comments, then verify:
// Write your code here
```

---

### Exercise 2 — medium

Write a function `deferredLogger(label)` that logs a message at each of the 4 scheduling levels:
- Synchronously (immediately)
- Via `process.nextTick`
- Via `Promise.resolve().then`
- Via `setImmediate`
- Via `setTimeout(fn, 0)`

Call `deferredLogger('REQUEST-1')` and `deferredLogger('REQUEST-2')` back to back.

Before running, predict: will all of REQUEST-1's logs appear before REQUEST-2's, or will they interleave? Write the expected output as a comment first, then verify.

```js
// Write your code here
```

---

### Exercise 3 — hard

You are building a request pipeline for an HTTP server. Implement a `runRequestPipeline(requestId)` function that simulates the following stages in the correct order:

1. **Auth check** — must happen via `process.nextTick` (highest priority, before any I/O)
2. **Cache lookup** — must happen via `Promise.resolve().then` (async but before I/O)
3. **Database read** — simulate with `setTimeout(fn, 10)` (actual async I/O)
4. **Response logging** — must happen via `setImmediate` (after I/O, in check phase)

Each stage must log: `[requestId] [stage] — executing`

Then call `runRequestPipeline('REQ-A')` and `runRequestPipeline('REQ-B')` simultaneously and observe how the two pipelines interleave. Add a comment explaining why they interleave the way they do.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
EVENT LOOP PHASES (in order)
  1. timers          → setTimeout, setInterval (when delay has expired)
  2. pending I/O     → I/O errors deferred from previous iteration
  3. idle/prepare    → internal only
  4. poll            → I/O callbacks (fs, net, http) — can BLOCK here waiting for I/O
  5. check           → setImmediate callbacks
  6. close callbacks → socket.on('close'), etc.

MICROTASK QUEUES (run between EVERY phase transition, highest priority)
  1. process.nextTick queue  ← drained first, completely
  2. Promise microtask queue ← drained second, completely

PRIORITY ORDER (top = runs first)
  Sync code
  → process.nextTick
  → Promise.then / Promise microtasks
  → setTimeout(fn, 0) / setInterval
  → I/O callbacks (fs.readFile, etc.)
  → setImmediate
  → close callbacks

KEY RULES
  - process.nextTick ALWAYS before Promise.then
  - setImmediate ALWAYS before setTimeout INSIDE an I/O callback
  - setImmediate vs setTimeout OUTSIDE I/O callback — ORDER NOT GUARANTEED
  - Recursive process.nextTick → starves I/O (bad) — use setImmediate instead
  - All microtasks drain completely before moving to next phase

WHEN TO USE WHAT
  process.nextTick  → error propagation, cleanup in same tick, high-priority defer
  Promise           → standard async control flow (prefer this over nextTick)
  setImmediate      → defer to after I/O callbacks, safe recursion
  setTimeout(fn,0)  → schedule for "next event loop iteration" loosely
```

---

## Connected topics

- **02 — How Node.js works internally** — the single-thread + libuv foundation this topic builds on
- **04 — process.nextTick vs setImmediate vs setTimeout(0)** — dedicated deep dive into the three deferred scheduling methods
- **30 — timers in Node** — full coverage of setTimeout/setInterval/setImmediate and their event loop interaction
