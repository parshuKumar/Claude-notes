# 41 — Observer Pattern (aka Publish–Subscribe)

## Category: LLD Patterns

---

## What is this?

The **Observer pattern** creates a **one-to-many** relationship between objects: one object (the **Subject**) holds a list of dependent objects (the **Observers**), and whenever the Subject's state changes, it **automatically notifies every observer** — without knowing or caring what any of them actually do.

Think of a **newspaper subscription**. The newspaper (Subject) doesn't know who you are or why you read it. You (Observer) hand over your address once — you *subscribe*. Every morning a new issue is printed and it lands on every subscriber's doorstep automatically. You can cancel anytime and the paper keeps running. The newspaper never calls you to ask "do you still want it?" — it just publishes, and delivery happens.

That's Observer: **subscribe once, get notified forever, unsubscribe when done.**

---

## Why does it matter?

Without Observer, the object that *knows something happened* has to **directly call** everyone who cares:

```javascript
// The nightmare: the Order class hard-codes every side effect
class OrderService {
  placeOrder(order) {
    this.db.save(order);
    this.emailService.sendConfirmation(order);      // ← coupling
    this.inventoryService.decrement(order.items);   // ← coupling
    this.analytics.track('order_placed', order);    // ← coupling
    this.loyaltyService.awardPoints(order);         // ← coupling
    this.fraudService.scan(order);                  // ← coupling
    // Product: "also send an SMS" → edit this file AGAIN
  }
}
```

`OrderService` now depends on five unrelated subsystems. Every new "when an order is placed, also do X" requirement means **editing the order code** — a direct violation of the Open/Closed Principle (recall from [15 — Open/Closed](./15-solid-open-closed.md): open for extension, closed for modification). Testing `placeOrder` now needs five mocks. If the analytics service throws, the order fails.

**Interview angle:** Observer is the single most-asked behavioral pattern. It shows up in *every* LLD problem that has notifications: "notify the rider when a driver is assigned," "notify subscribers when a video is uploaded," "update the display when the stock price changes." Interviewers also love the follow-up: *"how is this different from a message queue?"*

**Real-work angle:** You already use it every day. `button.addEventListener('click', fn)` is Observer. `emitter.on('data', fn)` is Observer. `useEffect` + a store subscription is Observer. And the #1 production bug it causes — **memory leaks from forgotten unsubscribes** — is something you *will* debug in your career.

---

## The core idea — explained simply

### The Newspaper Subscription Analogy

Imagine *The Daily Bugle*.

1. **The publisher** (the newspaper office) prints an issue whenever there's news. It maintains a **subscriber list** — just a list of addresses.
2. **Subscribers** sign up by giving their address. They don't tell the Bugle *why* they want it. One subscriber reads it at breakfast, another lines a birdcage with it, a third clips coupons. The Bugle doesn't care.
3. When a new issue is printed, the Bugle **walks the subscriber list** and delivers a copy to each address.
4. A subscriber can **cancel** — their address is struck off the list, and delivery stops.
5. The Bugle is **not blocked** waiting for you to read the paper. It just drops it and moves on. (…in a *good* implementation. We'll see that in-process Observer often *is* blocking — that's a real gotcha.)

Map it back:

| Newspaper world | Observer pattern | Node.js `EventEmitter` |
|---|---|---|
| The Daily Bugle office | **Subject** (a.k.a. Publisher / Observable) | The `EventEmitter` instance |
| Your home address | **Observer** (a listener object/function) | The callback function |
| "Sign me up" | `subject.subscribe(observer)` | `emitter.on('news', fn)` |
| "Cancel my subscription" | `subject.unsubscribe(observer)` | `emitter.off('news', fn)` |
| A new issue is printed | `subject.notify(data)` | `emitter.emit('news', data)` |
| Today's headlines | The **event payload** | The args passed to `emit` |
| Sports desk vs. finance desk | Multiple **event types / channels** | `'sports'` vs `'finance'` event names |
| The Bugle knows your address | **Subject holds direct references to observers** | The emitter holds the callbacks |

That last row is the one interviewers probe. In classic Observer, **the Subject keeps a reference to every Observer.** That reference is exactly what causes memory leaks, and it's exactly what distinguishes Observer from broker-based Pub/Sub (where publisher and subscriber never touch each other at all).

---

## Key concepts inside this topic

### 1. The problem it solves — start with the pain

Here's a stock ticker written *without* Observer.

