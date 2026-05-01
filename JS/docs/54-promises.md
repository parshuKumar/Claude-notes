# 54 — Promises

## What is this?

A Promise is an **object that represents the eventual result of an asynchronous operation** — a placeholder for a value that doesn't exist yet but will at some point in the future. Think of it like an order ticket at a restaurant: you place your order and receive a ticket (the Promise). You don't have the food yet, but you have a *guarantee* that you'll either receive the food (resolved) or be told it's unavailable (rejected). While waiting, you can describe what to do when the food arrives (`.then()`) or what to do if there's a problem (`.catch()`). The critical difference from callbacks: the ticket can be passed around, stored, combined with other tickets, and acted on at any time — even after the food has already arrived.

## Why does it matter?

Promises are the backbone of all modern JavaScript async code. Every `fetch()` call returns a Promise. Every `async/await` function works with Promises under the hood. Mongoose, Prisma, Sequelize, Axios, the entire Node.js `fs/promises` API — all Promises. Understanding Promises completely — their three states, the microtask queue, chaining, error propagation, and combinators — means you will never be confused by async code again. This is arguably the most important single topic in the entire async section.

---

## The three states of a Promise

A Promise is always in exactly one of three states:

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│   PENDING ──────────────────► FULFILLED (resolved)             │
│      │         resolve(value)       │                           │
│      │                              │ .then(callback) fires     │
│      │                              │                           │
│      │         reject(error)        │                           │
│      └─────────────────────► REJECTED                          │
│                                     │                           │
│                                     │ .catch(callback) fires    │
│                                                                 │
│  Rules:                                                         │
│  • State can only change ONCE — pending → fulfilled OR rejected │
│  • Once settled (fulfilled or rejected), state is IMMUTABLE     │
│  • resolve/reject calls after the first one are silently ignored│
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Syntax — creating a Promise

```js
const myPromise = new Promise((resolve, reject) => {
  // executor function — runs SYNCHRONOUSLY when the Promise is created

  // Do your async work here:
  setTimeout(() => {
    const success = true;

    if (success) {
      resolve("operation completed!");  // fulfils the Promise with this value
    } else {
      reject(new Error("operation failed"));  // rejects with this error
    }
  }, 1000);
});

// Consuming the Promise:
myPromise
  .then(value => console.log("Success:", value))   // runs if resolved
  .catch(err  => console.log("Error:", err.message)) // runs if rejected
  .finally(()  => console.log("Always runs"));       // runs either way
```

---

## How it works — line by line

```js
new Promise((resolve, reject) => { ... })
```
The `Promise` constructor takes one argument: an **executor function**. This executor runs *synchronously and immediately* when the Promise is created. The executor receives two functions — `resolve` and `reject` — that you call to settle the Promise.

```js
resolve("operation completed!")
```
Calling `resolve` transitions the Promise from `pending` → `fulfilled` and stores `"operation completed!"` as the resolution value. All `.then()` callbacks registered on this Promise are scheduled as microtasks.

```js
reject(new Error("operation failed"))
```
Calling `reject` transitions the Promise from `pending` → `rejected`. All `.catch()` callbacks are scheduled as microtasks. You should always reject with an `Error` object (not a string) — it preserves the stack trace.

```js
.then(value => ...)
```
Registers a callback to run when the Promise fulfils. Returns a **new Promise** — this is how chaining works.

```js
.catch(err => ...)
```
Registers a callback to run when the Promise rejects. Shorthand for `.then(undefined, onRejected)`. Also returns a new Promise.

```js
.finally(() => ...)
```
Runs whether the Promise fulfils or rejects. Does **not** receive any value — use it for cleanup (close connections, hide spinners, etc.).

---

## The executor runs synchronously

```js
console.log("1 — before new Promise");

const p = new Promise((resolve, reject) => {
  console.log("2 — executor runs NOW (synchronously)");
  resolve("done");
});

console.log("3 — after new Promise");

p.then(val => console.log("4 — then:", val));

console.log("5 — end of sync code");

// Output:
// 1 — before new Promise
// 2 — executor runs NOW (synchronously)
// 3 — after new Promise
// 5 — end of sync code
// 4 — then: done
//
// The .then() callback runs as a MICROTASK — after all sync code
```

---

## Promise chaining — the core power

Every `.then()` returns a new Promise. The value you `return` from a `.then()` callback becomes the resolved value of that new Promise — which feeds into the next `.then()`:

```js
fetch("/api/user/1")
  .then(response => response.json())        // returns a Promise (response.json() is async)
  .then(user     => fetch(`/api/orders/${user.id}`))  // returns another Promise
  .then(response => response.json())
  .then(orders   => {
    console.log("Orders:", orders);
    return orders.length;                   // returns a plain value
  })
  .then(count => console.log("Count:", count))  // receives the plain value
  .catch(err  => console.error("Something failed:", err));
```

**The chain rule:**
- Return a plain value from `.then()` → next `.then()` receives that value
- Return a Promise from `.then()` → next `.then()` waits for that Promise to resolve
- Throw an error in `.then()` → jumps to the nearest `.catch()`
- Return from `.catch()` → chain continues with `.then()` (error is handled)

---

## Example 1 — basic: simulating async operations

```js
function delay(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

function fetchUserFromDB(userId) {
  return new Promise((resolve, reject) => {
    // Simulate DB query taking 300ms
    setTimeout(() => {
      const users = {
        1: { id: 1, name: "Alice", email: "alice@example.com", role: "admin" },
        2: { id: 2, name: "Bob",   email: "bob@example.com",   role: "user"  },
      };
      const user = users[userId];
      if (user) {
        resolve(user);
      } else {
        reject(new Error(`User ${userId} not found`));
      }
    }, 300);
  });
}

function fetchOrdersFromDB(userId) {
  return new Promise((resolve, reject) => {
    setTimeout(() => {
      const orders = {
        1: [{ id: 101, item: "Laptop", price: 999 }, { id: 102, item: "Mouse", price: 29 }],
        2: [{ id: 103, item: "Keyboard", price: 79 }],
      };
      resolve(orders[userId] ?? []);
    }, 200);
  });
}

// Clean sequential chain — reads top-to-bottom, no nesting
fetchUserFromDB(1)
  .then(user => {
    console.log("User:", user.name);
    return fetchOrdersFromDB(user.id);   // returns a Promise
  })
  .then(orders => {
    const total = orders.reduce((sum, o) => sum + o.price, 0);
    console.log("Orders:", orders.length, "Total: $" + total);
  })
  .catch(err => console.error("Failed:", err.message));
```

---

## Example 2 — error propagation through the chain

```js
function step1() {
  return Promise.resolve("step1 result");
}

function step2(prev) {
  console.log("step2 received:", prev);
  throw new Error("step2 exploded");   // throwing inside .then() rejects the chain
}

function step3(prev) {
  console.log("step3 received:", prev);  // NEVER runs — skipped
  return "step3 result";
}

function recover(err) {
  console.log("catch received:", err.message);
  return "recovered value";    // returning from .catch() resumes the chain
}

function step4(prev) {
  console.log("step4 received:", prev);  // receives "recovered value"
}

step1()
  .then(step2)        // throws — rest of chain skips to .catch()
  .then(step3)        // SKIPPED
  .catch(recover)     // catches the error, returns "recovered value"
  .then(step4)        // resumes — receives "recovered value"
  .catch(err => console.error("Unhandled:", err));

// Output:
// step2 received: step1 result
// catch received: step2 exploded
// step4 received: recovered value
```

**Key rule:** When an error occurs in a `.then()`, every subsequent `.then()` is **skipped** until the nearest `.catch()`. After `.catch()` handles the error and returns normally, the chain **resumes** with the next `.then()`.

---

## Example 3 — real world: API request pipeline

```js
class ApiClient {
  constructor(baseUrl) {
    this.baseUrl = baseUrl;
  }

  #buildUrl(endpoint) {
    return `${this.baseUrl}${endpoint}`;
  }

  #handleResponse(response) {
    if (!response.ok) {
      // HTTP errors (4xx, 5xx) don't auto-reject fetch — we must reject manually
      return response.json().then(body => {
        throw new Error(`HTTP ${response.status}: ${body.message ?? response.statusText}`);
      });
    }
    return response.json();
  }

  get(endpoint) {
    return fetch(this.#buildUrl(endpoint))
      .then(res => this.#handleResponse(res));
  }

  post(endpoint, body) {
    return fetch(this.#buildUrl(endpoint), {
      method:  "POST",
      headers: { "Content-Type": "application/json" },
      body:    JSON.stringify(body),
    }).then(res => this.#handleResponse(res));
  }
}

const api = new ApiClient("https://api.example.com");

// Chained: login → get user data → get user's permissions
api.post("/auth/login", { email: "alice@example.com", password: "secret" })
  .then(session => {
    console.log("Logged in, token:", session.token);
    return api.get(`/users/${session.userId}`);   // use the token implicitly via cookies/headers
  })
  .then(user => {
    console.log("User loaded:", user.name);
    return api.get(`/users/${user.id}/permissions`);
  })
  .then(permissions => {
    console.log("Permissions:", permissions);
  })
  .catch(err => {
    console.error("Auth flow failed:", err.message);
    redirectToLogin();
  });
```

