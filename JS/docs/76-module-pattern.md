# 76 — Module Pattern

---

## 1. What is this?

The **module pattern** is a design pattern that bundles related state and behaviour into a single unit with a **controlled public interface** — exposing only what callers need while keeping everything else private.

Before ES Modules (topic 59) existed, JavaScript had no built-in concept of private variables. The module pattern uses **closures** to simulate privacy: variables declared inside a function are invisible from the outside, but functions returned from that same function can still access them.

**Analogy — a vending machine:**
A vending machine has internal mechanics (motor, coin counter, inventory database) that you never touch. It exposes exactly one public interface: buttons and a slot. You interact only through the interface. The module pattern is the same — internal state is hidden, a clean API is exposed.

**Analogy — an Express router:**
`express.Router()` gives you a router object with `.get()`, `.post()`, `.use()` — the public API. The internal routing table, middleware stack, and path matching logic are all private inside the router. You never access them directly.

---

## 2. Why does it matter?

- **Encapsulation** — internal implementation details are hidden. Callers can't accidentally break invariants by directly mutating internal state.
- **Intentional API** — you decide exactly what's public. Everything else is an implementation detail that can change without breaking callers.
- **Namespace pollution prevention** — before ES Modules, all JavaScript ran in a shared global scope. The module pattern was the only way to avoid name collisions.
- **Foundation for understanding** — every major pattern (classes, ES Modules, CommonJS, React components, Express routers) is built on this same idea. Understanding it deeply makes everything else click.

---

## 3. The Three Forms

The module pattern has evolved over time. All three forms are worth knowing.

### Form 1 — IIFE Module (classic, pre-ES6)

```js
const Counter = (() => {
  // Private — inaccessible from outside
  let count = 0;
  const STEP = 1;

  function validate(n) {
    if (typeof n !== "number") throw new TypeError("Expected a number");
  }

  // Public — returned object is the API
  return {
    increment() { count += STEP; },
    decrement() { count -= STEP; },
    incrementBy(n) { validate(n); count += n; },
    reset()     { count = 0; },
    getCount()  { return count; }
  };
})();

Counter.increment();
Counter.increment();
Counter.getCount(); // 2

// Private internals are inaccessible
Counter.count;    // undefined
Counter.STEP;     // undefined
Counter.validate; // undefined
```

The `(() => { ... })()` — Immediately Invoked Function Expression (IIFE) — runs once and returns the public object. The closure keeps `count`, `STEP`, and `validate` alive but private.

### Form 2 — Factory Function (preferred, reusable)

An IIFE creates one singleton. A factory function creates as many independent instances as you need:

```js
function createCounter(initialValue = 0, step = 1) {
  // Private state — each call to createCounter gets its own copy
  let count = initialValue;

  function validate(n) {
    if (typeof n !== "number") throw new TypeError("Expected a number");
    if (!Number.isFinite(n)) throw new RangeError("Expected a finite number");
  }

  // Public API
  return {
    increment()   { count += step; },
    decrement()   { count -= step; },
    incrementBy(n) { validate(n); count += n; },
    reset()       { count = initialValue; }, // resets to constructor value
    getCount()    { return count; },
    toString()    { return `Counter(${count})`; }
  };
}

const counterA = createCounter(0, 1);
const counterB = createCounter(100, 5);

counterA.increment(); // counterA.count = 1
counterB.increment(); // counterB.count = 105

// Fully independent — different closures, different state
counterA.getCount(); // 1
counterB.getCount(); // 105
```

### Form 3 — ES Module Singleton (modern)

With ES Modules (topic 59), the module itself is the encapsulation boundary. Top-level variables in a module are private by default. Only what you `export` is accessible.

```js
// counter.js
let count = 0;    // private — not exported
const STEP = 1;   // private

function validate(n) {  // private
  if (typeof n !== "number") throw new TypeError("Expected a number");
}

// Public — only exported functions
export function increment()    { count += STEP; }
export function decrement()    { count -= STEP; }
export function incrementBy(n) { validate(n); count += n; }
export function reset()        { count = 0; }
export function getCount()     { return count; }
```

```js
// main.js
import { increment, getCount } from "./counter.js";

increment();
increment();
getCount(); // 2

// count, STEP, validate are completely inaccessible from main.js
```

The module is a singleton — the same `count` is shared across all importers.

---

## 4. Revealing Module Pattern

