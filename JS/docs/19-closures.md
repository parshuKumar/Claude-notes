# 19 — Closures

## What is this?
A closure is what happens when a function **remembers the variables from the scope where it was created**, even after that outer scope has finished running. Think of it like a backpack — when a function is born inside another function, it packs up the surrounding variables into that backpack and carries them wherever it goes, even long after the outer function is done executing.

Every function in JavaScript *is* a closure — but the concept only becomes visible and useful when a function outlives the scope it was created in.

## Why does it matter?
Closures are one of the most powerful features in JavaScript. They are the engine behind **private variables**, **stateful functions**, **factory functions**, **memoization**, **event handlers**, **partial application**, and the entire **module pattern**. If you don't understand closures, you will not understand why half of the JS code you read behaves the way it does.

---

## Syntax

```js
function outer() {
  let count = 0;                  // variable in outer's scope

  function inner() {              // inner is born inside outer
    count++;                      // inner "closes over" count
    console.log(count);
  }

  return inner;                   // return the function — NOT a call
}

const increment = outer();        // outer() runs, returns inner, outer scope "closes"
increment();                      // 1  — count is still alive inside the closure
increment();                      // 2
increment();                      // 3
```

`count` is NOT destroyed when `outer()` finishes. Because `inner` (now stored in `increment`) still holds a reference to it, JS keeps it alive in memory.

---

## How it works — line by line

```js
function outer() {
```
A regular function. When called, it creates a new local scope.

```js
  let count = 0;
```
`count` lives in `outer`'s scope. Normally it would be destroyed when `outer` returns.

```js
  function inner() {
    count++;
    console.log(count);
  }
```
`inner` is defined inside `outer`. At the moment `inner` is created, JS attaches a reference to `outer`'s current scope (the "closure environment"). `inner` now *owns* a live link to `count`.

```js
  return inner;
```
We return the function *object* itself — not the result of calling it. This is what makes the closure escape `outer`'s scope.

```js
const increment = outer();
```
`outer()` runs and returns `inner`. Now `increment` holds the `inner` function. `outer` has finished running — but `count` is **not garbage collected** because `increment` still references it.

```js
increment(); // 1
increment(); // 2
```
Each call reads and modifies the *same* `count` variable that was created when `outer()` ran. The closure acts like a private memory cell attached to `increment`.

---

## Example 1 — basic: counter factory

```js
function createCounter(startValue) {
  // startValue lives in createCounter's scope
  let currentCount = startValue;    // private to this closure

  function increment() {
    currentCount++;                 // reads and writes the closed-over variable
    return currentCount;
  }

  function decrement() {
    currentCount--;
    return currentCount;
  }

  function reset() {
    currentCount = startValue;      // startValue is also closed over
    return currentCount;
  }

  function getCount() {
    return currentCount;            // read-only access
  }

  // Return an object of functions — the only way to interact with currentCount
  return { increment, decrement, reset, getCount };
}

const pageViews = createCounter(0);
console.log(pageViews.increment()); // 1
console.log(pageViews.increment()); // 2
console.log(pageViews.increment()); // 3
console.log(pageViews.decrement()); // 2
console.log(pageViews.reset());     // 0
console.log(pageViews.getCount());  // 0

// currentCount is COMPLETELY inaccessible from outside
// console.log(pageViews.currentCount); // undefined — not exposed
```

This is the **closure-based module pattern**: expose only what you want, hide everything else.

---

## Example 2 — real world: API request wrapper with auth token

