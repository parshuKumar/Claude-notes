# 61 — WeakMap and WeakSet

## What is this?

A **WeakMap** is a collection of key-value pairs where the **keys must be objects** (or non-registered Symbols in ES2023+), and the entries are held **weakly** — meaning the garbage collector can reclaim the key object (and its WeakMap entry) when nothing else in the program references that key. A **WeakSet** is the same idea for a set of objects — it holds references weakly so objects can be garbage collected when no other references exist.

Think of a WeakMap like a coat-check room that uses the actual coat as the ticket. When someone takes their coat and leaves, the coat-check entry disappears automatically — no need for staff to clean up. With a regular `Map`, the coat-check keeps a copy of the ticket forever, even after the coat is long gone.

---

## Why does it matter?

The core problem both structures solve is **memory leaks caused by forgotten references**. In a regular `Map`, storing an object as a key keeps that object alive forever — even if you want it to be garbage collected. This is the most common cause of memory leaks in long-running Node.js servers.

```
Regular Map:
  dom element → data
  When the DOM element is removed, Map still holds it → memory leak

WeakMap:
  dom element → data
  When the DOM element is removed (no other references), entry is auto-deleted
```

Use cases:
- Storing private/metadata on objects without preventing garbage collection
- Caching computed results keyed by objects (auto-evict when object is gone)
- Tracking DOM nodes or request objects with associated state
- Polyfilling private class fields (what `#` does internally)

---

## Syntax

```js
// ── WeakMap ──────────────────────────────────────────────────────
const wm = new WeakMap();

const key1 = {};
const key2 = {};

wm.set(key1, "value1");
wm.set(key2, { data: 42 });

wm.get(key1);         // "value1"
wm.has(key1);         // true
wm.delete(key1);      // true (returns boolean)
wm.has(key1);         // false

// Only objects (and ES2023+ non-registered Symbols) can be keys:
wm.set("string", 1);   // TypeError: Invalid value used as weak map key
wm.set(42, 1);         // TypeError
wm.set(null, 1);       // TypeError

// ── WeakSet ──────────────────────────────────────────────────────
const ws = new WeakSet();

const obj1 = { id: 1 };
const obj2 = { id: 2 };

ws.add(obj1);
ws.add(obj2);

ws.has(obj1);         // true
ws.delete(obj1);      // true
ws.has(obj1);         // false

ws.add("string");     // TypeError: Invalid value used in weak set
```

---

## How it works — weak references and garbage collection

### The difference: strong vs weak reference

```
let obj = { name: "Alice" };

// Strong reference — Map:
const map = new Map();
map.set(obj, "some data");

obj = null;  // you dropped your reference to the object

// But map still holds a strong reference to the original { name: "Alice" }
// → Garbage collector cannot collect it — memory is NOT freed

// ---

// Weak reference — WeakMap:
const wm = new WeakMap();
wm.set(obj, "some data");

obj = null;  // you dropped your reference

// Now there are NO more strong references to { name: "Alice" }
// → Garbage collector CAN collect it → WeakMap entry disappears automatically
```

### Why WeakMap/WeakSet have no iteration methods

WeakMap and WeakSet intentionally have **no `size`, no `.keys()`, no `.values()`, no `.entries()`, no `forEach()`**. This is not an oversight — it's required by the design:

- Because the garbage collector can remove entries at any time between two JavaScript operations, the set of entries is non-deterministic
- If you could iterate, you might see different results before vs after a GC run
- The spec forbids iteration to prevent this undefined behaviour
- `size` is excluded for the same reason

This also means you can't accidentally enumerate the contents — which is a useful security/privacy property.

---

## WeakMap — practical examples

### Example 1 — private data for objects (pre-`#` pattern)

