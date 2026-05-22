# 74 — Debugging

---

## 1. What is this?

**Debugging** is the process of finding and fixing the root cause of incorrect behaviour in your code. It is not guessing — it is a systematic process of forming a hypothesis, proving or disproving it with evidence, and narrowing down the location of the bug.

**Analogy — a doctor diagnosing a patient:**
A good doctor doesn't prescribe medicine at random. They take symptoms, form a hypothesis, run targeted tests, and narrow down the cause. A good debugger does the same: read the error message (symptoms), form a theory, add breakpoints or logs (tests), confirm the theory, fix it.

**Most developers waste time debugging because they:**
- Change code randomly hoping something fixes it
- Add `console.log` everywhere instead of at the right place
- Don't read the full error message and stack trace

---

## 2. Why does it matter?

You will spend at least as much time debugging as writing code — probably more. The difference between a developer who takes 3 minutes to fix a bug and one who takes 3 hours is almost entirely technique, not knowledge. Learning to use the debugger, read stack traces, and isolate problems systematically is one of the highest-leverage skills in software development.

---

## 3. The Debugging Toolkit

| Tool | When to use |
|---|---|
| `console.log` / `console.dir` | Quick inspection of a value |
| `console.error` / `console.warn` | Flag problems in the output stream |
| `console.table` | Arrays of objects — readable in the console |
| `console.group` / `console.groupEnd` | Collapsible log groups for complex flows |
| `console.time` / `console.timeEnd` | Measure how long a block takes |
| `console.trace` | Print a stack trace at the current point |
| `console.assert` | Log only if a condition is false |
| `debugger` statement | Pause execution in DevTools |
| Browser DevTools — Sources | Set breakpoints, step through code, watch variables |
| Node.js `--inspect` | Same DevTools experience for Node.js |
| VS Code debugger | Breakpoints with full IDE integration |
| Error `.stack` property | Full call stack in a string |

---

## 4. Reading Error Messages and Stack Traces

This is the single most important skill. Most developers skip to the code too fast.

```
TypeError: Cannot read properties of undefined (reading 'email')
    at formatUser (utils.js:14:20)
    at users.map (routes/users.js:38:12)
    at Array.map (<anonymous>)
    at getUsers (routes/users.js:37:22)
    at Layer.handle [as handle_request] (express/lib/router/layer.js:95:5)
    ...
```

How to read this:

1. **Error type** — `TypeError` — wrong type, probably `null`/`undefined` somewhere
2. **Message** — `Cannot read properties of undefined (reading 'email')` — something is `undefined` and we tried to access `.email` on it
3. **Top of stack** — `formatUser (utils.js:14:20)` — **this is where it broke** — line 14, column 20 of `utils.js`
4. **Caller** — `users.map (routes/users.js:38:12)` — `formatUser` was called here, inside a `.map()`
5. **Root caller** — `getUsers (routes/users.js:37:22)` — this is the function that started the chain

**Read the stack from top to bottom when following the path. The bug is almost always at the top 1–3 frames in YOUR code** (not the framework frames at the bottom).

---

## 5. `console` Methods in Depth

### `console.log` — the basics and beyond

```js
// Plain logging
console.log("user:", user);

// Multiple values
console.log("id:", id, "name:", name);

// Object shorthand (shows variable name)
console.log({ user, id, name }); // { user: {...}, id: 42, name: "Alice" }

// Styled output in browser
console.log("%cERROR%c message", "color:red;font-weight:bold", "color:inherit");

// Formatted string
console.log("Processed %d items in %dms", count, elapsed);
```

### `console.dir` — inspect object structure

```js
// console.log might show [object Object] for deeply nested things
// console.dir always shows the full object tree
console.dir(document.querySelector("body"), { depth: 5 });

// In Node.js — use util.inspect for full depth
const util = require("util");
console.log(util.inspect(complexObject, { depth: null, colors: true }));
```

### `console.table` — arrays of objects

```js
const users = [
  { id: 1, name: "Alice", role: "admin" },
  { id: 2, name: "Bob",   role: "user" },
];
console.table(users);
// ┌─────────┬────┬───────┬─────────┐
// │ (index) │ id │ name  │  role   │
// ├─────────┼────┼───────┼─────────┤
// │    0    │  1 │ Alice │  admin  │
// │    1    │  2 │  Bob  │  user   │
// └─────────┴────┴───────┴─────────┘

// Show only specific columns
console.table(users, ["name", "role"]);
```

### `console.group` — collapsible sections

