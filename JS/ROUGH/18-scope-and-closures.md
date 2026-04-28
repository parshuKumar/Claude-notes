# 18 — Scope and Closures

## What is this?

**Scope** is the set of rules that determines where a variable can be accessed from. Every variable you declare lives inside a scope — and the scope decides who can see it and who can't.

**Closure** is what happens when a function "remembers" the variables from the scope where it was created — even after that scope has finished executing and would normally be gone.

These two concepts are deeply connected. You cannot understand closures without understanding scope first.

```js
function outer() {
  const secret = "I'm in outer's scope";

  function inner() {
    console.log(secret);  // inner can see outer's variable — that's a closure
  }

  inner();
}

outer();  // "I'm in outer's scope"
```

---

## Why does it matter?

Scope and closures are not optional advanced knowledge — they are the mechanism that controls every variable in every JavaScript program you write. When something is `undefined` and you don't know why, or a variable update doesn't seem to stick, or a loop behaves weirdly in async code — it's almost always a scope or closure problem.

Closures specifically power:
- **Private state** — data that only specific functions can access
- **Factory functions** — functions that create other functions with baked-in behaviour
- **Callbacks and event handlers** — accessing outer variables inside an async callback
- **React hooks** — `useState`, `useEffect`, and almost everything in React is closures
- **Module pattern** — the original way to hide implementation details before ES6 modules
- **Memoization** — caching expensive results by closing over a cache object

If you understand closures, you understand how JavaScript actually works.

---

## Part 1 — Scope

### The three scopes

**Global scope** — the outermost level. Variables declared here are accessible everywhere.

```js
const appName = "ShopFlow";  // global

function showName() {
  console.log(appName);  // accessible inside functions
}

showName();             // "ShopFlow"
console.log(appName);  // "ShopFlow"
```

**Function scope** — variables declared inside a function exist only inside that function. They are created when the function runs and destroyed when it ends.

```js
function processOrder() {
  const orderId = "ORD-001";  // function scope
  console.log(orderId);        // ✅
}

processOrder();
console.log(orderId);  // ❌ ReferenceError: orderId is not defined
```

**Block scope** (`let` and `const` only) — variables declared inside `{}` (if, for, while, standalone blocks) exist only inside those braces.

```js
if (true) {
  const blockVar = "inside block";
  let anotherBlock = "also inside";
  var leaking = "I escape!";      // var ignores block scope
}

console.log(blockVar);    // ❌ ReferenceError
console.log(anotherBlock);// ❌ ReferenceError
console.log(leaking);     // ✅ "I escape!" — var leaks out of blocks
```

`var` only respects function scope, not block scope. This is one of the biggest historical JavaScript problems and why `let`/`const` were introduced.

---

### The scope chain

When JavaScript looks up a variable, it doesn't just check the current scope — it climbs up through every outer scope until it finds the variable or reaches global scope:

```js
const globalLevel = "global";

function outer() {
  const outerLevel = "outer";

  function middle() {
    const middleLevel = "middle";

    function inner() {
      const innerLevel = "inner";

      // inner can see ALL of these:
      console.log(innerLevel);  // own scope
      console.log(middleLevel); // parent's scope
      console.log(outerLevel);  // grandparent's scope
      console.log(globalLevel); // great-grandparent (global)
    }

    inner();
  }

  middle();
}

outer();
```

This chain of scopes — from inner to outer — is the **scope chain**. The lookup always goes **outward**, never inward. `outer` cannot see `innerLevel`.

**Shadowing** — if a variable has the same name in an inner scope as an outer scope, the inner one takes over and "shadows" the outer one:

```js
const username = "GlobalUser";

function showUser() {
  const username = "LocalUser";  // shadows the global
  console.log(username);          // "LocalUser"
}

showUser();
console.log(username);  // "GlobalUser" — unchanged
```

---

### Lexical scope (static scope)

JavaScript uses **lexical scope** — the scope of a variable is determined by where it is **written** in the source code, not by where the function is called from.

```js
const environment = "production";

function readEnvironment() {
  console.log(environment);  // always reads from where it was defined
}

function configureApp() {
  const environment = "development";  // local variable
  readEnvironment();  // ← what does this log?
}

configureApp();  // "production" — NOT "development"
```

`readEnvironment` was *written* in the global scope, so it reads `environment` from the global scope — even when *called* from inside `configureApp`. The call location doesn't matter. The write location does. This is lexical scope.

---

## Part 2 — Closures

### What a closure is