```js
// Imagine this token is fetched once at login and stored privately
function createApiClient(authToken) {
  const baseUrl = "https://api.example.com";  // also closed over

  function get(endpoint) {
    const url = baseUrl + endpoint;
    console.log(`GET ${url} | Token: ${authToken}`);
    // In real code: return fetch(url, { headers: { Authorization: authToken } })
  }

  function post(endpoint, data) {
    const url = baseUrl + endpoint;
    console.log(`POST ${url} | Data:`, data, `| Token: ${authToken}`);
    // In real code: return fetch(url, { method: "POST", body: JSON.stringify(data), ... })
  }

  function refreshToken(newToken) {
    authToken = newToken;              // closes over authToken — can even mutate it
    console.log("Token refreshed");
  }

  return { get, post, refreshToken };
}

// Token is captured once — every method shares the same closed-over reference
const apiClient = createApiClient("Bearer eyJhbGciOiJIUzI1NiJ9...");

apiClient.get("/users/profile");
// GET https://api.example.com/users/profile | Token: Bearer eyJhbGciOiJIUzI1NiJ9...

apiClient.post("/orders", { productId: 42, quantity: 2 });
// POST https://api.example.com/orders | Data: { productId: 42, quantity: 2 } | Token: ...

apiClient.refreshToken("Bearer newToken456");
apiClient.get("/dashboard");
// GET https://api.example.com/dashboard | Token: Bearer newToken456  ← updated!
```

Key point: `authToken` is truly private. Nothing outside `createApiClient` can read or modify it except through the returned methods.

---

## Example 3 — real world: partial application (pre-filling arguments)

```js
// A general discount calculator
function createDiscountApplier(discountPercent) {
  // discountPercent is closed over — baked in for this particular applier
  return function applyDiscount(originalPrice) {
    const multiplier = 1 - discountPercent / 100;
    const discountedPrice = originalPrice * multiplier;
    return parseFloat(discountedPrice.toFixed(2));
  };
}

// Create specialised functions by closing over different discount values
const applyBlackFridayDiscount = createDiscountApplier(30);  // 30% off
const applyMemberDiscount       = createDiscountApplier(10); // 10% off
const applyFlashSaleDiscount    = createDiscountApplier(50); // 50% off

console.log(applyBlackFridayDiscount(199.99)); // 139.99
console.log(applyMemberDiscount(199.99));       // 179.99
console.log(applyFlashSaleDiscount(199.99));    // 100.00

// Each function remembers its OWN discountPercent — they don't interfere
```

---

## Example 4 — real world: memoization (caching expensive results)

```js
function createMemoizedFetcher(fetchFn) {
  const cache = {};                    // closed over — private to this memoizer

  return async function fetchWithCache(userId) {
    if (cache[userId]) {
      console.log(`Cache hit for user ${userId}`);
      return cache[userId];            // return cached result — no network call
    }

    console.log(`Fetching user ${userId} from network...`);
    const result = await fetchFn(userId);  // call the real fetch function
    cache[userId] = result;            // store in the private cache
    return result;
  };
}

// Mock fetch function (pretend this hits a real API)
async function fetchUserFromApi(userId) {
  return { id: userId, name: "User " + userId, email: `user${userId}@example.com` };
}

const getUser = createMemoizedFetcher(fetchUserFromApi);

await getUser(1);   // Fetching user 1 from network...
await getUser(2);   // Fetching user 2 from network...
await getUser(1);   // Cache hit for user 1  ← no network call
await getUser(2);   // Cache hit for user 2  ← no network call
```

`cache` is completely private — nothing outside can clear it, inspect it, or corrupt it, unless you explicitly expose a `clearCache` method.

---

## Tricky things you'll encounter in the real world

### 1 — The classic loop bug (every closure captures the same variable)

```js
// ❌ BROKEN — all three callbacks print 3
const buttons = ["Home", "About", "Contact"];

for (var i = 0; i < buttons.length; i++) {
  setTimeout(function () {
    console.log("Clicked button index:", i); // always 3 — there's only ONE i
  }, 1000);
}
```

**Why it breaks:** `var` is function-scoped, so there is only ONE `i` variable for the entire loop. By the time the callbacks run, `i` is already `3` (the value after the loop ends). All three closures share a reference to the same `i`.

**Fix 1 — use `let` (creates a new binding per iteration):**
```js
for (let i = 0; i < buttons.length; i++) {   // ✅ each iteration gets its OWN i
  setTimeout(() => console.log("Clicked button index:", i), 1000);
}
// Prints: 0, 1, 2  ✅
```

