# 56 — async/await

## What is this?

`async/await` is **syntactic sugar over Promises** that lets you write asynchronous code as if it were synchronous — line by line, top to bottom, without chaining `.then()` callbacks. The `async` keyword marks a function as asynchronous (it always returns a Promise). The `await` keyword pauses execution *inside* that function until a Promise settles, then resumes with the resolved value. Think of it like a polite pause: when you `await` something, you're saying "I'll wait right here for this result before moving on" — but critically, you're only pausing *your function*, not the entire program. Everything else keeps running normally.

## Why does it matter?

`async/await` is how virtually all modern JavaScript and Node.js code is written. Express handlers, NestJS controllers, Mongoose queries, Prisma operations, AWS SDK calls — all use async/await. It eliminates the mental overhead of `.then()` chains, makes error handling with `try/catch` natural, and makes complex async flows (branching, looping, error recovery) readable. Mastering async/await — including its edge cases — is the single most impactful skill for a backend JavaScript developer.

---

## Syntax

```js
// 'async' before function — marks it as async, makes it always return a Promise
async function fetchUser(userId) {
  // 'await' pauses HERE until the Promise settles
  const response = await fetch(`/api/users/${userId}`);

  // Code here only runs after fetch resolves
  const user = await response.json();

  return user;   // implicitly wrapped in Promise.resolve(user)
}

// Consuming an async function — it returns a Promise
fetchUser(1)
  .then(user => console.log(user))
  .catch(err  => console.error(err));

// Or await it inside another async function:
async function main() {
  const user = await fetchUser(1);
  console.log(user);
}
```

---

## How it works — under the hood

`async/await` is 100% Promises. The JavaScript engine transforms it:

```js
// What you write:
async function getUser(id) {
  const res  = await fetch(`/api/users/${id}`);
  const user = await res.json();
  return user;
}

// What JS does internally (approximate):
function getUser(id) {
  return fetch(`/api/users/${id}`)
    .then(res => res.json())
    .then(user => user);
}
```

Key facts:
- `async` function always returns a Promise — even if you `return 42`, the caller gets `Promise.resolve(42)`
- `await` can only be used **inside** an `async` function (or at the top level of an ES module)
- `await` suspends the current function and schedules the continuation as a **microtask** — other code keeps running
- If the awaited Promise rejects, `await` **throws** the rejection as an error (catchable with `try/catch`)

---

## The execution flow — step by step

```js
console.log("1 — sync");

async function run() {
  console.log("2 — inside async fn, before await");

  const result = await Promise.resolve("hello");
  // ↑ suspends run(), returns control to the caller

  console.log("4 — after await, result:", result);
}

run();  // starts executing run(), returns a Promise

console.log("3 — sync, after calling run()");

// Output:
// 1 — sync
// 2 — inside async fn, before await
// 3 — sync, after calling run()
// 4 — after await, result: hello
//
// "3" runs before "4" because run() is SUSPENDED at await,
// control returns to the caller, then resumes as a microtask
```

---

## Error handling with try/catch

When an awaited Promise rejects, it throws the rejection reason. `try/catch` catches it:

```js
async function loadUserData(userId) {
  try {
    const response = await fetch(`/api/users/${userId}`);

    if (!response.ok) {
      throw new Error(`HTTP ${response.status}: ${response.statusText}`);
    }

    const user = await response.json();
    return user;

  } catch (err) {
    // Catches: network errors, HTTP errors we threw, JSON parse errors
    console.error("Failed to load user:", err.message);
    throw err;   // re-throw so the caller knows it failed
  }
}

// Caller handles it:
async function main() {
  try {
    const user = await loadUserData(999);
    console.log("User:", user.name);
  } catch (err) {
    console.error("Main caught:", err.message);
  }
}
```

---

## `async` functions always return a Promise

```js
async function greet(name) {
  return `Hello, ${name}!`;   // plain value
}

async function greetAsync(name) {
  await delay(100);
  return `Hello, ${name}!`;   // after async work
}

async function explode() {
  throw new Error("boom");   // async function that rejects
}

// All return Promises:
greet("Alice").then(v => console.log(v));         // "Hello, Alice!"
greetAsync("Bob").then(v => console.log(v));      // "Hello, Bob!" (after 100ms)
explode().catch(e => console.error(e.message));   // "boom"

// You can verify:
console.log(greet("Alice") instanceof Promise);   // true

// Returning a Promise from async function — it adopts that Promise's state:
async function wrapper() {
  return fetch("/api/data");   // returns the fetch Promise — outer Promise waits for it
}
// wrapper() resolves when fetch resolves (not when wrapper returns the Promise object)
```

