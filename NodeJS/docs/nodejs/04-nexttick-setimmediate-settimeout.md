# 04 — process.nextTick vs setImmediate vs setTimeout(0)

## What is this?

These three are all ways to say "run this code later, not right now" — but *when* exactly they run is very different. `process.nextTick` fires before any I/O or timer callbacks in the current iteration of the event loop. `Promise.then` fires right after nextTick. `setImmediate` fires in the **check phase**, after all I/O callbacks. `setTimeout(fn, 0)` fires in the **timers phase**, which comes before I/O. Think of them as different lanes at a checkpoint: nextTick is the emergency lane, Promises are the fast lane, setImmediate is the regular lane after traffic, and setTimeout(0) is the regular lane before traffic.

## Why does it matter for backend development?

In a backend server you constantly defer work — emitting events, propagating errors, scheduling cleanup, logging after a response. Choosing the wrong deferral mechanism causes:
- Callbacks firing in the wrong order breaking your middleware chain
- Starving I/O (blocking all file reads and network responses) with recursive `nextTick`
- Race conditions where a listener is attached after an event was already emitted
- Subtle test failures where async order assumptions break under load

Knowing exactly which queue each method uses lets you write correct, predictable async code.

---

## Syntax / API

```js
// ── process.nextTick(callback) ──────────────────────────────────────────────
// Queues callback in the nextTick queue.
// Drains completely before Promise microtasks and before any event loop phase.
process.nextTick(() => {
  console.log('nextTick fires');   // before Promises, before I/O, before timers
});

// ── Promise microtask (for comparison) ─────────────────────────────────────
Promise.resolve().then(() => {
  console.log('Promise fires');    // after nextTick queue drains, before I/O/timers
});

// ── setImmediate(callback) ──────────────────────────────────────────────────
// Queues callback in the check phase — runs AFTER the poll (I/O) phase completes.
setImmediate(() => {
  console.log('setImmediate fires');   // after I/O callbacks in same loop iteration
});

// ── setTimeout(fn, 0) ───────────────────────────────────────────────────────
// Queues callback in the timers phase with a minimum delay of ~1ms (not truly 0).
// Outside an I/O callback, order vs setImmediate is non-deterministic.
setTimeout(() => {
  console.log('setTimeout(0) fires');
}, 0);
```

---

## How it works — line by line

### Execution order visualised

```
  [ Call Stack — sync code runs here ]
           │
           │ sync code completes
           ▼
  ┌─────────────────────────────────────────┐
  │  MICROTASK QUEUES (run between phases)  │
  │  1. process.nextTick queue  ← first     │
  │  2. Promise microtask queue ← second    │
  └──────────────────┬──────────────────────┘
                     │ both queues empty
                     ▼
  ┌─────────────────────────────────────────┐
  │  EVENT LOOP PHASES                      │
  │                                         │
  │  timers      → setTimeout / setInterval │
  │     ↓ (microtasks drain between each)   │
  │  poll        → I/O callbacks            │
  │     ↓                                   │
  │  check       → setImmediate             │
  │     ↓                                   │
  │  close cbs   → socket.on('close')       │
  └─────────────────────────────────────────┘
```

### The determinism rule for setImmediate vs setTimeout(0)

```js
// OUTSIDE an I/O callback — order is NON-DETERMINISTIC
// (depends on how long OS took to prepare the timer)
setTimeout(() => console.log('timeout'), 0);
setImmediate(() => console.log('immediate'));
// Could print either order

// INSIDE an I/O callback — setImmediate ALWAYS wins
// Because we are already in the poll phase — check phase (setImmediate) is next
// timers phase comes only in the NEXT loop iteration
const fs = require('fs');
fs.readFile(__filename, () => {
  setTimeout(() => console.log('timeout'), 0);    // next loop iteration's timers phase
  setImmediate(() => console.log('immediate'));   // this loop iteration's check phase
  // immediate ALWAYS prints first here
});
```

### process.nextTick recursion danger

```js
// If a nextTick callback schedules another nextTick, it runs in the same drain pass
// This can STARVE the event loop — I/O callbacks never get to run
function dangerousRecursion() {
  process.nextTick(dangerousRecursion);   // ← never lets the loop advance
}

// Safe equivalent — setImmediate allows the loop to advance each iteration
function safeRecursion() {
  setImmediate(safeRecursion);   // ← loop advances, I/O can fire between calls
}
```

