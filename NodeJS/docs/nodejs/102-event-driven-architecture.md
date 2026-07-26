# 102 — Event-driven architecture basics

## What is this?

Event-driven architecture is a way of designing a system where parts communicate by announcing "something happened" (an **event**) instead of directly telling another part "go do this" (a **command**). The part that caused the change fires off a fact — `OrderPlaced`, `UserRegistered`, `PaymentFailed` — and any number of other parts can react to it without the original part knowing or caring who is listening. Think of a school fire alarm: the person who pulls it doesn't call every classroom individually and command them to evacuate — they just trigger one signal, and every room reacts independently, on its own terms.

## Why does it matter for backend development?

Backend systems grow by adding features, and every feature you bolt directly onto existing code (send a welcome email, update analytics, notify a Slack channel — all inside the `createUser` function) makes that function more fragile and harder to test. Event-driven design lets you emit `UserRegistered` once and have the email service, analytics service, and notification service each subscribe independently — none of them touch the original registration code, and you can add or remove a subscriber without redeploying the core logic. This is the foundation behind message queues (Topic 100), Redis pub/sub (Topic 101), RabbitMQ (Topic 103), and webhooks (Topic 104) — every one of those is "event-driven architecture" wearing a different transport.

---

## Syntax / API

```js
// events is Node's built-in module for emitting and listening to events IN-PROCESS
const EventEmitter = require('events');

// An event bus is just an EventEmitter used as a shared "announcement board"
const eventBus = new EventEmitter();

// ── Command style (what we are moving AWAY from) ───────────────────────────
// The caller directly invokes every dependent action — tightly coupled
function createUserCommandStyle(userId) {
  sendWelcomeEmail(userId);   // caller must know this exists
  logToAnalytics(userId);     // caller must know this exists too
  notifySlackChannel(userId); // adding a 4th action means editing THIS function
}

// ── Event style (what this topic teaches) ───────────────────────────────────
// The caller only announces a FACT — it has no idea who (if anyone) is listening
function createUserEventStyle(userId) {
  // emit() fires the event; 'user.registered' is the event NAME (a fact, past tense)
  eventBus.emit('user.registered', { userId, registeredAt: new Date() });
}

// Subscribers register themselves independently, anywhere in the codebase
eventBus.on('user.registered', (payload) => {
  console.log(`Send welcome email to ${payload.userId}`); // reacts to the fact
});

eventBus.on('user.registered', (payload) => {
  console.log(`Log signup for analytics: ${payload.userId}`); // another reaction
});
```

---

## How it works — line by line

`require('events')` loads Node's built-in `EventEmitter` class — the simplest possible in-process event bus. `new EventEmitter()` creates one shared object that acts as a bulletin board: anyone can post a notice (`emit`), and anyone can read notices of a certain type (`on`).

In the **command style** function, `createUserCommandStyle` directly calls three other functions by name. This means that function has to *know about* email sending, analytics, and Slack notifications — three unrelated concerns crammed into one place. If you want a fourth action, you edit this function again, and if `sendWelcomeEmail` throws, it can block the rest.

In the **event style** function, `createUserEventStyle` does exactly one thing: it calls `eventBus.emit('user.registered', payload)`. That line means "the fact that a user registered is now true — I am not commanding anyone to do anything, I am simply recording that it happened." The event name `'user.registered'` is written as a **noun + past-tense verb** — a fact, not an instruction — which is the key naming convention that separates events from commands (a command would be named `'sendWelcomeEmail'` — a verb telling someone what to do).

The two `eventBus.on(...)` calls register separate, independent listener functions. Neither listener knows about the other, and the code that emitted the event has no idea they exist. You could delete both listeners and `createUserEventStyle` would still run correctly — that is the definition of **decoupling**.

### Events vs commands — the core distinction