---

## Example 1 — sequential async operations

Operations that depend on each other must be sequential:

```js
async function processOrder(orderId) {
  console.log(`Processing order ${orderId}...`);

  // Step 1: load the order
  const order = await fetchOrder(orderId);
  console.log(`Order loaded: ${order.item}, qty: ${order.quantity}`);

  // Step 2: check inventory (depends on the order we just loaded)
  const inventory = await checkInventory(order.productId);
  if (inventory.stock < order.quantity) {
    throw new Error(`Insufficient stock: need ${order.quantity}, have ${inventory.stock}`);
  }

  // Step 3: charge the customer (depends on the order)
  const payment = await chargeCustomer(order.customerId, order.total);
  console.log(`Payment ${payment.id} confirmed`);

  // Step 4: ship the order (depends on payment)
  const shipment = await createShipment(order, payment.id);
  console.log(`Shipment ${shipment.trackingId} created`);

  return {
    order,
    payment,
    shipment,
    status: "complete",
  };
}

processOrder("ORD-001")
  .then(result => console.log("Done:", result.status))
  .catch(err   => console.error("Order failed:", err.message));
```

---

## Example 2 — parallel operations with `await Promise.all`

When operations are independent, run them in parallel:

```js
async function loadDashboard(userId) {
  // WRONG — sequential (wastes time)
  // const user          = await fetchUser(userId);       // 200ms
  // const orders        = await fetchOrders(userId);     // 400ms
  // const notifications = await fetchNotifications(userId); // 150ms
  // Total: 750ms

  // RIGHT — parallel (saves time)
  const [user, orders, notifications] = await Promise.all([
    fetchUser(userId),           // 200ms
    fetchOrders(userId),         // 400ms
    fetchNotifications(userId),  // 150ms
  ]);
  // Total: 400ms (slowest)

  return { user, orders, notifications };
}
```

### When some are independent and some are dependent — hybrid approach

```js
async function getUserDashboard(userId) {
  // Phase 1: get user (required for phase 2)
  const user = await fetchUser(userId);

  // Phase 2: now that we have user, fetch dependent data in parallel
  const [orders, preferences, activity] = await Promise.all([
    fetchOrders(user.id),
    fetchPreferences(user.id),
    fetchRecentActivity(user.id),
  ]);

  // Phase 3: depends on orders
  const orderDetails = await fetchOrderDetails(orders.map(o => o.id));

  return { user, orders, preferences, activity, orderDetails };
}
```

---

## Example 3 — real world: Express.js async route handlers

```js
const express = require("express");
const app     = express();

// Async route handler — Express catches unhandled Promise rejections in v5+
// In Express v4, you must wrap manually (see asyncHandler below)
app.get("/users/:id", async (req, res, next) => {
  try {
    const userId = parseInt(req.params.id);

    if (isNaN(userId) || userId <= 0) {
      return res.status(400).json({ error: "Invalid user ID" });
    }

    const [user, orders] = await Promise.all([
      User.findById(userId),
      Order.find({ userId }),
    ]);

    if (!user) {
      return res.status(404).json({ error: "User not found" });
    }

    res.json({ user, orders, orderCount: orders.length });

  } catch (err) {
    next(err);   // pass to Express error middleware
  }
});

// Express v4 helper: wrap async functions to catch rejected Promises
const asyncHandler = (fn) => (req, res, next) =>
  Promise.resolve(fn(req, res, next)).catch(next);

// Cleaner with wrapper:
app.get("/products/:id", asyncHandler(async (req, res) => {
  const product = await Product.findById(req.params.id);
  if (!product) return res.status(404).json({ error: "Not found" });
  res.json(product);
}));

// Global error middleware (must have 4 params)
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(err.statusCode ?? 500).json({
    error: err.message ?? "Internal server error"
  });
});
```

---

## Async in loops — a critical source of bugs

### `for...of` — correct way to await inside a loop

