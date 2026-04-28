# 20 — Higher-Order Functions

## What is this?
A higher-order function (HOF) is a function that either **accepts another function as an argument**, **returns a function**, or both. Think of it like a manager at a company — the manager doesn't do the work directly, they take instructions (a function) from someone and coordinate when and how that work gets done. The manager is the HOF; the instructions are the function passed in.

Every time you've used `.map()`, `.filter()`, `.forEach()`, `setTimeout`, or an event listener, you've already used a higher-order function.

## Why does it matter?
HOFs are the foundation of **functional programming** in JavaScript. They let you write reusable, composable, declarative code instead of repeating logic. Without HOFs, you'd write the same loop structure dozens of times with tiny variations. With HOFs, you write the *shape* of an operation once and swap in different behaviours.

---

## Syntax

```js
// --- HOF that ACCEPTS a function ---
function runTwice(fn) {          // fn is a function passed in as an argument
  fn();
  fn();
}

runTwice(() => console.log("Hello")); // Hello  Hello

// --- HOF that RETURNS a function ---
function multiplierOf(factor) {
  return function (number) {     // returns a new function
    return number * factor;
  };
}

const double = multiplierOf(2);
const triple = multiplierOf(3);

console.log(double(5));  // 10
console.log(triple(5));  // 15
```

---

## How it works — line by line

```js
function runTwice(fn) {
```
`fn` is just a parameter — but its *type* is a function. Functions are **first-class values** in JS, meaning you can pass them around like numbers or strings.

```js
  fn();
  fn();
```
We call whatever function was passed in, twice. `runTwice` doesn't know or care what `fn` does — it just knows when to call it.

```js
function multiplierOf(factor) {
  return function (number) {
    return number * factor;
  };
}
```
`multiplierOf` returns a function. The returned function *closes over* `factor` (this is where HOFs and closures intersect). Each call to `multiplierOf` creates a fresh function with a different `factor` baked in.

```js
const double = multiplierOf(2);
```
`double` is now a function — not the number 2, not `undefined` — a callable function that multiplies its argument by 2.

---

## Example 1 — basic: custom array iterator

Before you knew about `forEach`, you would write the loop yourself. A HOF lets you extract the loop and inject the behaviour:

```js
// Our own simple forEach
function eachItem(array, callback) {   // callback = the function passed in
  for (let i = 0; i < array.length; i++) {
    callback(array[i], i);             // call it with current item and index
  }
}

const productNames = ["Keyboard", "Monitor", "Mouse"];

eachItem(productNames, function (product, index) {
  console.log(index + 1 + ". " + product);
});
// 1. Keyboard
// 2. Monitor
// 3. Mouse

// Swap the callback — completely different behaviour, same HOF
eachItem(productNames, function (product) {
  console.log(product.toUpperCase());
});
// KEYBOARD
// MONITOR
// MOUSE
```

Same `eachItem` function, completely different results by changing what you pass in. That's the power of HOFs.

---

## Example 2 — real world: pipeline / data transformer

A HOF that returns a function, used to build a reusable data processing pipeline:

```js
// A HOF that wraps any transform function with logging
function withLogging(transformFn, label) {
  return function (data) {
    console.log(`[${label}] Input:`, data);
    const result = transformFn(data);
    console.log(`[${label}] Output:`, result);
    return result;
  };
}

// Raw transform functions (pure, simple)
function normaliseEmail(email) {
  return email.trim().toLowerCase();
}

function formatPhoneNumber(phone) {
  return phone.replace(/\D/g, "").replace(/(\d{3})(\d{3})(\d{4})/, "($1) $2-$3");
}

// Wrap them with logging
const loggedNormaliseEmail    = withLogging(normaliseEmail, "EmailNormaliser");
const loggedFormatPhoneNumber = withLogging(formatPhoneNumber, "PhoneFormatter");

loggedNormaliseEmail("  Alice@Example.COM  ");
// [EmailNormaliser] Input:   Alice@Example.COM  
// [EmailNormaliser] Output: alice@example.com

loggedFormatPhoneNumber("8005551234");
// [PhoneFormatter] Input: 8005551234
// [PhoneFormatter] Output: (800) 555-1234
```

