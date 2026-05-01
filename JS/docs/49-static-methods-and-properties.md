# 49 — Static Methods and Properties

## What is this?

A static method or property belongs to the **class itself**, not to any instance created from it. Think of it like a department's shared rulebook versus an individual employee's personal notes — the rulebook lives at the department level and every employee can reference it, but it isn't copied into each person's desk drawer. When you write `static` before a method or property inside a class, that member lives directly on the class object and is accessed via the class name, never via `new`.

## Why does it matter?

Statics are how you attach utility functions, factory methods, configuration constants, and class-level counters to a class without polluting every instance. You'll find them everywhere in real codebases: `Array.isArray()`, `Object.keys()`, `Math.random()`, `Promise.all()` — all static methods. Knowing when to reach for `static` versus an instance method is a core design decision in object-oriented JavaScript.

---

## Syntax

```js
class MathHelper {
  static PI = 3.14159265358979;      // static property — belongs to MathHelper

  static square(n) {                  // static method — called on the class
    return n * n;
  }

  static cube(n) {
    return n * n * n;
  }
}

// Called on the CLASS — no 'new' needed
console.log(MathHelper.PI);           // 3.14159265358979
console.log(MathHelper.square(4));    // 16
console.log(MathHelper.cube(3));      // 27

// NOT available on instances
const m = new MathHelper();
console.log(m.square);                // undefined
```

---

## How it works — line by line

```js
static PI = 3.14159265358979;
```
This property is set directly on the `MathHelper` function object itself (remember: a class IS a function). It is **not** on `MathHelper.prototype`, and it is **not** copied onto instances.

```js
static square(n) { return n * n; }
```
This method is also set directly on `MathHelper`, equivalent to writing `MathHelper.square = function(n) { return n * n; }` after the class definition.

```js
MathHelper.square(4)
```
Calls the function stored at `MathHelper.square`. No instance involved.

```js
const m = new MathHelper();
m.square   // undefined
```
Instances look up properties via `[[Prototype]]` → `MathHelper.prototype`. Static members live on `MathHelper` directly, which is NOT in the instance's prototype chain.

---

## Where static members live (memory map)

```
MathHelper (the class / function object)
├── PI: 3.14159...          ← static property
├── square: fn              ← static method
├── cube: fn                ← static method
└── prototype ────────────► MathHelper.prototype
                            ├── constructor: MathHelper
                            └── (instance methods would live here)

instance (new MathHelper())
└── [[Prototype]] ─────────► MathHelper.prototype
                             (no path to MathHelper's own statics)
```

---

## Example 1 — utility class (pure static)

When a class only contains static members it acts as a pure namespace for related functions:

```js
class StringUtils {
  static capitalize(str) {
    if (!str) return "";
    return str.charAt(0).toUpperCase() + str.slice(1).toLowerCase();
  }

  static truncate(str, maxLength, suffix = "...") {
    if (str.length <= maxLength) return str;
    return str.slice(0, maxLength - suffix.length) + suffix;
  }

  static toCamelCase(str) {
    return str
      .toLowerCase()
      .split(/[\s_-]+/)
      .map((word, i) => i === 0 ? word : StringUtils.capitalize(word))
      .join("");
  }

  static countWords(str) {
    return str.trim().split(/\s+/).filter(Boolean).length;
  }
}

console.log(StringUtils.capitalize("hello world"));   // "Hello world"
console.log(StringUtils.truncate("A very long sentence here", 15));  // "A very long ..."
console.log(StringUtils.toCamelCase("my-component-name")); // "myComponentName"
console.log(StringUtils.countWords("  hello   world  "));  // 2
```

---

## Example 2 — factory methods

Static methods are the standard way to provide **alternative constructors** — different ways to create an instance:

```js
class Color {
  constructor(r, g, b) {
    this.r = r;  // red   0-255
    this.g = g;  // green 0-255
    this.b = b;  // blue  0-255
  }

  // Factory: create from hex string "#ff6347"
  static fromHex(hex) {
    const clean = hex.replace("#", "");
    return new Color(
      parseInt(clean.slice(0, 2), 16),
      parseInt(clean.slice(2, 4), 16),
      parseInt(clean.slice(4, 6), 16)
    );
  }

  // Factory: create from CSS rgb string "rgb(255, 99, 71)"
  static fromCssString(css) {
    const [r, g, b] = css.match(/\d+/g).map(Number);
    return new Color(r, g, b);
  }

  // Factory: create a random color
  static random() {
    return new Color(
      Math.floor(Math.random() * 256),
      Math.floor(Math.random() * 256),
      Math.floor(Math.random() * 256)
    );
  }

  toHex() {
    return "#" + [this.r, this.g, this.b]
      .map(n => n.toString(16).padStart(2, "0"))
      .join("");
  }

  toString() {
    return `rgb(${this.r}, ${this.g}, ${this.b})`;
  }
}

const tomato  = Color.fromHex("#ff6347");
const crimson = Color.fromCssString("rgb(220, 20, 60)");
const rnd     = Color.random();

console.log(tomato.toString());    // "rgb(255, 99, 71)"
console.log(crimson.toHex());      // "#dc143c"
console.log(rnd.toHex());          // some random hex
```

---

## Example 3 — class-level counter (instance tracking)

```js
class Connection {
  static #count    = 0;         // private static — tracks total across all instances
  static maxConnections = 10;   // public static — configurable limit

  constructor(host, port) {
    if (Connection.#count >= Connection.maxConnections) {
      throw new Error(`Max connections (${Connection.maxConnections}) reached.`);
    }
    Connection.#count++;
    this.id   = Connection.#count;
    this.host = host;
    this.port = port;
    this.open = true;
  }

  close() {
    if (!this.open) return;
    this.open = false;
    Connection.#count--;   // decrement the class-level counter
  }

  static getActiveCount() {
    return Connection.#count;
  }

  static setMaxConnections(n) {
    Connection.maxConnections = n;
  }

  describe() {
    return `Connection #${this.id} → ${this.host}:${this.port} [${this.open ? "open" : "closed"}]`;
  }
}

const c1 = new Connection("db.example.com", 5432);
const c2 = new Connection("cache.example.com", 6379);
console.log(Connection.getActiveCount());   // 2
c1.close();
console.log(Connection.getActiveCount());   // 1
console.log(c1.describe());                 // "Connection #1 → db.example.com:5432 [closed]"
```

---

## Example 4 — validation / type checking (like built-in statics)

Mirroring the style of `Array.isArray()`, `Number.isNaN()`:

```js
class Validator {
  static isEmail(value) {
    return typeof value === "string" && /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(value);
  }

  static isPositiveInteger(value) {
    return Number.isInteger(value) && value > 0;
  }

  static isNonEmptyString(value) {
    return typeof value === "string" && value.trim().length > 0;
  }

  static assertEmail(value) {
    if (!Validator.isEmail(value)) {
      throw new TypeError(`"${value}" is not a valid email address.`);
    }
  }
}

console.log(Validator.isEmail("alice@example.com"));  // true
console.log(Validator.isEmail("not-an-email"));        // false
console.log(Validator.isPositiveInteger(5));           // true
console.log(Validator.isPositiveInteger(-3));          // false

Validator.assertEmail("bad-email");  // TypeError: "bad-email" is not a valid email address.
```

---

## Static methods are inherited

When a child class `extends` a parent, it inherits the parent's static methods too — because `Dog.__proto__ === Animal` (the class objects themselves are linked):

```js
class Animal {
  static create(name) {
    return new this(name);   // 'this' = the class the method is called ON
  }
  static kingdom() {
    return "Animalia";
  }
}

class Dog extends Animal {
  constructor(name) {
    super();
    this.name = name;
  }
}

// Dog inherits Animal's statics:
console.log(Dog.kingdom());         // "Animalia"

// 'new this' is polymorphic — 'this' is Dog when called on Dog
const rex = Dog.create("Rex");
console.log(rex instanceof Dog);    // true  — NOT Animal, but Dog
console.log(rex.name);              // "Rex"
```

---

## Overriding static methods

Child classes can override inherited static methods just like instance methods:

```js
class Base {
  static describe() {
    return "I am Base";
  }
}

class Child extends Base {
  static describe() {
    return `${super.describe()} — and I am Child`;
  }
}

console.log(Base.describe());   // "I am Base"
console.log(Child.describe());  // "I am Base — and I am Child"
```

---

## `this` inside static methods

Inside a static method, `this` refers to the **class** (or subclass) it was called on — not an instance:

```js
class Registry {
  static items = [];

