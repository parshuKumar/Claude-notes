# 63 — Proxy and Reflect

## What is this?

A **Proxy** is an object that wraps another object (the "target") and intercepts fundamental operations — property reads, writes, function calls, `in` checks, `delete`, `new`, and more — letting you run custom code before, after, or instead of the default behaviour. Think of a Proxy like a security desk at a building: every request to enter (read a property) or exit (write a property) passes through the security guard first. The guard can allow it, modify it, block it, or log it — without the building itself (target object) knowing.

**Reflect** is a companion API — a set of static methods that perform the same fundamental operations a Proxy intercepts, but in a safe, consistent way. Where `Proxy` intercepts, `Reflect` does. They are designed to work together.

```js
const target = { name: "Alice", age: 30 };

const proxy = new Proxy(target, {
  get(target, prop, receiver) {
    console.log(`Reading: ${prop}`);
    return Reflect.get(target, prop, receiver);  // do the actual get
  },
  set(target, prop, value, receiver) {
    console.log(`Setting: ${prop} = ${value}`);
    return Reflect.set(target, prop, value, receiver);  // do the actual set
  },
});

proxy.name;          // logs: Reading: name → "Alice"
proxy.age = 31;      // logs: Setting: age = 31
```

---

## Why does it matter?

Proxy is the mechanism behind some of JavaScript's most powerful patterns:

- **Vue 3 reactivity system** — built entirely on `Proxy` (Vue 2 used `Object.defineProperty` which is far more limited)
- **Validation and type checking** — enforce schemas on plain objects without classes
- **Logging and debugging** — trace every property access during development
- **Access control** — hide private properties, enforce read-only fields
- **Default values** — return sensible defaults for missing properties
- **Negative array indices** — `arr[-1]` to access the last element (like Python)
- **Auto-creating nested objects** — `obj.a.b.c = 1` without `obj.a` existing first
- **ORM / database models** — auto-generate SQL when you access object properties
- **Mocking in tests** — intercept any property access or method call

---

## Syntax

```js
const proxy = new Proxy(target, handler);
// target  — the object to wrap
// handler — object with "trap" methods; empty {} = transparent proxy

// Transparent proxy — passes everything through:
const p = new Proxy({}, {});

// Proxy with get trap:
const p = new Proxy(target, {
  get(target, prop, receiver) {
    // target   = the wrapped object
    // prop     = the property name (string or Symbol)
    // receiver = the proxy itself (or object that inherited from proxy)
    return Reflect.get(target, prop, receiver);
  },
});

// Proxy with set trap:
const p = new Proxy(target, {
  set(target, prop, value, receiver) {
    // value    = the value being assigned
    // Must return true (success) or false/throw (failure)
    return Reflect.set(target, prop, value, receiver);
  },
});
```

---

## How it works — traps

A **trap** is a method on the handler object that intercepts a specific operation. Every fundamental JavaScript operation has a corresponding trap. If a trap is not defined in the handler, the operation is forwarded transparently to the target.

### All 13 traps

```
┌─────────────────────────┬──────────────────────────────────────────────────┐
│ Trap                    │ Triggered by                                     │
├─────────────────────────┼──────────────────────────────────────────────────┤
│ get                     │ proxy.prop, proxy["prop"], Reflect.get()         │
│ set                     │ proxy.prop = val, Reflect.set()                  │
│ has                     │ "prop" in proxy, Reflect.has()                   │
│ deleteProperty          │ delete proxy.prop, Reflect.deleteProperty()      │
│ apply                   │ proxy(), proxy.call(), proxy.apply()             │
│ construct               │ new proxy(), Reflect.construct()                 │
│ getPrototypeOf          │ Object.getPrototypeOf(proxy), proxy instanceof X │
│ setPrototypeOf          │ Object.setPrototypeOf(proxy, proto)              │
│ isExtensible            │ Object.isExtensible(proxy)                       │
│ preventExtensions       │ Object.preventExtensions(proxy)                  │
│ getOwnPropertyDescriptor│ Object.getOwnPropertyDescriptor(proxy, prop)     │
│ defineProperty          │ Object.defineProperty(proxy, prop, desc)         │
│ ownKeys                 │ Object.keys(), Object.getOwnPropertyNames(), etc.│
└─────────────────────────┴──────────────────────────────────────────────────┘
```

---

## Reflect — what it is and why use it

