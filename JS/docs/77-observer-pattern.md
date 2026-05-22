# 77 — Observer Pattern

---

## 1. What is this?

The **observer pattern** (also called pub/sub — publish/subscribe) defines a one-to-many relationship between objects: when one object (the **subject** or **publisher**) changes state, all objects that depend on it (the **observers** or **subscribers**) are automatically notified and updated.

The subject doesn't know or care who its observers are. Observers don't know about each other. The only shared contract is the event name and the shape of the data.

**Analogy — a newsletter:**
A newspaper publishes one edition. Thousands of subscribers each receive it. The newspaper doesn't know who the subscribers are, doesn't call each one directly, and doesn't care what they do with the paper. Subscribers sign up and cancel independently. This is pub/sub.

**Analogy — you already know this:**
- `element.addEventListener("click", handler)` — browser events **are** the observer pattern
- `emitter.on("data", handler)` — Node.js `EventEmitter` **is** the observer pattern
- `store.subscribe(fn)` — Redux, Zustand, signals — **all** observer pattern
- WebSockets, MQTT, Server-Sent Events — **all** observer pattern at the protocol level

---

## 2. Why does it matter?

- **Decoupling** — the publisher and subscribers don't need to import or know about each other. You can add new subscribers without touching the publisher.
- **One-to-many** — a single event can trigger dozens of independent reactions
- **Inversion of control** — the publisher doesn't orchestrate what happens. Subscribers decide their own reactions.
- **Foundation for everything reactive** — React state, RxJS observables, event-driven Node.js architecture, message queues (Kafka, RabbitMQ) — all variations of this pattern
- **Easy feature addition** — to add logging, analytics, or email notifications when a user registers, you add a subscriber. You change zero existing code.

---

## 3. The Core Pattern — from scratch

```js
class EventEmitter {
  #listeners = new Map(); // event name → Set of handlers

  on(event, handler) {
    if (!this.#listeners.has(event)) {
      this.#listeners.set(event, new Set());
    }
    this.#listeners.get(event).add(handler);
    return this; // enables chaining: emitter.on("a", h).on("b", h)
  }

  off(event, handler) {
    this.#listeners.get(event)?.delete(handler);
    return this;
  }

  once(event, handler) {
    const wrapper = (...args) => {
      handler(...args);
      this.off(event, wrapper);
    };
    wrapper._original = handler; // store ref so off() can find it
    return this.on(event, wrapper);
  }

  emit(event, ...args) {
    this.#listeners.get(event)?.forEach(fn => fn(...args));
    return this;
  }

  removeAllListeners(event) {
    if (event) this.#listeners.delete(event);
    else       this.#listeners.clear();
    return this;
  }

  listenerCount(event) {
    return this.#listeners.get(event)?.size ?? 0;
  }

  eventNames() {
    return [...this.#listeners.keys()];
  }
}
```

Usage:

```js
const emitter = new EventEmitter();

// Subscribe
emitter.on("data", (payload) => {
  console.log("handler 1:", payload);
});

emitter.on("data", (payload) => {
  console.log("handler 2:", payload);
});

// Publish
emitter.emit("data", { id: 1, value: "hello" });
// handler 1: { id: 1, value: 'hello' }
// handler 2: { id: 1, value: 'hello' }

// Unsubscribe
const logHandler = (d) => console.log("log:", d);
emitter.on("data", logHandler);
emitter.off("data", logHandler);

// Once
emitter.once("connect", () => console.log("connected!"));
emitter.emit("connect"); // "connected!"
emitter.emit("connect"); // nothing — auto-removed
```

---

## 4. Variants of the Pattern

### Variant 1 — Simple function-based event bus

For cases where you just need a shared bus, not subclassing:

```js
function createEventBus() {
  const listeners = new Map();

  return {
    on(event, handler) {
      (listeners.get(event) ?? listeners.set(event, new Set()).get(event)).add(handler);
      return () => this.off(event, handler); // returns unsubscribe fn
    },
    off(event, handler) {
      listeners.get(event)?.delete(handler);
    },
    emit(event, data) {
      listeners.get(event)?.forEach(fn => {
        try { fn(data); } catch (e) { console.error(`Error in "${event}" handler:`, e); }
      });
    },
    once(event, handler) {
      const unsub = this.on(event, function wrapper(data) {
        handler(data);
        unsub();
      });
    }
  };
}

export const bus = createEventBus(); // shared singleton
```

Note the `try/catch` inside `emit` — one broken handler shouldn't stop others from running.

### Variant 2 — Observable value (reactive state)

A value that notifies observers whenever it changes:

```js
function createObservable(initialValue) {
  let value     = initialValue;
  const handlers = new Set();

  return {
    get() {
      return value;
    },
    set(newValue) {
      if (newValue === value) return; // no-op if unchanged
      const old = value;
      value = newValue;
      handlers.forEach(fn => fn(newValue, old));
    },
    subscribe(fn) {
      handlers.add(fn);
      return () => handlers.delete(fn); // unsubscribe
    },
    get value() { return value; }, // property access syntax
    set value(v) { this.set(v); }
  };
}

const count = createObservable(0);

const unsub = count.subscribe((next, prev) => {
  console.log(`count changed: ${prev} → ${next}`);
});

count.set(1);  // count changed: 0 → 1
count.set(1);  // no-op — value didn't change
count.set(5);  // count changed: 1 → 5

unsub();       // unsubscribe
count.set(10); // no output
```