```js
const userIds = [1, 2, 3, 4, 5];

// RIGHT — sequential: waits for each before moving to next
async function processUsersSequentially(userIds) {
  const results = [];
  for (const id of userIds) {
    const user = await fetchUser(id);   // awaits before next iteration
    results.push(user);
  }
  return results;
}

// RIGHT — parallel: all at once
async function processUsersParallel(userIds) {
  return Promise.all(userIds.map(id => fetchUser(id)));
}
```

### `forEach` — NEVER works with async/await

```js
// WRONG — forEach ignores the returned Promises from async callbacks
async function wrong(userIds) {
  userIds.forEach(async (id) => {
    const user = await fetchUser(id);  // forEach doesn't await this
    console.log(user);
  });
  // Function returns BEFORE any user is fetched!
  // The forEach loop fires all async functions and moves on
}

// Proof:
async function demo() {
  const ids = [1, 2, 3];
  ids.forEach(async (id) => {
    await delay(100);
    console.log("processed", id);
  });
  console.log("forEach returned — but nothing is processed yet!");
}
demo();
// Output:
// "forEach returned — but nothing is processed yet!"
// "processed 1"   ← arrives AFTER the function seemingly "finished"
// "processed 2"
// "processed 3"
```

### `map` — returns array of Promises (requires `Promise.all`)

```js
// WRONG — map with async returns [Promise, Promise, Promise], not results
async function wrong(ids) {
  const results = ids.map(async (id) => {
    return await fetchUser(id);
  });
  console.log(results);  // [Promise, Promise, Promise] — not users!
}

// RIGHT — wrap with Promise.all
async function correct(ids) {
  const results = await Promise.all(ids.map(async (id) => {
    return await fetchUser(id);
  }));
  console.log(results);  // [user1, user2, user3]
}
```

### Comparing loop strategies

```js
const ids = [1, 2, 3, 4, 5];

// Strategy 1: sequential with for...of (total time = sum of all)
async function sequential() {
  for (const id of ids) {
    const user = await fetchUser(id);   // waits for each
  }
}

// Strategy 2: parallel with Promise.all (total time = slowest)
async function parallel() {
  const users = await Promise.all(ids.map(fetchUser));
}

// Strategy 3: parallel with limit (controlled concurrency)
async function limited(limit = 2) {
  const results = [];
  for (let i = 0; i < ids.length; i += limit) {
    const batch = ids.slice(i, i + limit);
    const batchResults = await Promise.all(batch.map(fetchUser));
    results.push(...batchResults);
  }
  return results;
}
```

---

## `await` with non-Promise values

```js
// await on a non-Promise just returns it immediately (still yields to microtask queue once)
async function test() {
  const x = await 42;           // same as await Promise.resolve(42)
  const y = await "hello";      // same as await Promise.resolve("hello")
  const z = await null;         // null
  console.log(x, y, z);        // 42 "hello" null
}
```

---

## Top-level `await` (ES2022, ES modules only)

In `.mjs` files or files with `"type": "module"` in package.json, you can use `await` at the top level without wrapping in an async function:

```js
// config.mjs — top-level await
const config   = await fetch("/api/config").then(r => r.json());
const dbClient = await connectToDatabase(config.dbUrl);

export { config, dbClient };
```

This is especially useful for module initialisation that needs async setup. It also means anything that imports this module will wait for the top-level `await` to complete.

---

## `async` methods in classes

```js
class UserService {
  #db;

  constructor(db) {
    this.#db = db;
  }

  // async instance method
  async findById(id) {
    const user = await this.#db.query("SELECT * FROM users WHERE id = $1", [id]);
    if (!user.rows.length) throw new Error(`User ${id} not found`);
    return user.rows[0];
  }

  // async static method
  static async create(db, userData) {
    const service = new UserService(db);
    return service;
  }

  // async getter — NOT directly supported
  // get async userData() {}  — syntax error!
  // Instead, use an explicit async method:
  async getUserData() {
    return this.findById(this.#id);
  }
}

// Usage:
const userService = new UserService(db);
const user = await userService.findById(1);
```

---

## Error handling patterns — all variations

### Pattern 1: try/catch per async call (granular)

