# 60 — Symbol

## What is this?

A **Symbol** is a primitive value that is guaranteed to be **unique** — every time you call `Symbol()`, you get a brand-new value that is not equal to anything else, ever. Think of a Symbol as a serial number stamped on a key at the moment of creation: even if two keys look identical and are labelled the same, their serial numbers are different, so they open different locks.

```js
const a = Symbol("myKey");
const b = Symbol("myKey");

console.log(a === b);   // false — they are different symbols, even with the same label
```

The string you pass to `Symbol("label")` is just a description for debugging — it has no effect on uniqueness.

---

## Why does it matter?

Symbols solve a specific and important problem: **adding properties to objects without risking name collisions**. Before Symbols, if two libraries both added a property called `"id"` to the same object, they would silently overwrite each other. With Symbols, each library creates its own unique key that no other code can accidentally access or overwrite. JavaScript itself uses well-known Symbols to let you hook into core language behaviours (iteration, type coercion, custom `instanceof` logic, etc.).

---

## Syntax

```js
// Creating symbols
const id      = Symbol();              // no description
const userId  = Symbol("userId");      // with description (for debugging only)
const role    = Symbol("role");

// Using as object keys
const user = {
  name:    "Alice",
  [userId]: 123,   // Symbol as computed property key
  [role]:   "admin",
};

// Accessing symbol-keyed properties
console.log(user[userId]);   // 123
console.log(user.userId);    // undefined — dot notation doesn't work for symbols

// Global symbol registry
const s1 = Symbol.for("app.token");   // create or retrieve
const s2 = Symbol.for("app.token");   // retrieves same symbol
console.log(s1 === s2);               // true

Symbol.keyFor(s1);   // "app.token" — get the key back
Symbol.keyFor(Symbol("local"));  // undefined — not in global registry
```

---

## How it works

### Creating a Symbol

`Symbol()` is a factory function — not a constructor. Calling `new Symbol()` throws a `TypeError`. This is intentional: Symbols are primitives, like numbers and strings.

```js
typeof Symbol("x")   // "symbol"
typeof 42            // "number"
typeof "hello"       // "string"

new Symbol()         // TypeError: Symbol is not a constructor
```

### Symbols as object keys

Object keys are normally strings (or Symbols). When you use a string key, any code that knows the string can read the property. When you use a Symbol key, only code that has a reference to that specific Symbol can access the property.

```js
const SECRET = Symbol("secret");

const obj = {
  name: "public data",
  [SECRET]: "private data",
};

// Normal enumeration — Symbols are hidden:
Object.keys(obj)           // ["name"]
Object.values(obj)         // ["public data"]
Object.entries(obj)        // [["name", "public data"]]
JSON.stringify(obj)        // '{"name":"public data"}'  — Symbol key excluded
for (const k in obj) { }  // only iterates "name"

// You can still get them if you know to look:
Object.getOwnPropertySymbols(obj)      // [Symbol(secret)]
Reflect.ownKeys(obj)                   // ["name", Symbol(secret)]

// Access the value:
obj[SECRET]   // "private data"
```

---

## Example 1 — collision-free metadata on objects

```js
// Library A defines its own Symbol
const libA_id = Symbol("id");

// Library B defines its own Symbol
const libB_id = Symbol("id");

// A user object
const user = { name: "Alice" };

// Both libraries annotate the SAME object without conflicting:
user[libA_id] = "lib-a-internal-001";
user[libB_id] = "lib-b-uuid-xyz";

console.log(user[libA_id]);  // "lib-a-internal-001"
console.log(user[libB_id]);  // "lib-b-uuid-xyz"

// Neither library's property is visible to general iteration:
console.log(Object.keys(user));  // ["name"]
```

---

## Example 2 — constants with guaranteed uniqueness (replacing string enums)