```js
// Before private class fields (#), WeakMap was THE way to achieve true privacy.
// This pattern is still used in older codebases and some libraries.

const _balance = new WeakMap();
const _pin     = new WeakMap();

class BankAccount {
  constructor(initialBalance, pin) {
    _balance.set(this, initialBalance);
    _pin.set(this, pin);
  }

  deposit(amount) {
    if (amount <= 0) throw new RangeError("Amount must be positive");
    _balance.set(this, _balance.get(this) + amount);
  }

  withdraw(pin, amount) {
    if (pin !== _pin.get(this)) throw new Error("Invalid PIN");
    const bal = _balance.get(this);
    if (amount > bal)            throw new Error("Insufficient funds");
    _balance.set(this, bal - amount);
  }

  getBalance(pin) {
    if (pin !== _pin.get(this)) throw new Error("Invalid PIN");
    return _balance.get(this);
  }
}

const account = new BankAccount(1000, 1234);
account.deposit(500);
console.log(account.getBalance(1234));   // 1500

// _balance and _pin are module-scoped — not on the instance
console.log(Object.keys(account));       // []
console.log(account._balance);           // undefined

// When account is garbage collected, its entries in both WeakMaps are auto-removed
```

### Example 2 — metadata on DOM nodes without memory leaks

```js
// Storing click counts per DOM element

const clickCounts = new WeakMap();

function trackClicks(element) {
  element.addEventListener("click", () => {
    const count = (clickCounts.get(element) ?? 0) + 1;
    clickCounts.set(element, count);
    console.log(`Clicked ${count} times`);
  });
}

const button = document.querySelector("#my-button");
trackClicks(button);

// Later — element is removed from DOM
button.remove();
button = null;

// With a regular Map: { element: count } stays in memory forever → leak
// With WeakMap: entry is automatically removed when element is GC'd → no leak
```

### Example 3 — caching computed results (memoization on object instances)

```js
const cache = new WeakMap();

function processUser(user) {
  if (cache.has(user)) {
    console.log("cache hit");
    return cache.get(user);
  }

  // Expensive computation
  const result = {
    fullName:    `${user.firstName} ${user.lastName}`,
    permissions: computePermissions(user),
    score:       calculateScore(user),
  };

  cache.set(user, result);
  return result;
}

// When user objects are discarded (e.g., after a request ends in Node.js),
// their cache entries are automatically freed — no manual cache invalidation needed.
```

### Example 4 — marking objects as "seen" or "processed"

```js
const processed = new WeakMap();

async function processRequest(req, res, next) {
  // Middleware: mark request as processed to prevent double-processing
  if (processed.has(req)) {
    return res.status(400).json({ error: "Request already processed" });
  }

  processed.set(req, { timestamp: Date.now() });

  try {
    await handleRequest(req, res);
  } finally {
    next();
    // When req is garbage collected (after response sent), WeakMap entry is freed
  }
}
```

### Example 5 — storing state for external objects you don't own

```js
// You're building a library that needs to associate state with user-provided objects
// You can't add properties directly (you don't own the objects)
// You can't use a regular Map (memory leak if they never clean up)
// WeakMap is the perfect fit:

const elementAnimations = new WeakMap();

export function animate(element, config) {
  const animation = new Animation(element, config);
  elementAnimations.set(element, animation);
  animation.start();
}

export function stopAnimation(element) {
  const animation = elementAnimations.get(element);
  if (animation) {
    animation.stop();
    elementAnimations.delete(element);
  }
}

// Users call animate(el, {...}) and stopAnimation(el)
// If they forget to call stopAnimation but remove the element,
// the WeakMap entry and Animation instance are GC'd automatically
```

---

## WeakSet — practical examples

### Example 1 — tracking which objects have been initialised

```js
const initialised = new WeakSet();

function ensureInitialised(service) {
  if (initialised.has(service)) return;   // already done

  service.connect();
  service.loadConfig();
  initialised.add(service);

  console.log(`${service.name} initialised`);
}

// When service objects are garbage collected, their WeakSet entries vanish too
```

### Example 2 — preventing circular processing (graph traversal)

```js
function deepClone(obj, visited = new WeakSet()) {
  // Handle primitives
  if (obj === null || typeof obj !== "object") return obj;

  // Detect circular references
  if (visited.has(obj)) {
    throw new TypeError("Circular reference detected in object");
  }

  visited.add(obj);

  if (Array.isArray(obj)) {
    return obj.map(item => deepClone(item, visited));
  }

  const clone = {};
  for (const [key, value] of Object.entries(obj)) {
    clone[key] = deepClone(value, visited);
  }
  return clone;
}

// Works with deeply nested objects:
const original = { a: 1, b: { c: 2, d: { e: 3 } } };
const cloned   = deepClone(original);

// Correctly throws on circular:
const circular = { x: 1 };
circular.self  = circular;
deepClone(circular);   // TypeError: Circular reference detected
```