```javascript
// ---------- BAD: the subject hard-codes its dependents ----------
class StockTicker {
  constructor(priceDisplay, alertService, portfolioTracker) {
    this.price = 0;
    // The ticker must be handed every consumer up front.
    this.priceDisplay = priceDisplay;
    this.alertService = alertService;
    this.portfolioTracker = portfolioTracker;
  }

  setPrice(symbol, price) {
    this.price = price;
    // Every consumer is named here. Adding a 4th means editing this method.
    this.priceDisplay.render(symbol, price);
    this.alertService.checkThresholds(symbol, price);
    this.portfolioTracker.revalue(symbol, price);
  }
}
```

Problems, named precisely:
- **Tight coupling.** `StockTicker` imports and knows the API of three unrelated classes.
- **OCP violation.** A new consumer = modifying `setPrice`.
- **Untestable.** To test `setPrice` you must construct three collaborators.
- **No runtime flexibility.** You can't add or drop a consumer while the program runs.
- **Fragile.** If `alertService.checkThresholds` throws, the portfolio never gets revalued.

### 2. The structure — who plays which role

Four participants:

| Participant | Role | In our example |
|---|---|---|
| **Subject** (Observable) | Owns the state. Keeps the observer list. Exposes `subscribe` / `unsubscribe` / `notify`. | `StockTicker` |
| **Observer** (interface) | The contract every listener must satisfy — usually a single `update(data)` method. | `Observer` base class |
| **ConcreteObserver** | An actual listener with its own reaction. | `PriceDisplay`, `AlertService`, `PortfolioTracker` |
| **Client** | Wires them together: creates the subject, creates observers, subscribes them. | `main()` |

The critical inversion: **the Subject depends on the abstract `Observer` interface, not on any concrete observer.** That's Dependency Inversion (topic [18](./18-solid-dependency-inversion.md)) in action, and it's what makes the pattern work.

### 3. Full JavaScript implementation — from scratch

```javascript
// ============================================================
// observer.js — Observer pattern, hand-rolled
// Run: node observer.js
// ============================================================

/**
 * Observer (the "interface"). JS has no interfaces, so we use a base class
 * whose methods throw — this makes a missing implementation fail loudly at
 * call time instead of silently doing nothing.
 */
class Observer {
  update(_event) {
    throw new Error(`${this.constructor.name} must implement update(event)`);
  }
}

/**
 * Subject — owns state + the subscriber list.
 * It knows NOTHING about PriceDisplay, AlertService, etc.
 */
class Subject {
  // A Set, not an Array: subscribing the same observer twice is almost always
  // a bug (you'd get notified twice), and Set makes removal O(1).
  #observers = new Set();

  subscribe(observer) {
    if (typeof observer?.update !== 'function') {
      throw new TypeError('Observer must have an update(event) method');
    }
    this.#observers.add(observer);

    // Return an "unsubscribe handle". This is the single most important
    // ergonomic choice in the whole pattern: the caller who subscribed now
    // holds the exact token needed to clean up, and can't get it wrong.
    return () => this.unsubscribe(observer);
  }

  unsubscribe(observer) {
    this.#observers.delete(observer);
  }

  notify(event) {
    // Iterate a COPY. If an observer unsubscribes itself (or another observer)
    // inside update(), mutating the live Set mid-iteration would skip listeners.
    for (const observer of [...this.#observers]) {
      try {
        observer.update(event);
      } catch (err) {
        // One broken observer must not stop the others from being notified,
        // and must not blow up the subject. Isolate the failure.
        console.error(`[Subject] observer ${observer.constructor.name} threw:`, err.message);
      }
    }
  }

  get observerCount() {
    return this.#observers.size;
  }
}

// ---------- Concrete Subject ----------
class StockTicker extends Subject {
  #prices = new Map();

  setPrice(symbol, price) {
    const previous = this.#prices.get(symbol) ?? price;
    this.#prices.set(symbol, price);

    // The subject's ONLY job on change: announce it. It does not know or care
    // who is listening — zero observers is a perfectly valid state.
    this.notify({
      symbol,
      price,
      previous,
      changePct: previous === 0 ? 0 : ((price - previous) / previous) * 100,
      at: new Date().toISOString(),
    });
  }
}

// ---------- Concrete Observers ----------
class PriceDisplay extends Observer {
  update({ symbol, price, changePct }) {
    const arrow = changePct >= 0 ? '▲' : '▼';
    console.log(`[Display]   ${symbol} $${price.toFixed(2)} ${arrow} ${changePct.toFixed(2)}%`);
  }
}

class AlertService extends Observer {
  constructor(thresholds) {
    super();
    this.thresholds = thresholds; // e.g. { AAPL: 200 }
  }
  update({ symbol, price }) {
    const limit = this.thresholds[symbol];
    if (limit !== undefined && price >= limit) {
      console.log(`[Alert]     ${symbol} crossed $${limit} (now $${price.toFixed(2)})`);
    }
  }
}

class PortfolioTracker extends Observer {
  constructor(holdings) {
    super();
    this.holdings = holdings; // e.g. { AAPL: 10 }
  }
  update({ symbol, price }) {
    const shares = this.holdings[symbol];
    if (shares) {
      console.log(`[Portfolio] ${shares} × ${symbol} = $${(shares * price).toFixed(2)}`);
    }
  }
}

class FlakyAuditLogger extends Observer {
  update() {
    throw new Error('audit DB unreachable'); // proves one bad observer is contained
  }
}

// ---------- Demo ----------
function main() {
  const ticker = new StockTicker();

  const display = new PriceDisplay();
  const alerts = new AlertService({ AAPL: 200 });
  const portfolio = new PortfolioTracker({ AAPL: 10 });

  ticker.subscribe(display);
  ticker.subscribe(alerts);
  const unsubPortfolio = ticker.subscribe(portfolio); // keep the handle!
  ticker.subscribe(new FlakyAuditLogger());

  console.log(`--- ${ticker.observerCount} observers subscribed ---`);
  ticker.setPrice('AAPL', 195.5);

  console.log('\n--- price crosses the alert threshold ---');
  ticker.setPrice('AAPL', 203.25);

  console.log('\n--- user closes the portfolio widget: unsubscribe ---');
  unsubPortfolio();
  ticker.setPrice('AAPL', 210.0);
  console.log(`--- ${ticker.observerCount} observers remain ---`);
}

main();
```

