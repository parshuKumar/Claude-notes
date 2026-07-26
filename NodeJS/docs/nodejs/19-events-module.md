# 19 — events module

## What is this?

The `events` module gives you the `EventEmitter` class — the object that lets one part of your code "announce" that something happened, and lets other parts of your code "listen" for that announcement and react. Think of it like a radio station: the station broadcasts on a frequency without knowing who is tuned in, and any number of radios can tune into that frequency and react when a broadcast happens. `EventEmitter` is the same idea inside your Node.js process — one piece of code calls `.emit('somethingHappened')`, and any number of listeners registered with `.on('somethingHappened', ...)` react to it.

## Why does it matter for backend development?

Almost everything in Node.js is built on `EventEmitter` under the hood — HTTP servers emit `'request'`, streams emit `'data'` and `'end'`, TCP sockets emit `'connect'` and `'close'`, database drivers emit `'connect'` and `'error'`. As a backend developer you constantly listen to events emitted by Node's core modules and libraries, and you build your own emitters to decouple logic — for example, emitting a `'user:registered'` event so that sending a welcome email, logging an audit entry, and updating analytics can all happen independently without cluttering your registration function with unrelated code. This is the foundation of event-driven architecture, which shows up again in message queues, pub/sub, and microservices later in the curriculum.

---

## Syntax / API

```js
// The events module is built into Node.js — no install needed
const EventEmitter = require('events');

// Create an emitter instance — this object can emit and listen to events
const orderEmitter = new EventEmitter();

// on(eventName, listener) — register a listener that runs EVERY time the event fires
orderEmitter.on('order:placed', (order) => {
  console.log(`Order received: ${order.orderId}`);   // reacts every time
});

// once(eventName, listener) — register a listener that runs only ONE time, then auto-removes itself
orderEmitter.once('order:placed', (order) => {
  console.log('First order of the session!');         // runs only on the first emit
});

// emit(eventName, ...args) — fires the event, calling all registered listeners synchronously, in order
orderEmitter.emit('order:placed', { orderId: 'ORD-101' });
// → "Order received: ORD-101"
// → "First order of the session!"

orderEmitter.emit('order:placed', { orderId: 'ORD-102' });
// → "Order received: ORD-102"   (the once() listener is already gone, so it does not run again)

// off(eventName, listener) — removes ONE specific listener (alias for removeListener)
function logOrder(order) {
  console.log('Logging order:', order.orderId);
}
orderEmitter.on('order:placed', logOrder);
orderEmitter.off('order:placed', logOrder);   // this exact function reference is removed

// removeAllListeners(eventName?) — removes ALL listeners for one event, or every event if no name given
orderEmitter.removeAllListeners('order:placed');   // wipes every listener on this event
orderEmitter.removeAllListeners();                 // wipes every listener on every event — use with care

// listenerCount(eventName) — how many listeners are currently registered for an event
console.log(orderEmitter.listenerCount('order:placed'));   // → 0 after removeAllListeners

// emit() returns true if there were listeners, false if the event had no listeners
const hadListeners = orderEmitter.emit('order:cancelled', { orderId: 'ORD-103' });
console.log(hadListeners);   // → false — nobody was listening for 'order:cancelled'
```

---

## How it works — line by line

`require('events')` loads Node's built-in `EventEmitter` class — it ships with Node, so nothing needs to be installed. `new EventEmitter()` creates a fresh object that internally keeps a list (technically a map) of event names to arrays of listener functions — nothing is emitted or listened to yet, the object is just ready.

`.on(eventName, listener)` pushes `listener` onto the internal array for `eventName`. It does not run the function — it just remembers it for later. You can call `.on()` many times with the same event name to attach multiple independent listeners; all of them will run when that event fires.

`.once(eventName, listener)` does the same thing as `.on()`, but internally wraps your function so that the very first time it runs, it removes itself from the list. This means a `.once()` listener can never run twice, no matter how many times `.emit()` is called afterward.

`.emit(eventName, ...args)` looks up the array of listeners for `eventName` and calls every one of them **synchronously**, one after another, in the exact order they were registered, passing along any extra arguments you gave to `emit`. If two listeners are registered, `emit` does not return until both have finished running. If no listener is registered for that name, `emit` does nothing and simply returns `false`.

`.off(eventName, listener)` (an alias for the older `.removeListener()`) finds that exact function reference in the array for `eventName` and removes it — this is why you must save your listener in a named function or variable if you plan to remove it later; an anonymous inline arrow function can never be removed with `.off()` because you have no reference to pass back.

`.removeAllListeners(eventName)` clears the entire array for that one event name. Called with no argument at all, it clears every listener for every event on that emitter — a nuclear option best reserved for cleanup during shutdown.

---

## Example 1 — basic