A variation that defines everything privately first, then explicitly reveals the public surface:

```js
function createUserService(db) {
  // All functions defined privately first
  async function findById(id) {
    return db.query("SELECT * FROM users WHERE id = $1", [id]);
  }

  async function create(data) {
    validateUserData(data);
    const hash = await hashPassword(data.password);
    return db.query(
      "INSERT INTO users(name, email, password_hash) VALUES($1,$2,$3) RETURNING *",
      [data.name, data.email, hash]
    );
  }

  async function update(id, changes) {
    const user = await findById(id); // private functions call each other
    if (!user) throw new NotFoundError("User");
    return db.query(/* ... */);
  }

  async function remove(id) {
    return db.query("DELETE FROM users WHERE id = $1", [id]);
  }

  function validateUserData(data) {
    if (!data.name)  throw new ValidationError({ name: ["Required"] });
    if (!data.email) throw new ValidationError({ email: ["Required"] });
  }

  async function hashPassword(password) {
    return bcrypt.hash(password, 12);
  }

  // Reveal only what callers need
  return {
    findById,
    create,
    update,
    delete: remove  // rename at the reveal point
  };
  // validateUserData and hashPassword are NOT revealed — internal only
}
```

The revealing pattern is cleaner than inline method definitions when functions are complex — you write them all as proper named functions, then choose what to expose at the bottom.

---

## 5. Module Pattern with Private Mutable State

The key strength: callers can only change state through the functions you provide — they cannot directly mutate the internals.

```js
function createEventBus() {
  const listeners = new Map(); // private — callers can't add arbitrary listeners

  function on(event, handler) {
    if (!listeners.has(event)) listeners.set(event, new Set());
    listeners.get(event).add(handler);
    return () => off(event, handler); // return unsubscribe function
  }

  function off(event, handler) {
    listeners.get(event)?.delete(handler);
  }

  function emit(event, data) {
    listeners.get(event)?.forEach(fn => fn(data));
  }

  function once(event, handler) {
    const wrapper = (data) => {
      handler(data);
      off(event, wrapper);
    };
    on(event, wrapper);
  }

  function listenerCount(event) {
    return listeners.get(event)?.size ?? 0;
  }

  return { on, off, emit, once, listenerCount };
}

const bus = createEventBus();

const unsubscribe = bus.on("user:login", ({ username }) => {
  console.log(`Welcome back, ${username}`);
});

bus.emit("user:login", { username: "Alice" }); // "Welcome back, Alice"
unsubscribe(); // clean removal
bus.emit("user:login", { username: "Alice" }); // nothing — listener removed

// Caller cannot do this:
bus.listeners; // undefined — completely private
```

---

## 6. Real-World Examples

### Example 1 — API client module

```js
function createApiClient(baseUrl, defaultHeaders = {}) {
  let authToken = null; // private

  async function request(method, path, options = {}) {
    const headers = {
      "Content-Type": "application/json",
      ...defaultHeaders,
      ...(authToken ? { Authorization: `Bearer ${authToken}` } : {}),
      ...options.headers
    };

    const res = await fetch(`${baseUrl}${path}`, {
      method,
      headers,
      body: options.body ? JSON.stringify(options.body) : undefined
    });

    if (!res.ok) {
      const err = await res.json().catch(() => ({}));
      throw Object.assign(new Error(err.message || `HTTP ${res.status}`), {
        statusCode: res.status
      });
    }

    return res.status === 204 ? null : res.json();
  }

  return {
    setToken(token) { authToken = token; },
    clearToken()    { authToken = null; },

    get(path, options)          { return request("GET",    path, options); },
    post(path, body, options)   { return request("POST",   path, { ...options, body }); },
    put(path, body, options)    { return request("PUT",    path, { ...options, body }); },
    patch(path, body, options)  { return request("PATCH",  path, { ...options, body }); },
    delete(path, options)       { return request("DELETE", path, options); }
  };
}

// Usage
const api = createApiClient("https://api.example.com", {
  "X-Client-Version": "2.0"
});

const { token } = await api.post("/auth/login", { email, password });
api.setToken(token);

const user = await api.get("/me");
const orders = await api.get("/orders?limit=10");
```

### Example 2 — Logger module (ES Module style)