```js
async function robustLoad(userId) {
  let user;
  try {
    user = await fetchUser(userId);
  } catch (err) {
    console.error("User fetch failed, using guest:", err.message);
    user = { id: null, name: "Guest", role: "visitor" };  // fallback
  }

  let orders = [];
  try {
    orders = await fetchOrders(userId);
  } catch (err) {
    console.warn("Orders unavailable:", err.message);
    // non-critical — continue with empty orders
  }

  return { user, orders };
}
```

### Pattern 2: single try/catch for the whole flow

```js
async function strictLoad(userId) {
  try {
    const user   = await fetchUser(userId);
    const orders = await fetchOrders(userId);
    const prefs  = await fetchPreferences(userId);
    return { user, orders, prefs };
  } catch (err) {
    // Any failure aborts everything
    throw new AppError("Failed to load user data", { cause: err });
  }
}
```

### Pattern 3: `.catch()` on individual awaits (inline default)

```js
async function withDefaults(userId) {
  const user  = await fetchUser(userId)
    .catch(() => ({ id: userId, name: "Unknown", role: "guest" }));

  const posts = await fetchPosts(userId)
    .catch(() => []);  // default to empty array

  return { user, posts };
}
```

### Pattern 4: the `to` pattern (Go-style error handling)

```js
// Helper: converts Promise to [error, value] tuple — no try/catch needed
async function to(promise) {
  try {
    const data = await promise;
    return [null, data];
  } catch (err) {
    return [err, null];
  }
}

// Usage — clean, no nesting
async function loadPage(userId) {
  const [userErr, user] = await to(fetchUser(userId));
  if (userErr) {
    console.error("User load failed:", userErr.message);
    return null;
  }

  const [ordersErr, orders] = await to(fetchOrders(user.id));
  if (ordersErr) {
    console.warn("Orders unavailable — showing empty list");
    return { user, orders: [] };
  }

  return { user, orders };
}
```

### Pattern 5: `async` IIFE for top-level execution in scripts

```js
// CommonJS module — no top-level await — use an async IIFE
(async () => {
  try {
    const config = await loadConfig();
    const db     = await connectDB(config.dbUrl);
    await runMigrations(db);
    await startServer(db);
    console.log("Server running on port", config.port);
  } catch (err) {
    console.error("Startup failed:", err);
    process.exit(1);
  }
})();
```

---

## Async error propagation chain

```js
async function step3() {
  throw new Error("step3 failed");
}

async function step2() {
  await step3();   // throws → propagates up
  return "step2 done";   // never reached
}

async function step1() {
  await step2();   // receives the thrown error
  return "step1 done";   // never reached
}

// Entire chain unwinds via rejected Promises
step1()
  .catch(err => console.error("Caught at top:", err.message));
// "Caught at top: step3 failed"

// Same with try/catch:
async function main() {
  try {
    await step1();
  } catch (err) {
    console.error("Caught:", err.message);  // "step3 failed"
  }
}
```

---

## Tricky things

### 1. Missing `await` — the silent bug

```js
async function saveUser(userData) {
  validateUser(userData);    // sync — fine
  db.save(userData);         // async — but forgot await!
  return "saved";            // returns immediately before save completes
}

// Caller thinks save is done but the db.save() is still running
const result = await saveUser(data);
console.log(result);   // "saved" — but DB might not have saved yet!

// Fix:
async function saveUser(userData) {
  validateUser(userData);
  await db.save(userData);   // wait for it
  return "saved";
}
```

### 2. `await` inside `try` only catches awaited rejections

```js
async function tricky() {
  try {
    const p = fetchData();   // Promise created, NOT awaited
    return p;                // returned without awaiting
    // If fetchData() rejects, it WON'T be caught by this try/catch
    // The rejection escapes to the caller
  } catch (err) {
    console.log("Never catches fetchData rejection");
  }
}

// Fix — always await inside try:
async function correct() {
  try {
    const result = await fetchData();   // awaited inside try
    return result;
  } catch (err) {
    console.log("Catches properly");
  }
}
```

### 3. `async` in constructors — not directly possible

```js
// WRONG — constructors cannot be async
class Database {
  async constructor() {   // SyntaxError: constructor cannot be async
    this.connection = await connect();
  }
}

// Pattern 1: static async factory method
class Database {
  #connection;
  constructor(connection) {
    this.#connection = connection;
  }
  static async create(url) {
    const connection = await connect(url);
    return new Database(connection);
  }
}
const db = await Database.create("mongodb://localhost/mydb");

// Pattern 2: init() method
class Database {
  async init(url) {
    this.connection = await connect(url);
    return this;  // return this for chaining
  }
}
const db = await new Database().init("mongodb://localhost/mydb");
```