---

## Example 1 — basic

```js
// File: scheduling-order.js
// Maps out exactly when each scheduling method fires

console.log('1 — sync start');

setTimeout(() => console.log('6 — setTimeout(0)'), 0);

setImmediate(() => console.log('5 — setImmediate'));

Promise.resolve()
  .then(() => console.log('3 — Promise microtask 1'))
  .then(() => console.log('4 — Promise microtask 2 (chained)'));

process.nextTick(() => {
  console.log('2a — nextTick 1');
  process.nextTick(() => console.log('2b — nextTick 2 (scheduled inside nextTick)'));
  // nextTick scheduled inside a nextTick still runs before Promises
});

console.log('1b — sync end');

// EXACT OUTPUT:
// 1 — sync start
// 1b — sync end
// 2a — nextTick 1
// 2b — nextTick 2 (scheduled inside nextTick)  ← runs before Promises
// 3 — Promise microtask 1
// 4 — Promise microtask 2 (chained)
// 5 — setImmediate        (or 6 first — non-deterministic outside I/O)
// 6 — setTimeout(0)
```

---

## Example 2 — real world backend use case

```js
// File: event-emitter-safe-emit.js
// Real pattern: an EventEmitter that emits an event in its constructor.
// If you emit synchronously, any listener registered AFTER construction misses it.
// process.nextTick defers the emit to the next tick — giving callers time to attach listeners.

const EventEmitter = require('events');

class DataLoader extends EventEmitter {
  constructor(filePath) {
    super();
    this.filePath = filePath;

    // ❌ If we emitted 'ready' synchronously here, no one could listen yet
    // this.emit('ready');   ← listener registered below would miss this

    // ✅ Defer to nextTick — caller's code (synchronous) runs first,
    //    attaching the listener, then our emit fires
    process.nextTick(() => {
      this.emit('ready', { filePath: this.filePath });
    });
  }
}

// Caller code — synchronous, runs before the nextTick callback
const loader = new DataLoader('/data/users.json');

// This listener is attached synchronously, before the nextTick fires
loader.on('ready', ({ filePath }) => {
  console.log(`DataLoader ready — will load: ${filePath}`);
  // ✅ This fires correctly because process.nextTick gave us time to attach
});

// This is the same pattern used by Node's own stream and net modules internally
```

---

## Common mistakes

### Mistake 1 — Using process.nextTick where setImmediate is safer

```js
// ❌ WRONG — recursive nextTick blocks all I/O indefinitely
function watchConfig(configPath) {
  // Re-check config on every tick — starves everything else
  process.nextTick(() => {
    checkIfConfigChanged(configPath);
    watchConfig(configPath);   // schedules itself again immediately
  });
}

// ✅ CORRECT — setImmediate lets I/O callbacks run between iterations
function watchConfig(configPath) {
  setImmediate(() => {
    checkIfConfigChanged(configPath);
    watchConfig(configPath);   // re-schedules, but loop can advance between calls
  });
}
```

### Mistake 2 — Expecting setTimeout(fn, 0) to fire before setImmediate

```js
// ❌ WRONG — assuming this order is guaranteed
setTimeout(() => console.log('first'), 0);
setImmediate(() => console.log('second'));
// This assumption is wrong — the order can flip

// ✅ CORRECT — if you need setImmediate to always go first, be explicit about context
const fs = require('fs');
fs.readFile(__filename, () => {
  // Inside I/O callback: setImmediate always fires before setTimeout(0)
  setTimeout(() => console.log('second'), 0);
  setImmediate(() => console.log('first'));   // guaranteed first here
});
```

### Mistake 3 — Using nextTick for deferred I/O instead of setImmediate

```js
// ❌ WRONG — using nextTick to "batch" database writes
// This can delay I/O responses because nextTick drains before the poll phase
function saveUserData(userData) {
  process.nextTick(() => {
    dbConnection.query(`INSERT INTO users VALUES ?`, [userData]);
    // blocks other I/O callbacks until nextTick queue empties
  });
}

// ✅ CORRECT — use setImmediate for deferred I/O work
// It runs in the check phase — after other I/O callbacks have had their turn
function saveUserData(userData) {
  setImmediate(() => {
    dbConnection.query(`INSERT INTO users VALUES ?`, [userData]);
  });
}

// ✅ EVEN BETTER — just use async/await with Promises
async function saveUserData(userData) {
  await dbConnection.query(`INSERT INTO users VALUES ?`, [userData]);
}
```

