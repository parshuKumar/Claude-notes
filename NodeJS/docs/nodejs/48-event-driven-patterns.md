# 48 — Event-driven patterns

## What is this?

Event-driven patterns are ways of structuring code so that instead of one part of your app directly calling another, it **emits an event** and lets any number of listeners react to it, without either side knowing about the other. Think of a restaurant kitchen bell: the chef rings it when an order is ready — she doesn't know or care which waiter picks it up, or how many waiters are listening. Node.js's built-in `EventEmitter` is the tool that lets your code "ring bells" like this throughout an application.

## Why does it matter for backend development?

Backend systems constantly need to react to things happening — a user signed up, a payment succeeded, a file finished uploading, a job completed. If you hard-wire every one of these reactions directly into the code that triggers them (send welcome email, update analytics, notify Slack — all inside the signup handler), that handler becomes a tangled, hard-to-test, hard-to-extend mess. Event-driven patterns **decouple** the "what happened" from the "what to do about it," so you can add a new reaction (like a new analytics hook) without touching the original code at all. Node itself is built this way internally — `http.Server`, streams, and `process` are all EventEmitters — so understanding this pattern is understanding how Node thinks.

---

## Syntax / API

```js
// The events module ships with Node — no install needed
const EventEmitter = require('events');

// Create an emitter instance — this is the "bell" objects listen to
const orderEvents = new EventEmitter();

// Register a listener — runs every time the event fires
orderEvents.on('order:placed', (order) => {
  console.log(`Listener A: order ${order.orderId} placed`);
});

// Register a SECOND independent listener for the same event
orderEvents.on('order:placed', (order) => {
  console.log(`Listener B: sending confirmation email for ${order.orderId}`);
});

// Register a listener that only fires ONCE, then auto-removes itself
orderEvents.once('order:placed', (order) => {
  console.log(`Listener C: first order ever — send a welcome gift for ${order.orderId}`);
});

// Emit (fire) the event — every matching listener runs synchronously, in order added
orderEvents.emit('order:placed', { orderId: 'ord_101', total: 49.99 });

// Remove a specific listener when you no longer need it (prevents memory leaks)
function logOrder(order) { console.log(order); }
orderEvents.on('order:placed', logOrder);
orderEvents.off('order:placed', logOrder);      // 'off' is an alias for removeListener

// Remove ALL listeners for an event (use with care — usually only in tests/shutdown)
orderEvents.removeAllListeners('order:placed');

// Listen for the built-in 'error' event — EventEmitter throws if 'error' has no listener
orderEvents.on('error', (err) => console.error('Order events failed:', err.message));

// Custom class that IS an EventEmitter — the standard way to build your own emitters
class PaymentProcessor extends EventEmitter {
  charge(amount) {
    // ... real charge logic would go here ...
    this.emit('payment:success', { amount });   // 'this' refers to the emitter itself
  }
}
```

---

## How it works — line by line

`require('events')` loads Node's built-in event system — no third-party package needed. `new EventEmitter()` creates an object whose entire job is to keep an internal list of "who is listening for what." Calling `.on(eventName, callback)` adds a callback to that internal list under the given event name — nothing runs yet, it just registers interest. Calling `.emit(eventName, ...args)` looks up every callback registered under that name and calls each one, **synchronously**, in the exact order they were added, passing along any extra arguments. `.once()` behaves like `.on()` but Node automatically unregisters that listener right after it fires one time. `.off()` (or its older name, `.removeListener()`) deletes one specific callback from the list so it stops reacting to future emits. The special `'error'` event name is treated differently by Node: if you `emit('error', ...)` and there is no listener registered for it, Node throws the error and can crash the process — so any EventEmitter that might emit errors should always have an `'error'` listener attached. Finally, extending `EventEmitter` with `class PaymentProcessor extends EventEmitter` means every instance of `PaymentProcessor` inherits `.on()`, `.emit()`, `.once()`, etc. for free, which is exactly how Node's own core classes (like HTTP servers and streams) are built.

---

## Example 1 — basic

```js
// File: src/examples/eventBasics.js

const EventEmitter = require('events');   // built-in module, no npm install

// A bare emitter is enough to demonstrate the pattern before wrapping it in a class
const userEvents = new EventEmitter();

// Listener 1 — logs every signup to the console
userEvents.on('user:signup', (userId) => {
  console.log(`[log] new user signed up: ${userId}`);
});

// Listener 2 — pretends to send a welcome email
userEvents.on('user:signup', (userId) => {
  console.log(`[email] welcome email queued for ${userId}`);
});

// Listener 3 — runs only for the very first signup event ever fired
userEvents.once('user:signup', (userId) => {
  console.log(`[analytics] first signup observed this run: ${userId}`);
});

// Always attach an 'error' listener — otherwise an emitted error can crash the process
userEvents.on('error', (err) => {
  console.error('[user:events] unexpected error:', err.message);
});

// Fire the event twice — 'once' listener only reacts the first time
userEvents.emit('user:signup', 'user_501');
userEvents.emit('user:signup', 'user_502');

// Output order:
// [log] new user signed up: user_501
// [email] welcome email queued for user_501
// [analytics] first signup observed this run: user_501
// [log] new user signed up: user_502
// [email] welcome email queued for user_502
// (the analytics listener does NOT run again for user_502)
```