```js
console.group("Processing order #123");
  console.log("Validating items...");
  console.log("Calculating total...");
  console.group("Payment");
    console.log("Charging card...");
    console.log("Sending receipt...");
  console.groupEnd();
console.groupEnd();

// console.groupCollapsed — collapsed by default (good for verbose output)
console.groupCollapsed("Request details");
  console.log(req.headers);
  console.log(req.body);
console.groupEnd();
```

### `console.time` — performance measurement

```js
console.time("db-query");
const users = await db.query("SELECT * FROM users");
console.timeEnd("db-query"); // "db-query: 42.3ms"

// Multiple independent timers
console.time("parse");
const data = JSON.parse(largeJson);
console.timeEnd("parse"); // "parse: 12.1ms"

// timeLog — log intermediate times without stopping the timer
console.time("pipeline");
const raw  = await fetchData();
console.timeLog("pipeline", "after fetch");   // "pipeline: 200ms after fetch"
const parsed = transform(raw);
console.timeLog("pipeline", "after transform"); // "pipeline: 215ms after transform"
console.timeEnd("pipeline");
```

### `console.trace` — print stack at any point

```js
function deepFunction() {
  console.trace("How did I get here?");
}

// Output:
// Trace: How did I get here?
//     at deepFunction (app.js:2:11)
//     at middleFunction (app.js:6:3)
//     at topFunction (app.js:10:3)
//     ...
```

### `console.assert` — conditional logging

```js
// Only logs if condition is FALSE
console.assert(user !== null, "user should not be null", user);
console.assert(arr.length > 0, "Array must not be empty");

// No output when true
console.assert(1 + 1 === 2, "Math is broken"); // silent
```

### `console.count` — count how many times a code path runs

```js
function processItem(item) {
  console.count("processItem called");
  // ...
}
// After 5 calls:
// processItem called: 1
// processItem called: 2
// ...
// processItem called: 5

console.countReset("processItem called"); // reset to 0
```

---

## 6. The `debugger` Statement

Place `debugger` anywhere in your code. When DevTools is open, execution pauses at that exact line.

```js
async function processOrder(order) {
  const items = await getItems(order.id);
  debugger; // ← execution pauses here with DevTools open
  const total = calculateTotal(items);
  return total;
}
```

- In **browser DevTools** — pauses at the line, shows local variables in the Scope panel
- In **Node.js** with `--inspect` — same behaviour
- With **no DevTools open** — `debugger` is silently ignored

**Remove all `debugger` statements before committing.** Use a linter rule (`no-debugger`) to enforce this.

---

## 7. Browser DevTools — Sources Panel

The most powerful tool available. The key concepts:

### Breakpoints

```
Line breakpoint   — click the line number gutter — pauses at that line
Conditional breakpoint — right-click gutter → "Add conditional breakpoint"
                       e.g.: user.id === 42  — only pauses for that user
Logpoint          — right-click gutter → "Add logpoint"
                       e.g.: "user is {user.name}" — logs without pausing
Exception breakpoint — checkbox "Pause on caught/uncaught exceptions"
                       pauses the moment any error is thrown
```

### Stepping controls

```
▶  Resume (F8)         — continue until next breakpoint
↷  Step over (F10)     — run the next line, don't enter function calls
↓  Step into (F11)     — enter the function called on this line
↑  Step out  (Shift+F11) — run to the end of the current function and pause at caller
```

### The Scope panel

When paused at a breakpoint, the Scope panel shows:
- **Local** — variables in the current function
- **Closure** — variables captured from outer scopes
- **Global** — window / globalThis

You can **edit variable values** directly in the Scope panel during a pause — useful for testing "what if this was `null`?"

### The Watch panel

Add any expression to Watch — it re-evaluates every time execution pauses:
```
users.filter(u => u.active).length
order.items.reduce((s, i) => s + i.price, 0)
```

### The Call Stack panel

Shows the exact chain of function calls that led to the current pause point — the same information as a stack trace, but interactive. Click any frame to jump to that function and inspect its local variables.

---

## 8. Node.js Debugging

### `--inspect` flag

```bash
# Start Node with inspector
node --inspect server.js

# Or break immediately on start (useful for startup bugs)
node --inspect-brk server.js
```

Then open `chrome://inspect` in Chrome — click "inspect" under Remote Target. You get the exact same DevTools experience as browser code.

### VS Code debugger

Create `.vscode/launch.json`:

```json
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "launch",
      "name": "Debug server",
      "program": "${workspaceFolder}/server.js",
      "restart": true,
      "runtimeArgs": ["--inspect"]
    },
    {
      "type": "node",
      "request": "attach",
      "name": "Attach to running Node",
      "port": 9229
    }
  ]
}
```

Set breakpoints by clicking the gutter in VS Code — no `debugger` statement needed. The VS Code debugger has the same Scope, Watch, and Call Stack panels as browser DevTools.

### `util.inspect` — deep object printing in Node

```js
const { inspect } = require("util"); // or import

// Default console.log truncates deeply nested objects
console.log(bigObject); // [Object: null prototype] { ... } — truncated

// util.inspect shows everything
console.log(inspect(bigObject, {
  depth: null,    // no depth limit
  colors: true,   // ANSI colours in terminal
  compact: false  // one property per line
}));
```

---

## 9. Debugging Async Code

Async code is harder to debug because the call stack is often empty by the time an error surfaces.

### The async stack trace problem

```js
async function c() { throw new Error("async error"); }
async function b() { await c(); }
async function a() { await b(); }

// In Node.js < 12 — stack trace was nearly empty
// In modern Node.js / Chrome — async frames are shown:
// Error: async error
//     at c (app.js:1:27)
//     at async b (app.js:2:18)   ← "async" prefix
//     at async a (app.js:3:18)
```

Modern V8 tracks async call stacks. Keep your functions small and named (not anonymous arrows) so the trace is readable.

### Debugging unhandled rejections

```js
// Add this at the top of your entry file while debugging
process.on("unhandledRejection", (reason, promise) => {
  console.error("Unhandled Rejection at:", promise);
  console.error("Reason:", reason);
  console.error("Stack:", reason?.stack);
});
```

### Common async debugging technique — add await

```js
// ❌ Hard to debug — errors surface far from source
const [a, b, c] = await Promise.all([fa(), fb(), fc()]);

// ✅ During debugging — run sequentially to isolate which one failed
const a = await fa();
const b = await fb();
const c = await fc();
```

Once you know which one fails, restore `Promise.all`.

---

## 10. Debugging Patterns

### Pattern 1 — Binary search (bisect) for the bug

When you have a long function or many steps and you don't know where it breaks:

```js
async function longProcess(data) {
  const step1 = transform(data);      // ← add log here first
  const step2 = enrich(step1);
  const step3 = validate(step2);
  const step4 = await save(step3);    // ← if step1 log is good, add one here
  return step4;
}
```

Don't log everything — log the midpoint first. If midpoint is correct, the bug is in the second half. If incorrect, it's in the first half. Repeat. This finds bugs in O(log n) steps instead of O(n).

### Pattern 2 — Rubber duck debugging

Explain the code out loud (or in writing) line by line as if explaining to someone who knows nothing about JavaScript. The act of verbalising forces you to state every assumption explicitly — and the assumption you haven't stated is usually the bug.

### Pattern 3 — Reproduce minimally

Reduce the failing case to the smallest possible input that still fails. This eliminates noise and often reveals the bug immediately.

```js
// ❌ Debugging with full production data object — 50 fields
processUser(fullProductionUser);

// ✅ Find the minimal input that still fails
processUser({ id: null }); // if this fails too, the problem is about null id
processUser({ id: 1 });    // if this works, it confirms null id is the issue
```

### Pattern 4 — Check your assumptions with `console.assert`

```js
async function processOrder(orderId) {
  const order = await getOrder(orderId);
  console.assert(order !== null, "Expected order to exist", { orderId });
  console.assert(Array.isArray(order.items), "Expected items to be array", order);
  console.assert(order.items.length > 0, "Expected at least one item", order);

  // Now proceed knowing assertions passed
  const total = order.items.reduce((s, i) => s + i.price, 0);
}
```

### Pattern 5 — Log the type, not just the value

```js
// ❌ Value looks right but type is wrong — e.g., "42" instead of 42
console.log("id:", id); // "42" — looks fine but it's a string

// ✅ Always log type when debugging type issues
console.log("id:", id, typeof id);          // "42" string
console.log("id:", id, id instanceof Array); // check type more specifically
console.log({ id, type: typeof id });        // structured
```

---

## 11. Real-World Debugging Scenarios

### Scenario 1 — "Cannot read properties of undefined"

```
TypeError: Cannot read properties of undefined (reading 'name')
    at renderUser (ui.js:22:30)
```