Run it and you'll see: the flaky logger throws on every tick, is logged, and **the other three observers still get notified.** That error isolation is not optional polish — it's the difference between a resilient event system and one where a bug in analytics takes down checkout.

### 4. A real Node.js ecosystem example — `EventEmitter` *is* Observer

Node ships this pattern in core. `EventEmitter` is a Subject with **named channels** (event names), and observers are plain functions instead of objects.

```javascript
// ============================================================
// ticker-events.js — the SAME example, using Node's EventEmitter
// ============================================================
import { EventEmitter } from 'node:events';

class StockTicker extends EventEmitter {
  #prices = new Map();

  setPrice(symbol, price) {
    const previous = this.#prices.get(symbol) ?? price;
    this.#prices.set(symbol, price);

    // emit() === notify(). The event NAME is the channel.
    this.emit('price', { symbol, price, previous });

    // Named channels let observers subscribe to exactly what they care about.
    if (price >= 200) this.emit('threshold:crossed', { symbol, price });
  }
}

const ticker = new StockTicker();

// on() === subscribe(). The function itself is the observer.
const onPrice = ({ symbol, price }) => console.log(`[Display] ${symbol} $${price}`);
ticker.on('price', onPrice);

ticker.on('threshold:crossed', ({ symbol, price }) =>
  console.log(`[Alert] ${symbol} crossed 200 at $${price}`)
);

// once() = auto-unsubscribing observer. Fires exactly one time, then removes itself.
ticker.once('price', () => console.log('[Boot] first tick received, warm-up done'));

// EventEmitter treats 'error' specially: emit('error') with NO listener
// attached will THROW and crash the process. Always attach one.
ticker.on('error', (err) => console.error('[Ticker] recoverable error:', err.message));

ticker.setPrice('AAPL', 195.5);
ticker.setPrice('AAPL', 203.25);

// off() === unsubscribe(). You need the SAME function reference you passed to on().
ticker.off('price', onPrice);
ticker.setPrice('AAPL', 210); // [Display] no longer fires

console.log('listeners on "price":', ticker.listenerCount('price'));
```

Everything in Node that streams or signals is built on this: `stream.on('data')`, `server.on('request')`, `process.on('SIGTERM')`, `socket.on('message')`. Learn `EventEmitter` and you've learned Observer.

### 5. Memory leaks from forgotten unsubscribes — the #1 real bug

