# 21 — Callback Functions

## What is this?
A callback is a function you **pass as an argument to another function**, with the expectation that it will be called at some point — either immediately (synchronous callback) or later (asynchronous callback). Think of it like leaving your phone number at a restaurant — you don't wait at the counter, you go sit down, and they *call you back* when your table is ready. The restaurant is the higher-order function; your number is the callback.

## Why does it matter?
Callbacks are the original mechanism JavaScript uses to handle **asynchronous operations** — things that take time (network requests, file reads, timers, user events). Even after Promises and async/await were introduced, callbacks are still everywhere: every event listener, every `setTimeout`, every `forEach`, every `.map()` uses them. You cannot write real JavaScript without understanding callbacks deeply.

---

## Syntax

```js
// --- Synchronous callback (called immediately, inline) ---
function greetUser(name, formatFn) {   // formatFn is the callback
  const formatted = formatFn(name);    // called synchronously, right now
  console.log("Hello, " + formatted);
}

greetUser("alice", function (name) {   // anonymous function as callback
  return name.toUpperCase();
});
// Hello, ALICE

// --- Asynchronous callback (called later, by the runtime) ---
console.log("Before timer");

setTimeout(function () {               // this callback runs AFTER ~1 second
  console.log("Timer fired");
}, 1000);

console.log("After timer");
// Before timer
// After timer
// Timer fired    ← arrives last, even though it's in the middle of the code
```

---

## How it works — line by line

```js
function greetUser(name, formatFn) {
```
`greetUser` accepts two arguments. The second, `formatFn`, is expected to be a function. JavaScript doesn't enforce this — if you pass a non-function, you'll get a `TypeError` when it's called.

```js
  const formatted = formatFn(name);
```
`greetUser` calls the callback here, passing `name` as the argument. The callback decides what to do with it and returns a value. The HOF uses that return value.

```js
greetUser("alice", function (name) {
  return name.toUpperCase();
});
```
The second argument is a function literal — defined inline and passed directly. JavaScript evaluates this and hands the function object to `greetUser`.

For the async example:
```js
setTimeout(function () { ... }, 1000);
```
`setTimeout` is a browser/Node API. It accepts your callback and a delay. It immediately returns — it does NOT wait. The JS engine continues executing the rest of your code. The callback is placed into a queue and only called after the delay AND after the current call stack is empty. This is the **event loop**.

---

## Synchronous vs asynchronous callbacks

```js
// SYNCHRONOUS — callback runs immediately, in order, blocking
const numbers = [3, 1, 4, 1, 5];

numbers.forEach(function (n) {         // callback runs right now for each item
  console.log(n);
});
console.log("forEach done");
// 3, 1, 4, 1, 5, "forEach done"  — in order, no surprises

// ASYNCHRONOUS — callback runs later, after current execution finishes
console.log("Start");

setTimeout(function () {
  console.log("Async callback");       // runs AFTER "End"
}, 0);                                 // even 0ms delay is async!

console.log("End");
// Start
// End
// Async callback   ← comes last, even with 0ms delay
```

**The 0ms delay myth:** `setTimeout(..., 0)` does NOT run immediately — it schedules the callback at the end of the current event loop turn. This is a common interview question.

---

## Example 1 — basic: synchronous callbacks

```js
// A function that processes a list of cart items using a callback
function processCartItems(cartItems, processOneFn) {
  const results = [];

  for (const item of cartItems) {
    const result = processOneFn(item);   // callback called for each item
    results.push(result);
  }

  return results;
}

const cart = [
  { name: "Keyboard", price: 79.99 },
  { name: "Mouse",    price: 29.99 },
  { name: "Monitor",  price: 349.99 }
];

// Callback 1: format for display
const displayList = processCartItems(cart, function (item) {
  return item.name + " — $" + item.price.toFixed(2);
});

console.log(displayList);
// ["Keyboard — $79.99", "Mouse — $29.99", "Monitor — $349.99"]

// Callback 2: apply discount — same HOF, different callback
const discountedList = processCartItems(cart, function (item) {
  return { ...item, price: parseFloat((item.price * 0.9).toFixed(2)) };
});

console.log(discountedList);
// [{ name: 'Keyboard', price: 71.99 }, ...]
```

---

## Example 2 — real world: async callbacks with timers and events