**Fix 2 — use `forEach` (each callback gets its own parameter):**
```js
buttons.forEach((buttonName, index) => {
  setTimeout(() => console.log(buttonName, "at index", index), 1000);
});
// Home at index 0 | About at index 1 | Contact at index 2  ✅
```

**Fix 3 — IIFE to create a fresh scope (older code, pre-ES6):**
```js
for (var i = 0; i < buttons.length; i++) {
  (function (capturedIndex) {
    setTimeout(() => console.log("index:", capturedIndex), 1000);
  })(i);  // immediately invoke with current i — captured in capturedIndex
}
```

---

### 2 — Closures capture variables by REFERENCE, not by value

```js
function createMultipliers() {
  const multipliers = [];

  for (var i = 1; i <= 3; i++) {
    multipliers.push(function (x) {
      return x * i;    // captures i by reference — NOT the value of i at push time
    });
  }

  return multipliers;
}

const fns = createMultipliers();
console.log(fns[0](10)); // 40  ← NOT 10! i is 4 after the loop
console.log(fns[1](10)); // 40
console.log(fns[2](10)); // 40

// Fix: use let
function createMultipliersFixed() {
  const multipliers = [];

  for (let i = 1; i <= 3; i++) {    // ✅ each iteration has its own i
    multipliers.push(function (x) { return x * i; });
  }

  return multipliers;
}

const fixed = createMultipliersFixed();
console.log(fixed[0](10)); // 10  ✅
console.log(fixed[1](10)); // 20  ✅
console.log(fixed[2](10)); // 30  ✅
```

---

### 3 — Stale closure (a closure reads an old value of a variable)

This is extremely common in React and event handlers:

```js
let notificationCount = 0;

function setupNotificationButton() {
  // This handler closes over notificationCount AT THE TIME it's created
  // But if notificationCount is a primitive, the closure has a stale copy
  // (This doesn't apply to objects/arrays — those are passed by reference)

  const countAtSetup = notificationCount; // snapshots the value
  document.getElementById("btn").addEventListener("click", function () {
    console.log("Count when handler was set up:", countAtSetup);
    // This will NOT reflect future changes to notificationCount
  });
}

notificationCount = 5;
setupNotificationButton(); // handler closes over countAtSetup = 5
notificationCount = 10;    // too late — handler already captured 5
```

**Fix:** Close over a mutable reference (object or array) or re-read the source of truth:
```js
const appState = { notificationCount: 0 };  // object — passed by reference

document.getElementById("btn").addEventListener("click", function () {
  console.log("Current count:", appState.notificationCount); // always current ✅
});

appState.notificationCount = 10; // handler will see 10 on next click ✅
```

---

### 4 — Multiple closures sharing the same closed-over variable

```js
function createBankAccount(initialBalance) {
  let balance = initialBalance;   // ONE balance shared by ALL returned functions

  return {
    deposit(amount)  { balance += amount; console.log("Balance:", balance); },
    withdraw(amount) { balance -= amount; console.log("Balance:", balance); },
    getBalance()     { return balance; }
  };
}

const account = createBankAccount(1000);
account.deposit(500);   // Balance: 1500
account.withdraw(200);  // Balance: 1300
console.log(account.getBalance()); // 1300

// ALL THREE methods (deposit, withdraw, getBalance) share the SAME `balance`
```

This is intentional and powerful — it's how you create truly private shared state.

---

### 5 — Each call to the outer function creates a FRESH closure

```js
function createScoreTracker(playerName) {
  let score = 0;  // each call to createScoreTracker creates a NEW, separate score

  return {
    addPoints(pts)  { score += pts; },
    getScore()      { return `${playerName}: ${score}`; }
  };
}

const player1 = createScoreTracker("Alice");
const player2 = createScoreTracker("Bob");

player1.addPoints(10);
player1.addPoints(5);
player2.addPoints(20);

console.log(player1.getScore()); // "Alice: 15"
console.log(player2.getScore()); // "Bob: 20"

// player1 and player2 have completely SEPARATE score variables
// They don't share anything — each createScoreTracker() call is its own world
```