---

## Example 2 — real world backend use case

```js
// File: src/services/orderService.js
// A checkout flow that decouples "an order was placed" from everything that
// reacts to it — email, inventory, analytics — using an internal event bus.

const EventEmitter = require('events');

// One shared emitter acts as the app's internal "event bus" for order lifecycle events
class OrderEventBus extends EventEmitter {}
const orderBus = new OrderEventBus();

// Required 'error' listener — prevents a single failing handler from crashing the app
orderBus.on('error', (err) => {
  console.error('[orderBus] listener threw:', err.message);
});

// ── Listener: reserve inventory when an order is placed ─────────────────────
orderBus.on('order:placed', async ({ orderId, items }) => {
  try {
    // await inventoryService.reserve(items);  // real DB call would go here
    console.log(`[inventory] reserved stock for order ${orderId}`);
  } catch (err) {
    orderBus.emit('error', err);   // route failures through the bus, not a crash
  }
});

// ── Listener: send confirmation email — independent from inventory logic ────
orderBus.on('order:placed', async ({ orderId, customerEmail }) => {
  try {
    // await mailService.send(customerEmail, 'Order confirmed', ...);
    console.log(`[email] confirmation sent to ${customerEmail} for ${orderId}`);
  } catch (err) {
    orderBus.emit('error', err);
  }
});

// ── Listener: record an analytics event — again, fully independent ─────────
orderBus.on('order:placed', ({ orderId, total }) => {
  console.log(`[analytics] order ${orderId} recorded — $${total}`);
});

// ── The function callers actually invoke — it knows NOTHING about email/inventory ──
function placeOrder(requestBody) {
  const order = {
    orderId: `ord_${Date.now()}`,
    items: requestBody.items,
    customerEmail: requestBody.customerEmail,
    total: requestBody.total,
  };

  // saveOrderToDb(order) would happen here in a real app

  // Firing one event triggers every reaction above — none of them are hard-wired in
  orderBus.emit('order:placed', order);

  return order;   // caller (an Express route, for example) responds to the client immediately
}

module.exports = { placeOrder, orderBus };

// Usage in an Express route handler:
// app.post('/checkout', (req, res) => {
//   const order = placeOrder(req.body);
//   res.status(201).json(order);   // client gets a fast response — side effects run via events
// });
```

---

## Common mistakes

### Mistake 1 — Emitting an 'error' event with no listener attached

```js
// ❌ WRONG — emitting 'error' with no 'error' listener CRASHES the whole process
const EventEmitter = require('events');
const jobQueue = new EventEmitter();

jobQueue.emit('error', new Error('job failed'));
// Node throws this uncaught, even inside a try/catch around emit() — process exits

// ✅ CORRECT — always register an 'error' listener before anything can emit one
const jobQueue2 = new EventEmitter();
jobQueue2.on('error', (err) => {
  console.error('[jobQueue] handled:', err.message);   // logged safely, process survives
});
jobQueue2.emit('error', new Error('job failed'));
```

### Mistake 2 — Assuming emit() waits for async listeners to finish

```js
// ❌ WRONG — emit() is synchronous; it does NOT await async listener callbacks
orderBus.on('order:placed', async (order) => {
  await sendConfirmationEmail(order);   // this runs, but emit() doesn't wait for it
});

function placeOrderWrong(requestBody) {
  const order = { orderId: 'ord_1', ...requestBody };
  orderBus.emit('order:placed', order);
  return order;   // returns BEFORE the email listener's await resolves — fine for
                   // fire-and-forget, but a bug if you assumed side effects finished
}

// ✅ CORRECT — if you truly need to wait, don't rely on events for that step;
// call the async function directly, or track promises the listeners kick off
async function placeOrderRight(requestBody) {
  const order = { orderId: 'ord_1', ...requestBody };
  orderBus.emit('order:placed', order);       // fire independent side effects
  await sendConfirmationEmail(order);         // await the one thing you truly need done
  return order;
}
```

### Mistake 3 — Registering listeners repeatedly, causing a memory leak

