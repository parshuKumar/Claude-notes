# 64 — Logical Assignment Operators

## What is this?

**Logical assignment operators** are three shorthand operators introduced in ES2021 that combine a logical operation (`&&`, `||`, `??`) with assignment. They only assign a value if the logical condition determines it's needed. Think of them as "smart assignment" — they ask a question first, and only write a value if the answer says to.

```js
// These three are logical assignment operators:
x ||= y    // "assign y to x ONLY if x is falsy"
x &&= y    // "assign y to x ONLY if x is truthy"
x ??= y    // "assign y to x ONLY if x is null or undefined"
```

---

## Why does it matter?

These operators eliminate common verbose patterns in JavaScript — setting defaults, lazy initialisation, and conditional updates — replacing 3–5 line patterns with a single expressive line. You'll see them constantly in modern Node.js codebases, configuration handling, and Vue/React code.

---

## Syntax

```js
// Long form → short form:

// OR assignment (||=)
if (!x) x = y;          // old — but semantically wrong (tests truthiness)
x = x || y;             // old
x ||= y;                // ✓ new — assign y if x is falsy

// AND assignment (&&=)
if (x) x = y;           // old
x = x && y;             // old
x &&= y;                // ✓ new — assign y if x is truthy

// Nullish assignment (??=)
if (x === null || x === undefined) x = y;   // old
x = x ?? y;             // old — but doesn't assign (just evaluates)
x ??= y;                // ✓ new — assign y if x is null/undefined
```

---

## How each operator works

### `||=` — OR assignment

Assigns the right-hand value only if the left-hand side is **falsy** (`false`, `0`, `""`, `null`, `undefined`, `NaN`).

```js
let a = 0;
a ||= 42;
console.log(a);   // 42  — 0 is falsy → assigned

let b = 5;
b ||= 42;
console.log(b);   // 5   — 5 is truthy → NOT assigned

let c = "";
c ||= "default";
console.log(c);   // "default" — empty string is falsy → assigned

let d = false;
d ||= true;
console.log(d);   // true — false is falsy → assigned
```

**Key distinction:** `||=` treats `0`, `""`, and `false` as "missing" — it will overwrite them. This is often a bug when `0` or `""` are valid values.

### `&&=` — AND assignment

Assigns the right-hand value only if the left-hand side is **truthy**.

```js
let a = 5;
a &&= 99;
console.log(a);   // 99  — 5 is truthy → assigned

let b = 0;
b &&= 99;
console.log(b);   // 0   — 0 is falsy → NOT assigned

let c = null;
c &&= "value";
console.log(c);   // null — null is falsy → NOT assigned

// Common use: update a property only if the object exists
let user = { name: "Alice" };
user &&= { ...user, updatedAt: new Date() };
// user is truthy → assigns the updated object

let guest = null;
guest &&= { ...guest, updatedAt: new Date() };
// guest is null (falsy) → nothing happens, no error
```

### `??=` — Nullish assignment

Assigns the right-hand value only if the left-hand side is **`null` or `undefined`** (and nothing else).

```js
let a = null;
a ??= "default";
console.log(a);   // "default" — null → assigned

let b = undefined;
b ??= "default";
console.log(b);   // "default" — undefined → assigned

let c = 0;
c ??= "default";
console.log(c);   // 0 — 0 is NOT null/undefined → NOT assigned ← key difference from ||=

let d = "";
d ??= "default";
console.log(d);   // "" — empty string is NOT null/undefined → NOT assigned

let e = false;
e ??= true;
console.log(e);   // false — false is NOT null/undefined → NOT assigned
```

**`??=` is almost always what you actually want** when setting defaults, because `0`, `""`, and `false` are legitimate values that shouldn't be overwritten.

---

## How it works — short-circuit evaluation

These operators short-circuit: the right-hand side is **not evaluated at all** if the condition is not met. This matters when the right side has side effects (function calls, expensive computations).