```js
// PROBLEM with string-based "enums"
const STATUS_PENDING  = "pending";
const STATUS_APPROVED = "approved";

// A typo or collision makes the check silently pass with any string "pending":
if (order.status === "pending") { /* ... */ }  // any code can fake this

// SOLUTION with Symbols — only your code can create these values
const Status = Object.freeze({
  PENDING:  Symbol("pending"),
  APPROVED: Symbol("approved"),
  REJECTED: Symbol("rejected"),
});

order.status = Status.PENDING;

// Check:
if (order.status === Status.PENDING) { /* ... */ }

// Can't be faked from outside — no one else can create Symbol("pending") and have it === Status.PENDING
// Status.PENDING === Symbol("pending")  // false — different symbol!
```

---

## Example 3 — hidden state in objects (semi-private properties)

```js
const _version = Symbol("version");
const _validate = Symbol("validate");

class Document {
  constructor(content) {
    this.content  = content;
    this[_version] = 1;
  }

  update(newContent) {
    this[_validate](newContent);
    this.content = newContent;
    this[_version]++;
  }

  [_validate](content) {
    if (typeof content !== "string") throw new TypeError("Content must be a string");
    if (content.length === 0)        throw new Error("Content cannot be empty");
  }

  get version() { return this[_version]; }
}

const doc = new Document("Hello");
doc.update("World");

console.log(doc.version);           // 2
console.log(Object.keys(doc));      // ["content"] — symbol properties hidden
console.log(doc[_version]);         // 2 — accessible if you have the Symbol reference
```

Note: Symbol properties are **not truly private** — someone with access to `Object.getOwnPropertySymbols()` can find them. For true privacy, use `#private` fields (topic 51). Symbols give you **namespace safety**, not security.

---

## Well-known Symbols — hooking into the language

JavaScript has a set of built-in Symbols (on `Symbol.*`) that let you customise how your objects interact with language features. These are sometimes called "well-known Symbols".

### `Symbol.iterator` — make any object iterable

```js
// An object is iterable if it has a [Symbol.iterator]() method
// that returns an iterator (an object with a .next() method)

class Range {
  constructor(start, end) {
    this.start = start;
    this.end   = end;
  }

  [Symbol.iterator]() {
    let current = this.start;
    const end   = this.end;

    return {
      next() {
        if (current <= end) {
          return { value: current++, done: false };
        }
        return { value: undefined, done: true };
      },
    };
  }
}

const range = new Range(1, 5);

for (const n of range) {
  console.log(n);   // 1, 2, 3, 4, 5
}

console.log([...range]);          // [1, 2, 3, 4, 5]
const [first, second] = range;   // destructuring works too
console.log(first, second);       // 1 2
```

### `Symbol.toPrimitive` — control type coercion

```js
class Money {
  constructor(amount, currency) {
    this.amount   = amount;
    this.currency = currency;
  }

  [Symbol.toPrimitive](hint) {
    // hint: "number", "string", or "default"
    if (hint === "number")  return this.amount;
    if (hint === "string")  return `${this.amount} ${this.currency}`;
    return this.amount;  // default: used in arithmetic and comparisons
  }
}

const price = new Money(9.99, "USD");

console.log(`Price: ${price}`);    // "Price: 9.99 USD"  (string hint)
console.log(price + 0.01);         // 10       (default hint → number)
console.log(price * 2);            // 19.98    (number hint)
console.log(price > 5);            // true     (number hint)
```

### `Symbol.toStringTag` — control `Object.prototype.toString` output

```js
class Queue {
  #items = [];

  get [Symbol.toStringTag]() { return "Queue"; }

  enqueue(item) { this.#items.push(item); }
  dequeue()     { return this.#items.shift(); }
}

const q = new Queue();
Object.prototype.toString.call(q);    // "[object Queue]"
// Without Symbol.toStringTag: "[object Object]"

// Built-in uses:
Object.prototype.toString.call([]);          // "[object Array]"
Object.prototype.toString.call(new Map());   // "[object Map]"
Object.prototype.toString.call(Promise.resolve()); // "[object Promise]"
```