`withLogging` never touches the transform logic — it just wraps any function with consistent logging behaviour. This is called a **decorator pattern**.

---

## Example 3 — real world: retry wrapper

```js
// A HOF that returns a version of any async function that retries on failure
function withRetry(asyncFn, maxAttempts) {
  return async function (...args) {            // accepts whatever args the original fn takes
    let lastError;

    for (let attempt = 1; attempt <= maxAttempts; attempt++) {
      try {
        const result = await asyncFn(...args); // call the original function
        console.log(`Succeeded on attempt ${attempt}`);
        return result;
      } catch (error) {
        lastError = error;
        console.log(`Attempt ${attempt} failed: ${error.message}`);
      }
    }

    throw new Error(`All ${maxAttempts} attempts failed. Last error: ${lastError.message}`);
  };
}

// Simulated unreliable API call
let callCount = 0;
async function fetchOrderStatus(orderId) {
  callCount++;
  if (callCount < 3) throw new Error("Network timeout");
  return { orderId, status: "shipped" };
}

// Wrap with retry — now fetchOrderStatus gets 3 chances automatically
const reliableFetch = withRetry(fetchOrderStatus, 3);

const order = await reliableFetch("ORD-001");
// Attempt 1 failed: Network timeout
// Attempt 2 failed: Network timeout
// Succeeded on attempt 3
console.log(order); // { orderId: 'ORD-001', status: 'shipped' }
```

`withRetry` has no knowledge of what `fetchOrderStatus` does — it just knows *how* to retry any async function.

---

## Example 4 — real world: `compose` and `pipe`

Two classic HOF utilities that chain functions together:

```js
// pipe(f, g, h)(x) = h(g(f(x))) — left to right
function pipe(...fns) {
  return function (initialValue) {
    return fns.reduce(function (value, fn) {
      return fn(value);
    }, initialValue);
  };
}

// Individual transform steps (each one is a simple function)
const trim        = (str) => str.trim();
const toLowerCase = (str) => str.toLowerCase();
const removeSpaces = (str) => str.replace(/\s+/g, "-");
const addSlug     = (str) => "/posts/" + str;

// Compose them into a URL slug generator
const generateSlug = pipe(trim, toLowerCase, removeSpaces, addSlug);

console.log(generateSlug("  Getting Started With JavaScript  "));
// /posts/getting-started-with-javascript

console.log(generateSlug("  Higher Order Functions Explained  "));
// /posts/higher-order-functions-explained
```

`pipe` returns a function. That returned function runs the data through every transform in sequence. Add or remove steps without touching `pipe` itself.

---

## Tricky things you'll encounter in the real world

### 1 — Passing a function reference vs calling it immediately

```js
const prices = [10, 25, 5, 40, 15];

// ❌ WRONG — you're calling Math.round() yourself and passing the RESULT (NaN/undefined)
const rounded = prices.map(Math.round());    // TypeError or unexpected output

// ✅ RIGHT — pass the reference; map will call it with each element
const rounded = prices.map(Math.round);      // [10, 25, 5, 40, 15]
```

No parentheses = pass the function. Parentheses = call it immediately. This trips everyone up at least once.

---

### 2 — `parseInt` passed to `.map()` — a classic gotcha

```js
const stringNumbers = ["1", "2", "3", "4"];

// ❌ Looks innocent but produces [1, NaN, NaN, NaN]
const ints = stringNumbers.map(parseInt);
// Why? map passes (value, index, array) to the callback
// parseInt("1", 0, [...]) — 0 is treated as base 10 → 1  ✅
// parseInt("2", 1, [...]) — base 1 is invalid → NaN  ❌
// parseInt("3", 2, [...]) — "3" is invalid in base 2 → NaN  ❌

// ✅ Fix — wrap it so only the value is passed
const ints = stringNumbers.map((str) => parseInt(str, 10));
// [1, 2, 3, 4]
```

Any time you pass a named function as a callback to `map`/`filter`/etc., double-check it doesn't misbehave when it receives extra arguments (value, index, array).

---

### 3 — HOFs can be difficult to read when over-nested