**This is the bug you will actually hit in production.** The Subject holds a *strong reference* to every observer. If the observer's owner is destroyed but never unsubscribes, the Subject keeps that reference alive forever — the observer can never be garbage collected, and *its whole closure scope goes with it*.

```javascript
// ---------- BAD: the leak ----------
import { EventEmitter } from 'node:events';
const ticker = new EventEmitter(); // long-lived: lives for the whole process

class DashboardWidget {
  constructor(userId) {
    this.userId = userId;
    this.cache = new Array(100_000).fill('…'); // ~1 MB of state per widget
    // Subscribes on construction, NEVER unsubscribes.
    ticker.on('price', (p) => this.render(p));
  }
  render(p) { /* ... */ }
}

// A widget is created per request. Each one is "closed" and forgotten.
for (let i = 0; i < 5000; i++) {
  new DashboardWidget(i); // ← the arrow function closes over `this` (the widget)
}
// ticker now holds 5000 listeners, each pinning a 1 MB widget alive.
// Heap grows forever. Every price tick runs 5000 render() calls on dead widgets.
console.log(ticker.listenerCount('price')); // 5000 — and climbing
```

Two symptoms, both classic:
1. **Heap grows monotonically** until the process OOMs (`JavaScript heap out of memory`).
2. Node prints: `MaxListenersExceededWarning: Possible EventEmitter memory leak detected. 11 price listeners added.` — Node's built-in early-warning system, tripped at 11 listeners by default. **Never "fix" this by calling `setMaxListeners(1000)`.** That's disabling the smoke alarm.

```javascript
// ---------- GOOD: own the subscription, always provide a teardown ----------
class DashboardWidget {
  #unsubscribe;

  constructor(userId, ticker) {
    this.userId = userId;
    const handler = (p) => this.render(p);
    ticker.on('price', handler);
    // Store the teardown at subscription time, right next to the setup.
    this.#unsubscribe = () => ticker.off('price', handler);
  }

  render(p) { /* ... */ }

  destroy() {
    this.#unsubscribe();   // symmetric: every subscribe has exactly one unsubscribe
    this.#unsubscribe = () => {}; // idempotent — destroy() twice must be safe
  }
}

const w = new DashboardWidget(1, ticker);
w.destroy(); // now the widget is collectable
```

**The rules that prevent this bug:**
- Every `subscribe`/`on` must have a paired `unsubscribe`/`off` in a teardown path (`destroy()`, React's `useEffect` cleanup return, a `finally`).
- Prefer `subscribe()` returning an unsubscribe function — it makes forgetting harder.
- Use `once()` for one-shot listeners; it removes itself.
- Modern Node supports `AbortSignal`: `emitter.on('price', fn, { signal })` — abort the controller and **all** listeners registered with that signal are removed at once.
- Treat `MaxListenersExceededWarning` as a bug report, not noise.

### 6. Synchronous notification, ordering, and error handling

Three properties of in-process Observer that interviewers dig into:

**a) `emit()` is SYNCHRONOUS and BLOCKING.** `emitter.emit()` does not queue anything. It calls every listener, one after another, **on the current call stack**, and only returns when the last one has finished. So:

```javascript
ticker.on('price', () => { for (let i = 0; i < 1e9; i++); }); // 2s of CPU
ticker.setPrice('AAPL', 200); // ← blocks HERE for 2 seconds
console.log('this line waits');
```

One slow observer stalls the emitter **and** the Node event loop — every other request on the server is now waiting. If listener work is heavy, hand it off:

```javascript
ticker.on('price', (p) => {
  setImmediate(() => heavyWork(p)); // yield back to the event loop first
});
```

Note also: an `async` listener returns a promise that `emit()` **ignores**. `emit()` does *not* await it, and a rejection inside it becomes an unhandled rejection. If you need "wait for all listeners," you must build that yourself (collect promises, `await Promise.allSettled`).

**b) Ordering is registration order — and that's a weak contract.** Listeners fire in the order they were added (`prependListener` jumps the queue). But **never build logic that depends on it.** "The audit listener must run before the email listener" is a design smell: if two observers are ordered, they aren't independent, and you should model that as one observer or an explicit pipeline ([47 — Chain of Responsibility](./47-pattern-chain-of-responsibility.md)).

