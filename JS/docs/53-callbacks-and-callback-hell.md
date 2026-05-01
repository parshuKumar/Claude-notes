# 53 — Callbacks and Callback Hell

## What is this?

A callback is simply **a function you pass as an argument to another function, to be called later** — either after some time has passed, after an async operation completes, or after some event occurs. It's the oldest async pattern in JavaScript and the direct predecessor of Promises and async/await. Think of it like leaving your phone number at a restaurant when there's no table available: you don't stand at the door waiting (blocking), you go about your day, and they *call you back* when your table is ready. Callback hell is what happens when you have to chain many of these "call me back" requests one after another — each dependent on the previous — and you end up with code nested so deeply it becomes impossible to read, maintain, or debug.

## Why does it matter?

Even though modern code uses Promises and async/await, callbacks are still everywhere: `addEventListener`, `Array.forEach/map/filter/reduce`, `setTimeout`, `fs.readFile`, Express.js middleware, error-first callbacks in Node.js core modules, and every library that hasn't fully migrated to Promises. You cannot read Node.js source code, old libraries, or legacy backend code without understanding callbacks. More importantly, Promises and async/await were *invented* specifically to solve callback hell — and you won't appreciate the solution until you've felt the problem.

---

## Syntax

```js
// The simplest callback: a function passed as an argument
function doWork(data, onSuccess, onError) {
  if (!data) {
    onError(new Error("No data provided"));   // call the error callback
    return;
  }
  const result = data.toUpperCase();
  onSuccess(result);                          // call the success callback
}

// Using it:
doWork(
  "hello",
  (result) => console.log("Success:", result),   // success callback
  (err)    => console.error("Error:", err.message) // error callback
);

// Output: "Success: HELLO"
```

---

## How it works — line by line

```js
function doWork(data, onSuccess, onError) {
```
`doWork` accepts a value AND two functions. The names `onSuccess`/`onError` are just parameter names — they're regular function references.

```js
  onError(new Error("No data provided"));
```
Calls the error function that was passed in. The caller decides what to do when something goes wrong.

```js
  onSuccess(result);
```
Calls the success function with the computed result.

```js
doWork("hello", (result) => ..., (err) => ...)
```
Arrow functions passed inline as callbacks. They're defined here, called inside `doWork`.

---

## Synchronous callbacks (you already use these daily)

Not all callbacks are async. Many are called synchronously within the same tick:

```js
// forEach — callback called synchronously, once per element
const numbers = [1, 2, 3, 4, 5];

numbers.forEach(function(num) {
  console.log(num * 2);   // 2, 4, 6, 8, 10 — all print before next line
});

console.log("this runs AFTER forEach is done");

// filter — synchronous callback
const evens = numbers.filter(n => n % 2 === 0);  // [2, 4]

// sort — synchronous callback
const names = ["Charlie", "Alice", "Bob"];
names.sort((a, b) => a.localeCompare(b));  // ["Alice", "Bob", "Charlie"]
```

Synchronous callbacks don't involve the event loop at all — they run inline, immediately, on the call stack.

---

## Asynchronous callbacks — where timing matters

```js
console.log("1 — before setTimeout");

setTimeout(function onTimeout() {
  console.log("3 — inside callback (async)");
}, 1000);

console.log("2 — after setTimeout");

// Output:
// 1 — before setTimeout
// 2 — after setTimeout
// (1 second pause)
// 3 — inside callback (async)
```

`onTimeout` is placed in the macrotask queue by the timer API after 1000ms. It only runs when the call stack is empty.

---

## The Node.js error-first callback convention

Node.js standardised a specific callback signature: **the first argument is always an error (or `null` if none), the second is the result**. This is called the "error-first" or "Node.js callback" convention:

```js
const fs = require("fs");

// Signature: fs.readFile(path, options, callback)
// callback signature: (error, data) => void
fs.readFile("./config.json", "utf8", function(error, data) {
  if (error) {
    console.error("Failed to read file:", error.message);
    return;   // ALWAYS return after handling error — prevents running success code
  }
  const config = JSON.parse(data);
  console.log("Config loaded:", config);
});
```