### 4. Sequential `await` accidentally — the most common performance bug

```js
// WRONG — looks parallel, IS sequential (3 separate awaits)
async function wrong() {
  const a = await getA();   // starts, waits
  const b = await getB();   // only starts AFTER a finishes
  const c = await getC();   // only starts AFTER b finishes
  // Total time: getA + getB + getC
}

// RIGHT — truly parallel
async function correct() {
  const [a, b, c] = await Promise.all([getA(), getB(), getC()]);
  // Total time: max(getA, getB, getC)
}

// SUBTLE — even this is sequential (common mistake):
async function subtle() {
  const promiseA = getA();   // starts A
  const promiseB = getB();   // starts B immediately after (both running now)
  const a = await promiseA;  // waits for A
  const b = await promiseB;  // waits for B (but B has been running since line 2)
  // This IS parallel! Both started before either await
  // Equivalent to Promise.all([getA(), getB()])
}
```

### 5. `await` in `catch` and `finally`

```js
async function withCleanup() {
  let connection;
  try {
    connection = await openConnection();
    const data = await fetchData(connection);
    return data;
  } catch (err) {
    // You can await inside catch:
    await logErrorToService(err);   // async error logging
    throw err;
  } finally {
    // You can await inside finally too:
    if (connection) await connection.close();   // always close the connection
  }
}
```

### 6. Unhandled rejection from a floating async function

```js
async function riskyOp() {
  throw new Error("something failed");
}

// WRONG — no .catch(), no try/catch — unhandled rejection
async function main() {
  riskyOp();   // missing await — floating Promise, rejection ignored
  console.log("this runs fine but riskyOp's error is lost");
}

// RIGHT
async function main() {
  await riskyOp();   // await it, so error propagates to main's try/catch
  // OR:
  riskyOp().catch(err => console.error("handled:", err.message));
}
```

### 7. `return await` vs `return` inside async — subtle difference

```js
// These look the same but behave differently in try/catch:

async function withoutAwait() {
  try {
    return fetchData();   // returns the Promise — NOT awaited inside the try
    // If fetchData() rejects, it happens OUTSIDE this try/catch
  } catch (err) {
    console.log("may not catch this rejection");
  }
}

async function withAwait() {
  try {
    return await fetchData();   // awaited INSIDE the try
    // If fetchData() rejects, it IS caught by this try/catch
  } catch (err) {
    console.log("definitely catches this rejection");
  }
}
```

Rule: inside a `try/catch` block, always use `return await`, not just `return`.

---

## Common mistakes

### Mistake 1 — `await` inside `.forEach()` doesn't work

```js
// WRONG — forEach doesn't await the async callbacks
async function processAll(items) {
  items.forEach(async item => {
    await processItem(item);   // forEach ignores this Promise
  });
  // Function returns before any processing is done!
}

// RIGHT — for...of awaits each
async function processAll(items) {
  for (const item of items) {
    await processItem(item);
  }
}

// RIGHT — parallel: map + Promise.all
async function processAll(items) {
  await Promise.all(items.map(item => processItem(item)));
}
```

### Mistake 2 — Sequential `await` for independent operations

```js
// WRONG — 600ms (200+200+200)
async function loadProfile(userId) {
  const name    = await fetchName(userId);    // 200ms
  const avatar  = await fetchAvatar(userId);  // 200ms
  const bio     = await fetchBio(userId);     // 200ms
  return { name, avatar, bio };
}

// RIGHT — 200ms (parallel)
async function loadProfile(userId) {
  const [name, avatar, bio] = await Promise.all([
    fetchName(userId),
    fetchAvatar(userId),
    fetchBio(userId),
  ]);
  return { name, avatar, bio };
}
```

### Mistake 3 — Not re-throwing errors when needed

```js
// WRONG — error swallowed, caller thinks operation succeeded
async function saveRecord(data) {
  try {
    await db.insert(data);
  } catch (err) {
    console.error("Insert failed:", err);
    // No re-throw — function returns undefined (implicitly resolved)
  }
}

const result = await saveRecord(data);
// result = undefined — caller has no idea the save failed!

// RIGHT — re-throw or return a meaningful error indicator
async function saveRecord(data) {
  try {
    return await db.insert(data);
  } catch (err) {
    console.error("Insert failed:", err);
    throw new Error(`Failed to save record: ${err.message}`);
  }
}
```