`Reflect` is a plain object with static methods matching each Proxy trap. Every trap has a corresponding `Reflect` method with the same name and signature.

```js
// Old way — multiple inconsistent APIs:
obj[prop]                              // property access
Object.getOwnPropertyDescriptor(o, p) // descriptor
"key" in obj                           // has check
delete obj.key                         // delete (returns boolean)
Function.prototype.apply.call(fn, t, a)// apply

// Reflect — consistent, safe, same interface as Proxy traps:
Reflect.get(target, prop, receiver)
Reflect.getOwnPropertyDescriptor(target, prop)
Reflect.has(target, prop)
Reflect.deleteProperty(target, prop)
Reflect.apply(fn, thisArg, args)
Reflect.construct(Cls, args, newTarget)
```

**Why use `Reflect` inside traps?**
1. **Correct `receiver` propagation** — ensures getters/setters use the right `this` (the proxy, not the target)
2. **Returns boolean for set/deleteProperty** — `Reflect.set` returns `true`/`false`; direct assignment throws in strict mode
3. **Symmetry** — every trap should call its `Reflect` counterpart to maintain expected behaviour

```js
// Without Reflect — receiver is wrong for inherited getters:
const proto = {
  get fullName() { return this.first + " " + this.last; }
};
const obj = Object.create(proto);
obj.first = "John";
obj.last  = "Doe";

const proxy = new Proxy(obj, {
  get(target, prop) {
    return target[prop];   // WRONG — "this" inside getter = target, not proxy
  },
});

// With Reflect — receiver is correctly the proxy:
const proxy = new Proxy(obj, {
  get(target, prop, receiver) {
    return Reflect.get(target, prop, receiver);  // "this" inside getter = proxy
  },
});
```

---

## Example 1 — validation proxy

```js
function createValidated(schema) {
  return new Proxy({}, {
    set(target, prop, value) {
      const rule = schema[prop];
      if (!rule) throw new TypeError(`Unknown property: "${prop}"`);

      if (rule.type && typeof value !== rule.type) {
        throw new TypeError(`${prop} must be ${rule.type}, got ${typeof value}`);
      }
      if (rule.min !== undefined && value < rule.min) {
        throw new RangeError(`${prop} must be >= ${rule.min}`);
      }
      if (rule.max !== undefined && value > rule.max) {
        throw new RangeError(`${prop} must be <= ${rule.max}`);
      }
      if (rule.pattern && !rule.pattern.test(value)) {
        throw new Error(`${prop} does not match required pattern`);
      }

      return Reflect.set(target, prop, value);  // store value
    },

    get(target, prop, receiver) {
      const value = Reflect.get(target, prop, receiver);
      return value ?? schema[prop]?.default;    // return default if undefined
    },
  });
}

const user = createValidated({
  name:  { type: "string", min: 1, max: 50 },
  age:   { type: "number", min: 0, max: 120, default: 18 },
  email: { type: "string", pattern: /^[^\s@]+@[^\s@]+\.[^\s@]+$/ },
});

user.name  = "Alice";               // ✓
user.age   = 30;                    // ✓
user.email = "alice@example.com";   // ✓

user.age = -5;         // RangeError: age must be >= 0
user.email = "bad";    // Error: email does not match required pattern
user.unknown = "x";    // TypeError: Unknown property: "unknown"
user.age;              // 18 (default)
```

---

## Example 2 — logging / tracing proxy

```js
function createTraceable(target, label = "obj") {
  const handler = {
    get(target, prop, receiver) {
      const value = Reflect.get(target, prop, receiver);
      if (typeof value === "function") {
        return function(...args) {
          console.log(`[${label}] ${String(prop)}(${args.map(a => JSON.stringify(a)).join(", ")})`);
          const result = value.apply(this === receiver ? target : this, args);
          console.log(`[${label}] ${String(prop)} returned:`, result);
          return result;
        };
      }
      console.log(`[${label}] GET .${String(prop)} =`, value);
      return value;
    },

    set(target, prop, value, receiver) {
      console.log(`[${label}] SET .${String(prop)} =`, value);
      return Reflect.set(target, prop, value, receiver);
    },

    deleteProperty(target, prop) {
      console.log(`[${label}] DELETE .${String(prop)}`);
      return Reflect.deleteProperty(target, prop);
    },
  };

  return new Proxy(target, handler);
}

const api = createTraceable({
  users: ["Alice", "Bob"],
  getUser(id) { return this.users[id]; },
}, "UserAPI");

api.users;          // [UserAPI] GET .users = ["Alice", "Bob"]
api.getUser(0);     // [UserAPI] getUser("0") called; [UserAPI] getUser returned: "Alice"
api.name = "Test";  // [UserAPI] SET .name = "Test"
delete api.name;    // [UserAPI] DELETE .name
```