This convention exists so every callback handles errors the same way. Libraries like Express, mongoose (older API), and all of Node.js core use it.

```js
// Pattern to remember — error-first always
function myAsyncOperation(input, callback) {
  if (!input) {
    callback(new Error("Input required"), null);   // error first, result null
    return;
  }
  // ... do work ...
  callback(null, result);   // null error, result second
}

// Using it:
myAsyncOperation("hello", (err, result) => {
  if (err) {
    // handle error
    return;
  }
  // use result
});
```

---

## Example 1 — basic async chain: reading a file then processing

```js
const fs = require("fs");

function readConfig(callback) {
  fs.readFile("./config.json", "utf8", (err, data) => {
    if (err) return callback(err, null);
    let config;
    try {
      config = JSON.parse(data);
    } catch (parseErr) {
      return callback(parseErr, null);
    }
    callback(null, config);
  });
}

function connectToDatabase(config, callback) {
  // Simulate async DB connection
  setTimeout(() => {
    if (!config.dbUrl) {
      return callback(new Error("Missing dbUrl in config"));
    }
    const connection = { url: config.dbUrl, status: "connected" };
    callback(null, connection);
  }, 500);
}

// Using them in sequence — each depends on the previous result:
readConfig((configErr, config) => {
  if (configErr) {
    console.error("Config read failed:", configErr.message);
    return;
  }
  connectToDatabase(config, (dbErr, connection) => {
    if (dbErr) {
      console.error("DB connection failed:", dbErr.message);
      return;
    }
    console.log("Connected to DB:", connection.url);
  });
});
```

Two levels deep — already getting nested. This is the beginning of callback hell.

---

## Callback hell — the real problem

When operations must happen **in sequence** and **each step depends on the previous**, callbacks nest deeper and deeper. This is called "callback hell" (also called "the pyramid of doom" because of its triangular shape):

```js
const fs   = require("fs");
const path = require("path");

// Real scenario: authenticate user → load their profile → fetch their orders
//                → calculate totals → save a report → send a confirmation email

function processUserReport(userId, callback) {
  authenticateUser(userId, (authErr, user) => {              // level 1
    if (authErr) return callback(authErr);

    loadUserProfile(user.id, (profileErr, profile) => {      // level 2
      if (profileErr) return callback(profileErr);

        fetchUserOrders(profile.id, (ordersErr, orders) => { // level 3
          if (ordersErr) return callback(ordersErr);

          calculateTotals(orders, (calcErr, totals) => {     // level 4
            if (calcErr) return callback(calcErr);

              saveReport(totals, (saveErr, report) => {      // level 5
                if (saveErr) return callback(saveErr);

                  sendEmail(user.email, report, (emailErr) => { // level 6
                    if (emailErr) return callback(emailErr);

                    callback(null, "Report sent successfully!");
                  });                                         // end level 6
              });                                             // end level 5
          });                                                 // end level 4
        });                                                   // end level 3
    });                                                       // end level 2
  });                                                         // end level 1
}
```

This is the "pyramid of doom". Problems with this code:
1. **Horizontal drift** — code moves far to the right; you can't see what level you're at
2. **Error handling repeated at every level** — `if (err) return callback(err)` 6 times
3. **Hard to modify** — adding a step in the middle requires re-indenting everything
4. **Hard to reuse** — the inner steps are buried inside closures
5. **Variable scoping confusion** — all variables from outer scopes are visible everywhere
6. **Early returns don't stop outer functions** — `return callback(authErr)` returns from the arrow function, not from `processUserReport`

---

## The problems visualised one by one

### Problem 1: The forgotten return

```js
function getUser(id, callback) {
  findInDatabase(id, (err, user) => {
    if (err) {
      callback(err);        // WRONG — forgot return
      // code continues! falls through to callback(null, user) below
    }
    callback(null, user);   // called twice if there was an error!
  });
}

// RIGHT
function getUser(id, callback) {
  findInDatabase(id, (err, user) => {
    if (err) {
      return callback(err);  // return prevents the rest from running
    }
    callback(null, user);
  });
}
```

Calling a callback twice causes mysterious double-execution bugs that are extremely hard to track down.