A closure is formed when a function accesses a variable from an **outer scope**, AND that function outlives the scope that created that variable.

```js
function makeCounter() {
  let count = 0;  // this variable lives in makeCounter's scope

  function increment() {
    count++;            // increment accesses count from outer scope
    return count;
  }

  return increment;  // ← we return the function itself (not its result)
}

const counter = makeCounter();
// makeCounter() has now finished executing
// normally 'count' would be destroyed
// but 'increment' closed over it — so it survives

console.log(counter());  // 1
console.log(counter());  // 2
console.log(counter());  // 3
```

**Step by step:**
1. `makeCounter()` runs — `count` is created as `0`
2. `increment` is defined — it closes over `count`
3. `makeCounter()` returns `increment` and finishes
4. Normally `count` would be garbage collected — but `increment` still holds a reference to it
5. JavaScript keeps `count` alive in memory because `increment` needs it
6. Each call to `counter()` reads and updates the same `count` in memory

`count` is now **private** — only `increment` can access it. No outside code can touch it directly.

---

### Closures remember the variable, not the value

This is the single most misunderstood thing about closures.

```js
function makeAdder(x) {
  return function(y) {
    return x + y;  // closes over x
  };
}

const add5  = makeAdder(5);
const add10 = makeAdder(10);

console.log(add5(3));   // 8   — x is 5 in this closure
console.log(add10(3));  // 13  — x is 10 in this closure
console.log(add5(7));   // 12  — same add5, x is still 5
```

Each call to `makeAdder` creates a **new scope** with a new `x`. `add5` and `add10` are two separate closures each capturing their own `x`.

But closures capture the *variable* (the memory location), not a snapshot of its value. If the variable changes after the closure is created, the closure sees the new value:

```js
function makeFlexCounter() {
  let count = 0;

  return {
    increment() { count++; },
    decrement() { count--; },
    reset()     { count = 0; },
    get()       { return count; },
  };
}

const myCounter = makeFlexCounter();
myCounter.increment();
myCounter.increment();
myCounter.increment();
myCounter.decrement();
console.log(myCounter.get());  // 2 — all methods share the same 'count'
```

All four methods close over the *same* `count` variable — they share it. Changes made by one method are visible to all others.

---

## Example 1 — basic closure patterns

### Private variable with getter/setter

```js
function createTemperatureMonitor(unit = "C") {
  let currentTemp = null;
  let readingCount = 0;

  return {
    record(temp) {
      currentTemp = temp;
      readingCount++;
    },
    getCurrent() {
      return currentTemp === null
        ? "No reading yet"
        : `${currentTemp}°${unit}`;
    },
    getReadingCount() {
      return readingCount;
    },
  };
}

const sensor = createTemperatureMonitor("F");
sensor.record(98.6);
sensor.record(99.1);
console.log(sensor.getCurrent());       // "99.1°F"
console.log(sensor.getReadingCount());  // 2
console.log(sensor.currentTemp);        // undefined — private!
```

### Memoization (caching expensive results)

```js
function createMemoizedSquareRoot() {
  const cache = {};  // this object persists across calls via closure

  return function(number) {
    if (cache[number] !== undefined) {
      console.log("Cache hit for", number);
      return cache[number];
    }
    const result = Math.sqrt(number);
    cache[number] = result;
    return result;
  };
}

const sqrtOf = createMemoizedSquareRoot();
console.log(sqrtOf(144));  // calculates: 12
console.log(sqrtOf(144));  // Cache hit for 144: 12  — no recalculation
console.log(sqrtOf(225));  // calculates: 15
```

### Once function (runs only the first time)

```js
function createOnceFunction(fn) {
  let hasRun = false;
  let result;

  return function(...args) {
    if (!hasRun) {
      result = fn(...args);
      hasRun = true;
    }
    return result;
  };
}

const initializeApp = createOnceFunction(() => {
  console.log("App initialized!");
  return "initialized";
});

initializeApp();  // "App initialized!"
initializeApp();  // nothing — already ran
initializeApp();  // nothing
```

---

## Example 2 — real world

### Rate limiter

```js
function createRateLimiter(maxCalls, windowMs) {
  let callCount = 0;
  let windowStart = Date.now();

  return function(fn) {
    const now = Date.now();

    if (now - windowStart > windowMs) {
      callCount = 0;
      windowStart = now;
    }

    if (callCount >= maxCalls) {
      console.log("Rate limit exceeded — try again shortly");
      return;
    }

    callCount++;
    return fn();
  };
}

const limiter = createRateLimiter(3, 5000);  // max 3 calls per 5 seconds

limiter(() => console.log("API call 1"));  // API call 1
limiter(() => console.log("API call 2"));  // API call 2
limiter(() => console.log("API call 3"));  // API call 3
limiter(() => console.log("API call 4"));  // Rate limit exceeded
```