### Example 3 — marking frozen or sealed objects

```js
const frozenObjects = new WeakSet();

function deepFreeze(obj) {
  if (frozenObjects.has(obj)) return obj;  // already frozen, prevent infinite loop
  frozenObjects.add(obj);

  Object.freeze(obj);

  for (const value of Object.values(obj)) {
    if (value !== null && typeof value === "object") {
      deepFreeze(value);
    }
  }

  return obj;
}

export function isFrozenByUs(obj) {
  return frozenObjects.has(obj);
}
```

### Example 4 — brand checking (has this object been through our factory?)

```js
const validInstances = new WeakSet();

class SecureToken {
  constructor(secret) {
    if (secret !== INTERNAL_SECRET) {
      throw new Error("Use SecureToken.create() — do not call constructor directly");
    }
    validInstances.add(this);
    this.value = generateTokenValue();
  }

  static create() {
    return new SecureToken(INTERNAL_SECRET);
  }

  static isValid(token) {
    return validInstances.has(token);
  }
}

const token  = SecureToken.create();
SecureToken.isValid(token);   // true

const fake = { value: "forged" };
SecureToken.isValid(fake);    // false — wasn't created by our factory
```

---

## WeakRef and FinalizationRegistry (ES2021)

Related to weak references — not always used, but important to know:

```js
// WeakRef — hold a weak reference to an object
// .deref() returns the object, or undefined if it was GC'd

let obj = { data: "important" };
const ref = new WeakRef(obj);

// ... time passes, potentially GC happens ...

const val = ref.deref();
if (val !== undefined) {
  console.log(val.data);   // "important" if still alive
} else {
  console.log("Object was garbage collected");
}

// ---

// FinalizationRegistry — run a callback when an object is GC'd
const registry = new FinalizationRegistry((heldValue) => {
  console.log(`Object associated with "${heldValue}" was collected`);
});

let user = { name: "Alice" };
registry.register(user, "user-Alice");   // second arg is the "held value" passed to callback

user = null;   // when GC'd: "Object associated with 'user-Alice' was collected"
```

**Important caution:** GC timing is implementation-defined and non-deterministic. Don't write code that requires finalisation to happen at a specific time. Use `FinalizationRegistry` for cleanup of external resources (native handles, file descriptors) — not for program logic.

---

## Comparison: Map vs WeakMap vs Object

```
┌──────────────────────┬────────────────┬────────────────┬────────────────┐
│ Feature              │ Object         │ Map            │ WeakMap        │
├──────────────────────┼────────────────┼────────────────┼────────────────┤
│ Key types            │ String, Symbol │ Any value      │ Objects only   │
│ Key uniqueness       │ By string/sym  │ By ===         │ By ===         │
│ Iteration            │ Yes            │ Yes            │ NO             │
│ size property        │ No (use keys)  │ Yes            │ NO             │
│ GC of keys           │ No (strong)    │ No (strong)    │ YES (weak)     │
│ Memory safe for DOM  │ No             │ No             │ YES            │
│ JSON serialisable    │ Yes            │ No             │ No             │
│ Prototype pollution  │ Risk           │ No             │ No             │
│ Performance (large)  │ OK             │ Better         │ N/A (no size)  │
└──────────────────────┴────────────────┴────────────────┴────────────────┘
```

---

## Comparison: Set vs WeakSet

```
┌──────────────────────┬────────────────┬────────────────┐
│ Feature              │ Set            │ WeakSet        │
├──────────────────────┼────────────────┼────────────────┤
│ Value types          │ Any value      │ Objects only   │
│ Iteration            │ Yes            │ NO             │
│ size property        │ Yes            │ NO             │
│ GC of values         │ No (strong)    │ YES (weak)     │
│ Main use             │ Unique values  │ Object tagging │
└──────────────────────┴────────────────┴────────────────┘
```

---

## Tricky things

### 1. You cannot store primitives as WeakMap keys