```js
// Simulating a sequence of async operations using callbacks
function showLoadingSpinner() {
  console.log("⏳ Loading...");
}

function hideLoadingSpinner() {
  console.log("✅ Done!");
}

function fetchUserData(userId, onSuccess, onError) {
  showLoadingSpinner();

  // Simulate a network request with setTimeout
  setTimeout(function () {
    if (userId > 0) {
      const user = { id: userId, name: "Parsh", role: "admin" };
      onSuccess(user);          // call the success callback with the result
    } else {
      onError(new Error("Invalid user ID"));  // call the error callback
    }

    hideLoadingSpinner();
  }, 1500);
}

// Call fetchUserData and provide callbacks for both outcomes
fetchUserData(
  42,
  function (user) {
    console.log("User loaded:", user.name, "| Role:", user.role);
  },
  function (error) {
    console.error("Failed to load user:", error.message);
  }
);

console.log("Waiting for user data...");
// ⏳ Loading...
// Waiting for user data...
// (1.5 seconds later...)
// User loaded: Parsh | Role: admin
// ✅ Done!
```

This **error-first callback** pattern (or success/error pair) was the dominant async pattern in Node.js before Promises.

---

## Example 3 — real world: event-driven callbacks

```js
// Event listeners are callbacks — registered now, called later when the event fires
const submitButton = document.getElementById("submitBtn");
const emailInput   = document.getElementById("emailInput");

function validateEmail(email) {
  return email.includes("@") && email.includes(".");
}

function handleFormSubmit(event) {                 // this IS the callback
  event.preventDefault();                         // stop page reload

  const email = emailInput.value.trim();

  if (!validateEmail(email)) {
    console.error("Invalid email address");
    return;
  }

  console.log("Submitting form for:", email);
  // submitButton.disabled = true;
  // sendToServer(email, handleServerResponse);   // another callback!
}

// Register the callback — it will be called whenever 'click' fires
submitButton.addEventListener("click", handleFormSubmit);

// You can also remove it:
// submitButton.removeEventListener("click", handleFormSubmit);
// (This only works if you used a named function — anonymous functions can't be removed!)
```

---

## Callback hell — the problem that led to Promises

When async operations depend on each other, callbacks nest, creating the infamous "pyramid of doom":

```js
// ❌ Callback hell — each step depends on the previous
loginUser(credentials, function (user) {
  fetchUserProfile(user.id, function (profile) {
    fetchUserOrders(profile.id, function (orders) {
      fetchOrderDetails(orders[0].id, function (details) {
        renderDashboard(details, function (html) {
          displayPage(html, function () {
            console.log("Finally done");
            // error handling? Where do you even put it?
          });
        });
      });
    });
  });
});
```

**Problems with deeply nested callbacks:**
1. Hard to read — indentation grows rightward
2. Error handling is a nightmare — you have to check for errors at every level
3. Nearly impossible to add logic between steps
4. Debugging is painful — stack traces look strange

**The solution** is Promises (topic 54) and async/await (topic 56) — but they are built on top of callbacks, so you must understand callbacks first.

---

## Error-first callback pattern (Node.js convention)

In Node.js, the convention for async callbacks is: **first argument is the error, second is the result**.

```js
function readUserFromDb(userId, callback) {
  // Simulate DB query
  setTimeout(function () {
    if (userId === 0) {
      callback(new Error("User not found"), null);   // error first, result null
    } else {
      callback(null, { id: userId, name: "Parsh" }); // null error, result second
    }
  }, 500);
}

// Usage
readUserFromDb(1, function (error, user) {
  if (error) {
    console.error("DB error:", error.message);
    return;                                           // early return on error
  }
  console.log("Got user:", user.name);
});

readUserFromDb(0, function (error, user) {
  if (error) {
    console.error("DB error:", error.message);        // "DB error: User not found"
    return;
  }
  console.log("Got user:", user.name);               // never reached
});
```

---

## Tricky things you'll encounter in the real world

### 1 — Forgetting that async callbacks run AFTER the rest of your code

```js
let userData = null;

setTimeout(function () {
  userData = { name: "Parsh" };   // set inside callback — runs later
}, 500);

console.log(userData);            // ❌ null — the callback hasn't run yet!

// Fix: always work with async data INSIDE the callback (or use Promises/async-await)
setTimeout(function () {
  userData = { name: "Parsh" };
  console.log(userData);          // ✅ { name: "Parsh" } — guaranteed to be set
}, 500);
```

---

### 2 — Losing `this` inside a callback

```js
const userDashboard = {
  userName: "Parsh",
  orders: [101, 102, 103],

  printOrders() {
    // ❌ Regular function callback — `this` is lost
    this.orders.forEach(function (orderId) {
      console.log(this.userName + "'s order: " + orderId); // TypeError: this is undefined
    });
  },

  printOrdersFixed() {
    // ✅ Arrow function — `this` is inherited from printOrdersFixed
    this.orders.forEach((orderId) => {
      console.log(this.userName + "'s order: " + orderId); // Parsh's order: 101, etc.
    });
  }
};

userDashboard.printOrdersFixed();
```