### `Symbol.hasInstance` — customise `instanceof`

```js
class EvenNumber {
  static [Symbol.hasInstance](value) {
    return typeof value === "number" && value % 2 === 0;
  }
}

console.log(4  instanceof EvenNumber);   // true
console.log(5  instanceof EvenNumber);   // false
console.log(12 instanceof EvenNumber);   // true
// EvenNumber was never instantiated — instanceof is pure custom logic!
```

### `Symbol.species` — control what class is used for derived objects

```js
class MyArray extends Array {
  static get [Symbol.species]() { return Array; }  // map/filter return plain Array, not MyArray
}

const myArr = new MyArray(1, 2, 3);
const mapped = myArr.map(x => x * 2);

console.log(mapped instanceof MyArray);  // false — returns plain Array due to Symbol.species
console.log(mapped instanceof Array);    // true
```

### `Symbol.asyncIterator` — make objects async iterable

```js
class DatabaseCursor {
  constructor(query) {
    this.query = query;
    this.page  = 0;
  }

  async [Symbol.asyncIterator]() {
    return {
      page:  0,
      query: this.query,
      async next() {
        const results = await fetchPage(this.query, this.page++);
        if (results.length === 0) return { value: undefined, done: true };
        return { value: results, done: false };
      },
    };
  }
}

// Usage with for-await-of:
const cursor = new DatabaseCursor("SELECT * FROM users");

for await (const batch of cursor) {
  processBatch(batch);
}
```

### All well-known Symbols summary

| Symbol | Purpose |
|---|---|
| `Symbol.iterator` | `for...of`, spread `[...]`, destructuring |
| `Symbol.asyncIterator` | `for await...of` |
| `Symbol.toPrimitive` | Type coercion: `+`, template literals, comparisons |
| `Symbol.toStringTag` | `Object.prototype.toString.call(x)` result |
| `Symbol.hasInstance` | `instanceof` operator customisation |
| `Symbol.species` | Which class derived methods (map/filter/slice) return |
| `Symbol.isConcatSpreadable` | Whether `[].concat(x)` spreads `x` as elements |
| `Symbol.match` | `string.match(x)` — used in RegExp |
| `Symbol.replace` | `string.replace(x)` — used in RegExp |
| `Symbol.search` | `string.search(x)` — used in RegExp |
| `Symbol.split` | `string.split(x)` — used in RegExp |

---

## Global Symbol registry — `Symbol.for` and `Symbol.keyFor`

The global registry lets you share Symbols across different parts of an application — even across different module files or `<iframe>`s in the browser:

```js
// File A
const TOKEN_KEY = Symbol.for("app.auth.token");

// File B (completely separate)
const TOKEN_KEY = Symbol.for("app.auth.token");
// Same Symbol! Symbol.for looks up by key in the global registry.

// Check:
Symbol.keyFor(TOKEN_KEY);   // "app.auth.token"

// Compare to local Symbol:
const local = Symbol("app.auth.token");
Symbol.keyFor(local);       // undefined — not in registry
local === TOKEN_KEY;        // false — different Symbol

// Use case: when you need the same Symbol across modules without importing it
```

---

## Tricky things

### 1. Symbols are not auto-converted to strings

```js
const sym = Symbol("test");

console.log("Key: " + sym);   // TypeError: Cannot convert a Symbol to a string

// Must be explicit:
console.log("Key: " + sym.toString());   // "Key: Symbol(test)"
console.log("Key: " + sym.description); // "Key: test"
console.log(`Key: ${sym}`);             // TypeError too — template literal also coerces

// Convert safely:
String(sym)           // "Symbol(test)"
sym.toString()        // "Symbol(test)"
sym.description       // "test" (just the label, not "Symbol(...)")
```

### 2. `description` can be `undefined`

```js
Symbol().description           // undefined
Symbol("").description         // ""
Symbol("myKey").description    // "myKey"
```

