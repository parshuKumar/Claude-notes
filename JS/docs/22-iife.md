# 22 — IIFE (Immediately Invoked Function Expression)

## What is this?
An IIFE (pronounced "iffy") is a function that is **defined and called in the same expression** — it runs the instant the JavaScript engine reaches it. Think of it like a self-destructing note: you write it, it executes once, and the variables inside it vanish with it. Nothing leaks out into the surrounding scope unless you deliberately return it.

## Why does it matter?
IIFEs were the primary way JavaScript developers created **isolated scopes** before ES6 introduced `let`, `const`, and modules. You'll encounter them constantly in older code, third-party libraries, minified bundles, and polyfills. Even in modern code they still have specific use cases: isolating setup logic, avoiding global pollution, and creating async blocks inline.

---

## Syntax

```js
// Basic IIFE — anonymous
(function () {
  console.log("I run immediately");
})();

// Arrow function IIFE
(() => {
  console.log("Arrow IIFE");
})();

// IIFE with parameters
(function (appName, version) {
  console.log(appName + " v" + version);
})("MyApp", "2.0");
// MyApp v2.0

// IIFE that returns a value
const config = (function () {
  const env = "production";          // private
  return {                           // only this is exposed
    env,
    debug: env !== "production"
  };
})();

console.log(config.env);   // "production"
console.log(config.debug); // false
// console.log(env);        // ❌ ReferenceError — env is private
```

---

## How it works — line by line

```js
(function () {
  console.log("I run immediately");
})();
```

Breaking this apart:

```
(  function () { ... }  )  ()
 ↑                      ↑  ↑
 wrapping parens         |  call parens — invoke it NOW
 make it an expression   |
                         function expression (not a declaration)
```

**Why the outer parentheses?**  
Without them, JavaScript sees `function () { ... }` at the start of a statement and parses it as a **function declaration**. Declarations require a name and cannot be called immediately. Wrapping in `( )` forces JavaScript to treat it as an **expression**, which can be invoked. The trailing `()` then calls it.

```js
(function (appName, version) { ... })("MyApp", "2.0");
```
The call parentheses `()` at the end can also receive arguments — they are passed to the function like a normal call.

```js
const config = (function () {
  return { env: "production" };
})();
```
The IIFE runs, returns an object, and that object is assigned to `config`. The inner `env` variable is gone — only the returned value survives.

---

## Example 1 — basic: isolated setup block

```js
// Without IIFE — these variables pollute the global/module scope
const baseUrl = "https://api.example.com";
const timeout = 5000;
const retries = 3;
console.log("Config ready");

// With IIFE — setup runs once, nothing leaks out
(function () {
  const baseUrl = "https://api.example.com";  // private to this IIFE
  const timeout = 5000;
  const retries = 3;

  // Attach only what's needed to a single global object
  window.AppConfig = { baseUrl, timeout, retries };
  console.log("Config ready");
})();

console.log(window.AppConfig.baseUrl); // "https://api.example.com"
// console.log(baseUrl);               // ❌ ReferenceError — gone
```

---

## Example 2 — real world: module-like pattern (pre-ES6)

Before `import`/`export` existed, entire libraries were wrapped in IIFEs to avoid polluting the global scope:

```js
// How jQuery, Lodash, and most old libraries were structured
const CartModule = (function () {
  // Private state — completely inaccessible from outside
  let items = [];
  let totalPrice = 0;

  // Private helper — not exposed
  function recalculateTotal() {
    totalPrice = items.reduce((sum, item) => sum + item.price * item.qty, 0);
  }

  // Public API — what we choose to expose
  return {
    addItem(name, price, qty) {
      items.push({ name, price, qty });
      recalculateTotal();
      console.log(`Added ${qty}x ${name}`);
    },

    removeItem(name) {
      items = items.filter((item) => item.name !== name);
      recalculateTotal();
      console.log(`Removed ${name}`);
    },

    getTotal() {
      return parseFloat(totalPrice.toFixed(2));
    },

    getItems() {
      return [...items];   // return a copy — prevents direct mutation
    }
  };
})();

CartModule.addItem("Keyboard", 79.99, 1);
CartModule.addItem("Mouse", 29.99, 2);
console.log(CartModule.getTotal());    // 139.97
CartModule.removeItem("Mouse");
console.log(CartModule.getTotal());    // 79.99

// console.log(CartModule.items);      // undefined — private
// CartModule.recalculateTotal();      // ❌ TypeError — not exposed
```