```js
let calls = 0;
function expensive() {
  calls++;
  return "computed";
}

let a = "existing";
a ||= expensive();   // a is truthy → expensive() is NEVER called
console.log(calls);  // 0

let b = null;
b ??= expensive();   // b is null → expensive() IS called
console.log(calls);  // 1
console.log(b);      // "computed"
```

This is different from:
```js
// This ALWAYS evaluates expensive() even when it won't be assigned:
a = a || expensive();   // expensive() always runs, result just might not be used
// ↑ Not true — || also short-circuits in expressions. But the key difference is:
// a = a || y   ALWAYS performs the assignment (even if value doesn't change)
// a ||= y      ONLY performs the assignment if needed (important for reactive systems!)
```

---

## The assignment difference — why it matters for reactivity

The subtle but important difference between `a = a || y` and `a ||= y`:

```js
// a = a || y — ALWAYS assigns (triggers setters/Proxy traps every time)
// a ||= y    — only assigns when the value actually changes

const obj = {
  get count() { return this._count ?? 0; },
  set count(v) {
    console.log("setter called with", v);
    this._count = v;
  },
};

obj._count = 5;

// Old way — ALWAYS triggers setter, even when value doesn't change:
obj.count = obj.count || 5;   // logs: setter called with 5 (even though value was already 5)

// New way — only triggers setter when actually assigning:
obj.count ||= 5;               // obj.count is 5 (truthy) → setter NOT called
obj._count = 0;
obj.count ||= 5;               // obj.count is 0 (falsy) → setter called with 5
```

In Vue 3 / reactive systems, unnecessary assignment triggers re-renders. `||=` avoids this.

---

## Real world examples

### Setting default configuration values

```js
// Reading config with defaults — common in Node.js

// BEFORE (verbose):
function initConfig(options) {
  if (!options.port)    options.port    = 3000;
  if (!options.host)    options.host    = "localhost";
  if (!options.timeout) options.timeout = 5000;
  return options;
}

// WITH ??= (correct — 0 and "" are valid config values):
function initConfig(options) {
  options.port    ??= 3000;
  options.host    ??= "localhost";
  options.timeout ??= 5000;
  return options;
}

// PORT=0 (disable server) would be preserved correctly:
initConfig({ port: 0, timeout: 10000 });
// → { port: 0, host: "localhost", timeout: 10000 }
// (||= would have replaced port: 0 with 3000 — wrong!)
```

### Lazy initialisation of object properties

```js
// Build up nested data structures safely

const cache = {};

function getOrCreate(key) {
  cache[key] ??= [];   // only initialise if not already there
  return cache[key];
}

getOrCreate("users").push("Alice");
getOrCreate("users").push("Bob");
getOrCreate("posts").push("Post 1");

console.log(cache);
// { users: ["Alice", "Bob"], posts: ["Post 1"] }

// ---

// Building grouped data (like reduce + groupBy):
function groupBy(array, key) {
  const result = {};
  for (const item of array) {
    result[item[key]] ??= [];       // create group only if missing
    result[item[key]].push(item);
  }
  return result;
}
```

### Conditional update

```js
// Only update a user's last-seen if they are logged in
function recordActivity(session) {
  session.user &&= {
    ...session.user,
    lastSeen:    new Date(),
    activityCount: (session.user.activityCount ?? 0) + 1,
  };
  // If session.user is null/undefined, nothing happens — no error
}

// ---

// Update cache entry only if it already exists:
function updateCacheIfPresent(cache, key, newData) {
  cache[key] &&= { ...cache[key], ...newData, updatedAt: Date.now() };
  // If cache[key] doesn't exist, no spurious creation
}
```

### Express.js / Node.js middleware

```js
// Ensure request context always has required fields:
app.use((req, res, next) => {
  req.context       ??= {};
  req.context.user  ??= null;
  req.context.flags ??= {};
  req.context.timing ??= { start: Date.now() };
  next();
});

// Accumulate request metadata safely:
app.use((req, res, next) => {
  req.metadata ??= { tags: [], breadcrumbs: [] };
  req.metadata.tags &&= [...req.metadata.tags, "api-v2"];
  next();
});
```