---

## Static Promise methods

### `Promise.resolve(value)` and `Promise.reject(error)`

Instantly create already-settled Promises:

```js
// Already fulfilled — .then() fires as microtask
Promise.resolve(42).then(v => console.log(v));  // 42

// Already rejected — .catch() fires as microtask
Promise.reject(new Error("oops")).catch(e => console.error(e.message));  // "oops"

// Useful to normalise sync values into a Promise pipeline:
function getData(useCache) {
  if (useCache) return Promise.resolve(cachedData);   // wrap sync in Promise
  return fetch("/api/data").then(r => r.json());       // async
}
```

### `Promise.resolve()` with a thenable

```js
// If you pass a thenable (object with .then method) to Promise.resolve(),
// it adopts its state:
const thenable = {
  then(resolve) {
    resolve("from thenable");
  }
};
Promise.resolve(thenable).then(v => console.log(v));  // "from thenable"
```

---

## `.then(onFulfilled, onRejected)` — two-argument form

`.then()` actually accepts two callbacks — the second handles rejection:

```js
somePromise.then(
  value => console.log("fulfilled:", value),
  err   => console.log("rejected:", err.message)
);
```

However, this is almost always worse than `.catch()` because:

```js
Promise.resolve()
  .then(
    () => { throw new Error("thrown in onFulfilled"); },
    err => console.log("caught:", err.message)   // does NOT catch the throw above
  );
// Uncaught! The second argument of .then() doesn't catch errors from the FIRST argument

// .catch() after .then() catches everything:
Promise.resolve()
  .then(() => { throw new Error("thrown"); })
  .catch(err => console.log("caught:", err.message));  // works correctly
```

---

## `.finally()` in depth

```js
function fetchWithSpinner(url) {
  showSpinner();   // start loading UI
  return fetch(url)
    .then(res => res.json())
    .finally(() => {
      hideSpinner();   // runs whether fetch succeeds or fails
      // .finally() does NOT receive the value or error
      // It passes through the original value/error to the next handler
    });
}

fetchWithSpinner("/api/data")
  .then(data => console.log(data))     // receives original resolved value
  .catch(err  => console.error(err));   // receives original rejection reason

// .finally() return value matters:
Promise.resolve("original")
  .finally(() => "ignored")             // plain value returned from finally is ignored
  .then(v => console.log(v));           // "original" — finally didn't change it

Promise.resolve("original")
  .finally(() => Promise.reject(new Error("finally error")))  // rejection from finally DOES propagate
  .then(v => console.log(v))            // skipped
  .catch(e => console.error(e.message)); // "finally error"
```

---

## Promise chaining vs nesting — don't nest Promises

```js
// WRONG — nesting Promises recreates callback hell
fetch("/api/user/1")
  .then(res => {
    return res.json().then(user => {        // ← nested .then() inside .then()
      return fetch(`/api/orders/${user.id}`)
        .then(res2 => {
          return res2.json().then(orders => { // ← deeper nesting
            console.log(user, orders);
          });
        });
    });
  });

// RIGHT — flat chaining
let savedUser;
fetch("/api/user/1")
  .then(res   => res.json())
  .then(user  => { savedUser = user; return fetch(`/api/orders/${user.id}`); })
  .then(res   => res.json())
  .then(orders => console.log(savedUser, orders));

// EVEN BETTER — async/await (topic 56):
async function loadUserAndOrders() {
  const user   = await fetch("/api/user/1").then(r => r.json());
  const orders = await fetch(`/api/orders/${user.id}`).then(r => r.json());
  console.log(user, orders);
}
```

---

## Creating Promises from callback-based functions

```js
// Generic promisify (you saw this in topic 53)
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

// Or use Node.js built-in:
const { promisify } = require("util");
const fs            = require("fs");

const readFile  = promisify(fs.readFile);
const writeFile = promisify(fs.writeFile);

// Now usable in a chain:
readFile("input.txt", "utf8")
  .then(content => content.toUpperCase())
  .then(upper   => writeFile("output.txt", upper))
  .then(() => console.log("File processed!"))
  .catch(err => console.error("Error:", err.message));
```