```js
// File: src/examples/basic-emitter.js

// Import the built-in events module — no npm install required
const EventEmitter = require('events');

// Create a plain EventEmitter instance
const taskEmitter = new EventEmitter();

// Register a listener for the 'task:started' event
taskEmitter.on('task:started', (taskName) => {
  console.log(`Task started: ${taskName}`);   // runs every time this event fires
});

// Register a listener for the 'task:completed' event
taskEmitter.on('task:completed', (taskName, durationMs) => {
  console.log(`Task completed: ${taskName} in ${durationMs}ms`);
});

// Register a ONE-TIME listener — useful for setup that should only happen once
taskEmitter.once('task:started', () => {
  console.log('Listener attached: this only logs on the FIRST task');
});

// Fire the 'task:started' event with an argument
taskEmitter.emit('task:started', 'send-email');
// → "Task started: send-email"
// → "Listener attached: this only logs on the FIRST task"

// Fire it again — the once() listener does NOT run this time
taskEmitter.emit('task:started', 'generate-report');
// → "Task started: generate-report"

// Fire 'task:completed' with two arguments — both get passed through emit()
taskEmitter.emit('task:completed', 'send-email', 245);
// → "Task completed: send-email in 245ms"

// Check how many listeners remain on 'task:started'
console.log(taskEmitter.listenerCount('task:started'));   // → 1 (the once() listener is gone)
```

---

## Example 2 — real world backend use case

```js
// File: src/services/userService.js
// A user registration flow that decouples side effects using events —
// instead of one giant function doing everything, the registration logic
// just emits an event and lets independent listeners react.

const EventEmitter = require('events');

// This app-wide emitter acts as a lightweight internal event bus
const appEvents = new EventEmitter();

// ── Listener 1: send a welcome email ────────────────────────────────────────
appEvents.on('user:registered', (user) => {
  // In real code this would call an email service (e.g. nodemailer/SES)
  console.log(`[email] Sending welcome email to ${user.email}`);
});

// ── Listener 2: write an audit log entry ────────────────────────────────────
appEvents.on('user:registered', (user) => {
  console.log(`[audit] New account created: userId=${user.userId} at ${new Date().toISOString()}`);
});

// ── Listener 3: update analytics, but only ONCE for the very first signup ──
appEvents.once('user:registered', (user) => {
  console.log(`[analytics] First user of this process: ${user.userId}`);
});

// ── Listener 4: error handling — always listen for 'error' on emitters ─────
appEvents.on('error', (err) => {
  console.error('[appEvents] Unhandled emitter error:', err.message);
});

// The actual registration function stays small and focused —
// it only creates the user, then announces that it happened
function registerUser(requestBody) {
  const newUser = {
    userId: `user_${Date.now()}`,
    email: requestBody.email,
    name: requestBody.name,
  };

  // Pretend to save to a database here (covered in Phase 12 — Databases)
  console.log(`[db] Saved user ${newUser.userId} to database`);

  // Emit the event — every listener above reacts independently
  appEvents.emit('user:registered', newUser);

  return newUser;
}

// Simulate an incoming registration request
registerUser({ email: 'vakeelsaab791213@gmail.com', name: 'Vakeel Saab' });
// → [db] Saved user user_...
// → [email] Sending welcome email to vakeelsaab791213@gmail.com
// → [audit] New account created: userId=...
// → [analytics] First user of this process: user_...

module.exports = { appEvents, registerUser };
```

---

## Common mistakes

### Mistake 1 — Emitting `'error'` events with no listener attached

```js
// ❌ WRONG — Node treats 'error' as a SPECIAL event name.
// If you emit('error', ...) and there is NO listener for 'error',
// Node throws the error and CRASHES the entire process.
const EventEmitter = require('events');
const dbConnection = new EventEmitter();

dbConnection.emit('error', new Error('Connection refused'));
// → Uncaught exception, process exits — even though you never called throw()

// ✅ CORRECT — always attach an 'error' listener on any emitter that might emit one
const dbConnection2 = new EventEmitter();

dbConnection2.on('error', (err) => {
  console.error('Database connection error:', err.message);   // handled safely
});

dbConnection2.emit('error', new Error('Connection refused'));
// → "Database connection error: Connection refused" — process keeps running
```

### Mistake 2 — Trying to remove an anonymous listener with `off()`

```js
// ❌ WRONG — you can't remove a listener you never kept a reference to
const requestEmitter = new EventEmitter();

requestEmitter.on('request:received', (requestBody) => {
  console.log('Handling request:', requestBody.path);
});

// This does NOTHING — it's a different function object, not the one registered above
requestEmitter.off('request:received', (requestBody) => {
  console.log('Handling request:', requestBody.path);
});
// The original listener is still attached — off() silently found no match

// ✅ CORRECT — name the function and pass the same reference to both on() and off()
function handleRequest(requestBody) {
  console.log('Handling request:', requestBody.path);
}

requestEmitter.on('request:received', handleRequest);
requestEmitter.off('request:received', handleRequest);   // removes the exact same function
```

### Mistake 3 — Registering listeners inside a loop and hitting the max listener warning