### Merging options safely

```js
class HttpClient {
  constructor(options = {}) {
    // ??= for things where 0/false/empty-string are valid:
    options.timeout    ??= 10_000;
    options.maxRetries ??= 3;
    options.baseUrl    ??= "";

    // ||= for things where only truthy values are meaningful:
    options.method     ||= "GET";   // "GET" is the only sensible default here

    this.options = options;
  }
}

new HttpClient({ timeout: 0 });    // timeout: 0 preserved (??= respects 0)
new HttpClient({ method: "" });    // method: "GET" (||= replaces empty string)
```

---

## The three operators compared side by side

```js
let x;

// Same initial value, different operators:
x = null;     x ||= "A";   console.log(x);  // "A"  — null is falsy
x = null;     x &&= "A";   console.log(x);  // null — null is falsy, no assign
x = null;     x ??= "A";   console.log(x);  // "A"  — null triggers ??=

x = undefined; x ||= "B";  console.log(x);  // "B"
x = undefined; x &&= "B";  console.log(x);  // undefined
x = undefined; x ??= "B";  console.log(x);  // "B"

x = 0;        x ||= "C";   console.log(x);  // "C"  — 0 is falsy → ||= triggers!
x = 0;        x &&= "C";   console.log(x);  // 0
x = 0;        x ??= "C";   console.log(x);  // 0    — 0 is NOT null/undefined → preserved

x = "";       x ||= "D";   console.log(x);  // "D"  — "" is falsy → ||= triggers!
x = "";       x &&= "D";   console.log(x);  // ""
x = "";       x ??= "D";   console.log(x);  // ""   — "" is NOT null/undefined → preserved

x = false;    x ||= "E";   console.log(x);  // "E"  — false is falsy → ||= triggers!
x = false;    x &&= "E";   console.log(x);  // false
x = false;    x ??= "E";   console.log(x);  // false — false is NOT null/undefined → preserved

x = "hello";  x ||= "F";   console.log(x);  // "hello" — truthy → not assigned
x = "hello";  x &&= "F";   console.log(x);  // "F"     — truthy → assigned
x = "hello";  x ??= "F";   console.log(x);  // "hello" — not null/undefined → not assigned
```

---

## Tricky things

### 1. `||=` will overwrite `0`, `""`, and `false` — usually a bug

```js
// WRONG — using ||= when ??= was intended
function setLimit(options) {
  options.maxItems ||= 100;   // BUG: maxItems: 0 (unlimited) → replaced with 100!
}
setLimit({ maxItems: 0 });
// → { maxItems: 100 } — 0 was "unlimited" but now it's 100

// RIGHT
function setLimit(options) {
  options.maxItems ??= 100;   // 0 is preserved
}
setLimit({ maxItems: 0 });
// → { maxItems: 0 }  — correct
```

### 2. These work on any assignable target — including object properties and array indices

```js
const user = { name: "Alice", role: null };
user.role ??= "viewer";   // role was null → assigned "viewer"
console.log(user.role);   // "viewer"

const arr = [1, null, 3];
arr[1] ??= 99;
console.log(arr);  // [1, 99, 3]
```

### 3. Cannot use these on `const`

```js
const x = null;
x ??= "default";   // TypeError: Assignment to constant variable
// These are still assignments — they follow the same const rules
```

### 4. `&&=` with functions — replaces the reference, not mutates

```js
// WRONG assumption — this reassigns the variable, not a property
let arr = [1, 2, 3];
arr &&= arr.filter(x => x > 1);   // arr is now [2, 3] — arr variable re-assigned to new array
// This is fine if intentional, but not a mutation of the original array

// For in-place mutation, use array methods directly:
if (arr) arr.splice(0, arr.length, ...arr.filter(x => x > 1));
```

### 5. Chaining logical assignments — reads left to right, may have unexpected truthiness