---

## Example 3 — default values (no undefined)

```js
// Python's defaultdict equivalent
function withDefaults(defaultFn) {
  return new Proxy({}, {
    get(target, prop, receiver) {
      if (!Reflect.has(target, prop)) {
        const defaultValue = defaultFn(prop);
        Reflect.set(target, prop, defaultValue, receiver);
        return defaultValue;
      }
      return Reflect.get(target, prop, receiver);
    },
  });
}

// A dict that auto-creates arrays for new keys (like groupBy):
const groups = withDefaults(() => []);
groups["admin"].push("Alice");
groups["admin"].push("Bob");
groups["user"].push("Carol");

console.log(groups);
// { admin: ["Alice", "Bob"], user: ["Carol"] }

// A config object that returns 0 for missing numeric keys:
const counters = withDefaults(() => 0);
counters["views"]++;
counters["clicks"]++;
counters["views"]++;
console.log(counters);  // { views: 2, clicks: 1 }
```

---

## Example 4 — negative array indices

```js
function withNegativeIndices(arr) {
  return new Proxy(arr, {
    get(target, prop, receiver) {
      // Convert string index to number and handle negatives
      const index = Number(prop);
      if (Number.isInteger(index) && index < 0) {
        return Reflect.get(target, target.length + index, receiver);
      }
      return Reflect.get(target, prop, receiver);
    },
  });
}

const arr = withNegativeIndices([1, 2, 3, 4, 5]);

arr[-1];    // 5  (last)
arr[-2];    // 4  (second to last)
arr[0];     // 1  (normal access still works)
arr.length; // 5  (methods still work)
```

---

## Example 5 — read-only / immutable proxy

```js
function readOnly(target) {
  return new Proxy(target, {
    set(target, prop) {
      throw new TypeError(`Cannot set "${String(prop)}" on read-only object`);
    },
    deleteProperty(target, prop) {
      throw new TypeError(`Cannot delete "${String(prop)}" from read-only object`);
    },
    defineProperty() {
      throw new TypeError("Cannot define properties on read-only object");
    },
  });
}

const config = readOnly({
  API_URL:   "https://api.example.com",
  TIMEOUT:   5000,
  MAX_RETRY: 3,
});

config.API_URL;           // "https://api.example.com" ✓
config.NEW_KEY = "bad";   // TypeError: Cannot set "NEW_KEY" on read-only object
delete config.TIMEOUT;    // TypeError: Cannot delete "TIMEOUT" from read-only object
```

---

## Example 6 — auto-creating nested paths (deep path proxy)

```js
// Like _.set(obj, 'a.b.c', value) but with natural syntax
function deepProxy(target = {}, onChange = () => {}) {
  return new Proxy(target, {
    get(target, prop, receiver) {
      const value = Reflect.get(target, prop, receiver);
      // Auto-create nested objects for missing properties
      if (value === undefined && prop !== "then") {  // exclude Promise.then check
        const nested = {};
        Reflect.set(target, prop, nested);
        return deepProxy(nested, onChange);
      }
      if (typeof value === "object" && value !== null) {
        return deepProxy(value, onChange);  // wrap nested objects too
      }
      return value;
    },

    set(target, prop, value, receiver) {
      const result = Reflect.set(target, prop, value, receiver);
      onChange(target, prop, value);
      return result;
    },
  });
}

const config = deepProxy({}, (obj, key, val) => {
  console.log(`Config changed: .${key} = ${JSON.stringify(val)}`);
});

config.database.host     = "localhost";    // auto-creates config.database = {}
config.database.port     = 5432;
config.server.port       = 3000;
config.server.ssl.enabled = true;          // auto-creates config.server.ssl = {}

console.log(JSON.stringify(config, null, 2));
// { database: { host: "localhost", port: 5432 }, server: { port: 3000, ssl: { enabled: true }}}
```

---

## Example 7 — `apply` trap (proxy a function)