---

## Unhandled Promise rejections — a critical production issue

```js
// In Node.js — unhandled rejections can crash your process (Node 15+)
const p = Promise.reject(new Error("nobody is listening"));
// If nothing calls .catch() on this, Node throws:
// UnhandledPromiseRejection: Error: nobody is listening

// Global handler in Node.js:
process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled rejection:", reason);
  // In production: log to monitoring service, then exit
  process.exit(1);
});

// Global handler in browser:
window.addEventListener("unhandledrejection", (event) => {
  console.error("Unhandled rejection:", event.reason);
  event.preventDefault();   // prevents default browser error logging
});
```

**Rule:** Every Promise chain that can reject must end with a `.catch()`. If you fire a Promise and don't chain anything, attach a `.catch()` anyway:

```js
// Even fire-and-forget should handle errors
sendAnalyticsEvent(data).catch(err => console.warn("Analytics failed:", err.message));
```

---

## Tricky things

### 1. `.then()` always returns a NEW Promise

```js
const p1 = Promise.resolve(1);
const p2 = p1.then(v => v + 1);
const p3 = p2.then(v => v + 1);

console.log(p1 === p2);  // false — different Promise objects
console.log(p2 === p3);  // false

Promise.all([p1, p2, p3]).then(values => console.log(values));  // [1, 2, 3]
```

### 2. `return` inside `.then()` matters

```js
// WRONG — not returning the inner Promise, chain doesn't wait for it
fetch("/api/data")
  .then(res => {
    res.json();         // ← no return! This Promise is ignored
  })
  .then(data => console.log(data));  // undefined — chain didn't wait for res.json()

// RIGHT
fetch("/api/data")
  .then(res => res.json())   // ← returned — chain waits for it
  .then(data => console.log(data));
```

### 3. Promise executor errors are caught automatically

```js
const p = new Promise((resolve, reject) => {
  throw new Error("executor threw");   // automatically converted to rejection
});

p.catch(err => console.log(err.message));  // "executor threw"

// Same as:
new Promise((resolve, reject) => {
  reject(new Error("executor threw"));
});
```

### 4. Calling resolve/reject multiple times — only first call counts

```js
const p = new Promise((resolve, reject) => {
  resolve("first");
  resolve("second");   // silently ignored
  reject(new Error("error"));  // silently ignored
});

p.then(v => console.log(v));  // "first" — only the first resolve counts
```

### 5. `async/await` errors vs `.catch()` — understanding the relationship

```js
// These two are equivalent:
somePromise
  .then(v => doSomething(v))
  .catch(e => handleError(e));

// same as:
async function run() {
  try {
    const v = await somePromise;
    doSomething(v);
  } catch(e) {
    handleError(e);
  }
}
```

### 6. `.then()` on an already-resolved Promise still runs asynchronously

```js
const p = Promise.resolve("already done");

p.then(v => console.log("A:", v));  // scheduled as microtask
console.log("B: sync");

// Output:
// B: sync
// A: already done
// The .then() ALWAYS runs async (as microtask), even if Promise is already resolved
```

### 7. Forgetting to `return` a rejected catch — swallowing errors

```js
somePromise
  .then(processData)
  .catch(err => {
    console.error("Caught:", err.message);
    // No return — catch resolves with undefined, chain continues as fulfilled
  })
  .then(() => console.log("This still runs after the error!"))  // runs!
  .catch(() => console.log("This will NOT run"));  // won't run

// If you want to re-throw:
.catch(err => {
  console.error("Caught:", err.message);
  throw err;  // re-throw to keep the chain in rejected state
})
```

---

## Common mistakes

### Mistake 1 — Not returning the inner Promise (broken chain)

```js
// WRONG — the chain doesn't wait for saveToDatabase
getUserData(userId)
  .then(user => {
    saveToDatabase(user);   // ← not returned — Promise ignored, chain moves on immediately
  })
  .then(() => console.log("saved!"));  // fires before saveToDatabase completes

// RIGHT
getUserData(userId)
  .then(user => saveToDatabase(user))  // returned — chain waits
  .then(() => console.log("saved!"));
```

### Mistake 2 — Nesting `.then()` instead of chaining

```js
// WRONG — recreates callback hell
getUser(id).then(user => {
  getOrders(user.id).then(orders => {     // nested
    processOrders(orders).then(result => { // deeper nest
      console.log(result);
    });
  });
});

// RIGHT — flat chain
getUser(id)
  .then(user   => getOrders(user.id))
  .then(orders => processOrders(orders))
  .then(result => console.log(result))
  .catch(err   => console.error(err));
```