```js
let a = 0;
let b = null;

// These are separate statements:
a ??= 5;   // a = 0 (not null/undefined — not assigned)
b ??= 10;  // b = 10

// You cannot chain them meaningfully in one expression:
a ??= b ??= 20;   // valid but unusual: b ??= 20 runs first → b = 20, then a ??= 20 → a = 0
```

---

## Common mistakes

### Mistake 1 — Using `||=` instead of `??=` for numeric or boolean defaults

```js
// WRONG — ||= with numbers
function createTimer(options = {}) {
  options.delay    ||= 1000;  // BUG: delay: 0 (immediate) → replaced with 1000
  options.interval ||= 500;   // BUG: interval: 0 (no repeat) → replaced with 500
}

// RIGHT — ??= for numeric/boolean/string defaults
function createTimer(options = {}) {
  options.delay    ??= 1000;
  options.interval ??= 500;
}
```

### Mistake 2 — Forgetting logical assignment short-circuits (no assignment when condition not met)

```js
// Thinking ??= always initialises the property:
const config = {};
config.timeout ??= computeTimeout();   // only calls computeTimeout() if config.timeout is null/undefined

// If config.timeout = 0 (falsy but not null/undefined):
const config2 = { timeout: 0 };
config2.timeout ??= computeTimeout();  // computeTimeout() NOT called, timeout stays 0
// This is correct behaviour — just know it
```

### Mistake 3 — Using `&&=` when you want to check existence before creating

```js
// WRONG intention — this REPLACES an existing truthy value
const obj = { list: [1, 2, 3] };
obj.list &&= [];     // obj.list is truthy → REPLACES with [] — wrong!

// RIGHT — use ??= to only set if missing
obj.list ??= [];     // obj.list exists → not replaced

// RIGHT — use && with method call to operate ON the existing value
obj.list &&= [...obj.list, 4];   // appends 4 — intentional replacement
```

---

## Practice exercises

### Exercise 1 — easy

Given this object:
```js
const settings = {
  theme:      "dark",
  fontSize:   0,        // 0 = use system default
  language:   "",       // empty = auto-detect
  timeout:    null,
  maxRetries: undefined,
  debug:      false,
};
```

Using only `??=`, `||=`, and `&&=`:
1. Set `theme` to `"light"` if not already set (truthy check)
2. Set `fontSize` to `14` only if null or undefined (not if 0)
3. Set `language` to `"en"` if falsy
4. Set `timeout` to `5000` only if null or undefined
5. Set `maxRetries` to `3` only if null or undefined
6. If `debug` is truthy, add a `debugLevel` property set to `"verbose"`

Log the final object and explain which operator you chose for each and why.

```js
// Write your code here
```

### Exercise 2 — medium

Build a `ConfigManager` class that:
- Stores config in a private object
- `set(key, value)` — sets the value (always)
- `setDefault(key, value)` — sets value ONLY if key is `null` or `undefined` — use `??=`
- `setIfTruthy(key, value)` — sets value ONLY if key is currently falsy — use `||=`
- `setIfPresent(key, value)` — updates value ONLY if key is currently truthy — use `&&=`
- `get(key)` — returns value
- `merge(defaults)` — applies `??=` for every key in `defaults` to this config
- `toString()` — returns JSON string of all non-null config values

```js
const cm = new ConfigManager();
cm.setDefault("host", "localhost");   // sets host
cm.setDefault("host", "example.com"); // no-op — host is already set
cm.set("port", 0);
cm.setIfTruthy("port", 3000);         // no-op — port is 0 (falsy)
cm.setDefault("port", 8080);          // no-op — port is 0 (not null/undefined)
cm.set("debug", true);
cm.setIfPresent("debug", false);      // sets — debug was truthy
cm.merge({ host: "override.com", timeout: 5000 });
// host stays "localhost" (??= — already set)
// timeout = 5000 (??= — was undefined)

console.log(cm.get("host"));     // "localhost"
console.log(cm.get("port"));     // 0
console.log(cm.get("debug"));    // false
console.log(cm.get("timeout"));  // 5000
```