```
COMMAND                          EVENT
─────────────────────────────    ─────────────────────────────
"CreateOrder"                    "OrderCreated"
Tells someone what to do         Announces something that happened
Has exactly one handler          Can have zero, one, or many handlers
Sender expects a specific result Sender does not expect any result
Named as an imperative verb      Named as a noun + past-tense verb
Failure usually means retry      Failure of ONE listener shouldn't
the whole operation               break the other listeners
```

A useful rule: if renaming an action to past tense feels natural (`"place order"` → `"order placed"`), it's an event. If it doesn't (you can't naturally say `"validated input"` as the *reason* something else runs — validation is a step, not a fact worth broadcasting), it's a command and belongs inside a normal function call.

### Event sourcing — a very brief introduction

Most apps store **current state** (a `users` table row that gets overwritten on every update). **Event sourcing** instead stores **every event that ever happened** — `AccountOpened`, `MoneyDeposited`, `MoneyWithdrawn` — as an immutable, append-only log, and the "current balance" is calculated by replaying all events in order. This gives you a perfect audit trail (you can always answer "how did the balance get to $340?") and lets you rebuild state at any point in history, at the cost of more complex queries (you often need a separate read-optimized "projection" table, built by replaying events — a pattern called CQRS). You do not need to implement full event sourcing to use event-driven architecture — most backends only borrow the "emit events, let others react" idea, without going all the way to "the event log IS the database."

---

## Example 1 — basic

```js
// File: basic-event-bus.js
// A minimal in-process event bus using Node's built-in EventEmitter.

const EventEmitter = require('events'); // Node's built-in event system

const eventBus = new EventEmitter(); // one shared bus for the whole app

// Subscriber 1 — reacts to the order.placed event
eventBus.on('order.placed', (order) => {
  console.log(`[email-service] Sending confirmation for order ${order.orderId}`);
});

// Subscriber 2 — a completely separate concern, also reacting to the same event
eventBus.on('order.placed', (order) => {
  console.log(`[inventory-service] Reducing stock for order ${order.orderId}`);
});

// Subscriber 3 — using once() so it only fires the FIRST time this event happens
eventBus.once('order.placed', () => {
  console.log('[metrics] First order of the session recorded');
});

// The "producer" side — it only announces a fact, it does not call any of the above
function placeOrder(orderId, userId, totalAmount) {
  const order = { orderId, userId, totalAmount, placedAt: new Date().toISOString() };

  // emit() runs ALL registered listeners synchronously, in registration order
  eventBus.emit('order.placed', order);

  return order; // the function returns normally, unaware of what listeners did
}

placeOrder('ord_1001', 'user_42', 59.97); // triggers all three listeners above
placeOrder('ord_1002', 'user_88', 12.50); // 'once' listener does NOT fire again

module.exports = { eventBus, placeOrder };
```

---

## Example 2 — real world backend use case

```js
// File: src/events/userEventBus.js
// A shared event bus used across a signup flow, decoupling the core
// registration logic from side effects like email, audit logging, and welcome bonuses.

const EventEmitter = require('events');

// A dedicated class (not the generic EventEmitter) so we control the API surface
class UserEventBus extends EventEmitter {}

const userEventBus = new UserEventBus(); // exported singleton, shared app-wide

module.exports = userEventBus;
```

```js
// File: src/services/userService.js
// The CORE registration logic — knows nothing about email, audit logs, or bonuses.

const userEventBus = require('../events/userEventBus');
const dbConnection = require('../db/connection'); // pretend db client

async function registerUser(requestBody) {
  const { email, passwordHash } = requestBody;

  // Insert the user — the ONLY responsibility of this function
  const result = await dbConnection.query(
    'INSERT INTO users (email, password_hash, created_at) VALUES ($1, $2, NOW()) RETURNING id',
    [email, passwordHash]
  );
  const userId = result.rows[0].id;

  // Announce the fact — past tense, no instructions attached
  userEventBus.emit('user.registered', { userId, email, registeredAt: new Date() });

  return { userId, email }; // caller gets a clean result, unaware of side effects
}

module.exports = { registerUser };
```

```js
// File: src/listeners/sendWelcomeEmail.js
// Independent listener — can be deleted or broken without affecting registration.

const userEventBus = require('../events/userEventBus');
const mailer = require('../lib/mailer'); // pretend email client

userEventBus.on('user.registered', async (payload) => {
  try {
    await mailer.send({
      to: payload.email,
      subject: 'Welcome!',
      body: `Your account (id: ${payload.userId}) is ready.`,
    });
  } catch (err) {
    // A failure here does NOT crash registerUser — that is the whole point
    console.error('[welcome-email] failed:', err.message);
  }
});
```

```js
// File: src/listeners/logAuditTrail.js
// Another independent listener — writes an append-only audit record.

const userEventBus = require('../events/userEventBus');
const auditLog = require('../lib/auditLog'); // pretend append-only log writer

userEventBus.on('user.registered', (payload) => {
  auditLog.append({
    action: 'USER_REGISTERED',
    userId: payload.userId,
    timestamp: payload.registeredAt,
  });
});

// Startup file (e.g. src/server.js) just needs to require both listener
// files once so they attach themselves — no other coupling required:
// require('./listeners/sendWelcomeEmail');
// require('./listeners/logAuditTrail');
```

---

## Common mistakes

### Mistake 1 — Naming events as commands instead of facts

```js
// ❌ WRONG — imperative verb name makes it look like a command, confuses intent
eventBus.emit('sendEmail', { userId }); // reads like "please do this one specific thing"

// ✅ CORRECT — past-tense noun phrase describes a fact that already happened
eventBus.emit('user.registered', { userId });
// Any number of listeners can react however they want — sending email is
// just ONE possible reaction, not baked into the event's name
```

### Mistake 2 — Treating emit() as if it guarantees the listener ran successfully

```js
// ❌ WRONG — emit() returns before async listeners finish, and swallows their errors
function placeOrderWrong(orderId) {
  eventBus.emit('order.placed', { orderId }); // fires and forgets
  return { success: true }; // returned BEFORE listeners like email-sending finish
  // If a listener throws inside an async function, it becomes an unhandled
  // rejection that crashes the process (Topic 07) — emit() cannot catch it
}

// ✅ CORRECT — listeners handle their own errors internally, and heavy/critical
// work is handed off to a real queue (Topic 100) instead of an in-process emit
eventBus.on('order.placed', async (order) => {
  try {
    await sendConfirmationEmail(order); // own try/catch — never trust the emitter
  } catch (err) {
    console.error('[email] failed for order', order.orderId, err.message);
  }
});
```

### Mistake 3 — Using events for logic that MUST happen and MUST succeed

```js
// ❌ WRONG — charging the card is critical-path logic, not a side effect;
// hiding it behind an event means a listener failure silently loses the charge
function checkoutWrong(cartId) {
  eventBus.emit('cart.checked_out', { cartId }); // 'charge card' listener might fail silently
  return { status: 'ok' }; // lies — payment may never have happened
}

// ✅ CORRECT — critical, must-succeed steps stay as direct function calls;
// only the OPTIONAL side effects (email, analytics, notifications) become events
async function checkoutCorrect(cartId) {
  const payment = await chargeCard(cartId); // direct call — errors propagate, checkout fails loudly if this fails
  eventBus.emit('cart.checked_out', { cartId, paymentId: payment.id }); // now announce the fact
  return { status: 'ok', paymentId: payment.id };
}
```

---

## Practice exercises

### Exercise 1 — easy

Create a file that builds a small event bus for a blog. Emit an event named `'post.published'` carrying a `postId` and `authorId`. Register **two** separate listeners on that event: one that logs `"Notify subscribers about post <postId>"`, and another that logs `"Update author <authorId>'s post count"`. Call your publish function twice with different post IDs and confirm both listeners fire both times.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a `NotificationCenter` module (using `EventEmitter`) for a chat app that emits a `'message.sent'` event with `{ messageId, senderId, chatRoomId, text }`. Add three listeners:
1. One that "delivers" the message (just logs it) to every OTHER member of the room (assume a hardcoded array of room members).
2. One that increments an in-memory unread-count object keyed by `chatRoomId`.
3. One that uses `once()` to log `"First message of the day"` only for the very first `'message.sent'` event fired, no matter how many messages follow.

Then write a `sendMessage(...)` function that only emits the event — it must not directly call any of the three listener behaviors.

```js
// Write your code here
```

---

### Exercise 3 — hard

Design a tiny **event-sourced** bank account. Do NOT store a `balance` field directly. Instead:
1. Build an `Account` class with an internal, private array of events (e.g. `{ type: 'AccountOpened', amount }`, `{ type: 'MoneyDeposited', amount }`, `{ type: 'MoneyWithdrawn', amount }`).
2. Methods `open(initialAmount)`, `deposit(amount)`, and `withdraw(amount)` should each just **append** an event to the internal array — they must not modify a stored balance directly. `withdraw` should throw an error if replaying the events shows insufficient funds (calculate the balance by replaying events, do not track it separately).
3. A `getBalance()` method that computes the current balance by replaying ALL stored events from scratch every time it's called.
4. A `getHistory()` method that returns the full list of events (for audit purposes).
5. Also emit an EventEmitter event (e.g. `'account.balance_changed'`) after every deposit/withdraw so an outside listener could react (e.g. send a low-balance alert), separate from the internal event-sourcing log.

Test it by opening an account with 100, depositing 50, withdrawing 30, attempting to withdraw 500 (should throw), and printing the final balance and full history.

```js
// Write your code here
```

---

## Quick reference cheat sheet

```
EVENTS vs COMMANDS
  Command  → imperative verb    → "CreateOrder"   → exactly one handler, expects a result
  Event    → noun + past tense  → "OrderCreated"   → zero, one, or many handlers, fire-and-forget

WHY DECOUPLE WITH EVENTS
  Producer never knows who (if anyone) is listening
  Add/remove a listener without touching the code that emits the event
  One listener crashing should not break the producer or other listeners
  Makes systems easier to extend (add features by adding a listener, not editing core logic)

NODE'S BUILT-IN TOOL
  const EventEmitter = require('events');
  const bus = new EventEmitter();
  bus.on('event.name', handler)     → subscribe (fires every time)
  bus.once('event.name', handler)   → subscribe (fires only the first time)
  bus.emit('event.name', payload)   → publish — runs listeners SYNCHRONOUSLY, in order
  bus.off('event.name', handler)    → unsubscribe

WHAT emit() DOES NOT DO
  Does not wait for async listeners to finish
  Does not catch errors thrown inside listeners
  Does not guarantee delivery across process restarts (that needs a real queue/broker)

EVENT SOURCING (INTRO)
  Store WHAT HAPPENED (events), not just the current state
  Current state = replay all events in order
  Gives a perfect audit trail + point-in-time reconstruction
  Trade-off: reads get more expensive → often paired with a separate
  read-optimized "projection" (CQRS)

WHEN NOT TO USE EVENTS
  Critical-path steps that MUST succeed (charging a card, saving the core record)
    → keep these as direct, awaited function calls
  Steps where the caller needs the RESULT back immediately
    → that's a command/function call, not an event

IN-PROCESS vs OUT-OF-PROCESS EVENTS
  EventEmitter        → same process only, lost on restart, fastest
  Redis pub/sub       → across processes/servers, no persistence (Topic 101)
  BullMQ / RabbitMQ   → across processes, persisted, retries, delayed jobs (Topics 100, 103)
```

---

## Connected topics

- **19 — events module** — the `EventEmitter` class used throughout this topic; this doc applies the module to architectural decision-making
- **48 — Event-driven patterns** — goes deeper into the event bus pattern and decoupling techniques introduced here
- **101 — Pub/Sub with Redis** — takes this same "emit a fact, let others react" idea and makes it work across multiple processes and servers, not just within one Node process