---

## Practice exercises

### Exercise 1 — easy

Predict the exact output of this script WITHOUT running it first. Write your prediction as comments, then run it and check.

```js
process.nextTick(() => console.log('A'));
Promise.resolve().then(() => console.log('B'));
setTimeout(() => console.log('C'), 0);
process.nextTick(() => console.log('D'));
Promise.resolve().then(() => console.log('E'));
setImmediate(() => console.log('F'));
console.log('G');
```

```js
// My prediction (fill in the order):
// 1:
// 2:
// 3:
// 4:
// 5:
// 6:
// 7:
// Write your code here if needed
```

---

### Exercise 2 — medium

Build a `SafeEmitter` class that:
1. Extends `EventEmitter`
2. Takes a `channelName` string in its constructor
3. Emits a `'connected'` event with `{ channelName, timestamp: Date.now() }` — but deferred using `process.nextTick` so listeners can be attached synchronously after construction
4. Has a method `publish(message)` that emits a `'message'` event, but deferred via `setImmediate` (so in-flight I/O completes before the message is processed)

Test it by:
- Creating a `SafeEmitter` instance
- Attaching a `'connected'` listener synchronously after creation
- Calling `publish('hello from backend')` and attaching a `'message'` listener

Verify both listeners fire at the right time.

```js
// Write your code here
```

---

### Exercise 3 — hard

You are diagnosing a production bug. An API server processes requests in 3 stages:
1. **Validate** the request body (must run first — use `process.nextTick`)
2. **Enrich** with user data from a simulated async source (use `Promise.resolve().then`)
3. **Persist** to a simulated database (use `setTimeout(fn, 5)`)
4. **Audit log** the completed request (use `setImmediate`)

Write a `processRequest(requestBody)` function that runs all 4 stages in this order for a given request. Then call it 3 times simultaneously:

```js
processRequest({ requestId: 'REQ-001', userId: 'user_1' });
processRequest({ requestId: 'REQ-002', userId: 'user_2' });
processRequest({ requestId: 'REQ-003', userId: 'user_3' });
```

Each stage must log: `[requestId] [stage] — done`

Observe how the 3 requests interleave and write a comment explaining exactly why the validate stage of all 3 requests fires before any enrich stage, which fires before any persist stage.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
SCHEDULING METHODS — SUMMARY TABLE
┌─────────────────────┬──────────────────┬──────────────────────────────────────┐
│ Method              │ Queue/Phase       │ Use when                             │
├─────────────────────┼──────────────────┼──────────────────────────────────────┤
│ process.nextTick    │ nextTick queue   │ Highest priority defer; error prop;  │
│                     │ (before phases)  │ safe emit pattern                    │
├─────────────────────┼──────────────────┼──────────────────────────────────────┤
│ Promise.then        │ microtask queue  │ Standard async flow; most defer work │
│                     │ (before phases)  │ should use this                      │
├─────────────────────┼──────────────────┼──────────────────────────────────────┤
│ setImmediate        │ check phase      │ After I/O completes; safe recursion; │
│                     │ (after poll)     │ defer without starving I/O           │
├─────────────────────┼──────────────────┼──────────────────────────────────────┤
│ setTimeout(fn, 0)   │ timers phase     │ Loose "next iteration" scheduling;   │
│                     │ (~1ms minimum)   │ avoid in favor of setImmediate       │
└─────────────────────┴──────────────────┴──────────────────────────────────────┘

GUARANTEED ORDER
  nextTick   > Promise   > timers/I/O/setImmediate

ONLY INSIDE AN I/O CALLBACK
  setImmediate fires before setTimeout(0) — guaranteed

OUTSIDE AN I/O CALLBACK
  setImmediate vs setTimeout(0) — NOT guaranteed

DANGER ZONE
  Recursive process.nextTick → starves I/O → use setImmediate instead
  Recursive setImmediate     → safe, I/O can still fire between iterations

process.nextTick MAX DEPTH
  Node.js has a configurable limit — but hitting it is a design smell
  Default: 1000 (process.maxCallDepth)
```

---

## Connected topics

- **03 — The event loop in depth** — the phases and queues that explain why this order exists
- **19 — events module** — the EventEmitter safe-emit pattern that relies on process.nextTick
- **30 — timers in Node** — full coverage of setTimeout / setInterval / setImmediate