### Problem 2: Parallel async — naive callback approach

```js
// Naive — sequential even though operations are independent
function loadDashboard(userId, callback) {
  getUser(userId, (err, user) => {
    if (err) return callback(err);
    getPosts(userId, (err2, posts) => {          // waits for getUser
      if (err2) return callback(err2);
      getNotifications(userId, (err3, notes) => { // waits for getPosts
        if (err3) return callback(err3);
        callback(null, { user, posts, notes });
      });
    });
  });
}
// Total time = getUser + getPosts + getNotifications  (all sequential)

// Better — parallel with counter
function loadDashboard(userId, callback) {
  const results = {};
  let completed = 0;
  let hasError  = false;
  const total   = 3;

  function done(key) {
    return (err, value) => {
      if (hasError) return;
      if (err) { hasError = true; return callback(err); }
      results[key] = value;
      completed++;
      if (completed === total) callback(null, results);
    };
  }

  getUser(userId,          done("user"));
  getPosts(userId,         done("posts"));
  getNotifications(userId, done("notifications"));
}
// Total time = max(getUser, getPosts, getNotifications)  (parallel)
```

### Problem 3: Error handling in loops with callbacks

```js
const userIds = [1, 2, 3, 4, 5];

// WRONG — async operations in a loop, race conditions
userIds.forEach((id) => {
  getUserData(id, (err, data) => {
    if (err) {
      console.error(err);
      return;  // this only stops THIS callback, not the loop
    }
    processData(data);
  });
});
// All 5 requests fire at once, results come back in random order

// Manageable — sequential with recursion
function processAll(ids, index = 0, results = [], callback) {
  if (index >= ids.length) return callback(null, results);

  getUserData(ids[index], (err, data) => {
    if (err) return callback(err);
    results.push(data);
    processAll(ids, index + 1, results, callback);  // recurse for next
  });
}

processAll(userIds, 0, [], (err, allResults) => {
  if (err) return console.error(err);
  console.log("All results:", allResults);
});
```

---

## Escaping callback hell without Promises

Before Promises existed, developers used several patterns to manage callback complexity:

### Pattern 1: Named functions (instead of inline lambdas)

```js
// BEFORE — anonymous inline callbacks — pyramid of doom
getUser(id, (err, user) => {
  getProfile(user.id, (err, profile) => {
    getOrders(profile.id, (err, orders) => { ... });
  });
});

// AFTER — named, flat functions
function handleUser(err, user) {
  if (err) return handleError(err);
  getProfile(user.id, handleProfile);
}

function handleProfile(err, profile) {
  if (err) return handleError(err);
  getOrders(profile.id, handleOrders);
}

function handleOrders(err, orders) {
  if (err) return handleError(err);
  displayOrders(orders);
}

function handleError(err) {
  console.error("Operation failed:", err.message);
}

getUser(id, handleUser);  // flat, readable, each function is independently testable
```

### Pattern 2: Async control flow libraries (pre-Promise era)

The `async` npm library (not to be confused with `async/await`) was widely used:

```js
const async = require("async");   // npm install async

// series — sequential, stops on first error
async.series([
  (cb) => getUser(userId, cb),
  (cb) => getProfile(userId, cb),
  (cb) => getOrders(userId, cb)
], (err, results) => {
  if (err) return handleError(err);
  const [user, profile, orders] = results;
  displayDashboard(user, profile, orders);
});

// parallel — all at once
async.parallel([
  (cb) => getUser(userId, cb),
  (cb) => getProfile(userId, cb),
  (cb) => getOrders(userId, cb)
], (err, results) => {
  if (err) return handleError(err);
  displayDashboard(...results);
});

// waterfall — sequential, passes result of each to the next
async.waterfall([
  (cb)           => readConfig(cb),
  (config, cb)   => connectDB(config, cb),
  (conn, cb)     => fetchData(conn, cb),
  (data, cb)     => processData(data, cb)
], (err, finalResult) => {
  if (err) return handleError(err);
  console.log("Done:", finalResult);
});
```

---

## Promisifying callbacks — the bridge to modern async

You can convert any error-first callback function into a Promise-returning function. Node.js even has a built-in utility for this:

```js
const fs   = require("fs");
const util = require("util");

// util.promisify converts error-first callback → Promise
const readFile = util.promisify(fs.readFile);

// Before promisify:
fs.readFile("config.json", "utf8", (err, data) => {
  if (err) { console.error(err); return; }
  console.log(data);
});

// After promisify:
readFile("config.json", "utf8")
  .then(data => console.log(data))
  .catch(err => console.error(err));

// With async/await:
async function loadConfig() {
  const data = await readFile("config.json", "utf8");
  return JSON.parse(data);
}
```

Most of `fs`, `dns`, `crypto` etc. can be promisified. Node.js 10+ also ships `fs/promises` which is already Promise-based:

```js
const fs = require("fs/promises");  // Node.js 10+

async function setup() {
  const config = JSON.parse(await fs.readFile("config.json", "utf8"));
  await fs.writeFile("output.txt", JSON.stringify(config, null, 2));
  console.log("Setup complete");
}
```

### Manual promisification pattern

```js
// Generic promisify function — wraps any error-first callback function
function promisify(fn) {
  return function(...args) {
    return new Promise((resolve, reject) => {
      fn(...args, (err, result) => {
        if (err) reject(err);
        else     resolve(result);
      });
    });
  };
}

// Use it:
const readFile = promisify(fs.readFile);
const writeFile = promisify(fs.writeFile);
```

---

## When callbacks are still the right choice

Even in modern codebases, callbacks are appropriate for:

1. **Event listeners** — inherently callback-based (DOM events, Node.js EventEmitter)
```js
button.addEventListener("click", (event) => {
  console.log("Clicked:", event.target);
});
```

2. **Array methods** — `forEach`, `map`, `filter`, `reduce`, `sort` — synchronous callbacks
```js
const totals = orders.map(order => order.price * order.quantity);
```

3. **Simple one-off timers** — `setTimeout` for simple delays
```js
setTimeout(() => modal.remove(), 300);  // dismiss after animation
```

4. **Streaming data** — chunk-by-chunk processing, not one big result
```js
readStream.on("data", (chunk) => process(chunk));
readStream.on("end",  () => finalise());
```

5. **Performance-critical inner loops** — callbacks are slightly faster than Promise chains
```js
// In a tight loop processing millions of items, callbacks may be preferred
items.forEach((item) => fastProcessor(item));
```

---

## Example 2 — real world: Express middleware (still callbacks)

Express.js, one of the most popular Node.js frameworks, is entirely callback-based:

```js
const express = require("express");
const app     = express();

// Middleware: (req, res, next) is the callback signature
// next() = "call the next middleware/handler"
// next(err) = "skip to error handler"

function authenticate(req, res, next) {
  const token = req.headers.authorization;
  if (!token) {
    return res.status(401).json({ error: "No token" });
  }
  verifyToken(token, (err, decoded) => {    // callback-based verify
    if (err) return next(err);              // pass to Express error handler
    req.user = decoded;
    next();                                  // continue to next middleware
  });
}

function logRequest(req, res, next) {
  console.log(`${req.method} ${req.path} — user: ${req.user?.id ?? "anonymous"}`);
  next();
}

app.get("/dashboard", authenticate, logRequest, (req, res) => {
  res.json({ dashboard: "data", user: req.user });
});

// Error handling middleware — 4 parameters = Express error handler
app.use((err, req, res, next) => {
  console.error(err.stack);
  res.status(500).json({ error: "Internal server error" });
});
```

---

## Tricky things

### 1. `this` context is lost inside callbacks

```js
class DataService {
  constructor() {
    this.data = [];
    this.url  = "https://api.example.com/items";
  }

  fetchData() {
    // WRONG — regular function loses 'this' binding
    fetch(this.url).then(function(response) {
      return response.json();
    }).then(function(items) {
      this.data = items;   // TypeError: Cannot set properties of undefined
                           // 'this' is undefined (strict mode) or global
    });
  }

  fetchDataCorrect() {
    // FIX 1 — arrow function (lexical this)
    fetch(this.url)
      .then(response => response.json())
      .then(items => { this.data = items; });   // 'this' is the DataService instance

    // FIX 2 — bind
    fetch(this.url)
      .then(function(response) {
        return response.json();
      }.bind(this))
      .then(function(items) {
        this.data = items;
      }.bind(this));

    // FIX 3 — save reference
    const self = this;
    fetch(this.url).then(function(items) {
      self.data = items;   // 'self' is captured from outer scope
    });
  }
}
```