```js
// ❌ WRONG — attaching a new listener every time a function runs causes a
// memory leak: listeners pile up forever and are never cleaned up.
// Node warns you after 10 listeners on the same event by default.
function handleIncomingSocket(socketEmitter) {
  socketEmitter.on('data', (chunk) => {
    console.log('Received chunk:', chunk.length);
  });
}

for (let i = 0; i < 15; i++) {
  handleIncomingSocket(sessionEmitter);   // adds a NEW listener every call
}
// → (node) MaxListenersExceededWarning: Possible EventEmitter memory leak detected.
//    15 data listeners added. Use emitter.setMaxListeners() to increase limit

// ✅ CORRECT — attach the listener ONCE, outside any loop, or clean up old ones first
sessionEmitter.on('data', (chunk) => {
  console.log('Received chunk:', chunk.length);
});
// If you genuinely need many listeners on purpose, raise the limit intentionally:
sessionEmitter.setMaxListeners(20);   // signals "this is expected", not a leak
```

---

## Practice exercises

### Exercise 1 — easy

Create an `EventEmitter` called `orderEmitter`. Register:
1. A listener on `'order:placed'` that logs `Order placed: {orderId}`
2. A `once()` listener on `'order:placed'` that logs `Welcome bonus applied!` (should only ever log once)
3. A listener on `'error'` that logs the error message safely

Then emit `'order:placed'` twice with different order objects, and emit an `'error'` event once to confirm your handler catches it without crashing the process.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a small `NotificationCenter` using `EventEmitter` composition (an object that **has** an emitter, not a class that extends one) with these requirements:
1. It exposes `subscribe(channel, listener)`, `unsubscribe(channel, listener)`, and `publish(channel, payload)` methods that internally call `.on()`, `.off()`, and `.emit()` on a private `EventEmitter`
2. It exposes a `subscriberCount(channel)` method using `listenerCount()`
3. Register two named (non-anonymous) listeners on a `'payment:success'` channel, publish an event, then unsubscribe one of them and publish again to prove only the remaining one reacts
4. Log the subscriber count before and after unsubscribing

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a `TaskQueue` class that **extends `EventEmitter`** and simulates processing background jobs:
1. Constructor takes no arguments and initializes an internal array `this.tasks = []`
2. `addTask(taskName)` pushes a task object (`{ taskName, status: 'pending' }`) into `this.tasks` and emits a `'task:added'` event with the task
3. `processNext()` takes the first `'pending'` task, marks it `'processing'`, emits `'task:started'`, then uses `setTimeout` to simulate async work (random 100–500ms) before marking it `'completed'` and emitting `'task:completed'` with the task
4. If `this.tasks` is empty when `processNext()` is called, emit an `'error'` event with a descriptive `Error` object instead of throwing
5. Track a running count of completed tasks with `getCompletedCount()`
6. Wire up listeners for all four events (`task:added`, `task:started`, `task:completed`, `error`) that log clearly labeled messages, then add three tasks and call `processNext()` for each one

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE API
  new EventEmitter()                   → create an emitter instance
  emitter.on(name, listener)           → register a listener that fires EVERY time
  emitter.once(name, listener)         → register a listener that fires ONCE, then self-removes
  emitter.emit(name, ...args)          → fire the event synchronously, calling listeners in order
  emitter.off(name, listener)          → remove ONE specific listener (alias: removeListener)
  emitter.removeAllListeners([name])   → remove all listeners for one event, or all events if omitted
  emitter.listenerCount(name)          → number of listeners currently registered for that event
  emitter.setMaxListeners(n)           → raise the default 10-listener warning threshold on purpose

BEHAVIOR
  emit() runs listeners SYNCHRONOUSLY, in registration order, before returning
  emit() returns true if there were listeners, false if none
  Listeners registered with on() run every time; once() listeners run only the first time
  off() only removes a listener if you pass the EXACT SAME function reference used in on()

THE SPECIAL 'error' EVENT
  emit('error', err) with NO 'error' listener attached → Node THROWS and crashes the process
  ALWAYS attach an .on('error', ...) handler on any emitter that might emit errors

BUILDING CUSTOM EMITTERS
  Option A — extend it:      class MyThing extends EventEmitter { ... }
  Option B — compose it:     class MyThing { constructor() { this.emitter = new EventEmitter(); } }
  Prefer composition for public APIs — it hides internal event wiring from consumers

COMMON GOTCHAS
  Anonymous listeners can never be removed with off() — always keep a named reference
  More than 10 listeners on one event triggers MaxListenersExceededWarning (likely a leak)
  emit() does NOT catch errors thrown inside listeners — one throwing listener stops the rest
  Listener order matters — they run in the exact order .on()/.once() were called

WHERE THIS SHOWS UP IN CORE NODE
  http.Server        → emits 'request', 'connection', 'close'
  net.Socket         → emits 'data', 'end', 'error', 'close'
  fs.ReadStream       → emits 'open', 'data', 'end', 'error'
  process             → emits 'exit', 'uncaughtException', 'SIGTERM'
```

---

## Connected topics

- **17 — fs module (streams and large files)** — every stream (`ReadStream`, `WriteStream`) is itself an `EventEmitter`, emitting `'data'`, `'end'`, and `'error'`
- **20 — http module** — Node's `http.Server` is built entirely on `EventEmitter`, emitting `'request'` for every incoming call
- **48 — event-driven patterns** — goes deeper into the event bus pattern used in Example 2, and how to decouple large backend systems using events instead of direct function calls