**c) Errors do not propagate back to the emitter.** A throwing listener on a raw `EventEmitter` propagates up through `emit()` and can crash the caller — which is why our hand-rolled `Subject` wraps each `update()` in `try/catch`. Rule: **a subject should never fail because an observer failed.** Log it, count it, move on.

### 7. Observer vs. Pub/Sub with a message broker

These get used interchangeably in conversation, and the distinction is a **guaranteed interview follow-up**.

| | **Observer (in-process)** | **Pub/Sub (broker-based)** |
|---|---|---|
| Who knows whom | Subject holds **direct references** to observers | Publisher knows only a **channel/topic name**; never sees subscribers |
| Process boundary | Same process, same heap | **Cross-process, cross-machine** |
| Delivery | Synchronous function call | Asynchronous message over the network |
| Durability | None — if the process dies, everything is gone | Broker can **persist** and replay messages |
| If nobody is listening | Nothing happens, event is lost | Broker may queue it until a subscriber appears |
| Failure of a subscriber | Caller's problem (or swallowed) | Broker **retries**, dead-letter queue |
| Backpressure | None — the subject just blocks | Queue depth, consumer lag, throttling |
| Examples | `EventEmitter`, DOM events, RxJS | Redis Pub/Sub, Kafka, RabbitMQ, SNS/SQS |

The mental model: **Observer is a design pattern inside one process. Pub/Sub is an architecture across processes** — and it needs a middleman (the **broker**) that both sides talk to instead of each other.

```javascript
// Observer: subject → observer, direct
orderService.on('order:placed', (o) => emailService.send(o));

// Pub/Sub: publisher → BROKER → subscriber, in different processes
await redis.publish('orders', JSON.stringify(order));   // order-service (process A)
await redis.subscribe('orders', (msg) => send(JSON.parse(msg))); // email-service (process B)
```

Note that **Redis Pub/Sub is fire-and-forget**: if the email service is restarting, the message is simply gone. That's why real systems reach for Kafka or RabbitMQ, which persist. That whole world — delivery guarantees, at-least-once vs exactly-once, consumer groups, dead-letter queues — is [67 — Message Queues](./67-message-queues.md).

**The upgrade path is real and common:** you start with an in-process `EventEmitter`, you split the monolith into services, and the emitter becomes a queue. The *pattern* survives the migration; only the transport changes. That's the whole point.

---

## Visual / Diagram description

### Diagram 1: Class structure

```
                     ┌───────────────────────────────┐
                     │        Subject                │
                     │  (abstract / base class)      │
                     ├───────────────────────────────┤
                     │ - observers: Set<Observer>    │
                     ├───────────────────────────────┤
                     │ + subscribe(o) : unsubFn      │
                     │ + unsubscribe(o) : void       │
                     │ + notify(event) : void        │
                     └──────────────┬────────────────┘
                                    │ extends
                                    │
                     ┌──────────────▼────────────────┐          ┌──────────────────────┐
                     │       StockTicker             │  holds   │   Observer           │
                     │  (Concrete Subject)           │ 0..* ───▶│   (interface)        │
                     ├───────────────────────────────┤          ├──────────────────────┤
                     │ - prices: Map<string, number> │          │ + update(event): void│
                     ├───────────────────────────────┤          └──────────┬───────────┘
                     │ + setPrice(sym, price)        │                     │ implements
                     │     → calls this.notify(...)  │        ┌────────────┼────────────┐
                     └───────────────────────────────┘        │            │            │
                                                   ┌──────────▼───┐ ┌──────▼──────┐ ┌───▼────────────┐
                                                   │ PriceDisplay │ │AlertService │ │PortfolioTracker│
                                                   ├──────────────┤ ├─────────────┤ ├────────────────┤
                                                   │ + update()   │ │ + update()  │ │ + update()     │
                                                   └──────────────┘ └─────────────┘ └────────────────┘
```

Read the arrows: `StockTicker` points at the **`Observer` interface** — never at `PriceDisplay`. That single arrow is why you can add a tenth observer without touching the ticker. The concrete observers point *up* at the interface. Dependencies flow toward the abstraction.

### Diagram 2: The notification sequence (and the blocking problem)