### 3. `JSON.stringify` silently drops Symbol keys AND Symbol values

```js
const KEY = Symbol("id");

const obj = {
  name: "Alice",
  [KEY]: 123,
  getValue: Symbol("someSymbol"),  // Symbol as value, not key
};

JSON.stringify(obj);
// '{"name":"Alice"}'
// — KEY property silently dropped
// — getValue property silently dropped (Symbol values are omitted)
// No warning, no error — this is a silent data loss trap!

// If you need to serialise Symbol-keyed data, convert manually:
function serializeWithSymbols(obj) {
  const result = { ...obj };  // copies string-keyed enumerable props
  for (const sym of Object.getOwnPropertySymbols(obj)) {
    result[sym.toString()] = obj[sym];  // convert Symbol to string key
  }
  return JSON.stringify(result);
}
```

### 4. Well-known Symbols are NOT in the global registry

```js
Symbol.iterator === Symbol.for("Symbol.iterator");   // false
// Well-known symbols live directly on the Symbol constructor, not in the registry
// They are pre-defined, shared Symbols — not created via Symbol.for()
```

### 5. Symbol properties survive `Object.assign` and spread

```js
const SECRET = Symbol("secret");

const source = { visible: 1, [SECRET]: "hidden" };
const target = Object.assign({}, source);
const spread = { ...source };

// Both methods DO copy Symbol-keyed properties:
console.log(target[SECRET]);   // "hidden"
console.log(spread[SECRET]);   // "hidden"

// But JSON.stringify and for...in still ignore them.
// Object.keys() still ignores them.
```

### 6. Symbols cannot be used as WeakMap/WeakSet keys in older specs (changed in ES2023)

```js
// ES2023 (Stage 4): registered Symbols (Symbol.for) CANNOT be WeakMap keys
// Non-registered (local) Symbols CAN be WeakMap keys in ES2023+
const sym = Symbol("key");
const wm  = new WeakMap();
wm.set(sym, "value");   // valid in ES2023+
```

---

## Common mistakes

### Mistake 1 — Using dot notation to access Symbol properties

```js
const KEY = Symbol("key");
const obj = { [KEY]: "value" };

// WRONG — dot notation with the variable name
obj.KEY;    // undefined — looks for string key "KEY"
obj.key;    // undefined — looks for string key "key"

// RIGHT — bracket notation with the Symbol variable
obj[KEY];   // "value"
```

### Mistake 2 — Expecting Symbol description to identify it

```js
// WRONG assumption — "same description = same Symbol"
const a = Symbol("id");
const b = Symbol("id");

if (a === b) { /* never runs */ }   // they are DIFFERENT symbols

// RIGHT — use Symbol.for() if you need the same symbol in multiple places
const a = Symbol.for("app.id");
const b = Symbol.for("app.id");
a === b;  // true
```

### Mistake 3 — Forgetting `Symbol.for` when sharing across modules

```js
// module-a.js
const EVENT_CLICK = Symbol("click");  // local symbol
export { EVENT_CLICK };

// module-b.js
const EVENT_CLICK = Symbol("click");  // different local symbol!
// Even with the same description, these are different — won't match

// RIGHT — use global registry
const EVENT_CLICK = Symbol.for("myapp.events.click");
// Any module calling Symbol.for("myapp.events.click") gets the SAME symbol
```

---

## Practice exercises

### Exercise 1 — easy

1. Create a Symbol `ID` and a Symbol `ROLE`
2. Create an object `user` with string properties `name` and `email`, and Symbol-keyed properties `ID = 42` and `ROLE = "admin"`
3. Demonstrate that `Object.keys()`, `JSON.stringify()`, and `for...in` skip Symbol properties
4. Use `Object.getOwnPropertySymbols()` and `Reflect.ownKeys()` to retrieve all keys
5. Create `Symbol.for("app.session")` in two separate variables — prove they are `===`

```js
// Write your code here
```