### 2. Callback called synchronously vs asynchronously — Zalgo

A famous async bug: if a function sometimes calls its callback synchronously and sometimes asynchronously, code that assumes one behaviour breaks under the other:

```js
// DANGEROUS — inconsistent: sometimes sync, sometimes async
function getFromCache(key, callback) {
  if (cache.has(key)) {
    callback(null, cache.get(key));   // SYNC — called immediately
    return;
  }
  fetchFromDB(key, (err, value) => {
    cache.set(key, value);
    callback(err, value);             // ASYNC — called later
  });
}

// Code that breaks with Zalgo:
let result;
getFromCache("user:1", (err, value) => {
  result = value;   // If sync: result is set. If async: result is still undefined
});
console.log(result);  // undefined if async, value if sync — unpredictable

// FIX — always async (even for cached values)
function getFromCache(key, callback) {
  if (cache.has(key)) {
    process.nextTick(() => callback(null, cache.get(key)));  // force async
    return;
  }
  fetchFromDB(key, callback);
}
```

### 3. Callback called multiple times

```js
function riskyOperation(input, callback) {
  if (!input) {
    callback(new Error("No input"));
    // falls through — callback called twice on the next line!
  }
  callback(null, processInput(input));
}

// Always use return before callback when in error branch:
function safeOperation(input, callback) {
  if (!input) {
    return callback(new Error("No input"));  // return prevents fall-through
  }
  callback(null, processInput(input));
}
```

### 4. Swallowed errors in callbacks

```js
// WRONG — silent failure
getUser(id, (err, user) => {
  if (err) return;   // error silently swallowed, no logging, no propagation
  process(user);
});

// RIGHT — always handle or propagate errors
getUser(id, (err, user) => {
  if (err) {
    console.error("getUser failed:", err.message);
    return callback(err);    // propagate to caller
  }
  process(user);
});
```

### 5. Memory leaks with long-lived callbacks

```js
// Event listeners accumulate if not removed — memory leak
function startPolling(element) {
  function handler() { console.log("clicked"); }
  element.addEventListener("click", handler);
  // If startPolling is called repeatedly, handlers accumulate forever

  // FIX — return a cleanup function
  return () => element.removeEventListener("click", handler);
}

const stop = startPolling(button);
// Later:
stop();   // removes the listener
```

---

## Common mistakes

### Mistake 1 — Not returning after calling callback in error branch

```js
// WRONG — callback called twice when err exists
function processData(data, callback) {
  if (!data) {
    callback(new Error("No data"));   // ← no return
  }
  callback(null, data.toUpperCase()); // ← also runs if !data — double callback!
}

// RIGHT
function processData(data, callback) {
  if (!data) {
    return callback(new Error("No data"));  // return stops execution
  }
  callback(null, data.toUpperCase());
}
```

### Mistake 2 — Nesting when sequential isn't required

```js
// WRONG — sequential when parallel is possible
getUserName(id, (err, name) => {
  getAvatar(id, (err2, avatar) => {    // waits for name even though it doesn't use it
    getScore(id, (err3, score) => {    // waits for avatar even though it doesn't use it
      render({ name, avatar, score });
    });
  });
});
// Total time: getName + getAvatar + getScore

// RIGHT — parallel (all three fire at once)
const results = {};
let count = 0;
const done = (key) => (err, val) => {
  if (err) return console.error(err);
  results[key] = val;
  if (++count === 3) render(results);
};
getUserName(id, done("name"));
getAvatar(id,   done("avatar"));
getScore(id,    done("score"));
// Total time: max(getName, getAvatar, getScore)
```

### Mistake 3 — Ignoring the error argument

