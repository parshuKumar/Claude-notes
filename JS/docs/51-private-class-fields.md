# 51 — Private Class Fields (#)

## What is this?

A private class field is a property that is **completely inaccessible from outside the class** — no reading, no writing, no deleting, no even knowing it exists. You declare it with a `#` prefix, and only code written inside that exact class body can touch it. Think of it like a locked safe inside a building: the building (class) has it, employees working inside (class methods) can open it, but visitors from outside (any other code) literally cannot reach it — not even with a master key. This is **true encapsulation**, not just convention.

## Why does it matter?

Before private fields, the only way to signal "this is internal" was the `_underscore` convention — but that was just a polite request, not a lock. Anyone could still write `user._password = "hacked"` and nothing would stop them. Private fields (`#`) enforce the boundary at the language level. This matters for security-sensitive data (tokens, passwords, internal state), for building reliable APIs where internal implementation can change without breaking external code, and for preventing accidental mutations that cause hard-to-trace bugs.

---

## Syntax

```js
class BankAccount {
  #balance;                    // private field — declared in class body
  #owner;                      // must be declared before use (unlike regular properties)
  #transactionHistory = [];    // private field with default value

  constructor(owner, initialBalance) {
    this.#owner   = owner;          // write to private field
    this.#balance = initialBalance;
  }

  deposit(amount) {
    this.#validateAmount(amount);   // call private method
    this.#balance += amount;
    this.#transactionHistory.push({ type: "deposit", amount });
  }

  get balance() {                   // public getter — controlled read access
    return this.#balance;
  }

  #validateAmount(amount) {         // private method — only callable inside class
    if (typeof amount !== "number" || amount <= 0) {
      throw new TypeError("Amount must be a positive number.");
    }
  }
}

const account = new BankAccount("Alice", 500);
account.deposit(200);
console.log(account.balance);          // 700  — allowed via getter

console.log(account.#balance);         // SyntaxError: Private field '#balance' must be
                                       // declared in an enclosing class
console.log(account["#balance"]);      // undefined  — not accessible via bracket notation
```

---

## How it works — line by line

```js
#balance;
```
Declares a private field. It must appear at the **top of the class body**, before the constructor. It starts as `undefined` until assigned.

```js
#transactionHistory = [];
```
Declaration with a default value. Every instance gets its own fresh copy of this array.

```js
this.#balance = initialBalance;
```
Reading and writing private fields uses the same `this.#fieldName` syntax inside any method of the class.

```js
#validateAmount(amount) { ... }
```
A **private method** — the same `#` prefix applies to methods too. Completely invisible outside the class.

```js
account.#balance  // SyntaxError
```
The JavaScript engine rejects this at **parse time** (before the code even runs). It's not a runtime error you can catch — the file simply won't execute.

---

## Private fields vs `_` convention

| Feature | `_property` (convention) | `#property` (private field) |
|---|---|---|
| Actually private? | No — just a naming hint | Yes — engine-enforced |
| Accessible from outside? | Yes — `obj._prop` works | No — SyntaxError |
| Accessible via `obj["_prop"]`? | Yes | No — returns `undefined` |
| Shows up in `Object.keys()`? | Yes | No |
| Shows up in `for...in`? | Yes | No |
| JSON serialized? | Yes (`JSON.stringify`) | No — completely invisible |
| Works in subclasses? | Yes | Only in the declaring class |
| Performance | Same | Slightly optimised by engines |

---

## Example 1 — basic: user with private credentials

```js
class User {
  #passwordHash;
  #loginAttempts = 0;
  #maxAttempts   = 5;
  #locked        = false;

  constructor(username, password) {
    this.username    = username;           // public — OK to expose
    this.#passwordHash = this.#hash(password);  // private — never stored raw
  }

  // Primitive hash (demo only — use bcrypt in production)
  #hash(value) {
    return value.split("").reverse().join("") + "_hashed";
  }

  login(password) {
    if (this.#locked) {
      return "Account locked. Contact support.";
    }
    if (this.#hash(password) === this.#passwordHash) {
      this.#loginAttempts = 0;
      return `Welcome back, ${this.username}!`;
    }
    this.#loginAttempts++;
    if (this.#loginAttempts >= this.#maxAttempts) {
      this.#locked = true;
      return "Too many failed attempts. Account locked.";
    }
    return `Wrong password. ${this.#maxAttempts - this.#loginAttempts} attempts remaining.`;
  }

  changePassword(oldPassword, newPassword) {
    if (this.#hash(oldPassword) !== this.#passwordHash) {
      throw new Error("Current password is incorrect.");
    }
    this.#passwordHash = this.#hash(newPassword);
    return "Password changed successfully.";
  }

  get isLocked() { return this.#locked; }
}