---

### 6 — Closures and memory leaks

Because closures keep variables alive, they can prevent garbage collection:

```js
function attachHandler() {
  const hugeDataSet = new Array(1_000_000).fill("data"); // 1 million items

  document.getElementById("btn").addEventListener("click", function () {
    // This closure keeps hugeDataSet alive as long as the event listener exists
    console.log("Button clicked, data length:", hugeDataSet.length);
  });
}

attachHandler();

// hugeDataSet will NOT be garbage collected until the event listener is removed
// Fix: remove the listener when no longer needed, or don't close over large data unnecessarily
```

---

### 7 — A closure over `this` can surprise you

```js
function UserWidget(userName) {
  this.userName = userName;
  this.clickCount = 0;

  // ❌ Regular function — `this` inside setTimeout is NOT the UserWidget instance
  setTimeout(function () {
    this.clickCount++; // `this` is undefined (strict) or window (sloppy) — BUG
    console.log(this.clickCount);
  }, 1000);
}

function UserWidgetFixed(userName) {
  this.userName = userName;
  this.clickCount = 0;

  // ✅ Arrow function — inherits `this` from the enclosing scope (the constructor)
  setTimeout(() => {
    this.clickCount++;   // `this` is the UserWidgetFixed instance ✅
    console.log(this.clickCount);
  }, 1000);
}
```

Arrow functions do NOT have their own `this` — they close over the `this` of the surrounding lexical scope. This is one of the main reasons arrow functions were introduced.

---

## Common mistakes

### ❌ Mistake 1 — Calling the function instead of returning it

```js
// WRONG — you return the RESULT of calling inner, not inner itself
function createLogger(prefix) {
  function log(message) {
    console.log(prefix + ": " + message);
  }
  return log();    // ❌ calls log immediately, returns undefined
}

// RIGHT
function createLogger(prefix) {
  function log(message) {
    console.log(prefix + ": " + message);
  }
  return log;      // ✅ returns the function object
}

const errorLog = createLogger("[ERROR]");
errorLog("Something went wrong"); // [ERROR]: Something went wrong
```

---

### ❌ Mistake 2 — Expecting independent state when re-using the same closure

```js
// WRONG mental model — thinking each variable is independent
const counter = createCounter(0);
const alsoCounter = counter;      // ❌ this is the SAME object — same closure

alsoCounter.increment();
console.log(counter.getCount()); // 1 — they ARE the same, so this is correct behaviour
                                  // but people expect 0 and are confused

// RIGHT — call the factory again to get a fresh, independent closure
const counter1 = createCounter(0);
const counter2 = createCounter(0); // ✅ totally separate state
```

---

### ❌ Mistake 3 — Mutating a closed-over object reference unexpectedly

```js
function createConfig() {
  const settings = { theme: "dark", language: "en" };

  return {
    getSettings() { return settings; },        // ⚠️ returns the actual object reference
    setTheme(t)    { settings.theme = t; }
  };
}

const config = createConfig();
const s = config.getSettings();
s.theme = "light";                              // ⚠️ mutates the private settings object!
console.log(config.getSettings().theme);        // "light" — private state corrupted

// Fix: return a COPY
return {
  getSettings() { return { ...settings }; },   // ✅ spread creates a shallow copy
  setTheme(t)    { settings.theme = t; }
};
```

---

### ❌ Mistake 4 — Forgetting that `var` in loops creates one shared closure variable

(Covered in detail in the Tricky section — see loop bug above.)

---

## Frequently asked questions

**Q: Is a closure a special type of function?**  
No. Every function in JS is technically a closure — it just means the function has access to its own scope, the outer scope, and the global scope. The term "closure" usually describes the interesting case where a function *outlives* its creating scope and keeps those variables alive.

**Q: When does JS garbage collect a closed-over variable?**  
When no function holds a reference to it anymore. As long as a closure (function) that references the variable exists somewhere in memory (event listener, timer, array, etc.), the variable stays alive.