### Configuration with partial application

```js
function createApiClient(baseUrl, authToken) {
  const defaultHeaders = {
    "Authorization": `Bearer ${authToken}`,
    "Content-Type": "application/json",
  };

  return {
    get(endpoint) {
      return fetch(`${baseUrl}${endpoint}`, {
        method: "GET",
        headers: defaultHeaders,
      });
    },
    post(endpoint, body) {
      return fetch(`${baseUrl}${endpoint}`, {
        method: "POST",
        headers: defaultHeaders,
        body: JSON.stringify(body),
      });
    },
  };
}

const apiClient = createApiClient("https://api.example.com", "tok_abc123");
// baseUrl and authToken are private — closed over
// no need to pass them on every call
apiClient.get("/users");
apiClient.post("/orders", { productId: "P-01", quantity: 2 });
```

---

## Tricky things you'll encounter in the real world

### 1. The classic loop closure bug (most asked interview question)

```js
// ❌ The infamous var + closure bug
for (var i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i);
  }, 1000);
}
// Logs: 3, 3, 3  — NOT 0, 1, 2
```

**Why?** `var` is function-scoped — there is only **one** `i` shared across all three iterations. The `setTimeout` callbacks don't run for 1 second. By then, the loop has already finished and `i` is 3. All three callbacks close over the same `i`, which is now 3.

**Fix 1 — use `let` (block-scoped, creates a new `i` per iteration):**

```js
for (let i = 0; i < 3; i++) {
  setTimeout(function() {
    console.log(i);
  }, 1000);
}
// Logs: 0, 1, 2  ✅ — each iteration has its own i
```

**Fix 2 — IIFE to create a new scope per iteration (old ES5 pattern):**

```js
for (var i = 0; i < 3; i++) {
  (function(capturedI) {
    setTimeout(function() {
      console.log(capturedI);  // captured in a new scope
    }, 1000);
  })(i);
}
// Logs: 0, 1, 2  ✅
```

**Fix 3 — arrow function with a captured value:**

```js
for (var i = 0; i < 3; i++) {
  const current = i;  // new const per iteration captures current value
  setTimeout(() => console.log(current), 1000);
}
```

You will see this bug in interviews and in legacy codebases. `let` is the clean modern fix.

### 2. Closures over mutable variables — stale values

```js
let price = 100;

const getDiscounted = () => price * 0.9;

console.log(getDiscounted());  // 90

price = 200;

console.log(getDiscounted());  // 180 — closure reads CURRENT value of price
```

The closure doesn't take a snapshot of `price`. It holds a reference. If `price` changes, the next call to `getDiscounted()` sees the new value. This is correct behaviour, but it surprises people who expect closures to "freeze" values.

If you want to freeze a value, capture it in a constant:

```js
const capturedPrice = price;  // snapshot now
const getDiscounted = () => capturedPrice * 0.9;
```

### 3. Memory leaks with closures

```js
function attachHandler() {
  const massiveData = new Array(1000000).fill("data");  // 1M item array

  document.getElementById("btn").addEventListener("click", function() {
    console.log("clicked");  // this callback closes over massiveData's scope
    // even though massiveData isn't used, it can't be garbage collected
    // as long as this event listener exists
  });
}
```

A closure keeps the entire scope alive, not just the variables it uses. If a callback closes over a scope that contains large data, that data stays in memory as long as the closure exists.

**Fix:** explicitly remove event listeners when they're no longer needed, and avoid closing over large unnecessary objects.

### 4. Each function invocation creates a fresh closure

```js
function createGreeter(greeting) {
  return function(name) {
    return `${greeting}, ${name}!`;
  };
}

const sayHello = createGreeter("Hello");
const sayHi    = createGreeter("Hi");
const sayHey   = createGreeter("Hey");

console.log(sayHello("Ana"));  // "Hello, Ana!"
console.log(sayHi("Ana"));     // "Hi, Ana!"
console.log(sayHey("Ana"));    // "Hey, Ana!"
```

Three separate calls to `createGreeter` = three separate closures = three independent `greeting` variables. They don't interfere with each other. Each call creates its own scope.

### 5. Closures in callbacks — accessing outer state