```js
// Proxy wraps functions using the `apply` trap
function memoize(fn) {
  const cache = new Map();

  return new Proxy(fn, {
    apply(target, thisArg, args) {
      const key = JSON.stringify(args);

      if (cache.has(key)) {
        console.log("cache hit:", key);
        return cache.get(key);
      }

      const result = Reflect.apply(target, thisArg, args);
      cache.set(key, result);
      return result;
    },
  });
}

function expensiveCalc(n) {
  console.log("computing for", n);
  return n * n;
}

const fast = memoize(expensiveCalc);
fast(5);   // computing for 5 → 25
fast(5);   // cache hit: [5]  → 25
fast(3);   // computing for 3 → 9
fast(3);   // cache hit: [3]  → 9
```

---

## Example 8 — `construct` trap

```js
// Intercept new MyClass()
function withLogging(TargetClass) {
  return new Proxy(TargetClass, {
    construct(target, args, newTarget) {
      console.log(`new ${target.name}(${args.map(a => JSON.stringify(a)).join(", ")})`);
      const instance = Reflect.construct(target, args, newTarget);
      console.log(`Created:`, instance);
      return instance;
    },
  });
}

class User {
  constructor(name, role) {
    this.name = name;
    this.role = role;
  }
}

const TrackedUser = withLogging(User);
const u = new TrackedUser("Alice", "admin");
// logs: new User("Alice", "admin")
// logs: Created: User { name: 'Alice', role: 'admin' }
```

---

## Example 9 — hiding private properties

```js
function hidePrivate(target, prefix = "_") {
  const isPrivate = prop => typeof prop === "string" && prop.startsWith(prefix);

  return new Proxy(target, {
    get(target, prop, receiver) {
      if (isPrivate(prop)) return undefined;  // hide private props
      return Reflect.get(target, prop, receiver);
    },

    set(target, prop, value, receiver) {
      if (isPrivate(prop)) throw new TypeError(`Cannot set private property "${prop}"`);
      return Reflect.set(target, prop, value, receiver);
    },

    has(target, prop) {
      if (isPrivate(prop)) return false;  // "prop" in proxy → false for private
      return Reflect.has(target, prop);
    },

    ownKeys(target) {
      // Filter out private keys from Object.keys() etc.
      return Reflect.ownKeys(target).filter(key => !isPrivate(key));
    },

    getOwnPropertyDescriptor(target, prop) {
      if (isPrivate(prop)) return undefined;
      return Reflect.getOwnPropertyDescriptor(target, prop);
    },
  });
}

const obj = hidePrivate({ name: "Alice", _password: "secret", age: 30 });

obj.name;           // "Alice"
obj._password;      // undefined
"_password" in obj; // false
Object.keys(obj);   // ["name", "age"]
obj._newProp = "x"; // TypeError: Cannot set private property "_newProp"
```

---

## Vue 3 reactivity — how Proxy powers it

```js
// Simplified version of how Vue 3's reactive() works
function reactive(target) {
  const handlers = {
    get(target, prop, receiver) {
      track(target, prop);           // record that this prop was accessed by current effect
      const value = Reflect.get(target, prop, receiver);
      if (typeof value === "object" && value !== null) {
        return reactive(value);      // deeply wrap nested objects
      }
      return value;
    },

    set(target, prop, value, receiver) {
      const oldValue = target[prop];
      const result   = Reflect.set(target, prop, value, receiver);
      if (oldValue !== value) {
        trigger(target, prop);       // notify all effects that read this prop
      }
      return result;
    },

    deleteProperty(target, prop) {
      const result = Reflect.deleteProperty(target, prop);
      trigger(target, prop);
      return result;
    },
  };

  return new Proxy(target, handlers);
}

// Dependency tracking:
const targetMap   = new WeakMap();   // WeakMap so targets can be GC'd
let   activeEffect = null;

function track(target, prop) {
  if (!activeEffect) return;
  let propMap = targetMap.get(target);
  if (!propMap) targetMap.set(target, (propMap = new Map()));
  let effects = propMap.get(prop);
  if (!effects) propMap.set(prop, (effects = new Set()));
  effects.add(activeEffect);
}

function trigger(target, prop) {
  const propMap = targetMap.get(target);
  if (!propMap) return;
  propMap.get(prop)?.forEach(effect => effect());
}

function watchEffect(fn) {
  activeEffect = fn;
  fn();           // run immediately, recording all tracked props
  activeEffect = null;
}

// Usage:
const state = reactive({ count: 0, name: "Alice" });

watchEffect(() => {
  console.log(`count is: ${state.count}`);
});
// logs: "count is: 0" immediately

state.count = 1;   // logs: "count is: 1"
state.count = 2;   // logs: "count is: 2"
state.name  = "Bob"; // no log — effect didn't read .name
```