```js
const wm = new WeakMap();

wm.set(42, "value");       // TypeError: Invalid value used as weak map key
wm.set("key", "value");    // TypeError
wm.set(true, "value");     // TypeError
wm.set(null, "value");     // TypeError

// Only objects (and ES2023+ non-registered Symbols):
wm.set({}, "value");       // ✓
wm.set([], "value");       // ✓ (arrays are objects)
wm.set(function(){}, "v"); // ✓ (functions are objects)
```

### 2. There is no way to check how many entries are in a WeakMap

```js
const wm = new WeakMap();
wm.set({}, 1);
wm.set({}, 2);

wm.size;   // undefined — WeakMap has no size property
// You cannot count entries, cannot check if it's empty
// This is by design
```

### 3. WeakMap entries can disappear between synchronous operations — in theory

```js
// In practice, GC does not run in the middle of synchronous JS
// But in theory:
let key = {};
const wm = new WeakMap();
wm.set(key, "data");

// Between these two lines, could GC run? Technically yes in spec.
// In practice: no — JS is single-threaded, GC only runs between tasks.
// But you MUST keep a strong reference if you need to retrieve the value:
const result = wm.get(key);   // safe — key is still in scope here

// DANGEROUS:
wm.set({}, "data");           // anonymous object — immediately eligible for GC
wm.get(??);                   // can't get it back — you lost the key reference
```

### 4. WeakMap does not prevent GC of the VALUE — only the key triggers deletion

```js
const wm = new WeakMap();

let key  = { id: 1 };
let data = { heavy: new Array(1_000_000).fill(0) };

wm.set(key, data);

data = null;   // data variable is gone — but WeakMap still holds a strong reference to the array object!
// The VALUE is strongly held by the WeakMap until the KEY is GC'd

key = null;   // NOW: key is eligible for GC → WeakMap entry deleted → data array is freed too
```

### 5. Using object literals directly as keys loses the reference immediately

```js
const wm = new WeakMap();

// WRONG — object literal creates an anonymous object with no other reference
wm.set({ id: 1 }, "user data");
wm.get({ id: 1 });   // undefined — this is a NEW object, not the same one!

// You can never retrieve this value — the key is immediately GC-eligible

// RIGHT — keep a reference to the key
const userKey = { id: 1 };
wm.set(userKey, "user data");
wm.get(userKey);   // "user data" ✓
```

---

## Common mistakes

### Mistake 1 — Using Map when WeakMap is appropriate (memory leak)

```js
// WRONG — in a Node.js server handling thousands of requests
const requestCache = new Map();   // grows forever!

app.use((req, res, next) => {
  requestCache.set(req, { startTime: Date.now() });
  // When request finishes, req is no longer needed
  // But Map keeps a strong reference → memory leak over time
  next();
});

// RIGHT — WeakMap auto-cleans when req is GC'd
const requestCache = new WeakMap();

app.use((req, res, next) => {
  requestCache.set(req, { startTime: Date.now() });
  next();
});
```

### Mistake 2 — Trying to iterate a WeakMap

```js
const wm = new WeakMap();
wm.set({ a: 1 }, "first");
wm.set({ b: 2 }, "second");

// WRONG — these don't exist
for (const [key, val] of wm) { }   // TypeError: wm is not iterable
wm.forEach((v, k) => { });          // TypeError: wm.forEach is not a function
[...wm];                            // TypeError

// If you need iteration → use a regular Map
// If you need weak keys → use WeakMap (accept no iteration)
```

### Mistake 3 — Expecting WeakSet to deduplicate by value

```js
const ws = new WeakSet();

// WRONG assumption — WeakSet compares by identity (===), not by value
const a = { id: 1 };
const b = { id: 1 };

ws.add(a);
ws.add(b);  // adds b — it's a DIFFERENT object even though it has the same shape

ws.has(a);  // true
ws.has(b);  // true — both are in the set (two separate objects)
ws.has({ id: 1 });  // false — yet another new object

// WeakSet membership = "is this exact object reference in the set?"
```

---

## Practice exercises

### Exercise 1 — easy