```
Client        StockTicker         PriceDisplay    AlertService    PortfolioTracker
  │                │                    │               │                │
  │ subscribe(d)   │                    │               │                │
  ├───────────────▶│  observers = {d}   │               │                │
  │ subscribe(a)   │                    │               │                │
  ├───────────────▶│  observers = {d,a} │               │                │
  │ subscribe(p)   │                    │               │                │
  ├───────────────▶│  observers={d,a,p} │               │                │
  │                │                    │               │                │
  │ setPrice(203)  │                    │               │                │
  ├───────────────▶│                    │               │                │
  │                │── update(evt) ────▶│               │                │
  │                │◀── returns ────────┤   ▲           │                │
  │                │── update(evt) ─────────────────────▶│               │
  │                │◀── returns ────────────────────────┤ │ ALL of this  │
  │                │── update(evt) ─────────────────────────────────────▶│
  │                │◀── returns ────────────────────────────────────────┤ │ SYNCHRONOUS
  │◀── returns ────┤                    │               │                │ ▼ on ONE stack
  │                │                    │               │                │
  │  ← setPrice() only returns after the LAST observer finishes.         │
  │    A slow observer blocks the ticker AND the whole event loop.       │
```

### Diagram 3: Observer vs. broker Pub/Sub — the shape of the coupling

```
  OBSERVER (in-process)                 PUB/SUB (cross-process, via broker)

  ┌──────────┐                          ┌──────────────┐
  │ Subject  │                          │ order-service│  (process A)
  └────┬─────┘                          └──────┬───────┘
       │ holds refs                            │ publish("orders", msg)
       │ (knows exactly who)                   ▼
  ┌────┼────┬──────────┐            ┌────────────────────────┐
  ▼    ▼    ▼          ▼            │   BROKER  (Kafka /     │
┌───┐┌───┐┌───┐     ┌──────┐        │   RabbitMQ / Redis)    │
│Ob1││Ob2││Ob3│     │ Ob4  │        │  topic: "orders"       │
└───┘└───┘└───┘     └──────┘        │  [msg][msg][msg]  ←──── persisted, replayable
                                    └───┬──────────┬─────────┘
  Same heap. Sync calls.                │ subscribe│ subscribe
  Zero durability.                      ▼          ▼
  Subject → Observer coupling.   ┌────────────┐ ┌──────────────┐
                                 │email-svc   │ │analytics-svc │  (processes B, C)
                                 └────────────┘ └──────────────┘

  The publisher NEVER learns who subscribed. Total decoupling — at the cost
  of a network hop, a broker to operate, and delivery semantics to reason about.
```

---

## Real world examples

### 1. The DOM and the browser event system