```js
// Write your code here
```

### Exercise 3 — hard

Build a **request context accumulator** for an Express.js-style middleware pipeline:

```js
// Each middleware receives `req` and can attach data to `req.ctx`
// using logical assignment to safely build up context

function createMiddlewarePipeline(...middlewares) {
  // Returns an async function (req, res) that runs all middlewares in order
  // Each middleware gets (req, res, next) where next() proceeds to the next one
}

// Middlewares:
const initContext    = (req, res, next) => { req.ctx ??= {}; next(); };
const attachUser     = (req, res, next) => { req.ctx.user ??= { id: null, role: "guest" }; next(); };
const setAuthUser    = (req, res, next) => { /* simulates auth — sets req.ctx.user if header present */
  req.ctx.user &&= { ...req.ctx.user, id: req.headers["x-user-id"] ?? null };
  next();
};
const trackTiming    = (req, res, next) => { req.ctx.timing ??= { start: Date.now() }; next(); };
const addFlags       = (req, res, next) => {
  req.ctx.flags ??= {};
  req.ctx.flags.authenticated &&= !!req.ctx.user?.id;
  next();
};
const finalise       = (req, res, next) => {
  req.ctx.timing &&= { ...req.ctx.timing, end: Date.now() };
  next();
};

// Test the pipeline with two simulated requests
const pipeline = createMiddlewarePipeline(
  initContext, attachUser, setAuthUser, trackTiming, addFlags, finalise
);

// Request 1: authenticated user
const req1 = { headers: { "x-user-id": "user-42" } };
await pipeline(req1, {});
console.log(req1.ctx);

// Request 2: unauthenticated
const req2 = { headers: {} };
await pipeline(req2, {});
console.log(req2.ctx);
```

Also write a `mergeContexts(contexts)` function that merges an array of partial context objects using logical assignment rules:
- For each key: if the merged result is `null`/`undefined`, take the value from the next context (`??=`)
- For array values: concatenate them (`&&=` the spread)
- For number values: take the maximum

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Operator | Full form | Assigns when |
|---|---|---|
| `x \|\|= y` | `x \|\| (x = y)` | `x` is **falsy**: `false`, `0`, `""`, `null`, `undefined`, `NaN` |
| `x &&= y` | `x && (x = y)` | `x` is **truthy**: any value except falsy ones |
| `x ??= y` | `x ?? (x = y)` | `x` is **nullish**: only `null` or `undefined` |

| Situation | Best operator | Why |
|---|---|---|
| Set default for unknown/unset | `??=` | Preserves `0`, `""`, `false` |
| Set default only if "empty" | `\|\|=` | Treats `0`/`""`/`false` as "empty" |
| Update only if exists/truthy | `&&=` | Skips null/undefined/falsy |
| Lazy initialise object prop | `??=` | First access creates, subsequent ones skip |
| Config defaults | `??=` | Config values of `0`/`false` must be respected |
| Conditional object update | `&&=` | Spread update only if object isn't null |

| Fact | Detail |
|---|---|
| Short-circuits | RHS not evaluated if condition not met |
| Only assigns when needed | Unlike `x = x \|\| y` which always assigns |
| Works on properties | `obj.prop ??= default` — property access works |
| Works on array indices | `arr[0] ??= "fill"` |
| Not valid on `const` | These are assignments — `const` disallows reassignment |
| ES2021+ | Available natively in Node.js 15+ and all modern browsers |

---

## Connected topics

- **04 — Operators** — `||`, `&&`, `??` are the logical operators these build on
- **07 — Truthy and falsy** — `||=` and `&&=` depend on JS truthiness rules
- **08 — Nullish coalescing (`??`)** — `??=` is the assignment version; understanding `??` vs `||` is essential
- **56 — async/await** — `??=` used frequently for lazy async initialisation patterns
- **63 — Proxy and Reflect** — logical assignment operators interact with Proxy `set` traps; unlike `x = x || y`, `x ||= y` does NOT trigger the setter when the condition is not met