### Mistake 3 — Forgetting `.catch()` at the end

```js
// WRONG — unhandled rejection will crash Node.js 15+
fetchData()
  .then(processData)
  .then(saveData);
  // No .catch() — if anything throws, it's an unhandled rejection

// RIGHT
fetchData()
  .then(processData)
  .then(saveData)
  .catch(err => {
    logger.error("Pipeline failed:", err);
    // handle or re-throw
  });
```

---

## Practice exercises

### Exercise 1 — easy

Write a function `fetchWithTimeout(url, timeoutMs)` that returns a Promise:
- If `fetch(url)` completes within `timeoutMs`, resolve with the parsed JSON
- If `timeoutMs` elapses first, reject with `new Error("Request timed out")`
- Use `Promise.race()` ... wait, that's topic 55. Instead: create a timeout Promise using `setTimeout` and `reject`, and manually race them with a flag variable.

Also write a `retry(fn, times)` function that calls `fn()` (a function returning a Promise) up to `times` attempts, resolving on the first success, and rejecting with the final error after all attempts are exhausted.

```js
// Write your code here
```

### Exercise 2 — medium

Build a Promise-based pipeline for processing user registration:

```
validateInput(data)
  → checkEmailNotTaken(email)
  → hashPassword(password)
  → createUserInDB(userData)
  → sendWelcomeEmail(user)
  → return { user, message: "Registration successful" }
```

Each step is an async function returning a Promise. Simulate them with `setTimeout`. Include:
- `validateInput` rejects if `email` or `password` is missing
- `checkEmailNotTaken` rejects if email is `"taken@example.com"`
- Each other step has a 10% random failure chance (simulate with `Math.random()`)

Write the chain so that a single `.catch()` at the end handles all possible failures with a descriptive error message.

```js
// Write your code here
```

### Exercise 3 — hard

Implement `Promise.allSettled` from scratch (as `myAllSettled`) without using the built-in. It should:
- Accept an array of Promises
- Return a Promise that **always resolves** (never rejects)
- Resolves with an array of result objects: `{ status: "fulfilled", value }` or `{ status: "rejected", reason }`
- Results must be in the original order
- Works with empty arrays

Then implement `myRace(promises)` which resolves or rejects with the first Promise that settles.

Test `myAllSettled` with a mix of resolved and rejected Promises (including some that resolve after delays). Test `myRace` with Promises that resolve at different times.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Detail |
|---|---|
| Create Promise | `new Promise((resolve, reject) => { ... })` |
| Resolve | `resolve(value)` — fulfils, value passed to `.then()` |
| Reject | `reject(error)` — rejects, error passed to `.catch()` |
| States | `pending` → `fulfilled` or `rejected` (one-way, immutable after settling) |
| `.then(fn)` | Runs `fn` on fulfil; returns NEW Promise with `fn`'s return value |
| `.catch(fn)` | Runs `fn` on reject; returns new Promise (chain can resume) |
| `.finally(fn)` | Runs `fn` always; passes original value/error through unchanged |
| Chain rule | Return plain value → next `.then()` gets it; Return Promise → chain waits |
| Error propagation | Error in `.then()` skips all `.then()` until nearest `.catch()` |
| Executor is sync | Runs immediately when `new Promise(...)` is called |
| `.then()` is async | Always runs as microtask — even if Promise already resolved |
| `Promise.resolve(v)` | Already-fulfilled Promise wrapping `v` |
| `Promise.reject(e)` | Already-rejected Promise wrapping `e` |
| Don't nest `.then()` | Chain flat instead — nesting recreates callback hell |
| Always return | Forgetting `return` inside `.then()` breaks the chain |
| Always `.catch()` | Unhandled rejections crash Node.js 15+ |
| Unhandled rejection | `process.on("unhandledRejection", ...)` in Node.js |

---

## Connected topics

- **53 — Callbacks** — Promises replace callback hell; understanding both shows you exactly what problem Promises solve.
- **55 — Promise combinators** — `Promise.all`, `.race`, `.allSettled`, `.any` — running multiple Promises together.
- **56 — async/await** — Syntactic sugar over Promises; every `await` is a `.then()` internally.
- **57 — Error handling** — `try/catch` with async/await vs `.catch()` in Promise chains — when to use each.