**Step 1:** Go to `ui.js:22`. See `user.name` — something called `renderUser` with `undefined`.
**Step 2:** Find all callers of `renderUser`. Look for where user data comes from.
**Step 3:** Add log before the call:
```js
console.log("renderUser called with:", user, typeof user);
```
**Step 4:** Likely cause — async data not awaited, array index out of bounds, or missing optional chaining. Fix:
```js
// If user might be undefined:
user?.name ?? "Unknown"

// If user comes from an array:
const user = users.find(u => u.id === id); // returns undefined if not found
// Add guard:
if (!user) throw new NotFoundError("User");
```

### Scenario 2 — Infinite loop / hang

If the page freezes or Node stops responding:

```js
// ❌ Classic infinite loop — condition never becomes false
let i = 0;
while (i < arr.length) {
  processItem(arr[i]);
  // forgot i++
}

// Debugging technique: add a safety counter
let i = 0, safety = 0;
while (i < arr.length) {
  if (++safety > 10000) { console.error("Possible infinite loop at i:", i); break; }
  processItem(arr[i]);
  i++;
}
```

### Scenario 3 — Event listener fires too many times

```js
// Problem: handler runs 5 times per click
// Cause: addEventListener called inside a re-render loop

// Debugging: add a counter
let callCount = 0;
btn.addEventListener("click", () => {
  console.log("click fired, total:", ++callCount);
  console.trace("listener registered at:"); // shows where addEventListener was called
});
```

Fix: move `addEventListener` outside the loop, or use `{ once: true }`, or use event delegation (topic 69).

### Scenario 4 — Wrong value in async code (stale closure)

```js
// Bug: all console.logs print the same value (5)
for (var i = 0; i < 5; i++) {
  setTimeout(() => console.log(i), 100); // 5 5 5 5 5 — var is function-scoped
}

// Fix: use let (block-scoped) or wrap in IIFE
for (let i = 0; i < 5; i++) {  // let creates a new binding per iteration
  setTimeout(() => console.log(i), 100); // 0 1 2 3 4
}
```

### Scenario 5 — Express route not matching

```js
// Problem: route handler never fires

// Debug with a global wildcard first:
app.use("*", (req, res, next) => {
  console.log(`${req.method} ${req.path}`); // see what path Express actually receives
  next();
});

// Common causes:
// 1. Path mismatch — "/users/:id" vs "/user/:id" (typo)
// 2. Middleware order — route defined after wildcard catch-all
// 3. Method mismatch — app.get() but client sends POST
// 4. Body parser not set up — req.body is undefined
```

---

## 12. Tricky Things

### 1. `console.log` shows objects by reference — not by value at that moment

```js
const obj = { a: 1 };
console.log(obj);     // in browser: shows CURRENT state when you expand it in DevTools
obj.a = 999;
// if you expand the log now, you see { a: 999 } — not { a: 1 }!

// ✅ Snapshot with JSON or spread
console.log(JSON.parse(JSON.stringify(obj))); // value at this moment
console.log({ ...obj });                       // shallow copy — safe for flat objects
```

### 2. `debugger` in production code is invisible but wasteful

```js
debugger; // silently ignored without DevTools, but still parsed
// Use linting: "no-debugger": "error" in ESLint config
```

### 3. Minified code makes stack traces unreadable — use source maps

```js
// In webpack/Vite/esbuild config:
// { devtool: "source-map" } — maps minified lines back to original source
// Then DevTools shows your actual file and line, not bundle.min.js:1
```

### 4. `typeof null === "object"` — classic JS quirk in debugging

```js
// When checking for objects, null passes object check
typeof null;           // "object" — not "null"!
typeof {};             // "object"
typeof [];             // "object"
typeof null === "object" && null !== null; // never happens — null !== null is false

// ✅ Null check
if (value !== null && typeof value === "object") { ... }
// or:
if (value && typeof value === "object") { ... }
```

### 5. Async stack traces are only available in modern environments

```js
// Node.js < 12, or with --harmony flags, async stack traces may be empty
// Upgrade Node.js, or use --async-stack-traces flag in older versions
node --async-stack-traces server.js
```

---

## 13. Common Mistakes

### Mistake 1 — Logging without structure

```js
// ❌ Hard to parse in logs with many messages
console.log("user");
console.log(user);
console.log("after filter");

// ✅ One structured log with context
console.log("Processing user:", { userId: user.id, step: "after-filter", count: users.length });
```

### Mistake 2 — Leaving debug logs in production

```js
// ❌ Sensitive data in production logs
console.log("password:", req.body.password);
console.log("token:", authToken);

// ✅ Use a logger with log levels (pino, winston)
logger.debug({ userId }, "Processing request"); // only outputs in development
logger.info({ path: req.path }, "Request received"); // always outputs
```