This is called the **Revealing Module Pattern** — one of the most important patterns in pre-ES6 JavaScript.

---

## Example 3 — real world: async IIFE (still common in modern JS)

`await` can only be used inside an `async` function. When you need top-level `await`-style code in older environments or at the top of a non-module script:

```js
// Without top-level await support, you need an async IIFE
(async function () {
  try {
    console.log("Fetching user...");

    // Simulate an async operation
    const user = await new Promise((resolve) => {
      setTimeout(() => resolve({ id: 1, name: "Parsh" }), 500);
    });

    console.log("Loaded user:", user.name);

    const orders = await new Promise((resolve) => {
      setTimeout(() => resolve([{ id: 101 }, { id: 102 }]), 300);
    });

    console.log("Orders:", orders.length);
  } catch (error) {
    console.error("Startup failed:", error.message);
  }
})();

// Arrow version (same thing)
(async () => {
  const data = await fetchSomething();
  console.log(data);
})();
```

Even with ES2022 top-level `await` in modules, async IIFEs appear frequently in blog posts, tutorials, and environments that don't support modules.

---

## Example 4 — real world: init scripts and configuration

```js
// App initialisation — run once, set up the environment, expose nothing unnecessary
const App = (function () {
  // Private constants
  const VERSION = "3.1.0";
  const BUILD_DATE = "2026-04-29";
  const FEATURE_FLAGS = {
    darkMode: true,
    betaSearch: false,
    newCheckout: true
  };

  // Private init logic
  function detectEnvironment() {
    return window.location.hostname === "localhost" ? "development" : "production";
  }

  function applyFeatureFlags() {
    if (FEATURE_FLAGS.darkMode) {
      document.body.classList.add("dark");
    }
  }

  // Run setup immediately
  const env = detectEnvironment();
  applyFeatureFlags();
  console.log(`App v${VERSION} (${BUILD_DATE}) starting in ${env} mode`);

  // Only expose what external code genuinely needs
  return {
    version: VERSION,
    env,
    isFeatureEnabled: (flag) => Boolean(FEATURE_FLAGS[flag])
  };
})();

console.log(App.version);                      // "3.1.0"
console.log(App.env);                          // "development" or "production"
console.log(App.isFeatureEnabled("darkMode")); // true
console.log(App.isFeatureEnabled("betaSearch")); // false
```

---

## Tricky things you'll encounter in the real world

### 1 — The four valid IIFE syntaxes (all are correct)

```js
// Style 1 — parens wrap the whole thing (most common)
(function () { console.log(1); }());

// Style 2 — parens wrap only the function (also very common)
(function () { console.log(2); })();

// Style 3 — arrow function (modern, concise)
(() => { console.log(3); })();

// Style 4 — using a unary operator to force expression mode (you'll see this in minified code)
!function () { console.log(4); }();
~function () { console.log(5); }();
void function () { console.log(6); }();
// The !, ~, void operators force expression parsing — same effect as wrapping parens
```

They're all equivalent. The unary operator styles are used by minifiers because they save one character.

---

### 2 — Semicolons before IIFEs (the "defensive semicolon")

```js
// Imagine two files concatenated by a build tool:
// file1.js ends with:
const x = 5

// file2.js starts with:
(function () { ... })()
```

Without a semicolon, these two lines merge into `const x = 5(function () { ... })()` — JavaScript tries to call `5` as a function → `TypeError`.

**Fix — start IIFEs with a defensive semicolon:**
```js
;(function () {
  // safe even if the previous file had no semicolon
})();
```

This is why you'll see `; (function...` in production code.

---

### 3 — IIFEs don't need to be anonymous