```js
function setupForm() {
  let submitCount = 0;

  document.getElementById("form").addEventListener("submit", function(event) {
    event.preventDefault();
    submitCount++;  // closes over submitCount
    console.log(`Form submitted ${submitCount} time(s)`);
  });
}
```

`submitCount` lives in `setupForm`'s scope. The event listener callback closes over it. Every time the form is submitted, it increments the same `submitCount`. This is how event handlers track state without polluting global scope.

### 6. IIFE — Immediately Invoked Function Expression

An IIFE creates a scope and executes it immediately. Historically used before modules to avoid polluting global scope:

```js
(function() {
  const privateVar = "only inside here";
  console.log(privateVar);  // works
})();

console.log(privateVar);  // ❌ ReferenceError — scope is gone
```

Arrow IIFE:

```js
(() => {
  const init = "setup";
  // run initialization code
})();
```

IIFEs are less common now (ES6 modules handle this), but you'll see them in older code and in some build tool patterns.

### 7. Closures don't capture `this` — arrow functions fix this

```js
function Timer() {
  this.ticks = 0;

  // Regular function callback — loses 'this'
  setInterval(function() {
    this.ticks++;  // ❌ 'this' is window/undefined, not the Timer instance
  }, 1000);

  // Arrow function callback — inherits 'this' from Timer
  setInterval(() => {
    this.ticks++;  // ✅ 'this' is the Timer instance
  }, 1000);
}
```

This is why arrow functions were introduced — lexical `this` (topic 15) works together with closure. The arrow function closes over the outer `this`, making it behave as expected.

---

## Common mistakes

### Mistake 1: Expecting `var` in a loop to capture per-iteration values

```js
// ❌ All click handlers will log the same final value of i
for (var i = 0; i < buttons.length; i++) {
  buttons[i].addEventListener("click", () => console.log(i));
}

// ✅ Use let
for (let i = 0; i < buttons.length; i++) {
  buttons[i].addEventListener("click", () => console.log(i));
}
```

### Mistake 2: Thinking closures freeze the variable's value

```js
let discount = 0.1;
const getDiscount = () => discount;

discount = 0.2;
console.log(getDiscount());  // 0.2 — not 0.1 — closure reads live variable
```

### Mistake 3: Creating closures in a loop over an object and expecting stable references

```js
const handlers = {};
const events = ["click", "hover", "focus"];

for (var event of events) {
  handlers[event] = () => console.log(event);  // var: only one 'event'
}

handlers.click();  // "focus" — all share the last value!
// Fix: use let (for...of with var leaks too in older setups)
```

Actually `for...of` with `var` does have a single shared binding. Use `let` or `const`:

```js
for (const event of events) {
  handlers[event] = () => console.log(event);  // ✅ const per iteration
}
handlers.click();  // "click"
```

### Mistake 4: Mutating a closed-over object and expecting isolation

```js
function createUser(userData) {
  return {
    getUser() { return userData; },      // returns reference to the same object
  };
}

const user = createUser({ name: "Sam" });
const retrieved = user.getUser();
retrieved.name = "Hacked";  // mutates the closed-over object!

console.log(user.getUser().name);  // "Hacked" — not "Sam"
```

Closures capture the reference to objects, not a copy. If you return a closed-over object directly, callers can mutate it. Return a copy if you want immutability:

```js
getUser() { return { ...userData }; }  // spread makes a shallow copy
```

---

## Frequently asked questions

**Q: Does every function create a closure?**  
Technically yes — every function has access to the scope chain at its definition point. But we only call it a "closure" in a meaningful sense when the function accesses a variable from an outer scope that has already finished executing (the outer scope has "closed").

**Q: Are closures slow?**  
No meaningfully. Modern JS engines (V8) optimize closures heavily. The memory concern is real for large data (see memory leak section), but performance overhead is negligible.

**Q: Closure vs global variable — what's the difference?**  
A global variable is accessible to all code. A closed-over variable is accessible only to the functions that close over it — it's private by design. Closures give you the persistence of a global without the pollution.

**Q: Can a closure outlive the function that created it?**  
Yes — that's exactly the point. The bank account example, the counter, the rate limiter — the inner function (the closure) outlives `createBankAccount()`, `makeCounter()`, `createRateLimiter()`. The closed-over variables persist in memory until the closure itself is garbage collected.

**Q: What's the difference between scope and closure?**  
Scope is a set of rules (where can this variable be accessed?). Closure is the mechanism that lets a function carry its scope with it — keeping outer variables alive after their function finishes.