### Mistake 3 — Modifying code to "see if it fixes it" without understanding why

This is the most common debugging mistake. Every code change should be based on a hypothesis: "I believe X is the cause because of Y evidence. I'll change Z to test that hypothesis."

### Mistake 4 — Not reading the full error message

The message almost always tells you exactly what went wrong. "Cannot read properties of undefined (reading 'email')" literally says: `undefined.email`. The fix is to find where `undefined` came from — which is one level up in the stack trace.

### Mistake 5 — Using `alert()` for debugging

```js
// ❌ Blocks execution, changes timing, impossible in Node.js
alert("got here");

// ✅ console.log, breakpoints, or debugger
console.log("got here");
debugger;
```

---

## 14. Practice Exercises

### Easy — console mastery

Write a function `debugLog(label, data)` that:
- Logs a collapsible group with `label` as the title
- Inside: logs `data` with `console.dir`
- Logs `typeof data` and (if object) `Object.keys(data).length`
- Measures time from function entry to exit with `console.time`

---

### Medium — bug hunt

The following code has three bugs. Find and fix all three without running the code first — use only reading and reasoning. Then verify with a debugger or logs.

```js
async function loadDashboard(userId) {
  const user = await fetchUser(userId);
  const orders = await fetchOrders(user.id);

  const total = orders.reduce((sum, order) => {
    return sum + order.total;
  }, "0");                          // Bug 1

  const recent = orders
    .filter(o => o.date > Date.now() - 30 * 24 * 60 * 60)  // Bug 2
    .sort((a, b) => a.date - b.date);

  return {
    user,
    orders: recent,
    total,
    averageOrder: total / orders.length  // Bug 3
  };
}
```

---

### Hard — build a debug utility

Build a `Debugger` class that can be used as a debugging wrapper around any async function:

```js
const debug = new Debugger({ label: "processOrder", verbose: true });

const result = await debug.wrap(processOrder, orderId);
```

Requirements:
- Logs the function name, arguments (sanitised — no passwords), and start time
- Logs the result or the error (with full stack)
- Measures execution time with `console.time`
- Has a `verbose` flag — when false, only logs errors
- Has a `sanitize(args)` method that replaces any arg with key `"password"` or `"token"` with `"[REDACTED]"`
- Works with both sync and async functions

---

## 15. Quick Reference

```
CONSOLE METHODS
  console.log(val)              basic output
  console.log({ val })          shows variable name
  console.dir(obj)              always shows full object tree
  console.table(arr)            pretty-print arrays of objects
  console.group("label")        collapsible section start
  console.groupEnd()            end section
  console.groupCollapsed(lbl)   collapsed by default
  console.time("label")         start timer
  console.timeEnd("label")      stop and print elapsed
  console.timeLog("label", msg) print elapsed without stopping
  console.trace("msg")          print stack trace here
  console.assert(bool, msg)     log only when condition is false
  console.count("label")        count calls
  console.error(val)            red output, adds stack trace

BREAKPOINTS (DevTools)
  line breakpoint        click gutter
  conditional breakpoint right-click gutter → condition
  logpoint               right-click gutter → log message
  exception breakpoint   checkbox in DevTools Sources

STEP CONTROLS
  F8  / Resume           continue to next breakpoint
  F10 / Step over        next line, skip into function calls
  F11 / Step into        enter the called function
  Shift+F11 / Step out   finish current function, pause at caller

NODE.JS
  node --inspect server.js          open in chrome://inspect
  node --inspect-brk server.js      break immediately on start
  Error.stackTraceLimit = 50        increase stack depth

SNAPSHOT OBJECTS
  console.log(JSON.parse(JSON.stringify(obj)))   deep snapshot
  console.log({ ...obj })                        shallow snapshot

KEY RULES
  Read the full error message before touching code
  Top of stack = where it broke (in your code)
  Binary search — log midpoint, not everything
  Every code change = a testable hypothesis
  Never commit: debugger, console.log, alert()
```

---

## 16. Connected Topics

- **71 — Types of JS errors** — understanding what `TypeError`, `RangeError` etc. mean in a stack trace
- **72 — try/catch/finally** — catching and logging errors systematically
- **73 — Custom error classes** — custom `.name` and `.stack` in error output
- **52 — Async JS** — async call stacks, unhandled rejections
- **68 — Events** — debugging event listener accumulation
- **Node.js** — `--inspect`, `process.on("unhandledRejection")`, structured logging with pino/winston