```js
// ❌ Hard to read — deeply nested anonymous functions
processOrders(orders, function(order) {
  return filterValid(order, function(item) {
    return transformItem(item, function(result) {
      return result;
    });
  });
});

// ✅ Name your callbacks — much more readable
function transformResult(result) { return result; }
function transformItem(item)     { return transformItem(item, transformResult); }
function processOrder(order)     { return filterValid(order, transformItem); }

processOrders(orders, processOrder);
```

Arrow functions help, but named intermediate functions help more when logic gets complex.

---

### 4 — `this` inside a callback passed to a HOF

```js
const userManager = {
  adminPrefix: "[ADMIN]",
  users: ["Alice", "Bob", "Charlie"],

  // ❌ Regular function loses `this` — it becomes undefined (strict) or window
  listUsers_broken() {
    this.users.forEach(function (user) {
      console.log(this.adminPrefix + " " + user); // TypeError: cannot read properties of undefined
    });
  },

  // ✅ Arrow function inherits `this` from listUsers_fixed's context
  listUsers_fixed() {
    this.users.forEach((user) => {
      console.log(this.adminPrefix + " " + user); // [ADMIN] Alice, etc.
    });
  }
};

userManager.listUsers_fixed();
// [ADMIN] Alice
// [ADMIN] Bob
// [ADMIN] Charlie
```

Always use arrow functions for callbacks inside methods when you need `this`.

---

### 5 — Returning a function vs returning the function's result

```js
function createValidator(minLength) {
  // ❌ Wrong — this returns true/false immediately, not a function
  return "password".length >= minLength;

  // ✅ Right — wraps the logic in a function and returns THAT
  return function (value) {
    return value.length >= minLength;
  };
}

const isValidPassword = createValidator(8);
console.log(isValidPassword("hello"));      // false
console.log(isValidPassword("strongpass")); // true
```

---

### 6 — HOFs compose — you can wrap wrappers

```js
const reliableLoggedFetch = withLogging(withRetry(fetchOrderStatus, 3), "OrderFetcher");

// Now every call is both retried AND logged — no changes to the original function
await reliableLoggedFetch("ORD-002");
```

This is the real power: small, single-purpose HOFs that stack.

---

## Common mistakes

### ❌ Mistake 1 — Adding `()` when passing a function as a callback

```js
// WRONG — greet() executes immediately, undefined is passed as the callback
setTimeout(greet(), 1000);

// RIGHT — pass the reference
setTimeout(greet, 1000);

// Also right — arrow function wrapper
setTimeout(() => greet("Alice"), 1000); // use this when you need to pass arguments
```

---

### ❌ Mistake 2 — Forgetting to return inside a callback

```js
const orderIds = [101, 102, 103];

// WRONG — no return, results array is [undefined, undefined, undefined]
const orderUrls = orderIds.map(function (id) {
  "/orders/" + id;     // ❌ expression evaluated, result discarded
});

// RIGHT
const orderUrls = orderIds.map(function (id) {
  return "/orders/" + id;   // ✅
});

// Also right — arrow function with implicit return (no braces)
const orderUrls = orderIds.map((id) => "/orders/" + id); // ✅
```

---

### ❌ Mistake 3 — Mutating the original array inside `.map()`

```js
const products = [
  { name: "Keyboard", price: 100 },
  { name: "Mouse",    price: 50  }
];

// WRONG — mutates the original objects AND the resulting array is [undefined, undefined]
const discounted = products.map((product) => {
  product.price = product.price * 0.9;  // ❌ mutates the original
});

// RIGHT — return a new object
const discounted = products.map((product) => ({
  ...product,                            // ✅ copy all original properties
  price: parseFloat((product.price * 0.9).toFixed(2))
}));
```

---

## Frequently asked questions

**Q: What's the difference between a HOF and a callback?**  
A HOF is the *function that takes or returns another function*. A callback is the *function that gets passed in*. `forEach` is the HOF; the function you pass to `forEach` is the callback.

**Q: Are `.map()`, `.filter()`, and `.reduce()` higher-order functions?**  
Yes — all three accept a function as their argument, making them HOFs. They're built into JavaScript's Array prototype.