**Q: Does `const` vs `let` affect closure behaviour?**  
For closures, no — both can be closed over. The difference only matters for reassignment (`const` can't be reassigned). Inside a loop, `const` and `let` both create a new binding per iteration; `var` does not.

---

## Practice exercises

### Exercise 1 — easy

Write a factory function `createScoreTracker(playerName)` that returns an object with:
- `addScore(points)` — adds to the player's score
- `deductScore(points)` — subtracts (minimum 0 — score can't go negative)
- `getScore()` — returns current score
- `getSummary()` — returns `"Priya: 350 points"`

Create two separate trackers and verify they don't share state:

```js
const player1 = createScoreTracker("Priya");
const player2 = createScoreTracker("Jordan");

player1.addScore(200);
player1.addScore(150);
player2.addScore(400);
player1.deductScore(50);

console.log(player1.getSummary());  // "Priya: 300 points"
console.log(player2.getSummary());  // "Jordan: 400 points"
```

---

### Exercise 2 — medium

Fix the classic loop closure bug in this code AND explain in a comment why the original was broken:

```js
// ❌ This logs 5, 5, 5, 5, 5 instead of 0, 1, 2, 3, 4
const funcs = [];

for (var i = 0; i < 5; i++) {
  funcs.push(function() {
    return i;
  });
}

funcs.forEach(fn => console.log(fn()));
```

Then write a second version of the same loop that collects functions returning **squares** of their index (`0, 1, 4, 9, 16`) using `let`.

---

### Exercise 3 — hard

Build a **task queue system** using closures:

```js
function createTaskQueue(concurrencyLimit) {
  // your code
}
```

The returned object should have:
- `add(taskName, duration)` — adds a task to the queue; logs `"Queued: <taskName>"`
- `run()` — starts processing tasks; respects `concurrencyLimit` (max that many tasks "running" at once); logs `"Running: <taskName>"` when a task starts and `"Done: <taskName>"` when it finishes (simulate duration with a counter, no real async needed)
- `getStats()` — returns:
  ```js
  { queued: 2, running: 1, completed: 3, total: 6 }
  ```
- `clear()` — empties the queue (does not stop running tasks); returns number of tasks removed

All state (`queue`, `running count`, `completed count`) must live in the closure — no external variables.

Test:
```js
const queue = createTaskQueue(2);

queue.add("fetchUsers", 3);
queue.add("processOrders", 2);
queue.add("sendEmails", 4);
queue.add("generateReport", 1);

queue.run();
console.log(queue.getStats());
```

---

## Quick reference

```
Scope and closures — cheat sheet
─────────────────────────────────────────────────────
SCOPE TYPES
  Global scope     accessible everywhere
  Function scope   let/const/var inside a function
  Block scope      let/const inside {} (var ignores this)

SCOPE CHAIN        inner → outer → global (never inward)
LEXICAL SCOPE      scope determined by WHERE written, not where called
SHADOWING          inner var with same name hides outer one
─────────────────────────────────────────────────────
CLOSURE
  Definition       function retains access to its outer scope variables
                   even after that outer scope has finished
  Captures         the VARIABLE (live reference), not a snapshot of its value
  Each invocation  creates a fresh, independent closure
─────────────────────────────────────────────────────
CLASSIC BUG        var in loop + async callback → all share same var
FIX                use let (new binding per iteration)

IIFE               (() => { })()  — create + run scope immediately
MEMORY             closures keep outer scopes alive — avoid closing over large data
─────────────────────────────────────────────────────
PATTERNS
  Private state       close over a variable, expose methods only
  Factory function    return a function with baked-in variables
  Memoization         close over a cache object
  Once function       close over a hasRun flag
  Module pattern      IIFE returning public API (pre-ES6)
─────────────────────────────────────────────────────
```

---

## Connected topics

- **03 — Type coercion / 08 — Truthy-Falsy** — scope rules apply to all variable types
- **14 — Function declarations** — hoisting moves declarations to top of their scope
- **15 — Arrow functions** — lexical `this`; arrow functions close over `this` from outer scope
- **17 — Return values** — returning a function IS how you create a closure
- **19 — Hoisting** — how `var`, `let`, `const`, and function declarations behave at scope entry (next topic)
- **41 — Higher-order functions** — functions returning functions; closure is what makes them work
- **42 — Currying** — `a => b => a + b` is two nested closures
- **React hooks** — `useState` setter closes over state; `useEffect` closes over props/state from render