---

### 3 — Named callbacks are better than anonymous for debugging

```js
// ❌ Anonymous — stack trace shows "anonymous" everywhere
document.getElementById("btn").addEventListener("click", function () {
  doSomethingComplex();
});

// ✅ Named — stack trace shows "handleButtonClick" → much easier to debug
document.getElementById("btn").addEventListener("click", function handleButtonClick() {
  doSomethingComplex();
});

// ✅ Even better — define separately, pass reference
function handleButtonClick() {
  doSomethingComplex();
}
document.getElementById("btn").addEventListener("click", handleButtonClick);
```

---

### 4 — Calling the callback multiple times by accident

```js
// ❌ BAD — callback could be called twice if you're not careful
function fetchData(url, callback) {
  if (!url) {
    callback(new Error("No URL provided")); // calls callback
    // missing return — falls through to the setTimeout!
  }

  setTimeout(function () {
    callback(null, { data: "..." }); // called again!
  }, 500);
}

// ✅ GOOD — use early return to guarantee callback is called exactly once
function fetchData(url, callback) {
  if (!url) {
    return callback(new Error("No URL provided")); // return stops execution
  }

  setTimeout(function () {
    callback(null, { data: "..." });
  }, 500);
}
```

---

### 5 — Callback vs Promise confusion: you can't `return` out of an async callback

```js
function getUsername(userId) {
  // ❌ This return does NOTHING — it returns from the callback, not from getUsername
  setTimeout(function () {
    return "Parsh";           // returns into the void — setTimeout ignores return values
  }, 500);
  // getUsername itself returns undefined
}

const name = getUsername(1);
console.log(name); // undefined — always!

// To "return" async data, you must either:
// 1. Use a callback parameter
// 2. Return a Promise
// 3. Use async/await
```

---

### 6 — Infinite callback loops

```js
// ❌ This causes an infinite loop — the callback keeps re-scheduling itself
function pollServer() {
  setTimeout(function () {
    fetchLatestData();
    pollServer();   // schedules itself again immediately after running — INFINITE
  }, 1000);
}

// ✅ Fine if that's actually what you want (polling), but make sure there's an exit condition
let polling = true;

function pollServer() {
  if (!polling) return;            // exit condition

  setTimeout(function () {
    fetchLatestData();
    pollServer();
  }, 1000);
}

// Call polling = false when done
```

---

## Common mistakes

### ❌ Mistake 1 — Invoking the callback instead of passing it

```js
function doAfterDelay(callback, ms) {
  setTimeout(callback, ms);
}

// WRONG — greetUser() runs immediately, undefined is passed to setTimeout
doAfterDelay(greetUser(), 1000);

// RIGHT — pass the reference
doAfterDelay(greetUser, 1000);

// RIGHT — if you need to pass arguments, wrap in arrow function
doAfterDelay(() => greetUser("Alice"), 1000);
```

---

### ❌ Mistake 2 — Not checking if the callback is actually a function

```js
// WRONG — crashes if someone forgets to pass a callback
function processData(data, callback) {
  const result = transform(data);
  callback(result);              // ❌ TypeError if callback is undefined
}

// RIGHT — guard against missing callback
function processData(data, callback) {
  const result = transform(data);
  if (typeof callback === "function") {
    callback(result);            // ✅ only call if it's actually a function
  }
}
```

---

### ❌ Mistake 3 — Forgetting to handle the error argument in error-first callbacks

```js
// WRONG — ignoring the error silently causes bugs that are impossible to trace
readUserFromDb(userId, function (error, user) {
  console.log(user.name);   // ❌ crashes if error occurred and user is null
});

// RIGHT — always check the error first
readUserFromDb(userId, function (error, user) {
  if (error) {
    console.error("Could not load user:", error.message);
    return;
  }
  console.log(user.name);   // ✅ safe — only reached if no error
});
```

---

## Frequently asked questions

**Q: Is every function passed as an argument a callback?**  
Yes, by definition. But "callback" usually implies the receiving function decides *when* to call it — not that you call it yourself at a specific time. The naming is about intent.

**Q: Are callbacks and higher-order functions the same thing?**  
Related but different. A callback is the function *being passed*. A higher-order function is the function *doing the receiving or returning*. `forEach` is the HOF; the function you pass to `forEach` is the callback.

**Q: Why do callbacks cause "callback hell"?**  
Because async operations often depend on each other. You need the result of step 1 before you can start step 2, so step 2's code goes inside step 1's callback — and so on. Each level of dependency adds another level of nesting.