---

## Revocable proxies

```js
// Create a proxy that can be disabled:
const { proxy, revoke } = Proxy.revocable(target, handler);

proxy.name;   // works

revoke();     // disable the proxy permanently

proxy.name;   // TypeError: Cannot perform 'get' on a proxy that has been revoked
```

Use cases:
- Grant temporary access to a resource, revoke when access period ends
- Pass a proxy to untrusted code, revoke it when the call returns
- Lifetime-limited API keys for sub-systems

```js
function withTimeout(target, handler, ms) {
  const { proxy, revoke } = Proxy.revocable(target, handler);
  setTimeout(revoke, ms);
  return proxy;
}

const tempAccess = withTimeout(sensitiveData, {}, 5000);
// tempAccess is valid for 5 seconds, then throws on access
```

---

## All Reflect methods

```js
Reflect.get(target, prop, receiver)          // like target[prop]
Reflect.set(target, prop, value, receiver)   // like target[prop] = value
Reflect.has(target, prop)                    // like prop in target
Reflect.deleteProperty(target, prop)         // like delete target[prop]
Reflect.apply(fn, thisArg, argsList)         // like fn.apply(thisArg, argsList)
Reflect.construct(Target, argsList, newTarget) // like new Target(...argsList)
Reflect.getPrototypeOf(target)               // like Object.getPrototypeOf(target)
Reflect.setPrototypeOf(target, proto)        // like Object.setPrototypeOf(target, proto)
Reflect.isExtensible(target)                 // like Object.isExtensible(target)
Reflect.preventExtensions(target)            // like Object.preventExtensions(target)
Reflect.getOwnPropertyDescriptor(t, prop)    // like Object.getOwnPropertyDescriptor(t, prop)
Reflect.defineProperty(target, prop, desc)   // like Object.defineProperty(target, prop, desc)
Reflect.ownKeys(target)                      // like Object.getOwnPropertyNames + getOwnPropertySymbols

// Returns: boolean for set/deleteProperty/setPrototypeOf/isExtensible/preventExtensions/defineProperty
// Returns: value for get/apply/construct/getPrototypeOf/getOwnPropertyDescriptor
// Returns: array for ownKeys
```

---

## Tricky things

### 1. Proxy identity — `proxy !== target`

```js
const target = {};
const proxy  = new Proxy(target, {});

proxy === target;   // false — they are different objects
proxy instanceof Object;   // true (via getPrototypeOf trap)

// Implications:
const map = new Map();
map.set(target, "real");
map.get(proxy);    // undefined — proxy and target are different keys!
map.get(target);   // "real"
```

### 2. The `set` trap MUST return `true` for success

```js
// WRONG — trap returns nothing (undefined → falsy)
const proxy = new Proxy({}, {
  set(target, prop, value) {
    target[prop] = value;
    // forgot to return true!
  },
});

proxy.x = 1;   // In strict mode: TypeError: 'set' on proxy: trap returned falsish

// RIGHT
const proxy = new Proxy({}, {
  set(target, prop, value, receiver) {
    return Reflect.set(target, prop, value, receiver);  // returns true/false
  },
});
```

### 3. Invariants — traps cannot violate the target object's constraints

The Proxy spec enforces **invariants**: if the target object has a non-configurable, non-writable property, the proxy's `get` trap MUST return that exact value. If it doesn't, a `TypeError` is thrown.

```js
const target = {};
Object.defineProperty(target, "frozen", {
  value:      42,
  writable:   false,
  configurable: false,
});

const proxy = new Proxy(target, {
  get(target, prop) {
    if (prop === "frozen") return 99;  // lying about a non-configurable property
    return target[prop];
  },
});

proxy.frozen;   // TypeError: 'get' on proxy: property 'frozen' is a non-configurable
                // data property that is not writable — must report value 42
```

### 4. `has` trap does NOT affect `for...in`