```js
// WRONG — error silently ignored
fs.readFile("config.json", "utf8", (err, data) => {
  const config = JSON.parse(data);  // crashes if err occurred and data is undefined
  startServer(config);
});

// RIGHT — always check err first
fs.readFile("config.json", "utf8", (err, data) => {
  if (err) {
    console.error("Cannot read config:", err.message);
    process.exit(1);  // or handle gracefully
    return;
  }
  const config = JSON.parse(data);
  startServer(config);
});
```

---

## Practice exercises

### Exercise 1 — easy

Write a function `delayedGreet(name, delayMs, callback)` that waits `delayMs` milliseconds then calls `callback` with the string `"Hello, [name]!"`. Write a second function `delayedGreetError(name, delayMs, callback)` that uses the **error-first callback convention**: if `name` is not a string or is empty, it calls `callback(new Error("Name must be a non-empty string"), null)`, otherwise calls `callback(null, "Hello, [name]!")`.

```js
// Write your code here
```

### Exercise 2 — medium

You have three async functions that simulate database queries (use `setTimeout` to simulate delay):
- `getUser(id, callback)` — returns `{ id, name, email }` after 300ms, error if id < 0
- `getOrders(userId, callback)` — returns an array of orders after 500ms
- `getProductDetails(orderId, callback)` — returns product info after 200ms

Write a function `getUserOrderDetails(userId, callback)` that:
1. Gets the user (error-first)
2. Gets their orders
3. For each order, gets the product details (run these **in parallel** — don't wait for one to start the next)
4. Calls `callback(null, { user, orders: [ { order, product }, ... ] })`
5. Calls `callback(err)` on any failure, only once

```js
// Write your code here
```

### Exercise 3 — hard

Implement a `callbackWaterfall(tasks, finalCallback)` function that:
- Takes an array of functions `tasks`, each with signature `(previousResult, callback) => void`
  - The first task has signature `(callback) => void` (no previous result)
- Runs each task sequentially, passing the result of each to the next
- If any task calls back with an error, stop immediately and call `finalCallback(err)`
- If all tasks succeed, call `finalCallback(null, lastResult)`

Also implement `callbackParallel(tasks, finalCallback)` that:
- Runs all tasks concurrently (each has signature `(callback) => void`)
- Collects results in original order
- Calls `finalCallback(null, [result1, result2, ...])` when all complete
- Calls `finalCallback(err)` immediately if any task errors

Test both with simulated async operations using setTimeout.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Detail |
|---|---|
| Callback | A function passed as argument, called when work completes |
| Sync callback | Called immediately within the same tick (`forEach`, `map`, `sort`) |
| Async callback | Called later via event loop (`setTimeout`, `fs.readFile`) |
| Error-first convention | `(err, result) => {}` — always check `err` first |
| Always return after error | `return callback(err)` — prevents double-calling |
| Callback hell | Deep nesting from sequential dependent async operations |
| Fix #1 | Named functions instead of inline anonymous ones |
| Fix #2 | `util.promisify()` — convert to Promise |
| Fix #3 | Use `async` npm library for series/parallel/waterfall |
| Fix #4 — modern | Promises + async/await (topics 54, 56) |
| Parallel callbacks | Fire all at once, use a counter to detect when all done |
| Zalgo | Bug where callback is sometimes sync, sometimes async — always pick one |
| `process.nextTick` | Force async for cache-hit paths in Node.js |
| `this` in callback | Use arrow functions or `.bind(this)` to preserve context |
| Memory leak | Always remove event listeners you no longer need |
| Swallowed errors | Always log or propagate `err` — never silently ignore it |
| `util.promisify` | `const readFile = util.promisify(fs.readFile)` |
| `fs/promises` | Modern Node.js — all `fs` methods already return Promises |

---

## Connected topics

- **52 — Sync vs async + the event loop** — Callbacks are the mechanism that puts work in the macrotask queue; understanding when they fire requires the event loop model.
- **54 — Promises** — Promises were designed specifically to eliminate callback hell. The next topic.
- **56 — async/await** — Syntactic sugar over Promises; makes async code look sequential without callbacks at all.
- **58 — fetch API** — A real-world async operation; uses Promises (not callbacks) but understanding the callback history explains why Promises exist.