```js
// logger.js
const LEVELS = { debug: 0, info: 1, warn: 2, error: 3 };
let minLevel = LEVELS.info;  // private
let transport = console;      // private — swappable for testing

function shouldLog(level) {
  return LEVELS[level] >= minLevel;
}

function format(level, message, meta) {
  return {
    level,
    message,
    timestamp: new Date().toISOString(),
    ...(meta && { meta })
  };
}

function log(level, message, meta) {
  if (!shouldLog(level)) return;
  const entry = format(level, message, meta);
  transport[level === "debug" ? "log" : level](JSON.stringify(entry));
}

export const debug = (msg, meta) => log("debug", msg, meta);
export const info  = (msg, meta) => log("info",  msg, meta);
export const warn  = (msg, meta) => log("warn",  msg, meta);
export const error = (msg, meta) => log("error", msg, meta);

export function setLevel(level) {
  if (!(level in LEVELS)) throw new RangeError(`Invalid log level: ${level}`);
  minLevel = LEVELS[level];
}

export function setTransport(t) {
  transport = t; // swap out console for testing
}

// shouldLog, format, log, LEVELS — all private
```

```js
// app.js
import * as logger from "./logger.js";

logger.setLevel("debug");
logger.info("Server started", { port: 3000 });
logger.error("DB connection failed", { host: "localhost" });
```

### Example 3 — Cache module with TTL

```js
function createCache(defaultTtlMs = 60_000) {
  const store = new Map();  // private

  function isExpired(entry) {
    return Date.now() > entry.expiresAt;
  }

  function cleanup() {
    for (const [key, entry] of store) {
      if (isExpired(entry)) store.delete(key);
    }
  }

  // Run cleanup every minute
  const cleanupTimer = setInterval(cleanup, 60_000);

  return {
    set(key, value, ttlMs = defaultTtlMs) {
      store.set(key, { value, expiresAt: Date.now() + ttlMs });
    },

    get(key) {
      const entry = store.get(key);
      if (!entry) return undefined;
      if (isExpired(entry)) { store.delete(key); return undefined; }
      return entry.value;
    },

    has(key) {
      return this.get(key) !== undefined;
    },

    delete(key) {
      store.delete(key);
    },

    clear() {
      store.clear();
    },

    get size() {
      cleanup(); // count only live entries
      return store.size;
    },

    destroy() {
      clearInterval(cleanupTimer); // cleanup resource
      store.clear();
    }
  };
}

const cache = createCache(30_000); // 30s TTL
cache.set("user:42", userData);
cache.get("user:42"); // userData
// 30 seconds later:
cache.get("user:42"); // undefined — expired
```

### Example 4 — Request rate limiter

```js
function createRateLimiter(maxRequests, windowMs) {
  const windows = new Map(); // private: ip → { count, resetAt }

  function getWindow(key) {
    const now = Date.now();
    const win = windows.get(key);

    if (!win || now > win.resetAt) {
      const fresh = { count: 0, resetAt: now + windowMs };
      windows.set(key, fresh);
      return fresh;
    }
    return win;
  }

  // Returns Express middleware
  return function rateLimitMiddleware(req, res, next) {
    const key = req.ip;
    const win = getWindow(key);
    win.count++;

    const remaining = Math.max(0, maxRequests - win.count);
    res.setHeader("X-RateLimit-Limit",     maxRequests);
    res.setHeader("X-RateLimit-Remaining", remaining);
    res.setHeader("X-RateLimit-Reset",     Math.ceil(win.resetAt / 1000));

    if (win.count > maxRequests) {
      return res.status(429).json({ error: "Too many requests" });
    }
    next();
  };
}

// Usage in Express
const apiLimiter = createRateLimiter(100, 60_000); // 100 req/min
app.use("/api", apiLimiter);

const authLimiter = createRateLimiter(5, 60_000);  // 5 req/min for auth
app.use("/api/auth", authLimiter);
```

### Example 5 — Connection pool (simplified)

```js
function createConnectionPool(config) {
  const available = [];  // private: idle connections
  const waiting   = [];  // private: pending acquire callbacks

  async function createConnection() {
    return db.connect(config); // hypothetical DB connect
  }

  async function acquire() {
    if (available.length > 0) {
      return available.pop();
    }

    // No available connection — wait for one
    return new Promise((resolve) => {
      waiting.push(resolve);
    });
  }

  function release(conn) {
    if (waiting.length > 0) {
      // Give it directly to the next waiter
      waiting.shift()(conn);
    } else {
      available.push(conn);
    }
  }

  async function withConnection(fn) {
    const conn = await acquire();
    try {
      return await fn(conn);
    } finally {
      release(conn);
    }
  }

  // Initialise pool
  async function init() {
    for (let i = 0; i < config.min; i++) {
      available.push(await createConnection());
    }
  }

  return { acquire, release, withConnection, init };
}
```