```js
// Named IIFE — the name is only visible INSIDE the function (useful for recursion or debugging)
(function initializeApp() {
  console.log("Initializing...");
  // You CAN call initializeApp() recursively here
  // But initializeApp is NOT accessible outside this expression
})();

// console.log(initializeApp); // ❌ ReferenceError
```

Naming IIFEs gives you better stack traces in DevTools without polluting scope.

---

### 4 — IIFEs vs blocks (modern alternative)

In modern code with `let`/`const`, a plain block `{ }` achieves the same scope isolation an IIFE used to:

```js
// Old way — IIFE for scope isolation
(function () {
  const tempResult = heavyCalculation();
  doSomethingWith(tempResult);
})();

// Modern way — block scope with let/const
{
  const tempResult = heavyCalculation();   // block-scoped with const
  doSomethingWith(tempResult);
}
// tempResult is gone here — same effect as the IIFE

// console.log(tempResult); // ❌ ReferenceError — same behaviour
```

Use a block for simple scope isolation. Use an IIFE when you need to:
- Return a value
- Use `async`/`await`
- Pass in arguments
- Be compatible with older environments

---

### 5 — IIFEs in `for` loops — the old closure fix

Before `let`, IIFEs were used to capture loop variables in closures (covered in the closures topic):

```js
// ES5 fix for the var loop bug
for (var i = 0; i < 3; i++) {
  (function (capturedI) {
    setTimeout(function () {
      console.log(capturedI);   // 0, 1, 2  ✅
    }, 100);
  })(i);
}

// Modern way — just use let
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100); // ✅
}
```

You'll see the IIFE version in older codebases and tutorials.

---

### 6 — IIFEs and `this`

```js
// In non-strict mode, `this` inside a plain IIFE is the global object (window/global)
(function () {
  console.log(this); // window (browser) or global (Node) in sloppy mode
                     // undefined in strict mode
})();

// If you need `this`, arrow IIFEs inherit it from the enclosing context
const obj = {
  name: "Parsh",
  init() {
    (() => {
      console.log(this.name); // "Parsh" — arrow IIFE inherits `this` from init()
    })();
  }
};
obj.init();
```

---

## Common mistakes

### ❌ Mistake 1 — Missing the wrapping parentheses

```js
// WRONG — SyntaxError: function declarations cannot be immediately invoked
function () {
  console.log("oops");
}();

// WRONG — named declaration still can't be immediately invoked this way
function init() {
  console.log("oops");
}();   // SyntaxError

// RIGHT — wrap to turn it into an expression
(function () {
  console.log("fixed");
})();
```

---

### ❌ Mistake 2 — Forgetting the call parentheses at the end

```js
// WRONG — this just defines a function expression and discards it
(function () {
  console.log("never runs");
});   // ← no () at the end — never called!

// RIGHT
(function () {
  console.log("runs");
})();  // ← () invokes it
```

---

### ❌ Mistake 3 — Trying to access IIFE-internal variables from outside

```js
const result = (function () {
  const privateData = { user: "Parsh", role: "admin" };
  return privateData.user;   // only the return value escapes
})();

console.log(result);       // "Parsh"  ✅
console.log(privateData);  // ❌ ReferenceError — gone forever
```

If you need data from an IIFE, you must `return` it.

---

## Frequently asked questions

**Q: Are IIFEs still used in modern JavaScript?**  
Yes, in specific situations: async IIFEs for inline async code, module setup in environments without native module support, and whenever you need a scope that runs once and returns a value. But for simple scope isolation, a block `{ }` with `let`/`const` is cleaner.

**Q: What's the difference between an IIFE and a regular function call?**  
A regular function is defined first, then called separately. An IIFE is defined and called in one expression. The function object from an IIFE is never assigned a name or stored — it executes once and is discarded.

**Q: Can an IIFE take parameters?**  
Yes — the call parentheses at the end accept arguments: `(function(a, b) { ... })(arg1, arg2)`. This was a common pattern for passing in `window`, `document`, or `undefined` in old code to protect against them being overridden.