`element.addEventListener('click', handler)` is textbook Observer: the DOM element is the Subject, the handler is the Observer, and clicking calls `notify`. The browser hands you `removeEventListener` for a reason — single-page apps that mount and unmount thousands of components leak listeners exactly as described above, which is why every SPA framework makes cleanup a first-class concern (React's `useEffect` return value, Vue's `onUnmounted`, Angular's `ngOnDestroy`). The modern `AbortController` API exists largely to make bulk-unsubscribe ergonomic.

### 2. RxJS and Angular — Observer, industrialized

RxJS is Observer with an entire algebra on top. An `Observable` is a Subject; `.subscribe()` returns a `Subscription` with an `.unsubscribe()` method; and operators (`map`, `filter`, `debounceTime`, `takeUntil`) transform the notification stream between subject and observer. Angular's HTTP client, router, and forms all expose observables. The community's standard leak-prevention idiom — piping through `takeUntil(this.destroy$)` — exists precisely because forgotten unsubscribes are the most common Angular memory bug.

### 3. Redis Pub/Sub — the same pattern, across the network

Redis exposes `PUBLISH channel message` and `SUBSCRIBE channel`. Publishers never learn who is subscribed; Redis is the broker in the middle. It's fire-and-forget — messages are **not** persisted, so a subscriber that's down misses them entirely. That trade-off makes it excellent for ephemeral fanout (live presence, cache-invalidation broadcasts, chat rooms in a small deployment) and unsuitable for anything that must not be lost, like order events — for which teams reach for Kafka or RabbitMQ instead.

### 4. React state libraries

Redux, Zustand, and Jotai are Subjects holding application state. Components subscribe on mount and unsubscribe on unmount; a `dispatch`/`set` mutates state and notifies every subscriber, which re-renders. `useSyncExternalStore` — the hook React added specifically for external stores — takes a `subscribe(callback) → unsubscribe` function as its first argument. That signature *is* the Observer contract, standardized into React itself.

---

## Trade-offs

| Pros | What it buys you |
|---|---|
| **Loose coupling** | Subject depends only on the `Observer` interface; observers can be added or removed without touching it |
| **Open/Closed** | New reaction to an event = new observer class, zero edits to the subject ([15](./15-solid-open-closed.md)) |
| **Runtime flexibility** | Subscribe/unsubscribe while the program is running |
| **Broadcast for free** | One event, N independent reactions, no `if` chains |
| **Testability** | Test the subject with zero observers; test an observer by calling `update()` directly |

| Cons | The cost you pay |
|---|---|
| **Memory leaks** | The subject strongly references observers. Forgotten unsubscribe = permanent leak. **The #1 real bug.** |
| **Hidden control flow** | Reading `setPrice()` tells you *nothing* about what actually happens. Debugging means grepping for every `on('price')`. |
| **Synchronous blocking** | One slow listener stalls the emitter and the event loop |
| **No delivery guarantee** | Zero listeners = event silently vanishes. No retry, no persistence. |
| **Ordering is implicit** | Registration order — a contract that's easy to break accidentally |
| **Cascade risk** | Observer A's `update` emits event B, whose observer emits event C… infinite loops and stack overflows are real |
| **Error handling is manual** | You must decide what a throwing observer means, or it will decide for you |

**Rule of thumb:** Use Observer when **one thing happens and N unrelated things must react, and the subject shouldn't care what they are.** If there's exactly one reaction, just call the method directly — a one-observer event system is indirection with no payoff. And the moment the reaction must survive a crash, cross a process boundary, or be retried, stop reaching for `EventEmitter` and reach for a queue ([67](./67-message-queues.md)).

---

## Common interview questions on this topic

### Q1: "What's the difference between the Observer pattern and Pub/Sub?"
**Hint:** In Observer, the **Subject holds direct references to its observers** — same process, synchronous call, no durability. In Pub/Sub, publisher and subscriber are decoupled by a **broker** and a **topic name**; neither knows the other exists, they can be on different machines, and the broker can persist, retry, and fan out. Observer is a design pattern; broker Pub/Sub is an architecture. Say the sentence: *"Observer = in-process, subject knows its observers. Pub/Sub = cross-process, mediated by a channel."*

### Q2: "How do you prevent memory leaks with Observer?"
**Hint:** The subject holds a strong reference, so an observer whose owner is destroyed but never unsubscribed can never be GC'd — and its whole closure scope stays alive with it. Fixes: (a) have `subscribe()` **return an unsubscribe function** so the teardown is impossible to lose; (b) pair every subscribe with an unsubscribe in a `destroy()` / `useEffect` cleanup / `finally`; (c) use `once()` for one-shot listeners; (d) use `AbortSignal` to bulk-remove; (e) treat Node's `MaxListenersExceededWarning` as a bug, never raise the limit to silence it.

### Q3: "Is `emit()` synchronous or asynchronous in Node?"
**Hint:** **Synchronous.** `emit()` invokes every listener in registration order on the current call stack and returns only when the last one finishes. It does **not** await async listeners — a promise returned by a listener is ignored, and a rejection becomes an unhandled rejection. So one slow listener blocks the emitter *and* the event loop. Offload heavy work with `setImmediate`, or collect promises and `Promise.allSettled` them yourself if you truly need to wait.

### Q4: "What happens if one observer throws?"
**Hint:** On a raw `EventEmitter`, the exception propagates out of `emit()` — the remaining listeners never run and the caller may crash. Correct design: **a subject must never fail because an observer failed.** Wrap each `update()` in `try/catch`, log it, and continue. Also note the special case: `emit('error')` with no `'error'` listener attached will throw and crash the Node process by design.

### Q5: "Can observers depend on being notified in a specific order?"
**Hint:** They *can* (registration order), but they **shouldn't**. Order-dependence means the observers aren't independent, which defeats the pattern. If step B genuinely must follow step A, that's a **pipeline**, not a broadcast — model it as one observer that runs both steps, or use Chain of Responsibility ([47](./47-pattern-chain-of-responsibility.md)).

### Q6: "How does Observer differ from Mediator?"
**Hint:** Observer is **one-to-many broadcast** in one direction: the subject announces, observers listen. Mediator ([48](./48-pattern-mediator.md)) is **many-to-many coordination**: a central object knows all the colleagues and orchestrates *bidirectional* interaction between them (a chat room, air traffic control). Mediators are often *implemented using* Observer internally — but a mediator contains logic about who talks to whom; a subject contains none.

---

## Practice exercise

### Build an Auction House event system

Build `auction.js` — a live-auction system, from scratch, then a second time with `EventEmitter`.

**Part A — hand-rolled Observer (~15 min)**
1. Write a generic `Subject` class with `subscribe(observer)` (returning an **unsubscribe function**), `unsubscribe(observer)`, and `notify(event)`. `notify` must **iterate a copy** of the observer list and **isolate errors** with `try/catch`.
2. Write `AuctionItem extends Subject` with `placeBid(bidder, amount)`. It must reject bids below the current highest, and on success `notify({ type: 'bid', item, bidder, amount, previousHigh })`. Add `close()` which notifies `{ type: 'sold', winner, amount }`.
3. Write four observers: `BidderNotifier` (tells the previous high bidder they were outbid), `PriceBoard` (prints the current price), `FraudMonitor` (flags a bidder who raises their own bid twice in a row), and `BrokenAnalytics` (always throws — prove the others still fire).

**Part B — the leak (~10 min)**
4. Create 1,000 `BidderNotifier` observers in a loop and subscribe them all, without unsubscribing. Print `observerCount`. Then run `node --expose-gc` and log `process.memoryUsage().heapUsed` before and after `global.gc()`. Now fix it by calling the returned unsubscribe handles, and show the heap drops.

**Part C — the same thing with `EventEmitter` (~10 min)**
5. Rewrite `AuctionItem` to `extends EventEmitter`, emitting named channels `'bid'` and `'sold'`. Use `once('sold', …)` for a listener that must fire exactly once. Attach one listener with an `AbortSignal` and demonstrate that aborting removes it.

**Deliverable:** one file that runs with `node auction.js` and prints a full auction: three bids, an outbid notice, a fraud flag, a contained analytics error, and a sale — plus a printed listener count that goes back to zero after teardown.

---

## Quick reference cheat sheet

- **Observer** = one-to-many. Subject changes → all registered observers are notified automatically.
- **Four participants:** Subject (holds the list), Observer (the interface), ConcreteObserver (the reaction), Client (wires them up).
- **Three methods:** `subscribe(o)`, `unsubscribe(o)`, `notify(event)`. In Node: `on`, `off`, `emit`.
- **The subject knows the interface, never the concrete observer** — that's what makes it Open/Closed.
- **Always return an unsubscribe function from `subscribe()`.** It makes cleanup impossible to lose.
- **The #1 bug is the memory leak:** the subject strongly references observers; a forgotten `off()` pins the observer *and its whole closure* in memory forever.
- **`MaxListenersExceededWarning` is a leak alarm.** Fix the leak; never raise the limit.
- **`emit()` is synchronous and blocking.** It runs listeners on the current stack and ignores promises returned by async listeners.
- **Isolate observer errors** with `try/catch` — a subject must never fail because an observer failed.
- **`emit('error')` with no `'error'` listener crashes the Node process.** Always attach one.
- **Ordering is registration order — don't depend on it.** Order-dependent observers should be a pipeline instead.
- **Observer ≠ broker Pub/Sub.** Observer: in-process, direct references, sync, no durability. Pub/Sub: cross-process, a channel in the middle, async, persistable, retryable.
- **`once()` and `AbortSignal`** are the built-in leak-proof subscription tools in Node.
- **In the wild:** DOM `addEventListener`, Node `EventEmitter`, RxJS `Observable`, Redux/Zustand `subscribe`, React `useSyncExternalStore`, Redis Pub/Sub.
- **Don't use it** when there's exactly one reaction (just call the method), or when the event must survive a crash (use a queue).

---

## Connected topics

| Direction | Topic |
|-----------|-------|
| **Previous** | [40 — Flyweight Pattern](./40-pattern-flyweight.md) — the last structural pattern; Observer begins the behavioral group |
| **Next** | [42 — Strategy Pattern](./42-pattern-strategy.md) — swapping algorithms at runtime, the other must-know behavioral pattern |
| **Related** | [67 — Message Queues](./67-message-queues.md) — what Observer becomes once the observers live in another process |
| **Related** | [68 — Event-Driven Architecture](./68-event-driven-architecture.md) — Observer scaled up to be the organizing principle of a whole system |
| **Related** | [48 — Mediator Pattern](./48-pattern-mediator.md) — many-to-many coordination, the pattern most often confused with Observer |