### Exercise 2 — medium

Build a `Collection` class that uses `Symbol.iterator` to be iterable in multiple ways:

- Constructor takes an array of items
- `[Symbol.iterator]()` — iterates items forward (enables `for...of`, spread, destructuring)
- `reverse()` — returns an object with its own `[Symbol.iterator]()` that iterates backwards
- `filter(fn)` — returns an object with its own `[Symbol.iterator]()` that yields only matching items
- `[Symbol.toStringTag]` — returns `"Collection"`
- `[Symbol.toPrimitive](hint)` — returns count for `"number"`, comma-joined for `"string"`, count for `"default"`

Test:
```js
const c = new Collection([1, 2, 3, 4, 5]);
console.log([...c]);               // [1,2,3,4,5]
console.log([...c.reverse()]);     // [5,4,3,2,1]
console.log([...c.filter(x => x % 2 === 0)]);  // [2,4]
console.log(`${c}`);               // "1,2,3,4,5"
console.log(+c);                   // 5
```

```js
// Write your code here
```

### Exercise 3 — hard

Design a Symbol-based **event system** for a Node.js-style `EventEmitter` that uses Symbols as event names:

Requirements:
- Event names are Symbols (prevents string collision between different subsystems)
- `EventBus` class with: `on(symbol, handler)`, `off(symbol, handler)`, `emit(symbol, ...args)`, `once(symbol, handler)`
- A `createEvent(namespace, name)` utility that creates namespaced Symbols via `Symbol.for(\`\${namespace}:\${name}\`)`
- Three separate modules (simulate as objects/classes) each defining their own event symbols via `createEvent`, none importing from each other — they all share events through the global registry via `EventBus`
- `[Symbol.iterator]` on `EventBus` that yields `[symbol, handlersCount]` pairs for all registered events
- `[Symbol.toStringTag]` returning `"EventBus"`

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Detail |
|---|---|
| Create Symbol | `Symbol("label")` — label is just for debugging |
| Uniqueness | Every `Symbol()` call produces a unique value |
| Not a constructor | `new Symbol()` throws `TypeError` |
| `typeof` | `typeof Symbol() === "symbol"` |
| As object key | `{ [sym]: value }` — bracket notation required |
| Access | `obj[sym]` — bracket notation required; `obj.sym` looks for string key |
| Hidden from iteration | `Object.keys`, `for...in`, `JSON.stringify`, `Object.values` all skip symbols |
| Retrieve symbol keys | `Object.getOwnPropertySymbols(obj)`, `Reflect.ownKeys(obj)` |
| Global registry | `Symbol.for("key")` — create or retrieve; shared across modules/iframes |
| Lookup key | `Symbol.keyFor(sym)` — returns string key, or `undefined` if local |
| Description | `sym.description` — the label string |
| Convert to string | `sym.toString()` → `"Symbol(label)"` or `String(sym)` |
| Coercion to string | `"" + sym` throws `TypeError` — must be explicit |
| JSON | Symbol keys and Symbol values are silently dropped |
| `Symbol.iterator` | Enables `for...of`, spread, destructuring |
| `Symbol.asyncIterator` | Enables `for await...of` |
| `Symbol.toPrimitive` | Controls `+`, template literals, comparisons |
| `Symbol.toStringTag` | Controls `Object.prototype.toString.call()` output |
| `Symbol.hasInstance` | Controls `instanceof` |
| `Symbol.species` | Controls which class derived array methods return |

---

## Connected topics

- **37 — Spread operator** — spread on objects does copy Symbol-keyed properties
- **51 — Private class fields (`#`)** — Symbols give namespace safety; `#` gives true privacy
- **61 — WeakMap and WeakSet** — Symbols can be WeakMap keys (ES2023+); both solve "hidden data" problems
- **62 — Generators** — generators use `Symbol.iterator` under the hood
- **63 — Proxy and Reflect** — Proxy traps fire even for Symbol-keyed property access