```js
// ❌ WRONG — adding a new listener on every request means listeners pile up forever
function handleConnection(dbConnection) {
  // Every call to this function adds ANOTHER listener that never gets removed
  dbConnection.on('data', (chunk) => processChunk(chunk));
}
// After 1000 requests, 'data' has 1000 listeners — Node warns:
// "MaxListenersExceededWarning: Possible EventEmitter memory leak detected"

// ✅ CORRECT — attach the listener once (outside the per-request path), or remove it after use
function setupConnectionOnce(dbConnection) {
  dbConnection.on('data', (chunk) => processChunk(chunk));   // attached one time only
}

// If a listener truly must be per-request, always clean it up:
function handleRequestScoped(dbConnection) {
  const onData = (chunk) => processChunk(chunk);
  dbConnection.on('data', onData);
  dbConnection.once('end', () => dbConnection.off('data', onData));  // clean up after use
}
```

---

## Practice exercises

### Exercise 1 — easy

Create an `EventEmitter` called `notificationBus`. Register two separate listeners on a `'notify'` event: one that logs `"[sms] Sending SMS: <message>"` and one that logs `"[push] Sending push notification: <message>"`. Also register an `'error'` listener that logs any error message safely. Emit the `'notify'` event once with a sample message, and confirm both listeners run. Then emit an `'error'` event with a test `Error` object and confirm the process does not crash.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a `UserService` class that `extends EventEmitter`. It should have a `register(userId, email)` method that:
1. Simulates saving the user (just log it)
2. Emits a `'user:registered'` event with `{ userId, email }`

Outside the class, attach three independent listeners to a `UserService` instance for `'user:registered'`:
- One that logs a welcome message
- One that logs "adding user to mailing list"
- One that uses `.once()` to log "first registration this session" — and prove it only fires on the first call by registering two different users and checking the output

Also attach an `'error'` listener on the instance before calling `register()` at all.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a small **event bus module** (`eventBus.js`) that any part of an app can import and use — this is the event bus pattern used to decouple whole modules from each other. Requirements:
1. Export a single shared `EventEmitter` instance (not a class — one instance everyone imports)
2. Attach a default `'error'` listener inside the module itself so consumers never crash the app by forgetting one
3. Add a helper function `emitSafely(eventName, payload)` that wraps `.emit()` in a `try/catch`, routing any thrown error to the bus's own `'error'` event instead of letting it escape
4. In a separate file, simulate three independent modules that all listen to a `'payment:completed'` event: an `invoicingModule` (logs "generating invoice"), a `loyaltyModule` (logs "adding loyalty points"), and a `fraudCheckModule` (deliberately throws an error inside its listener, to prove step 3 catches it and the app keeps running)
5. Fire the event once with a `{ userId, amount }` payload and show that all three modules react independently and the thrown error does not crash the process

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
CORE API (require('events'))
  new EventEmitter()              → creates an emitter instance
  emitter.on(name, fn)             → register a listener (runs every time)
  emitter.once(name, fn)           → register a listener (runs only the first time)
  emitter.emit(name, ...args)      → fire the event — calls listeners SYNCHRONOUSLY, in order
  emitter.off(name, fn)            → remove one specific listener (alias: removeListener)
  emitter.removeAllListeners(name) → remove every listener for that event name
  emitter.listenerCount(name)      → how many listeners are registered
  emitter.setMaxListeners(n)       → raise the default warning limit (default: 10)

BUILDING YOUR OWN EMITTER
  class MyService extends EventEmitter {}   → instances inherit on/emit/once for free
  this.emit('some:event', payload)          → called from inside the class's own methods

THE 'error' EVENT IS SPECIAL
  emit('error', err) with NO listener registered → Node throws it, can crash the process
  ALWAYS attach: emitter.on('error', (err) => { ... })  before anything can emit one

EMIT IS SYNCHRONOUS
  emit() does not await async listeners — it fires them and moves on immediately
  do not assume side effects finished just because emit() returned

EVENT BUS PATTERN
  One shared EventEmitter instance, imported by multiple unrelated modules
  Producers emit; consumers subscribe — neither side imports the other
  Great for decoupling: order placed → email, inventory, analytics all react independently

COMMON GOTCHAS
  Adding a listener inside a per-request function → memory leak (MaxListenersExceededWarning)
  Forgetting to .off() a listener you no longer need
  Expecting listener order across MULTIPLE emitters — order is only guaranteed within ONE emit() call
  Naming clashes — use namespaced event names like 'order:placed', not just 'placed'
```

---

## Connected topics

- **19 — events module** — the full `EventEmitter` API this document builds on: `on`, `emit`, `once`, `off`, `removeAllListeners`, and building custom emitters
- **102 — Event-driven architecture basics** — takes this same idea to a system-wide scale: events vs commands, and why whole services decouple with events instead of direct calls
- **100 — Message queues (BullMQ)** — the production-grade, persistent version of an event bus — jobs, retries, and delayed processing backed by Redis instead of an in-memory `EventEmitter`