**Q: Are callbacks still used in modern JavaScript?**  
Yes, constantly. Every event listener is a callback. Built-in array methods use callbacks. The difference is that for async *chaining*, Promises and async/await have replaced the error-first callback pattern. But callbacks themselves are not going away.

**Q: What is a "fire-and-forget" callback?**  
An async callback where you don't care about the return value or result — you just trigger something (e.g., logging an event to an analytics server). You pass a callback that does the work, and you don't wait for a response.

**Q: How does the event loop relate to async callbacks?**  
When an async operation completes (timer fires, network responds), its callback is placed in the **callback queue**. The **event loop** checks if the call stack is empty; if it is, it takes the next callback from the queue and runs it. This is why async callbacks always run *after* synchronous code — the stack must be empty first.

---

## Practice exercises

### Exercise 1 — easy

Write a function `transformList(items, transformFn)` that takes an array and a callback, applies the callback to each element, and returns a new array of results. (No `.map()` allowed — write the loop yourself.)

Test it three times with three different callbacks:
1. Double each number: `[2, 5, 8]` → `[4, 10, 16]`
2. Shorten each string to 5 characters with `"..."` if longer: `["JavaScript", "Go", "Python"]` → `["JavaS...", "Go", "Pytho..."]`
3. Convert each item to a `"$XX.XX"` price string: `[9.9, 4, 19.99]` → `["$9.90", "$4.00", "$19.99"]`

```js
// Write your code here
```

---

### Exercise 2 — medium

Implement the **error-first callback pattern** for an async operation.

Write a function `simulateLogin(username, password, callback)` that:
- Uses `setTimeout` to simulate a 1-second server delay
- If `username === ""` — calls `callback(new Error("Username is required"), null)`
- If `password.length < 6` — calls `callback(new Error("Password too short"), null)`
- Otherwise — calls `callback(null, { username, token: "tok_" + Date.now() })`

Call `simulateLogin` three times to demonstrate all three outcomes. In each callback, always check the error first.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a **sequential async queue** using callbacks — no Promises, no async/await.

Write a function `runSequential(tasks, onAllDone)` where:
- `tasks` is an array of functions. Each task function accepts a single `done` callback and must call `done(result)` when complete.
- `runSequential` runs each task **one after the other** (not in parallel) — task 2 only starts after task 1's `done` is called.
- After all tasks complete, call `onAllDone(results)` where `results` is an array of each task's result in order.

Create 3 tasks that simulate async work with `setTimeout`:
- Task 1 (500ms): returns `"user loaded"`
- Task 2 (300ms): returns `"permissions checked"`
- Task 3 (700ms): returns `"dashboard rendered"`

Log the final `results` array from `onAllDone`.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

**Definition:**
> A callback is a function passed as an argument to another function, which the receiving function calls at the appropriate time.

**Two types:**

| Type | When called | Examples |
|---|---|---|
| Synchronous | Immediately, inline | `forEach`, `map`, `filter`, `sort` |
| Asynchronous | Later, after an event or delay | `setTimeout`, `addEventListener`, `fs.readFile` |

**Error-first pattern (Node.js):**
```js
function doAsync(input, callback) {
  if (badInput) return callback(new Error("Bad input"), null);
  // ... do work ...
  callback(null, result);
}

doAsync(input, function (error, result) {
  if (error) { /* handle */ return; }
  // use result
});
```

**Key rules:**
- Pass function references, not calls: `fn` not `fn()`
- Always handle errors in async callbacks
- Use arrow functions in method callbacks to preserve `this`
- Name your callbacks — anonymous functions are hard to debug
- A callback should be called **exactly once** — guard with early returns
- You cannot `return` data out of an async callback — work with the data inside it

**Common callback consumers:**
```js
setTimeout(fn, ms)
setInterval(fn, ms)
element.addEventListener(event, fn)
array.forEach(fn)
array.map(fn)
array.filter(fn)
array.sort(fn)
fs.readFile(path, fn)         // Node.js
```

---

## Connected topics

- **20 — Higher-order functions** — every HOF that accepts a function is using the callback pattern; the two concepts are two sides of the same coin
- **22 — IIFE** — immediately invoked function expressions are closely related; they create scope instantly and are often used to wrap callback-heavy code
- **52 — The event loop** — understanding *why* async callbacks run after synchronous code requires understanding the call stack, callback queue, and event loop
- **54 — Promises** — Promises were invented specifically to solve callback hell; you must understand callbacks to appreciate why Promises are better for chaining async steps