1. Create a `WeakMap` to store "view counts" for objects (simulating DOM elements)
2. Write `trackView(element)` — increments the view count for that element
3. Write `getViews(element)` — returns view count (0 if never viewed)
4. Create 3 objects, track various views, retrieve counts
5. Set one object reference to `null` — explain what happens to its WeakMap entry (you can't verify it without devtools, just explain)
6. Repeat steps 1-4 with a `WeakSet` that tracks "visited" objects (just `true/false`)

```js
// Write your code here
```

### Exercise 2 — medium

Build a `Memoize` utility using WeakMap for memory-safe caching:

```js
class Memoize {
  // constructor(fn) — wraps a function
  // call(thisArg, ...args) — returns cached result for thisArg if available
  //   Cache key = thisArg (object) → WeakMap
  //   For each thisArg, inner cache stores results by serialised args
  // invalidate(thisArg) — removes all cached results for that object
  // stats() — returns { hits, misses } counts (use regular counters, not WeakMap)
}

// Example usage:
const memoized = new Memoize(function computeScore(weights) {
  console.log("computing...");
  return this.data.reduce((sum, v, i) => sum + v * (weights[i] ?? 1), 0);
});

const user1 = { data: [1, 2, 3] };
const user2 = { data: [4, 5, 6] };

memoized.call(user1, [1, 1, 1]);  // computing... → 6
memoized.call(user1, [1, 1, 1]);  // cache hit → 6
memoized.call(user2, [2, 2, 2]);  // computing... → 30
memoized.invalidate(user1);
memoized.call(user1, [1, 1, 1]);  // computing... (invalidated) → 6
console.log(memoized.stats());    // { hits: 1, misses: 3 }
```

```js
// Write your code here
```

### Exercise 3 — hard

Implement a `ReactiveSystem` that uses WeakMap to track object dependencies without memory leaks:

Requirements:
- `observable(target)` — wraps an object in a Proxy; stores reactive metadata in a WeakMap (dependency subscribers per property)
- `computed(fn)` — tracks which reactive properties `fn` reads and re-runs when they change; returns an object with `.value` getter
- `watch(reactiveObj, prop, callback)` — register a callback that fires when `reactiveObj[prop]` changes
- `effect(fn)` — run `fn` immediately and re-run it whenever any reactive property it accessed changes

All subscriber tracking must use WeakMap/WeakSet so observable objects can be GC'd when dropped, automatically cleaning up all watchers.

```js
const state = observable({ count: 0, name: "Alice" });

watch(state, "count", (newVal, oldVal) => {
  console.log(`count changed: ${oldVal} → ${newVal}`);
});

const doubled = computed(() => state.count * 2);
console.log(doubled.value);   // 0

state.count = 5;
// → "count changed: 0 → 5"
console.log(doubled.value);   // 10

effect(() => {
  console.log(`name is: ${state.name}`);
});
// → "name is: Alice" (immediately)

state.name = "Bob";
// → "name is: Bob" (re-ran because effect accessed state.name)
```

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | WeakMap | WeakSet |
|---|---|---|
| Create | `new WeakMap()` | `new WeakSet()` |
| Key/value types | Keys: objects only; Values: anything | Values: objects only |
| Add/set | `wm.set(key, val)` | `ws.add(obj)` |
| Get | `wm.get(key)` | — (no get) |
| Check | `wm.has(key)` | `ws.has(obj)` |
| Remove | `wm.delete(key)` | `ws.delete(obj)` |
| Iterate | NOT POSSIBLE | NOT POSSIBLE |
| Size | NOT AVAILABLE | NOT AVAILABLE |
| GC behaviour | Entry deleted when key is GC'd | Entry deleted when value is GC'd |
| Main use | Private/metadata on objects | Tagging objects as seen/processed |
| vs Map | Use when keys are objects that should be GC'd | — |
| vs Set | — | Use when tracked objects should be GC'd |
| ES2023+ | Non-registered Symbols allowed as keys | Same |
| `WeakRef` | Related: hold a weak reference explicitly | — |
| `FinalizationRegistry` | Related: run callback when object is GC'd | — |

---

## Connected topics

- **51 — Private class fields (`#`)** — `#` is implemented similarly to WeakMap internally in V8; WeakMap was the way to achieve privacy before `#`
- **60 — Symbol** — Symbols can be WeakMap keys in ES2023+; both are used for metadata/privacy
- **63 — Proxy and Reflect** — Proxy + WeakMap is the foundation of reactive systems (Vue 3 reactivity)
- **22 — Map and Set** — the strong-reference counterparts; choose between them based on GC needs