  static register(item) {
    this.items.push(item);     // 'this' is Registry (or a subclass if inherited)
    return this;               // enables static method chaining
  }

  static list() {
    return [...this.items];
  }

  static clear() {
    this.items = [];
    return this;
  }
}

Registry
  .register("itemA")
  .register("itemB")
  .register("itemC");

console.log(Registry.list());   // ["itemA", "itemB", "itemC"]
Registry.clear();
console.log(Registry.list());   // []
```

---

## Tricky things

### 1. Static properties are NOT shared with instances

```js
class Config {
  static version = "1.0.0";
}

const c = new Config();
console.log(Config.version);   // "1.0.0"  ✓
console.log(c.version);        // undefined — instances don't see statics
```

### 2. Each subclass gets its OWN static property (shadowing)

```js
class Parent {
  static count = 0;
}

class Child extends Parent {}

Parent.count++;
console.log(Parent.count);  // 1
console.log(Child.count);   // 1  — inherited (no own property yet)

Child.count = 99;            // creates an OWN static property on Child
console.log(Child.count);   // 99
console.log(Parent.count);  // 1  — Parent unaffected
```

This is the static equivalent of prototype property shadowing. It's why shared class-level state often needs careful design — if you want all classes to share a counter, you must read/write it explicitly on the base class.

### 3. Calling a static method from an instance method

```js
class Product {
  static taxRate = 0.2;

  constructor(price) {
    this.price = price;
  }

  getPriceWithTax() {
    // Access static via the constructor property — works even in subclasses
    return this.price * (1 + this.constructor.taxRate);
  }
}

class LuxuryProduct extends Product {
  static taxRate = 0.35;   // different tax rate
}

const p = new Product(100);
const l = new LuxuryProduct(100);

console.log(p.getPriceWithTax());  // 120 — uses Product.taxRate
console.log(l.getPriceWithTax());  // 135 — uses LuxuryProduct.taxRate (polymorphic!)
```

Using `this.constructor.staticProp` inside an instance method is the correct way to access a static property in a class-aware, polymorphic manner.

### 4. Static methods can't access instance properties

```js
class User {
  constructor(name) { this.name = name; }

  static greet() {
    return `Hello, ${this.name}!`;  // WRONG — 'this' is the class, not an instance
    // this.name here = "User" (the class's own .name property, which is the class name string)
  }
}
console.log(User.greet());   // "Hello, User!" — not what you expected

// Fix: pass the instance in as an argument
class User {
  constructor(name) { this.name = name; }
  static greet(user) {
    return `Hello, ${user.name}!`;   // correct
  }
}
const alice = new User("Alice");
console.log(User.greet(alice));   // "Hello, Alice!"
```

### 5. `static` before class fields vs methods — same rule, different data

```js
class Demo {
  static methodA() { return "I'm a static method"; }
  static fieldB = "I'm a static property";

  instanceMethod() { return "I'm an instance method (on prototype)"; }
  instanceField = "I'm an instance property (own property per instance)";
}

const d = new Demo();

// Static — on the class
console.log(Demo.methodA());   // "I'm a static method"
console.log(Demo.fieldB);      // "I'm a static property"

// Instance — on the prototype or instance
console.log(d.instanceMethod());  // "I'm an instance method"
console.log(d.instanceField);     // "I'm an instance property"

// Cross-access fails:
console.log(d.methodA);        // undefined — not on instance or prototype
console.log(Demo.instanceField); // undefined — not on the class
```

---

## Common mistakes

### Mistake 1 — Calling a static method with `new` or on an instance

```js
class DateUtils {
  static formatDate(date) {
    return date.toISOString().split("T")[0];
  }
}

const utils = new DateUtils();

// WRONG
console.log(utils.formatDate(new Date()));   // TypeError: utils.formatDate is not a function

// RIGHT
console.log(DateUtils.formatDate(new Date()));  // "2026-05-01"
```

### Mistake 2 — Expecting subclass to share parent's static property by reference

```js
// WRONG assumption: "Child.count and Parent.count are the same variable"
class Parent { static count = 0; }
class Child extends Parent {}

Child.count++;             // This shadows — creates Child's OWN count
console.log(Parent.count); // 0 — unaffected
console.log(Child.count);  // 1