---

## 7. Module Pattern vs Classes

Both achieve encapsulation but with different trade-offs:

```js
// Factory function (module pattern)
function createUser(name, email) {
  let loginCount = 0; // truly private

  return {
    get name()  { return name; },
    get email() { return email; },
    login()     { loginCount++; },
    getLoginCount() { return loginCount; }
  };
}

// Class (with private fields)
class User {
  #loginCount = 0; // private field (ES2022)

  constructor(name, email) {
    this.name  = name;
    this.email = email;
  }

  login()     { this.#loginCount++; }
  getLoginCount() { return this.#loginCount; }
}
```

| | Factory function | Class |
|---|---|---|
| Privacy mechanism | Closure | `#` private fields |
| `instanceof` | ❌ no | ✅ yes |
| Prototype chain | ❌ each instance has own methods | ✅ shared methods (memory efficient) |
| `this` binding issues | ✅ no — closures, not `this` | ⚠️ yes — must bind or use class fields |
| Inheritance | Via composition | `extends` |
| When to use | Simple objects, no inheritance needed | When you need `instanceof`, inheritance, or many instances |

For most utility modules (logger, cache, event bus, API client), factory functions are simpler and safer. For domain entities where `instanceof` and inheritance matter, use classes.

---

## 8. Tricky Things

### 1. Closure state is per-instance with factory functions

```js
const a = createCounter();
const b = createCounter();

a.increment();
a.increment();
b.increment();

a.getCount(); // 2 — its own private count
b.getCount(); // 1 — its own private count
// They share no state
```

### 2. Methods in the returned object have access to private state via closure — but only if defined inside the factory

```js
function createService() {
  let secret = 42;

  const api = {
    getSecret() { return secret; } // ✅ closure — works
  };

  // Adding a method AFTER creation does NOT get closure access
  api.anotherMethod = function() {
    return secret; // ❌ ReferenceError — `secret` is not in this scope
  };

  return api;
}
```

### 3. IIFE modules are singletons — factory functions are not

```js
// IIFE — one instance, shared state
const Logger = (() => {
  let count = 0;
  return { log(msg) { count++; console.log(msg); } };
})();

// Factory — independent instances
const logA = createLogger();
const logB = createLogger();
// logA and logB have separate state
```

### 4. The returned object is still mutable — callers can add properties

```js
const counter = createCounter();
counter.getCount();   // ✅ works
counter.secret = 999; // ✅ this adds a property to the returned object

// But this doesn't affect the private state — it just adds to the public object
// To prevent it: Object.freeze the returned object

function createCounter() {
  let count = 0;
  return Object.freeze({
    increment() { count++; },
    getCount()  { return count; }
  });
}
```

### 5. Circular dependencies between factory modules can cause `undefined`

```js
// a.js imports b.js which imports a.js
// At load time, a.js may not be fully evaluated yet
// Solution: lazy imports or restructure to avoid cycles
```

---

## 9. Common Mistakes

### Mistake 1 — Exposing internal state directly

```js
// ❌ Caller can mutate the internal array directly
function createTodoList() {
  const todos = [];
  return {
    todos,          // ← direct reference! caller can push/pop/splice
    add(text) { todos.push({ text, done: false }); }
  };
}

const list = createTodoList();
list.todos.push("INJECTED"); // bypasses all your logic

// ✅ Only expose methods, not the data itself
function createTodoList() {
  const todos = [];
  return {
    add(text)   { todos.push({ text, done: false }); },
    getAll()    { return [...todos]; }, // return a copy, not the reference
    getCount()  { return todos.length; }
  };
}
```

### Mistake 2 — Returning `this` accidentally creates a `this` binding problem

```js
function createService() {
  let data = [];

  return {
    add(item) { data.push(item); },
    addAndGet(item) {
      this.add(item); // ⚠️ fragile if method is destructured
      return data;
    }
  };
}

const { addAndGet } = createService();
addAndGet("x"); // TypeError: this.add is not a function — `this` is undefined

// ✅ Call the private function directly — no `this` needed
function createService() {
  let data = [];

  function add(item) { data.push(item); } // private function

  return {
    add,
    addAndGet(item) {
      add(item); // direct call — no `this`
      return [...data];
    }
  };
}
```