**Q: What's the difference between a closure and a class?**  
Both can store private state. Classes use `this` and are blueprints for objects. Closures use function scope. Closures are often simpler and more lightweight for single-purpose stateful things. Classes are better when you need inheritance or many methods that share complex state.

**Q: Do closures only work with `let`/`const`?**  
No — closures work with `var` too. The tricky parts (like the loop bug) come from *how* `var` is scoped (function-level, not block-level), not from whether closures work.

**Q: Can a closure modify the outer variable or just read it?**  
It can both read AND write. That's exactly how the counter and bank account examples work — the inner functions increment or change the closed-over variable.

**Q: Are closures slow?**  
For virtually all real-world applications, no. The minor overhead of maintaining the closure environment is negligible. Worry about closures for correctness and privacy, not performance.

---

## Practice exercises

### Exercise 1 — easy

Write a function `createGreeter(greeting)` that takes a greeting word (e.g. `"Hello"`) and returns a new function.  
The returned function should take a `name` parameter and log `"Hello, Parsh!"` (or whatever name + greeting is passed).

Create two greeters — one for `"Hello"` and one for `"Hi"` — and call each with a different name.

```js
// Write your code here
```

---

### Exercise 2 — medium

Build a **private click tracker** for a webpage button using closures.

Write a function `createClickTracker(buttonLabel)` that:
- Keeps a private `clickCount` starting at `0`
- Returns an object with three methods:
  - `click()` — increments `clickCount` by 1 and logs `"[buttonLabel] clicked N time(s)"`
  - `reset()` — sets `clickCount` back to `0` and logs `"[buttonLabel] counter reset"`
  - `getCount()` — returns the current count

Create two independent trackers (e.g. `"Subscribe"` and `"Share"`) and show that their counts are completely separate.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a **rate limiter** using closures.

Write a function `createRateLimiter(maxCalls, windowMs)` that:
- Keeps private state: `callTimestamps` (an array of timestamps of recent calls)
- Returns a function `attempt(actionName)` that:
  - Removes timestamps from `callTimestamps` that are older than `windowMs` milliseconds (they're outside the window)
  - If `callTimestamps.length < maxCalls`, adds the current timestamp and logs `"[actionName] allowed (N/maxCalls used)"`
  - Otherwise logs `"[actionName] BLOCKED — rate limit reached (maxCalls/maxCalls used)"`

Test it: create a limiter of 3 calls per 2000ms, call `attempt("save")` four times in a row.  
(The first 3 should be allowed, the 4th blocked.)

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

**What a closure is:**
```
outer function runs → creates scope with variables
  → inner function is created inside that scope
  → inner function is returned (or stored)
  → outer function finishes, but its scope is NOT destroyed
  → because inner still holds a reference to it
```

**The two requirements for a closure:**
1. A function defined inside another function (or any scope it can "close over")
2. The inner function is returned, stored, or passed somewhere that outlives the outer scope

**Common closure patterns:**

| Pattern | What it does |
|---|---|
| Counter / tracker | Inner fn reads/writes a private variable |
| Factory function | Outer fn takes config, returns specialised inner fn |
| Module pattern | Returns object of methods that share private state |
| Partial application | Outer fn bakes in some args, returns fn waiting for the rest |
| Memoization | Closes over a `cache` object, stores results to avoid recomputation |
| Once function | Uses a flag to ensure a function only runs once |

**Key rules:**
- Closures capture variables **by reference**, not by value (for primitives in `var` loops, this causes bugs)
- Each call to the outer function creates a **fresh, independent closure**
- All returned functions from the same outer call **share the same closed-over variables**
- Closures keep variables alive as long as any function referencing them exists

---

## Connected topics

- **18 — Scope** — closures are impossible to understand without scope; every closure IS a scope relationship
- **20 — Higher-order functions** — HOFs take or return functions; closures are what make returned functions useful and stateful
- **22 — IIFE** — immediately invoked function expressions are a classic closure pattern used to create isolated scopes without polluting the global namespace