// RIGHT — if you want truly shared state, always reference the base class explicitly:
Parent.count++;
console.log(Parent.count);  // 1
console.log(Child.count);   // 1 — still reads from Parent (no own property yet)
```

### Mistake 3 — Putting stateless utility logic on the prototype instead of static

```js
// WRONG — greet takes no instance data, wastes a prototype slot
class Formatter {
  constructor() {}
  currency(amount) {   // doesn't use 'this' at all
    return `$${amount.toFixed(2)}`;
  }
}
const f = new Formatter();
f.currency(9.9);   // works but forces unnecessary instantiation

// RIGHT
class Formatter {
  static currency(amount) {
    return `$${amount.toFixed(2)}`;
  }
}
Formatter.currency(9.9);   // clean, no instance needed
```

---

## Practice exercises

### Exercise 1 — easy

Create a class `Temperature` with a `celsius` instance property. Add **static** conversion methods (no instance needed):
- `static celsiusToFahrenheit(c)` → `c * 9/5 + 32`
- `static fahrenheitToCelsius(f)` → `(f - 32) * 5/9`
- `static celsiusToKelvin(c)` → `c + 273.15`

Also add a static factory method `static fromFahrenheit(f)` that creates a `Temperature` instance from a Fahrenheit value. Add an instance method `toString()` returning `"[celsius]°C"`. Verify you can convert without creating an instance, and also create an instance via the factory.

```js
// Write your code here
```

### Exercise 2 — medium

Create a class `Cache` that stores key-value pairs. Requirements:
- `static #store = new Map()` — one shared store for the whole class
- `static set(key, value, ttlMs)` — stores the value with an expiry timestamp (`Date.now() + ttlMs`)
- `static get(key)` — returns the value if not expired, or `null` if expired/missing; also **deletes** expired entries on access
- `static has(key)` — returns `true` only if key exists AND is not expired
- `static clear()` — wipes the entire store
- `static size()` — returns count of non-expired entries

Test with at least 3 entries, two of which have different TTLs. Show that an "expired" entry (simulate by setting a past expiry time directly) returns `null`.

```js
// Write your code here
```

### Exercise 3 — hard

Build a class `EventBus` that acts as a **global singleton** using a static property:
- `static #instance = null`
- `static getInstance()` — returns the single shared instance (creates it on first call — this is the Singleton pattern)
- Instance method `subscribe(event, handler)` — register handler
- Instance method `unsubscribe(event, handler)` — remove handler
- Instance method `publish(event, ...data)` — call all handlers for event
- Instance method `publishAsync(event, ...data)` — call all handlers asynchronously (return a `Promise.all` of all handler results if handlers return promises)
- Static method `reset()` — destroys the singleton (sets `#instance = null`) — useful for testing

Demonstrate that calling `EventBus.getInstance()` multiple times returns the **exact same object** (`===`). Subscribe two handlers to the same event, publish to it, then unsubscribe one and publish again to show only one handler fires.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Syntax / Rule |
|---|---|
| Static method | `static methodName() { ... }` — called as `ClassName.methodName()` |
| Static property | `static propName = value;` — accessed as `ClassName.propName` |
| `this` in static | Refers to the **class** (or subclass), not an instance |
| Access from instance | Use `this.constructor.staticProp` for polymorphic access |
| Static inheritance | Child inherits parent statics via `Child.__proto__ === Parent` |
| Static shadowing | Assigning to `Child.staticProp` creates a child-own copy |
| Can't access instance props | Static methods have no instance — pass one as argument if needed |
| Common use cases | Utility/helper functions, factory methods, constants, class counters, singletons |
| Built-in examples | `Array.isArray()`, `Object.keys()`, `Math.random()`, `Promise.all()`, `Number.isNaN()` |
| NOT on prototype | Static methods don't appear in instance's prototype chain |
| Method chaining (static) | Return `this` (= the class) from static methods |

---

## Connected topics

- **47 — ES6 Classes** — Static members are defined inside the class body; you need to understand the basic class syntax first.
- **48 — Inheritance** — Static members are inherited and can be overridden, and `super.staticMethod()` works in child statics.
- **51 — Private class fields (#)** — Static members are often combined with private fields (e.g., `static #instance` for singletons). That topic covers the `#` syntax in depth.