```js
const proxy = new Proxy({ a: 1 }, {
  has(target, prop) {
    console.log("has:", prop);
    return Reflect.has(target, prop);
  },
});

"a" in proxy;              // triggers has trap: "a"
for (const k in proxy) {} // does NOT trigger has trap — uses ownKeys instead
```

### 5. Proxy does not automatically deep-wrap nested objects

```js
const proxy = new Proxy({ nested: { x: 1 } }, {
  set(target, prop, value) {
    console.log("set:", prop);
    return Reflect.set(target, prop, value);
  },
});

proxy.nested.x = 99;
// logs NOTHING — proxy.nested returns the plain { x: 1 } object
// setting .x on that plain object bypasses the proxy

// Fix: in the get trap, return a Proxy for nested objects too (see Vue 3 example)
```

### 6. `ownKeys` trap must return ALL non-configurable own keys

The `ownKeys` trap cannot omit non-configurable own property keys — the engine will throw if it does:

```js
const target = {};
Object.defineProperty(target, "secret", { value: 1, configurable: false });

const proxy = new Proxy(target, {
  ownKeys() {
    return [];   // hiding "secret" — but it's non-configurable!
  },
});

Object.keys(proxy);   // TypeError: 'ownKeys' on proxy: trap result did not include
                      // all non-configurable own keys
```

---

## Common mistakes

### Mistake 1 — Not returning `true` from `set` trap

```js
// WRONG — silent failure or TypeError in strict mode
const proxy = new Proxy({}, {
  set(target, prop, value) {
    if (typeof value !== "number") return;   // falsy — throws in strict mode
    target[prop] = value;
    // no return statement — undefined is falsy
  },
});

// RIGHT
const proxy = new Proxy({}, {
  set(target, prop, value, receiver) {
    if (typeof value !== "number") {
      throw new TypeError(`${String(prop)} must be a number`);
    }
    return Reflect.set(target, prop, value, receiver);   // always return the result
  },
});
```

### Mistake 2 — Using `target[prop]` instead of `Reflect.get` (breaks inherited getters)

```js
const proto = {
  get computed() { return this.x * 2; }
};
const obj = Object.create(proto);
obj.x = 5;

// WRONG — getter's "this" = target (the raw object), not the proxy
const proxy = new Proxy(obj, {
  get(target, prop) { return target[prop]; }
});
proxy.computed;   // 10 — seems fine here, but breaks when proxy overrides this

// RIGHT — receiver ensures "this" in getter = proxy
const proxy = new Proxy(obj, {
  get(target, prop, receiver) { return Reflect.get(target, prop, receiver); }
});
```

### Mistake 3 — Mutating the target directly instead of through the proxy

```js
const proxy = new Proxy(target, handler);

// WRONG — bypasses all traps
target.newProp = "value";       // goes directly to target, no set trap fires

// RIGHT — always mutate through the proxy if you want traps to fire
proxy.newProp = "value";        // set trap fires
```

### Mistake 4 — Assuming `proxy instanceof TargetClass` works without a trap

```js
class Animal {}
const animal = new Animal();
const proxy  = new Proxy(animal, {});

proxy instanceof Animal;   // true — works because getPrototypeOf is not trapped

// If you define a getPrototypeOf trap that lies, instanceof breaks:
const proxy2 = new Proxy(animal, {
  getPrototypeOf() { return Object.prototype; }
});
proxy2 instanceof Animal;   // false — prototype chain lied
```

---

## Practice exercises

### Exercise 1 — easy

Build a `createTypedObject(types)` function that returns a Proxy enforcing type constraints:

```js
const point = createTypedObject({ x: "number", y: "number", label: "string" });

point.x = 10;          // ✓
point.y = 20;          // ✓
point.label = "A";     // ✓
point.x = "hello";     // TypeError: x must be of type number, got string
point.z = 1;           // TypeError: Unknown property: z
console.log("x" in point);  // true (has trap)
console.log("z" in point);  // false
```

Also implement:
- `get` trap returning `null` for unset properties (instead of `undefined`)
- `ownKeys` trap returning only the keys defined in `types`

```js
// Write your code here
```

### Exercise 2 — medium

Build an **observable store** similar to a minimal Vue `reactive()`:

```js
// Usage:
const store = observable({
  user: { name: "Alice", role: "admin" },
  items: [],
  count: 0,
});

// Subscribe to changes on any nested path:
store.$watch("count", (newVal, oldVal) => {
  console.log(`count: ${oldVal} → ${newVal}`);
});
store.$watch("user.name", (newVal) => {
  console.log(`name changed to: ${newVal}`);
});

store.count = 1;           // logs: count: 0 → 1
store.user.name = "Bob";   // logs: name changed to: Bob
store.items.push("x");     // should NOT trigger watch (array mutation — optional)
```

Requirements:
- `observable(target)` wraps an object (and all nested objects) with Proxy
- Changes to nested properties propagate watchers correctly
- `$watch(path, callback)` — path can be dot-separated like `"user.name"`
- `$unwatch(path, callback)` — removes a specific watcher
- Use WeakMap for storing subscribers so the store can be GC'd

```js
// Write your code here
```

### Exercise 3 — hard

Build a **database model proxy system** that generates SQL queries from property accesses:

```js
// Usage:
const db = createDB(executeQuery);   // executeQuery is an async fn that runs SQL

const users = db.table("users");

// These generate and execute SQL:
await users.findAll();
// SELECT * FROM users

await users.find({ id: 1 });
// SELECT * FROM users WHERE id = 1

await users.find({ role: "admin", active: true });
// SELECT * FROM users WHERE role = 'admin' AND active = true

await users.create({ name: "Alice", email: "alice@example.com" });
// INSERT INTO users (name, email) VALUES ('Alice', 'alice@example.com')

await users.update({ id: 1 }, { name: "Alicia" });
// UPDATE users SET name = 'Alicia' WHERE id = 1

await users.delete({ id: 1 });
// DELETE FROM users WHERE id = 1

// Chainable query builder via Proxy:
await users.where({ role: "admin" }).limit(10).orderBy("name").findAll();
// SELECT * FROM users WHERE role = 'admin' ORDER BY name LIMIT 10
```

The query builder methods (`where`, `limit`, `orderBy`) should return a Proxy that:
- Stores the chain state internally
- Any unknown property access on the chain returns another Proxy continuing the chain
- Terminal methods (`findAll`, `find`, `create`, `update`, `delete`) execute the SQL

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Detail |
|---|---|
| Create proxy | `new Proxy(target, handler)` |
| Revocable proxy | `const { proxy, revoke } = Proxy.revocable(target, handler)` |
| `get` trap | `get(target, prop, receiver)` — intercepts property read |
| `set` trap | `set(target, prop, value, receiver)` — intercepts write; must return `true` |
| `has` trap | `has(target, prop)` — intercepts `"x" in proxy` |
| `deleteProperty` trap | `deleteProperty(target, prop)` — intercepts `delete proxy.x` |
| `apply` trap | `apply(target, thisArg, args)` — intercepts function call |
| `construct` trap | `construct(target, args, newTarget)` — intercepts `new` |
| `ownKeys` trap | `ownKeys(target)` — intercepts `Object.keys()` etc. |
| `getPrototypeOf` trap | Intercepts `Object.getPrototypeOf()`, `instanceof` |
| Missing trap | Falls through to target — transparent |
| `Reflect.get` | Safe property read; correct receiver for inherited getters |
| `Reflect.set` | Safe property write; returns `true`/`false` |
| `Reflect.has` | Like `in` operator |
| `Reflect.apply` | Like `.call()` / `.apply()` |
| `Reflect.construct` | Like `new` |
| `Reflect.ownKeys` | Returns ALL own keys (string + Symbol) |
| Proxy identity | `proxy !== target` — different objects |
| Deep proxying | Must wrap nested objects in `get` trap manually |
| Invariants | Non-configurable/non-writable properties — trap cannot lie |
| `set` must return `true` | Falsy return → TypeError in strict mode |
| `ownKeys` rule | Cannot omit non-configurable own keys |
| Revoke | `revoke()` permanently disables proxy → TypeError on access |

---

## Connected topics

- **47 — ES6 Classes** — Proxy can wrap class instances to intercept method calls and property access
- **51 — Private class fields** — `#` fields bypass Proxy traps entirely — cannot be intercepted
- **60 — Symbol** — Proxy traps fire for Symbol-keyed property access too; `Symbol.toPrimitive` coercion passes through Proxy
- **61 — WeakMap/WeakSet** — Vue 3's reactivity system uses WeakMap (via Proxy) to track dependencies without memory leaks
- **62 — Generators** — `apply` trap can intercept generator function calls