**Q: When should I write my own HOF vs using a built-in?**  
Use built-ins (`map`, `filter`, etc.) for array operations. Write your own HOF when you find yourself repeating a *structural* pattern (retry logic, logging, rate limiting, caching) across multiple functions.

**Q: Is a function that returns a function always called a "factory function"?**  
Not necessarily. "Factory function" usually means a function that returns a *new object*. A HOF that returns a function is more specifically called a **function factory** or **curried function** depending on context.

**Q: Can a function be both a HOF and use closures?**  
Yes — they're related but distinct concepts. HOF describes the *interface* (takes/returns functions). Closure describes the *memory mechanism* (inner functions remember outer scope). HOFs that return functions almost always create closures.

---

## Practice exercises

### Exercise 1 — easy

Write a HOF called `applyToAll(array, fn)` that loops through `array`, calls `fn` on each element, and returns a new array of results. (This is basically your own `map` — do NOT use the built-in `.map()`.)

Test it:
- Call it with `[1, 2, 3, 4, 5]` and a function that squares each number.
- Call it again with `["alice", "bob", "charlie"]` and a function that capitalises the first letter.

```js
// Write your code here
```

---

### Exercise 2 — medium

Write a HOF called `createMultiplier(factor)` that returns a function. The returned function takes a `number` and returns `number * factor`.

Then:
- Create `double`, `triple`, and `tenX` using `createMultiplier`.
- Write a second HOF `applyAll(value, ...fns)` that takes a starting value and any number of functions, and applies them **one after another** (the output of each becomes the input of the next — like a pipe).
- Call `applyAll(3, double, triple, tenX)` and log the result. (3 → 6 → 18 → 180)

```js
// Write your code here
```

---

### Exercise 3 — hard

Write a HOF called `createThrottledFunction(fn, limitMs)` that returns a new version of `fn` which can only be called **once per `limitMs` milliseconds**. If it's called again within that window, the call is silently ignored.

Requirements:
- Keep `lastCalledAt` as private state (closed over) — initialise to `0`.
- On each call, check if `Date.now() - lastCalledAt >= limitMs`.
- If allowed: update `lastCalledAt`, call the original `fn` with the same arguments, and return its result.
- If blocked: log `"[throttled] call blocked"` and return `undefined`.

Test it by creating a throttled version of a `saveDocument(docName)` function that logs `"Saving: [docName]"`, with a 1000ms limit. Call it 3 times in a rapid loop and confirm only the first call goes through.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

**Definition:**
> A function is a higher-order function if it **accepts a function as an argument** OR **returns a function** (or both).

**Two forms:**

```js
// Form 1 — accepts a function
function doTwice(fn)    { fn(); fn(); }

// Form 2 — returns a function
function multiplierOf(n) { return (x) => x * n; }

// Form 3 — both (accepts AND returns)
function withLogging(fn) { return (...args) => { console.log(args); return fn(...args); }; }
```

**Built-in HOFs you use constantly:**

| HOF | What it does |
|---|---|
| `array.forEach(fn)` | Calls `fn` for each element, returns nothing |
| `array.map(fn)` | Calls `fn` for each element, returns new transformed array |
| `array.filter(fn)` | Returns new array of elements where `fn` returned `true` |
| `array.reduce(fn, init)` | Collapses array to single value using `fn` |
| `array.find(fn)` | Returns first element where `fn` returns `true` |
| `array.sort(fn)` | Sorts using `fn` as comparator |
| `setTimeout(fn, ms)` | Calls `fn` after `ms` milliseconds |
| `addEventListener(event, fn)` | Calls `fn` when `event` fires |

**Key rules:**
- Pass function references, not calls: `map(fn)` not `map(fn())`
- Arrow function callbacks with implicit return: `(x) => x * 2` (no braces, no `return`)
- Use arrow functions in method callbacks when you need `this`
- Name complex callbacks — anonymous functions nested 3 levels deep are unreadable

---

## Connected topics

- **19 — Closures** — HOFs that return functions almost always create closures; the returned function closes over the outer function's variables
- **21 — Callback functions** — callbacks are the functions *passed into* HOFs; they deserve their own deep-dive (async callbacks, error-first callbacks, etc.)
- **43 — Currying** — a specific HOF pattern where a multi-argument function is broken into a chain of single-argument functions