const alice = new User("alice99", "SecurePass1");
console.log(alice.login("wrongpass"));      // "Wrong password. 4 attempts remaining."
console.log(alice.login("SecurePass1"));    // "Welcome back, alice99!"
console.log(alice.changePassword("SecurePass1", "NewPass2"));

// Cannot read private fields from outside:
console.log(alice.#passwordHash);  // SyntaxError — blocked at parse time
console.log(Object.keys(alice));   // ["username"]  — private fields invisible
```

---

## Example 2 — private static fields (class-level private state)

```js
class IdGenerator {
  static #nextId    = 1;
  static #instances = new Map();  // track all created instances

  #id;
  #createdAt;

  constructor(label) {
    this.#id        = IdGenerator.#nextId++;
    this.#createdAt = new Date();
    this.label      = label;
    IdGenerator.#instances.set(this.#id, this);
  }

  get id()        { return this.#id; }
  get createdAt() { return this.#createdAt; }

  static getById(id) {
    return IdGenerator.#instances.get(id) ?? null;
  }

  static getCount() {
    return IdGenerator.#instances.size;
  }

  toString() {
    return `[${this.#id}] ${this.label} (created ${this.#createdAt.toISOString()})`;
  }
}

const a = new IdGenerator("Task");
const b = new IdGenerator("User");
const c = new IdGenerator("Product");

console.log(a.id);                        // 1
console.log(IdGenerator.getCount());      // 3
console.log(IdGenerator.getById(2).label); // "User"
console.log(c.toString());                // "[3] Product (created ...)"

// Private static is truly locked:
console.log(IdGenerator.#nextId);   // SyntaxError
```

---

## Example 3 — real world: rate limiter

```js
class RateLimiter {
  #limit;
  #windowMs;
  #requests = new Map();   // ip → [timestamps]

  constructor(limit = 100, windowMs = 60_000) {
    this.#limit    = limit;     // max requests per window
    this.#windowMs = windowMs;  // window size in ms
  }

  isAllowed(clientId) {
    const now  = Date.now();
    const hits  = this.#requests.get(clientId) ?? [];

    // Remove timestamps outside the window
    const recent = hits.filter(t => now - t < this.#windowMs);

    if (recent.length >= this.#limit) {
      this.#requests.set(clientId, recent);
      return { allowed: false, remaining: 0 };
    }

    recent.push(now);
    this.#requests.set(clientId, recent);
    return { allowed: true, remaining: this.#limit - recent.length };
  }

  getRemainingRequests(clientId) {
    const now    = Date.now();
    const hits   = this.#requests.get(clientId) ?? [];
    const recent = hits.filter(t => now - t < this.#windowMs);
    return Math.max(0, this.#limit - recent.length);
  }

  // Private helper — not callable from outside
  #cleanup() {
    const now = Date.now();
    for (const [id, hits] of this.#requests) {
      const recent = hits.filter(t => now - t < this.#windowMs);
      if (recent.length === 0) this.#requests.delete(id);
      else this.#requests.set(id, recent);
    }
  }
}

const limiter = new RateLimiter(3, 5000);  // 3 requests per 5 seconds (for demo)

console.log(limiter.isAllowed("user_1"));   // { allowed: true, remaining: 2 }
console.log(limiter.isAllowed("user_1"));   // { allowed: true, remaining: 1 }
console.log(limiter.isAllowed("user_1"));   // { allowed: true, remaining: 0 }
console.log(limiter.isAllowed("user_1"));   // { allowed: false, remaining: 0 }
console.log(limiter.isAllowed("user_2"));   // { allowed: true, remaining: 2 }
```

---

## Private fields in inheritance — the key restriction

Private fields are **NOT inherited**. A subclass cannot access the parent's private fields, even with `super`:

```js
class Animal {
  #name;

  constructor(name) {
    this.#name = name;
  }

  getName() {
    return this.#name;    // works — we're inside Animal
  }
}

class Dog extends Animal {
  constructor(name, breed) {
    super(name);
    this.breed = breed;
  }

  describe() {
    return `${this.getName()} is a ${this.breed}`;  // OK — using public method
    // return `${this.#name} is a ${this.breed}`;   // SyntaxError — can't access parent's private
  }
}

const d = new Dog("Rex", "Labrador");
console.log(d.describe());    // "Rex is a Labrador"
console.log(d.getName());     // "Rex"  — public method works fine
```

This is intentional. Private fields belong to the class that declares them — period. Subclasses must use the public or protected API (getters/methods) to access parent state.

---

## Checking if a private field exists: `#field in obj`

You can check whether an object has a particular private field without triggering an error:

```js
class Circle {
  #radius;
  constructor(r) { this.#radius = r; }

  static isCircle(obj) {
    return #radius in obj;   // 'in' operator with private field name
  }
}

class Square {
  #side;
  constructor(s) { this.#side = s; }
}

const c = new Circle(5);
const s = new Square(4);

console.log(Circle.isCircle(c));  // true
console.log(Circle.isCircle(s));  // false — Square doesn't have #radius
console.log(Circle.isCircle({})); // false
```

This is used to build **branded checks** — a way to verify that an object was genuinely created by a specific class, not just duck-typed.

---

## Private fields + WeakRef pattern (advanced)

Private fields can hold any value, including references, symbols, or even WeakRefs for memory-safe callbacks:

```js
class Timer {
  #intervalId = null;
  #tickCount   = 0;
  #callback;

  constructor(callback) {
    this.#callback = callback;
  }

  start(intervalMs = 1000) {
    if (this.#intervalId !== null) return;   // already running
    this.#intervalId = setInterval(() => {
      this.#tickCount++;
      this.#callback(this.#tickCount);
    }, intervalMs);
  }

  stop() {
    clearInterval(this.#intervalId);
    this.#intervalId = null;
  }

  get ticks() { return this.#tickCount; }
  get isRunning() { return this.#intervalId !== null; }

  reset() {
    this.stop();
    this.#tickCount = 0;
  }
}

const timer = new Timer(count => console.log(`Tick #${count}`));
// timer.#intervalId  — SyntaxError: no way to clear this from outside
// timer.stop()       — the only way to control it
```

---

## Tricky things

### 1. Private fields MUST be declared in the class body — no dynamic addition

```js
class Foo {
  // NOT declared: #bar

  constructor() {
    this.#bar = 5;   // SyntaxError: Private field '#bar' must be declared in an enclosing class
  }
}

// You cannot add private fields at runtime — unlike regular properties:
const obj = {};
obj.name = "Alice";       // fine — regular properties are dynamic
// obj.#secret = "x";    // SyntaxError — always
```

### 2. Private fields do NOT show up in any standard reflection API

```js
class Wallet {
  #pin = 1234;
  publicBalance = 500;
}

const w = new Wallet();
console.log(Object.keys(w));                          // ["publicBalance"]
console.log(Object.getOwnPropertyNames(w));           // ["publicBalance"]
console.log(JSON.stringify(w));                       // '{"publicBalance":500}'
console.log(w.hasOwnProperty("#pin"));                // false
console.log(w["#pin"]);                               // undefined — not the same as #pin
```

`#pin` and `"#pin"` are completely different things. The string `"#pin"` is just a regular property name starting with `#`. The actual private field `#pin` is invisible to all standard tools.

### 3. Two instances of the same class CAN access each other's private fields

This surprises most people — it's intentional for things like equality checks and copy constructors:

```js
class Point {
  #x;
  #y;

  constructor(x, y) {
    this.#x = x;
    this.#y = y;
  }

  // Can access OTHER Point instance's private fields — same class body
  distanceTo(other) {
    const dx = this.#x - other.#x;   // accessing other's #x — allowed!
    const dy = this.#y - other.#y;
    return Math.sqrt(dx ** 2 + dy ** 2);
  }

  equals(other) {
    return this.#x === other.#x && this.#y === other.#y;
  }
}

const p1 = new Point(0, 0);
const p2 = new Point(3, 4);
console.log(p1.distanceTo(p2));  // 5
console.log(p1.equals(p2));      // false
```

The rule is: **same class body** = access allowed. Not "same instance" — same *class*.

### 4. Private fields are not part of the prototype

```js
class Demo {
  #value = 42;
}

console.log(Demo.prototype);    // {} — no #value here
// Private fields live on INSTANCES only, stamped on during construction
```

### 5. Subclass cannot shadow a parent's private field

```js
class Parent {
  #secret = "parent";
  getSecret() { return this.#secret; }
}

class Child extends Parent {
  #secret = "child";            // This is a DIFFERENT field, not an override
  getChildSecret() { return this.#secret; }
}

const c = new Child();
console.log(c.getSecret());       // "parent"  — Parent's #secret
console.log(c.getChildSecret());  // "child"   — Child's own separate #secret
// They coexist — two different fields that happen to share a name
```

---

## Common mistakes

### Mistake 1 — Forgetting to declare the field at the top

```js
// WRONG — using without declaration
class Config {
  constructor(apiKey) {
    this.#apiKey = apiKey;   // SyntaxError: Private field '#apiKey' must be declared
  }
}

// RIGHT
class Config {
  #apiKey;                   // declare first
  constructor(apiKey) {
    this.#apiKey = apiKey;
  }
}
```

### Mistake 2 — Trying to access private fields in a subclass

```js
class Vehicle {
  #vin;
  constructor(vin) { this.#vin = vin; }
}

class Car extends Vehicle {
  getVin() {
    return this.#vin;   // SyntaxError — #vin belongs to Vehicle, not Car
  }
}

// RIGHT — expose it via a protected-style getter in the parent
class Vehicle {
  #vin;
  constructor(vin) { this.#vin = vin; }
  get vin() { return this.#vin; }   // public read access
}

class Car extends Vehicle {
  getVin() {
    return this.vin;   // uses parent's getter — correct
  }
}
```

### Mistake 3 — Using `"#field"` (string) thinking it's the same as `#field`

```js
class Secret {
  #code = 1234;
}

const s = new Secret();
console.log(s["#code"]);   // undefined — this is a regular string property lookup
                            // "#code" as a string is NOT the private field #code
```

---

## Practice exercises

### Exercise 1 — easy

Create a class `Counter` with:
- A private field `#count` initialised to `0`
- A private field `#step` (how much to increment/decrement, set in constructor, default `1`)
- Public methods: `increment()`, `decrement()`, `reset()`
- A getter `value` that returns the current count
- A getter `isZero` returning `true` when count is 0
- No direct way to set `#count` from outside — it can only change through the methods

Verify that `counter.#count` throws a SyntaxError (just note this — don't try to run it), and that `Object.keys(counter)` returns an empty array.

```js
// Write your code here
```

### Exercise 2 — medium

Build a class `SecureVault` that stores key-value secrets:
- Private field `#secrets = new Map()`
- Private field `#masterKey`
- Private field `#accessLog = []`
- Constructor takes a `masterKey` string
- `store(key, value, masterKey)` — throws if masterKey is wrong; otherwise stores and logs `{ action: "store", key, timestamp }`
- `retrieve(key, masterKey)` — throws if masterKey wrong or key not found; logs `{ action: "retrieve", key, timestamp }`
- `delete(key, masterKey)` — throws if wrong key; deletes entry; logs it
- `getAuditLog(masterKey)` — returns a **copy** of `#accessLog` (not the original array)
- `has(key)` — returns `true/false` without requiring masterKey (key existence is not sensitive)

```js
// Write your code here
```

### Exercise 3 — hard

Create a class `ObservableState` that acts like a reactive state container (simplified Redux/Zustand):
- Private field `#state` — the current state object (plain object)
- Private field `#listeners = new Map()` — maps `keyPath` strings to arrays of listener functions
- Private field `#history = []` — array of previous states for undo
- Private method `#deepGet(obj, keyPath)` — reads nested value by dot-separated path e.g. `"user.profile.name"`
- Private method `#deepSet(obj, keyPath, value)` — returns a new object with the nested value updated (immutably)
- `get(keyPath)` — public: read current value at path
- `set(keyPath, value)` — public: update state (immutably — replace, don't mutate), push old state to `#history`, notify all listeners registered on that keyPath
- `subscribe(keyPath, callback)` — register listener; returns unsubscribe function
- `undo()` — pop last state from `#history` and restore it; notify all listeners
- `getSnapshot()` — returns a deep copy of current state (not the reference)

Test with a state like `{ user: { name: "Alice", score: 0 }, theme: "light" }`. Subscribe to `"user.score"`, call `set("user.score", 10)`, then `undo()`, verify the listener was called and state rolled back.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Detail |
|---|---|
| Declare private field | `#fieldName;` or `#fieldName = defaultValue;` — at top of class body |
| Read/write | `this.#fieldName` — only inside the declaring class |
| Private method | `#methodName() { ... }` — same `#` prefix |
| Static private | `static #fieldName = value;` — one copy on the class |
| Subclass access | Forbidden — subclass must use public getters/methods |
| Two instances, same class | Can access each other's private fields — allowed |
| Outside access | `SyntaxError` at parse time — not a runtime error you can catch |
| Bracket notation | `obj["#field"]` returns `undefined` — NOT the private field |
| `Object.keys()` | Private fields invisible — not enumerated |
| `JSON.stringify` | Private fields NOT serialised |
| Existence check | `#field in obj` — works inside the class; returns `true/false` |
| Difference from `_` | `_` is convention only; `#` is engine-enforced true privacy |
| Cannot be added dynamically | Must be declared in class body — no runtime addition |
| Subclass shadow | Child `#field` and parent `#field` with same name are **two separate fields** |

---

## Connected topics

- **47 — ES6 Classes** — Private fields are a class-only feature; you need the class foundation.
- **50 — Getters and setters** — The standard pattern is to combine `#privateField` with a public getter/setter that controls access. Topics 50 and 51 are a natural pair.
- **49 — Static methods and properties** — `static #field` (private static) combines both topics. The `IdGenerator` example in topic 49 uses private statics.