### Mistake 3 — Creating the module inside a loop (loses shared state)

```js
// ❌ A new independent module is created on every iteration
const handlers = routes.map(route => {
  const cache = createCache(); // fresh cache per route — probably not intended
  return createHandler(route, cache);
});

// ✅ Create the module once and share it
const cache = createCache();
const handlers = routes.map(route => createHandler(route, cache));
```

### Mistake 4 — Forgetting cleanup (timers, listeners)

```js
// ❌ No way to stop the internal timer
function createPoller(fn, intervalMs) {
  setInterval(fn, intervalMs);
  return { getLastResult: () => lastResult };
}

// ✅ Always return a cleanup/destroy method
function createPoller(fn, intervalMs) {
  let lastResult;
  const timer = setInterval(async () => {
    lastResult = await fn();
  }, intervalMs);

  return {
    getLastResult: () => lastResult,
    stop: () => clearInterval(timer)  // ← caller can clean up
  };
}

const poller = createPoller(fetchStats, 5000);
// When done:
poller.stop();
```

### Mistake 5 — Mixing module pattern and classes unnecessarily

Pick one consistently per module/feature. Mixing both without reason leads to confusion about where state lives and how to access it.

---

## 10. Practice Exercises

### Easy — counter with limits

Build `createCounter(min, max)` that:
- Keeps a private `count` starting at `min`
- `increment()` — increases by 1, but not above `max`
- `decrement()` — decreases by 1, but not below `min`
- `reset()` — back to `min`
- `getCount()` — returns current count
- `isAtMax()` / `isAtMin()` — booleans

Create two independent counters and verify they don't share state.

---

### Medium — local storage wrapper

Build `createStorage(namespace)` that wraps `localStorage` with:
- `set(key, value)` — stores as JSON, namespaced: `"namespace:key"`
- `get(key, fallback)` — parses JSON, returns `fallback` if missing or parse fails
- `remove(key)` — deletes the namespaced key
- `clear()` — removes only keys with this namespace (not others)
- `keys()` — returns array of keys (without namespace prefix)

Private: the namespace string, the key formatter function.

---

### Hard — observable store

Build `createStore(reducer, initialState)` that:
- Holds state privately
- `dispatch(action)` — runs the reducer, updates state, notifies subscribers
- `getState()` — returns a frozen copy of current state
- `subscribe(fn)` — registers `fn(newState, action)` callback, returns unsubscribe function
- `getHistory()` — returns array of past `{ action, stateBefore, stateAfter }` entries (max 20)
- Works with the `cartReducer` from topic 75's exercises

---

## 11. Quick Reference

```
THREE FORMS
  IIFE module           — singleton, classic pre-ES6
    const M = (() => { let priv; return { pub }; })();

  Factory function      — creates independent instances
    function createM() { let priv; return { pub }; }

  ES Module             — file IS the module, top-level = private, export = public
    let priv; export function pub() { }

REVEALING MODULE
  Define everything as named private functions
  Return { publicName: privateFn, ... } at the bottom
  Rename at reveal point: { delete: remove }

KEY RULES
  Never expose internal data directly — expose copies or accessor methods
  Don't use `this` inside factory functions — call private functions directly
  Always return a cleanup method when you create timers/listeners
  Freeze the returned object to prevent callers from adding properties
  One module = one concern

FACTORY vs CLASS
  Factory   — no `instanceof`, methods per-instance (closure), no `this` issues
  Class     — `instanceof` works, shared prototype, use when you need inheritance

PRIVACY MECHANISMS
  Closure variables     — factory function / IIFE
  # private fields      — class syntax (ES2022)
  ES Module scope       — file-level privacy
```

---

## 12. Connected Topics

- **22 — Closures** — the mechanism that makes private state possible in factory functions
- **42 — Classes & OOP** — classes with `#` private fields achieve the same encapsulation via different syntax
- **59 — ES Modules** — `export`/`import` is the modern module pattern at file level
- **75 — Immutability** — `Object.freeze` on the returned public object
- **77 — Observer pattern** — `createEventBus` in this topic is the observer pattern
- **78 — Clean code principles** — single responsibility, intentional public API — both flow from module thinking