This is the foundation of signals (Solid.js, Angular signals, Vue's `ref()`).

### Variant 3 — Subject with typed events (TypeScript-style in JSDoc)

```js
/**
 * @template T
 * @param {T} initialValue
 */
function createSubject(initialValue) {
  let current = initialValue;
  const subscribers = new Set();

  return {
    /** @returns {T} */
    getValue() { return current; },

    /** @param {T} value */
    next(value) {
      current = value;
      subscribers.forEach(fn => fn(value));
    },

    /** @param {(value: T) => void} fn */
    subscribe(fn) {
      subscribers.add(fn);
      fn(current); // emit current value immediately on subscribe (like BehaviorSubject in RxJS)
      return () => subscribers.delete(fn);
    }
  };
}
```

### Variant 4 — DOM custom events as observer (browser-native)

```js
// The browser's own event system IS the observer pattern
// Use it for cross-component communication without a library

// Publisher (anywhere in the codebase)
function userLoggedIn(user) {
  document.dispatchEvent(new CustomEvent("app:userLogin", {
    bubbles: false,
    detail: { userId: user.id, username: user.name }
  }));
}

// Subscriber A — header component
document.addEventListener("app:userLogin", ({ detail }) => {
  headerEl.querySelector(".username").textContent = `Hi, ${detail.username}`;
});

// Subscriber B — analytics module
document.addEventListener("app:userLogin", ({ detail }) => {
  analytics.track("login", { userId: detail.userId });
});

// Subscriber C — notification module
document.addEventListener("app:userLogin", ({ detail }) => {
  toast.show(`Welcome back, ${detail.username}`);
});

// Publisher calls userLoggedIn — all three react. None know about each other.
```

---

## 5. Real-World Examples

### Example 1 — Domain event system for a backend service

```js
// events/domainEvents.js
const { createEventBus } = require("./eventBus");

const events = createEventBus();

module.exports = { events };

// ─── Event publishers (domain layer) ────────────────────────────────────────

// userService.js
async function createUser(data) {
  const user = await db.users.create(data);
  events.emit("user:created", { userId: user.id, email: user.email });
  return user;
}

async function deleteUser(userId) {
  await db.users.delete(userId);
  events.emit("user:deleted", { userId });
}

// ─── Event subscribers (side effects layer) ─────────────────────────────────

// emailService.js — subscribe without touching userService.js
events.on("user:created", async ({ email }) => {
  await sendWelcomeEmail(email);
});

// auditLog.js
events.on("user:created", ({ userId }) => {
  auditLog.write({ action: "user_created", userId, timestamp: new Date() });
});

events.on("user:deleted", ({ userId }) => {
  auditLog.write({ action: "user_deleted", userId, timestamp: new Date() });
});

// cacheInvalidation.js
events.on("user:deleted", ({ userId }) => {
  cache.delete(`user:${userId}`);
});
```

Adding email confirmation, Slack notifications, or data warehouse sync? Just add more subscribers. `userService.js` never changes.

### Example 2 — Store with reactive subscriptions (mini Redux)

```js
function createStore(reducer, initialState) {
  let state       = initialState;
  const listeners = new Set();
  const history   = [];

  function dispatch(action) {
    const prev = state;
    state = reducer(state, action);

    if (state !== prev) { // only notify if state actually changed
      history.push({ action, prev, next: state, at: Date.now() });
      if (history.length > 50) history.shift();
      listeners.forEach(fn => fn(state, action));
    }

    return action;
  }

  return {
    getState:   () => Object.freeze({ ...state }), // always a frozen copy
    dispatch,
    subscribe(fn) {
      listeners.add(fn);
      return () => listeners.delete(fn); // return unsubscribe
    },
    getHistory: () => [...history]
  };
}

// Usage
const store = createStore(cartReducer, { items: [], coupon: null });

// Multiple independent subscribers
const unsubUI = store.subscribe((state) => {
  renderCartUI(state.items);
  updateCartBadge(state.items.length);
});

const unsubAnalytics = store.subscribe((state, action) => {
  if (action.type === "ADD_ITEM") {
    analytics.track("cart_item_added", { item: action.item });
  }
});

const unsubPersist = store.subscribe((state) => {
  localStorage.setItem("cart", JSON.stringify(state));
});

store.dispatch({ type: "ADD_ITEM", item: { id: 1, name: "Widget", price: 9.99 } });
// All three subscribers fire

unsubAnalytics(); // analytics no longer tracks — others still work
```

### Example 3 — WebSocket with observer pattern

```js
function createWebSocketClient(url) {
  const emitter  = new EventEmitter();
  let   socket   = null;
  let   reconnectTimer = null;

  function connect() {
    socket = new WebSocket(url);

    socket.addEventListener("open", () => {
      emitter.emit("connect");
    });

    socket.addEventListener("message", ({ data }) => {
      try {
        const msg = JSON.parse(data);
        emitter.emit("message", msg);
        emitter.emit(`message:${msg.type}`, msg.payload); // typed events
      } catch {
        emitter.emit("error", new SyntaxError("Invalid JSON from server"));
      }
    });

    socket.addEventListener("close", ({ code, reason }) => {
      emitter.emit("disconnect", { code, reason });
      // Auto-reconnect
      reconnectTimer = setTimeout(connect, 3000);
    });

    socket.addEventListener("error", (e) => {
      emitter.emit("error", e);
    });
  }

  return {
    connect,
    disconnect() {
      clearTimeout(reconnectTimer);
      socket?.close();
    },
    send(type, payload) {
      if (socket?.readyState === WebSocket.OPEN) {
        socket.send(JSON.stringify({ type, payload }));
      }
    },
    on:   (event, fn) => emitter.on(event, fn),
    off:  (event, fn) => emitter.off(event, fn),
    once: (event, fn) => emitter.once(event, fn)
  };
}

// Usage
const ws = createWebSocketClient("wss://api.example.com/live");

ws.on("connect",          ()      => console.log("Connected"));
ws.on("disconnect",       (info)  => console.log("Disconnected:", info.code));
ws.on("message:chat",     (msg)   => appendChatMessage(msg));
ws.on("message:presence", (users) => updateOnlineList(users));
ws.on("error",            (e)     => showConnectionError(e));

ws.connect();
```

### Example 4 — Lifecycle hooks system

```js
function createHookSystem() {
  const hooks = new Map(); // hookName → [{ fn, priority }]

  function register(name, fn, priority = 10) {
    if (!hooks.has(name)) hooks.set(name, []);
    hooks.get(name).push({ fn, priority });
    hooks.get(name).sort((a, b) => a.priority - b.priority);
    return () => {
      const list = hooks.get(name);
      const idx  = list.findIndex(h => h.fn === fn);
      if (idx !== -1) list.splice(idx, 1);
    };
  }

  async function run(name, context) {
    const list = hooks.get(name) ?? [];
    for (const { fn } of list) {
      // Each hook can modify context or throw to abort
      await fn(context);
    }
    return context;
  }

  return { register, run };
}

const hooks = createHookSystem();

// Register hooks for "request" lifecycle
hooks.register("request:before", async (ctx) => {
  ctx.startTime = Date.now();
  console.log(`→ ${ctx.method} ${ctx.path}`);
}, 1); // priority 1 — runs first

hooks.register("request:before", async (ctx) => {
  ctx.user = await authenticate(ctx.headers.authorization);
}, 5); // priority 5 — runs second

hooks.register("request:before", async (ctx) => {
  await rateLimit(ctx.user?.id ?? ctx.ip);
}, 10); // priority 10 — runs third

hooks.register("request:after", async (ctx) => {
  const elapsed = Date.now() - ctx.startTime;
  console.log(`← ${ctx.statusCode} (${elapsed}ms)`);
});

// In request handler
async function handleRequest(req, res) {
  const ctx = { method: req.method, path: req.path, headers: req.headers };
  await hooks.run("request:before", ctx);
  // ... process request
  ctx.statusCode = 200;
  await hooks.run("request:after", ctx);
}
```

### Example 5 — File watcher with observer pattern

```js
// Node.js
import { watch } from "fs";
import { createEventBus } from "./eventBus.js";

function createFileWatcher(paths) {
  const bus      = createEventBus();
  const watchers = [];

  for (const filePath of paths) {
    const watcher = watch(filePath, { persistent: false }, (eventType, filename) => {
      bus.emit("change", { type: eventType, file: filename, path: filePath });
      bus.emit(`change:${eventType}`, { file: filename, path: filePath });
    });
    watchers.push(watcher);
  }

  return {
    on:      (event, fn) => bus.on(event, fn),
    off:     (event, fn) => bus.off(event, fn),
    close()  { watchers.forEach(w => w.close()); }
  };
}

const watcher = createFileWatcher(["./config.json", "./.env"]);

watcher.on("change:rename", ({ file }) => console.log(`File renamed: ${file}`));
watcher.on("change:change", ({ path }) => {
  console.log(`File changed: ${path}`);
  reloadConfig(path);
});
```

---

## 6. Observer vs Other Patterns

| Pattern | Who initiates? | How coupled? | When to use |
|---|---|---|---|
| **Observer** | Subject pushes to subscribers | Loose — via event name | One-to-many state change notifications |
| **Callback** | Caller passes function | Tight — direct reference | One-time async result |
| **Promise** | Resolved once | Loose | Single async value |
| **Iterator** | Consumer pulls values | Loose | Sequential data |
| **Mediator** | Central coordinator routes | Looser than direct | Complex multi-party coordination |

Observer vs Mediator:
- **Observer** — publisher broadcasts, subscribers self-select
- **Mediator** — central hub receives all messages and decides who gets what

---

## 7. Tricky Things

### 1. Synchronous emit — exceptions in one handler stop others

```js
emitter.on("data", () => { throw new Error("handler 1 broke"); });
emitter.on("data", () => { console.log("handler 2"); }); // never runs!

// ✅ Wrap handlers in try/catch inside emit
emit(event, ...args) {
  this.#listeners.get(event)?.forEach(fn => {
    try { fn(...args); }
    catch (e) { console.error(`Error in "${event}" handler:`, e); }
  });
}
```

### 2. Memory leaks from forgotten subscriptions

```js
class Component {
  mount() {
    store.subscribe(this.render.bind(this)); // ← no reference kept
  }
  unmount() {
    // can't unsubscribe — reference lost
    // store holds reference to this component — never garbage collected
  }
}

// ✅ Store the unsubscribe function
class Component {
  #unsub = null;

  mount() {
    this.#unsub = store.subscribe(this.render.bind(this));
  }

  unmount() {
    this.#unsub?.(); // clean up
  }
}
```

### 3. Emitting inside a handler can cause infinite loops

```js
emitter.on("count:change", (n) => {
  emitter.emit("count:change", n + 1); // ← infinite loop!
});
```

### 4. Async handlers are fire-and-forget unless you await them

```js
emitter.on("user:created", async ({ userId }) => {
  await sendWelcomeEmail(userId); // promise ignored by emit!
});

emitter.emit("user:created", { userId: 1 });
// The async handler starts but emit returns immediately
// Any errors inside the async handler become unhandled rejections

// ✅ For async side effects, explicitly handle errors inside the handler
emitter.on("user:created", async ({ userId }) => {
  try {
    await sendWelcomeEmail(userId);
  } catch (e) {
    logger.error("Failed to send welcome email", { cause: e, userId });
  }
});
```

### 5. `once` removes the wrapper, not the original — `off` with the original handler won't work without the `_original` trick

```js
// Without storing the original ref:
const handler = () => console.log("ran");
emitter.once("event", handler);
emitter.off("event", handler); // ← may not work — wrapper function is different

// ✅ Store the original ref on the wrapper (as done in the class at the top)
wrapper._original = handler;
// Then in off():
off(event, handler) {
  const set = this.#listeners.get(event);
  for (const fn of set ?? []) {
    if (fn === handler || fn._original === handler) {
      set.delete(fn);
      break;
    }
  }
}
```

---

## 8. Common Mistakes

### Mistake 1 — No error isolation in emit

```js
// ❌ One failing handler breaks all subsequent handlers
emit(event, data) {
  this.listeners.get(event)?.forEach(fn => fn(data));
}

// ✅ Isolate each handler
emit(event, data) {
  this.listeners.get(event)?.forEach(fn => {
    try { fn(data); } catch (e) { console.error(e); }
  });
}
```

### Mistake 2 — Forgetting to unsubscribe

```js
// ❌ Common in SPAs — component re-mounts add more listeners each time
function mountDashboard() {
  store.subscribe(updateDashboard);
  eventBus.on("notification", showBanner);
}

// ✅ Always clean up
function mountDashboard() {
  const cleanups = [
    store.subscribe(updateDashboard),
    eventBus.on("notification", showBanner)
  ];
  return () => cleanups.forEach(fn => fn()); // return cleanup
}
```

### Mistake 3 — Event name typos cause silent failures

```js
// ❌ Nothing happens — "user:loign" never fires (typo)
events.on("user:loign", handler);
events.emit("user:login", data); // "user:login" — different name

// ✅ Use constants for event names
const EVENTS = Object.freeze({
  USER_LOGIN:   "user:login",
  USER_LOGOUT:  "user:logout",
  ORDER_PLACED: "order:placed"
});

events.on(EVENTS.USER_LOGIN, handler);
events.emit(EVENTS.USER_LOGIN, data);
```

### Mistake 4 — Mutating the event data inside a handler

```js
events.on("cart:updated", (cart) => {
  cart.items.push({ id: 99 }); // ❌ mutates the data — other handlers see the modified version
});

// ✅ Treat event data as read-only, or freeze it in emit
emit(event, data) {
  const frozen = Object.freeze({ ...data });
  this.#listeners.get(event)?.forEach(fn => fn(frozen));
}
```

### Mistake 5 — Tight coupling through event payloads

```js
// ❌ Subscribers depend on the exact internal shape of the publisher's state
events.on("state:changed", (state) => {
  updateUI(state.users[0].profile.avatar.url); // deeply coupled to state shape
});

// ✅ Publish only what subscribers need
events.emit("user:avatarUpdated", { userId, avatarUrl }); // specific, stable payload
```

---

## 9. Practice Exercises

### Easy — typed event emitter

Build an `EventEmitter` class with `on`, `off`, `emit`, `once`, and `listenerCount`. Test with at least three different event types, multiple handlers per event, and verify that `once` handlers fire exactly once.

---

### Medium — reactive form fields

Build `createField(initialValue)` using the observable pattern:
- `getValue()` — returns current value
- `setValue(v)` — updates value, runs validators, notifies subscribers
- `subscribe(fn)` — calls `fn({ value, valid, errors })` on every change
- `addValidator(fn)` — adds a validator function `(value) => string | null` (null = valid)
- `reset()` — restores initial value, clears errors

Create three fields (name, email, password) and a form-level subscriber that logs `"form valid"` when all three are valid.

---

### Hard — event-sourced order system

Build an `OrderService` using event sourcing:

- All state changes are recorded as immutable events: `{ type, payload, timestamp }`
- State is derived by replaying events (a reducer)
- `placeOrder(items)` — emits `"order:placed"`
- `confirmOrder(orderId)` — emits `"order:confirmed"`
- `cancelOrder(orderId, reason)` — emits `"order:cancelled"`
- `getOrder(orderId)` — replays events to reconstruct current state
- `getHistory(orderId)` — returns all events for an order

Attach three observers:
1. Email sender (logs to console instead of real email)
2. Inventory updater (decrements a mock inventory map)
3. Analytics tracker (counts events by type)

Demonstrate adding a fourth observer (audit log) without changing any existing code.

---

## 10. Quick Reference

```
CORE API
  emitter.on(event, handler)        subscribe
  emitter.off(event, handler)       unsubscribe
  emitter.once(event, handler)      subscribe — auto-removes after first fire
  emitter.emit(event, ...args)      publish — calls all handlers synchronously
  emitter.removeAllListeners(event) remove all handlers for an event
  emitter.listenerCount(event)      number of active handlers

SUBSCRIBE WITH CLEANUP
  const unsub = bus.on("event", handler);
  unsub(); // removes listener

EVENT NAME CONSTANTS
  const EVENTS = Object.freeze({ USER_LOGIN: "user:login" });
  // prevents typos, enables IDE autocomplete

EMIT WITH ERROR ISOLATION
  listeners.forEach(fn => {
    try { fn(data); } catch (e) { console.error(e); }
  });

OBSERVABLE VALUE
  const obs = createObservable(value);
  obs.set(newValue);              // notifies if changed
  obs.subscribe((next, prev) => {}); // returns unsubscribe fn

ASYNC HANDLERS
  Emit is synchronous — async handlers are fire-and-forget
  Always wrap async handlers in try/catch — unhandled rejections otherwise

MEMORY LEAK PREVENTION
  Always store unsubscribe function
  Call unsubscribe on component unmount / module teardown
  Use AbortController.signal for DOM addEventListener cleanup

OBSERVER vs CALLBACK vs PROMISE
  Observer   — one-to-many, ongoing, multiple events over time
  Callback   — one-to-one, single use
  Promise    — one-to-one, resolves once
```

---

## 11. Connected Topics

- **22 — Closures** — private listener state in the EventEmitter is held by closure
- **52 — Async JS** — async handlers, unhandled rejections from async observers
- **59 — ES Modules** — single shared bus instance via module singleton guarantee
- **68 — Events** — browser DOM events are the observer pattern built into the platform
- **69 — Event delegation** — one listener pattern on a parent
- **76 — Module pattern** — `createEventBus` from topic 76 is observer pattern
- **75 — Immutability** — freeze event payloads before emitting to prevent mutation
- **78 — Clean code** — event naming conventions (`namespace:action`), single responsibility per handler
- **Node.js EventEmitter** — `require("events")` — same pattern, built into Node.js core
- **RxJS** — full reactive extension of this pattern with operators (`map`, `filter`, `debounce`, `switchMap`)