---

## Practice exercises

### Exercise 1 — easy

Write three async functions using `async/await` and `setTimeout`-based delays:
- `getWeather(city)` — resolves after 300ms with `{ city, temp: 22, condition: "sunny" }`; rejects if city is `"Atlantis"` with `"City not found"`
- `getForcast(city, days)` — resolves after 200ms with an array of `days` forecast objects
- `getWeatherDashboard(city)` — uses `Promise.all` to fetch both simultaneously and returns combined data

Wrap `getWeatherDashboard` in proper `try/catch` and test both success and the "Atlantis" error case.

```js
// Write your code here
```

### Exercise 2 — medium

Build a Node.js-style file processing pipeline using `async/await`. Create async functions that simulate:
1. `readFile(filename)` — resolves after 100ms; rejects if filename contains "missing"
2. `parseJSON(content)` — resolves with parsed object; rejects with `"Invalid JSON"` if parsing fails
3. `validateSchema(data)` — resolves if data has `name` and `email` fields; rejects otherwise
4. `saveToDatabase(record)` — has a 20% random failure rate; rejects with `"DB write failed"`
5. `sendConfirmationEmail(email)` — resolves after 50ms (non-critical — failure should NOT fail pipeline)

Write `processFile(filename)` that runs all steps in sequence. `sendConfirmationEmail` should be attempted but failures should be logged and ignored. Use the `to` pattern (from the tricky section) for error handling.

```js
// Write your code here
```

### Exercise 3 — hard

Build a `TaskRunner` class that manages async tasks with the following features:
- Constructor takes `{ concurrency: number, retries: number, retryDelay: number }`
- `add(taskFn, label)` — queues a task (function returning a Promise)
- `run()` — starts executing queued tasks, respecting `concurrency` limit
  - If a task fails, retry up to `retries` times with `retryDelay` ms between retries
  - After all retries exhausted, mark task as failed (don't throw — collect the failure)
- Returns a Promise resolving to `{ results: [...], errors: [...] }` where each item has `{ label, value/error, attempts }`
- While running: emit progress logs like `"[label] attempt 1/3 started"`, `"[label] succeeded after 2 attempts"`, `"[label] failed after 3 attempts: [error]"`

Test with 6 tasks of varying delays and some that always fail, concurrency=2, retries=3.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Syntax / Rule |
|---|---|
| Mark function async | `async function fn() {}` / `async () => {}` / `async method() {}` |
| Pause for Promise | `const val = await somePromise` |
| Return value | `return x` inside async fn → caller gets `Promise.resolve(x)` |
| Error handling | `try { await ... } catch (err) { ... }` |
| `await` non-Promise | Works fine — value passes through unchanged |
| Top-level await | Only in ES modules (`.mjs` or `"type":"module"`) |
| Parallel pattern | `const [a, b] = await Promise.all([getA(), getB()])` |
| Sequential in loop | `for...of` with `await` inside |
| Never use | `forEach` with async callbacks — forEach ignores the Promises |
| `map` + async | Returns `[Promise, ...]` — must `await Promise.all(arr.map(async ...))` |
| Async in class | `async methodName() {}` — works; `async constructor()` — SyntaxError |
| `return await` vs `return` | Inside `try/catch`, always `return await` to catch the rejection |
| Floating async call | `asyncFn()` without `await` → unhandled rejection risk |
| Error propagation | Unhandled `throw` inside async fn rejects its returned Promise |
| `await` in catch/finally | Allowed — can do async logging/cleanup |
| Sequential mistake | `await getA(); await getB()` = sequential; `await Promise.all(...)` = parallel |

---

## Connected topics

- **54 — Promises** — `async/await` is Promises under the hood. Every `await` is a `.then()`. Every `throw` is a `reject()`.
- **55 — Promise combinators** — `await Promise.all([...])` is the most important pattern in backend async code. These topics are inseparable.
- **57 — Error handling** — `try/catch/finally` with async/await is covered in depth in the next topic, including custom error types.
- **58 — fetch API** — The primary real-world use of async/await: making HTTP requests in Node.js and the browser.