**Q: Why did old libraries use IIFEs?**  
Before ES modules, every JS file shared the same global scope. If two libraries both defined `var helpers = ...`, they'd overwrite each other. Wrapping in an IIFE meant only the single return value (usually one global namespace object like `$` or `_`) was exposed.

**Q: What is the Revealing Module Pattern?**  
It's an IIFE that returns an object of methods. The private state and helper functions stay inside the IIFE; only the returned object is public. Example 2 above is a textbook Revealing Module Pattern.

**Q: Can I use `return` inside an IIFE?**  
Yes — `return` exits the IIFE's function body and the returned value is the result of the IIFE expression. This is exactly how you expose values (see the `config` and `CartModule` examples above).

---

## Practice exercises

### Exercise 1 — easy

Write an IIFE that:
- Has private variables `appName = "TaskTracker"`, `version = "1.0"`, and `maxTasks = 100`
- Logs `"TaskTracker v1.0 — max tasks: 100"` immediately when it runs
- Returns an object exposing only `appName` and `version` (not `maxTasks`)
- Assign the returned object to a `const appInfo`

After the IIFE, log `appInfo.appName`, `appInfo.version`, and confirm that `appInfo.maxTasks` is `undefined`.

```js
// Write your code here
```

---

### Exercise 2 — medium

Rebuild the counter from the closures topic, but this time as an IIFE that **immediately returns the counter API** — no outer factory function.

The IIFE should:
- Accept a `startValue` parameter (pass `10` when invoking it)
- Keep a private `currentCount` set to `startValue`
- Return an object with `increment()`, `decrement()`, `reset()`, and `getCount()` methods

Assign the result to `const scoreCounter`. Call all four methods and log results to confirm they work.

```js
// Write your code here
```

---

### Exercise 3 — hard

Build a **feature flag system** as a Revealing Module Pattern using an IIFE.

Requirements:
- Private `flags` object: `{ darkMode: true, betaSearch: false, newCheckout: true, offlineMode: false }`
- Private `changeLog` array that records every flag change as `"[timestamp] flagName: oldValue → newValue"`
- Expose only these methods:
  - `isEnabled(flagName)` — returns `true`/`false`
  - `enable(flagName)` — sets the flag to `true`, logs a change entry
  - `disable(flagName)` — sets the flag to `false`, logs a change entry
  - `getChangeLog()` — returns a **copy** of the changeLog array (not the original)
- Unknown flag names should log `"Unknown flag: [name]"` and do nothing

Assign the module to `const FeatureFlags`. Then:
1. Check if `betaSearch` is enabled
2. Enable `betaSearch`
3. Disable `darkMode`
4. Try to enable a flag called `nonExistentFlag`
5. Log the full change log

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

**IIFE anatomy:**
```
(  function (param) { ... }  )  (  arg  )
 ↑                            ↑  ↑
 expression wrapper            |  argument passed in
                               call operator — runs immediately
```

**Four valid forms:**
```js
(function () { })();           // classic — most common
(function () { }());           // Crockford style
(() => { })();                 // arrow IIFE
(async () => { await ...; })() // async IIFE
```

**When to use an IIFE:**

| Situation | Prefer |
|---|---|
| Simple scope isolation | Block `{ }` with `let`/`const` |
| Run once + return a value | IIFE |
| Inline async/await block | Async IIFE |
| Private module state (pre-ES6 code) | IIFE (Revealing Module Pattern) |
| Old loop closure bug fix | IIFE (or just use `let`) |
| Top-level await without module support | Async IIFE |

**Key rules:**
- Wrap the function in `( )` to make it an expression
- Add `()` at the end to invoke it — don't forget this
- Variables inside are private — use `return` to expose values
- Use `;` before an IIFE in files that may be concatenated
- Arrow IIFEs inherit `this`; regular IIFEs get their own `this`

---

## Connected topics

- **18 — Scope** — IIFEs create their own scope; everything inside is invisible outside unless returned
- **19 — Closures** — the Revealing Module Pattern returned by an IIFE IS a closure; the returned methods close over the private variables
- **59 — Modules (import/export)** — ES6 modules made IIFEs largely unnecessary for scope isolation; each module file automatically has its own scope
