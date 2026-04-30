# 47 — ES6 Classes

## What is this?

An ES6 class is a **cleaner, more readable syntax for creating constructor functions and wiring up prototype methods** — all in one block. Think of a class as a factory blueprint: you define the shape, properties, and behaviours of an object once, and then stamp out as many instances as you need. Under the hood, JavaScript translates `class` syntax into the exact same constructor function + `prototype` pattern from topics 45 and 46 — no new runtime mechanism is introduced. The keyword `class` is what developers call *syntactic sugar*: sweeter to write, identical in execution.

## Why does it matter?

Classes are the standard way to organise object-oriented code in modern JavaScript. Every major library and framework (React, Vue, Angular, NestJS) uses class syntax. Without understanding it you can't read most professional codebases. And because classes are just constructor functions in disguise, the bugs you can make with them are *exactly* the same bugs from the prototype chapter — so understanding the underlying mechanics makes you dangerous with both.

---

## Syntax

```js
class Person {                          // class keyword + PascalCase name
  constructor(name, age) {              // special method — runs when 'new' is called
    this.name = name;                   // instance properties set here
    this.age  = age;
  }

  greet() {                             // prototype method — defined without 'function'
    return `Hi, I'm ${this.name}.`;
  }

  getBirthYear() {                      // another prototype method
    return new Date().getFullYear() - this.age;
  }
}

const alice = new Person("Alice", 28);  // instantiate with 'new' — same as always
console.log(alice.greet());             // "Hi, I'm Alice."
```

---

## How it works — line by line

```js
class Person {
```
Declares a class named `Person`. JS internally creates a function called `Person` (you can verify: `typeof Person === "function"`).

```js
  constructor(name, age) {
    this.name = name;
    this.age  = age;
  }
```
The `constructor` method is called automatically when you do `new Person(...)`. It's exactly the body of the old-style constructor function. Properties set with `this.` become **own properties** of each instance.

```js
  greet() {
    return `Hi, I'm ${this.name}.`;
  }
```
This method is automatically placed on `Person.prototype.greet`. It is **not** inside the constructor — there's only one copy of it, shared by all instances.

```js
const alice = new Person("Alice", 28);
```
`new` does the same 4 steps as always: creates `{}`, sets `[[Prototype]] = Person.prototype`, runs `constructor`, returns `this`.

---

## Proof: class is constructor function in disguise

```js
class Animal {
  constructor(name) { this.name = name; }
  speak() { return `${this.name} speaks.`; }
}

// 1. The class IS a function
console.log(typeof Animal);                          // "function"

// 2. Methods live on the prototype
console.log(typeof Animal.prototype.speak);          // "function"

// 3. Instances are linked via prototype chain — identical to topic 46
const cat = new Animal("Whiskers");
console.log(Object.getPrototypeOf(cat) === Animal.prototype);  // true

// 4. The prototype's constructor points back to the class
console.log(Animal.prototype.constructor === Animal);          // true

// 5. instanceof works as expected
console.log(cat instanceof Animal);  // true
console.log(cat instanceof Object);  // true
```

---

## Example 1 — basic: BankAccount

```js
class BankAccount {
  constructor(owner, initialBalance = 0) {  // default parameter
    this.owner   = owner;
    this.balance = initialBalance;
    this.transactions = [];                  // own array per instance
  }

  deposit(amount) {
    if (amount <= 0) throw new Error("Deposit amount must be positive.");
    this.balance += amount;
    this.transactions.push({ type: "deposit", amount });
    return this.balance;
  }

  withdraw(amount) {
    if (amount <= 0)          throw new Error("Withdrawal amount must be positive.");
    if (amount > this.balance) throw new Error("Insufficient funds.");
    this.balance -= amount;
    this.transactions.push({ type: "withdrawal", amount });
    return this.balance;
  }

  getStatement() {
    const lines = this.transactions.map(
      t => `  ${t.type}: $${t.amount.toFixed(2)}`
    );
    return [
      `Account owner : ${this.owner}`,
      `Balance       : $${this.balance.toFixed(2)}`,
      `Transactions  :`,
      ...lines
    ].join("\n");
  }
}

const account = new BankAccount("Alice", 500);
account.deposit(200);
account.withdraw(75);
console.log(account.getStatement());
/*
Account owner : Alice
Balance       : $625.00
Transactions  :
  deposit: $200.00
  withdrawal: $75.00
*/
```

---

## Example 2 — real world: ProductCatalog

```js
class Product {
  constructor(id, name, price, stock) {
    this.id    = id;
    this.name  = name;
    this.price = price;
    this.stock = stock;
  }

  isAvailable() {
    return this.stock > 0;
  }

  applyDiscount(percent) {
    if (percent < 0 || percent > 100) throw new Error("Invalid discount percent.");
    this.price = parseFloat((this.price * (1 - percent / 100)).toFixed(2));
    return this;    // return 'this' to allow method chaining
  }

  restock(units) {
    this.stock += units;
    return this;
  }

  describe() {
    const availability = this.isAvailable() ? `${this.stock} in stock` : "out of stock";
    return `[${this.id}] ${this.name} — $${this.price} (${availability})`;
  }
}

const laptop = new Product("P001", "ThinkPad X1", 1299.99, 5);
const mouse  = new Product("P002", "Logitech MX", 79.99,   0);

console.log(laptop.describe());   // [P001] ThinkPad X1 — $1299.99 (5 in stock)
console.log(mouse.describe());    // [P002] Logitech MX — $79.99 (out of stock)

// Method chaining with 'return this'
laptop.applyDiscount(10).restock(3);
console.log(laptop.describe());   // [P001] ThinkPad X1 — $1169.99 (8 in stock)
```

---

## Class expressions (classes are values too)

Just like function expressions, classes can be assigned to variables:

```js
// Named class expression
const Rectangle = class RectShape {
  constructor(w, h) {
    this.width  = w;
    this.height = h;
  }
  area() { return this.width * this.height; }
};

const r = new Rectangle(5, 3);
console.log(r.area());   // 15

// Anonymous class expression
const Circle = class {
  constructor(radius) { this.radius = radius; }
  area() { return Math.PI * this.radius ** 2; }
};

// Class as an argument (factory pattern)
function createInstance(ClassRef, ...args) {
  return new ClassRef(...args);
}
const c = createInstance(Circle, 7);
console.log(c.area().toFixed(2));  // "153.94"
```

---

## Method types inside a class

```js
class Counter {
  // 1. Constructor — runs on 'new'
  constructor(start = 0) {
    this.count = start;
  }

  // 2. Regular (prototype) method — shared, lives on Counter.prototype
  increment() {
    this.count++;
    return this;
  }

  decrement() {
    this.count--;
    return this;
  }

  // 3. Getter — accessed like a property, no ()
  get value() {
    return this.count;
  }

  // 4. Setter — called when you assign to 'value'
  set value(newCount) {
    if (typeof newCount !== "number") throw new TypeError("Count must be a number.");
    this.count = newCount;
  }

  // 5. Static method — called on the CLASS, not instances
  static create(start) {
    return new Counter(start);
  }

  // 6. toString override — used when object is coerced to string
  toString() {
    return `Counter(${this.count})`;
  }
}

const c = Counter.create(10);    // static method
c.increment().increment();        // method chaining
console.log(c.value);             // 12  — getter (no parentheses)
c.value = 50;                     // setter
console.log(c.value);             // 50
console.log(`${c}`);              // "Counter(50)"  — toString
```

---

## Getters and setters in depth

```js
class Temperature {
  constructor(celsius) {
    this._celsius = celsius;    // _ prefix = "internal, don't touch directly"
  }

  // Getter — computed on read
  get fahrenheit() {
    return this._celsius * 9/5 + 32;
  }

  // Setter — validation on write
  set celsius(value) {
    if (value < -273.15) throw new RangeError("Temperature below absolute zero.");
    this._celsius = value;
  }

  get celsius() {
    return this._celsius;
  }
}

const temp = new Temperature(100);
console.log(temp.fahrenheit);  // 212 — calculated, not stored
temp.celsius = 0;
console.log(temp.fahrenheit);  // 32
temp.celsius = -300;           // RangeError: Temperature below absolute zero.
```

---

## Static methods and properties

Static members belong to the **class itself**, not to any instance. Use them for utility functions, factory methods, or class-level state.

```js
class MathUtils {
  // Static method — no 'this.property' because there's no instance
  static add(a, b)      { return a + b; }
  static multiply(a, b) { return a * b; }
  static clamp(val, min, max) { return Math.min(Math.max(val, min), max); }
}

// Called on the CLASS, never on an instance
console.log(MathUtils.add(3, 4));         // 7
console.log(MathUtils.clamp(150, 0, 100)); // 100

// Static property (class-level counter)
class IdGenerator {
  static #nextId = 1;    // private static field (see topic 51)

  static generate() {
    return IdGenerator.#nextId++;
  }
}

console.log(IdGenerator.generate());  // 1
console.log(IdGenerator.generate());  // 2
console.log(IdGenerator.generate());  // 3
```

---

## Class fields (modern syntax — ES2022)

You can declare instance properties at the top of the class body without putting them in the constructor:

```js
class GamePlayer {
  // Public class fields — initialised on every instance
  score  = 0;
  lives  = 3;
  name;           // declared but undefined until set

  // Private class fields — topic 51 covers these in depth
  #secretCode = Math.random();

  constructor(name) {
    this.name = name;    // can still set in constructor
  }

  addPoints(n) {
    this.score += n;
    return this;
  }

  loseLife() {
    if (this.lives === 0) throw new Error("Game over.");
    this.lives--;
    return this;
  }

  status() {
    return `${this.name} | Score: ${this.score} | Lives: ${this.lives}`;
  }
}

const player = new GamePlayer("Alice");
player.addPoints(100).addPoints(50).loseLife();
console.log(player.status());  // "Alice | Score: 150 | Lives: 2"
```

---

## Important class vs function constructor differences

| Feature | Constructor Function | ES6 Class |
|---|---|---|
| Syntax | `function Person() {}` | `class Person {}` |
| Must use `new`? | No (fails silently) | Yes (throws `TypeError` if omitted) |
| Hoisted? | Yes (fully) | No — TDZ applies |
| Methods auto on prototype? | No — you add manually | Yes — all non-static methods go on prototype |
| `"use strict"` | Opt-in | Always strict inside class body |
| Method enumerable? | `true` by default | `false` — won't show in `for...in` |
| Inheritance syntax | Manual prototype wiring | `extends` keyword (topic 48) |

### Class hoisting (Temporal Dead Zone)

```js
// Function constructor — hoisted, works
const a = new Animal("Rex");  // OK
function Animal(name) { this.name = name; }

// Class — NOT hoisted (TDZ)
const b = new Pet("Whiskers");  // ReferenceError: Cannot access 'Pet' before initialization
class Pet { constructor(name) { this.name = name; } }
```

### Class body is always strict mode

```js
class Demo {
  test() {
    undeclaredVar = 5;  // ReferenceError in strict mode — would silently work outside class
  }
}
```

### Methods are non-enumerable

```js
class Foo {
  bar() {}
}
const f = new Foo();
console.log(Object.keys(f));                    // []  — 'bar' is NOT enumerable
console.log(Object.getOwnPropertyNames(Foo.prototype));  // ["constructor", "bar"]

// Compare with constructor function:
function OldFoo() {}
OldFoo.prototype.bar = function() {};
for (const key in new OldFoo()) {
  console.log(key);  // "bar" — prototype methods are enumerable in old style
}
```

---

## Method chaining pattern

Returning `this` from methods allows you to chain calls:

```js
class QueryBuilder {
  constructor(table) {
    this._table      = table;
    this._conditions = [];
    this._limit      = null;
    this._orderBy    = null;
  }

  where(condition) {
    this._conditions.push(condition);
    return this;    // returns the instance so next method can chain
  }

  orderBy(field, direction = "ASC") {
    this._orderBy = `${field} ${direction}`;
    return this;
  }

  limit(n) {
    this._limit = n;
    return this;
  }

  build() {
    let query = `SELECT * FROM ${this._table}`;
    if (this._conditions.length > 0) {
      query += ` WHERE ${this._conditions.join(" AND ")}`;
    }
    if (this._orderBy) query += ` ORDER BY ${this._orderBy}`;
    if (this._limit)   query += ` LIMIT ${this._limit}`;
    return query;
  }
}

const sql = new QueryBuilder("users")
  .where("age > 18")
  .where("active = true")
  .orderBy("createdAt", "DESC")
  .limit(10)
  .build();

console.log(sql);
// SELECT * FROM users WHERE age > 18 AND active = true ORDER BY createdAt DESC LIMIT 10
```

---

## Tricky things

### 1. `this` is lost when you extract a method

```js
class Timer {
  constructor() { this.seconds = 0; }
  tick() { this.seconds++; console.log(this.seconds); }
}

const t = new Timer();
t.tick();            // 1  — works

const tick = t.tick; // extracted the method reference
tick();              // TypeError: Cannot read properties of undefined — 'this' is undefined
                     // (class bodies are strict mode, so 'this' is undefined, not global)

// Fix 1: bind
const tick2 = t.tick.bind(t);
tick2();             // 2

// Fix 2: arrow function in constructor (own property, not prototype method)
class Timer2 {
  constructor() {
    this.seconds = 0;
    this.tick = () => this.seconds++;   // 'this' is lexically captured
  }
}
```

### 2. Calling the class without `new` throws

```js
class Person { constructor(name) { this.name = name; } }

Person("Alice");   // TypeError: Class constructor Person cannot be invoked without 'new'
// Compare: old constructor functions silently corrupt global scope
```

### 3. Computed method names

```js
const methodName = "greet";

class Greeter {
  [methodName]() {              // computed method name
    return "Hello!";
  }
  [`get${methodName.charAt(0).toUpperCase() + methodName.slice(1)}`]() {
    return "Getting greeting";
  }
}

const g = new Greeter();
console.log(g.greet());         // "Hello!"
console.log(g.getGreet());      // "Getting greeting"
```

### 4. Classes are not hoisted — but class expressions assigned to `const` are in TDZ

```js
// Both of these fail:
const a = new MyClass();     // ReferenceError
class MyClass {}

const b = new AnotherClass();  // ReferenceError
const AnotherClass = class {};
```

### 5. `toString()` and `valueOf()` override

```js
class Money {
  constructor(amount, currency) {
    this.amount   = amount;
    this.currency = currency;
  }
  toString()  { return `${this.amount} ${this.currency}`; }
  valueOf()   { return this.amount; }  // used in arithmetic
}

const price = new Money(29.99, "USD");
console.log(`Price: ${price}`);    // "Price: 29.99 USD"  — toString
console.log(price + 10);          // 39.99  — valueOf
console.log(price > 20);          // true   — valueOf
```

---

## Common mistakes

### Mistake 1 — Forgetting that methods are NOT in the constructor

```js
// WRONG — method is re-created per instance (memory waste)
class User {
  constructor(name) {
    this.name = name;
    this.greet = function () { return `Hi, ${this.name}`; };  // ← bad
  }
}

// RIGHT — define outside constructor; JS puts it on prototype automatically
class User {
  constructor(name) {
    this.name = name;
  }
  greet() { return `Hi, ${this.name}`; }  // ← goes on User.prototype
}
```

### Mistake 2 — Mutating class fields across instances

```js
// WRONG — using a reference type as a class field shared value
class Group {
  members = [];   // this IS safe as a class field — each instance gets its own
}

// This is actually fine:
const g1 = new Group();
const g2 = new Group();
g1.members.push("Alice");
console.log(g2.members);  // []  — each instance has its own array ✓

// But this IS the problem — on the prototype directly:
function OldGroup() {}
OldGroup.prototype.members = [];   // WRONG — shared across all instances
```

### Mistake 3 — Using `this` inside a static method

```js
class Config {
  constructor(env) { this.env = env; }

  static getDefault() {
    return this.env;   // WRONG — 'this' inside static = the class, not an instance
    // 'this' here is Config, which has no .env property
  }

  static getDefault() {
    return "production";  // RIGHT — static methods don't have instance context
  }
}
```

---

## Practice exercises

### Exercise 1 — easy

Create a class `Rectangle` with properties `width` and `height`. Add methods:
- `area()` → returns `width * height`
- `perimeter()` → returns `2 * (width + height)`
- `isSquare()` → returns `true` if `width === height`
- `describe()` → returns `"Rectangle: [w]x[h], area=[area]"`

Create three rectangles (including one square) and call all methods on each.

```js
// Write your code here
```

### Exercise 2 — medium

Create a class `TodoList` with:
- An instance property `tasks` starting as an empty array
- A `title` set via the constructor
- `addTask(text)` — pushes `{ id: (auto-increment), text, done: false }` onto `tasks`
- `completeTask(id)` — sets matching task's `done` to `true`; throws if not found
- `deleteTask(id)` — removes the task; throws if not found
- `getPending()` — returns only tasks where `done === false`
- `getSummary()` — returns `"[title]: X pending, Y done"`

Add a **static** method `merge(listA, listB, newTitle)` that creates a new `TodoList` with all tasks from both lists combined.

```js
// Write your code here
```

### Exercise 3 — hard

Build a class `EventEmitter` from scratch. It should work like this:

```js
const emitter = new EventEmitter();

emitter.on("login", (user) => console.log(`${user} logged in`));
emitter.on("login", (user) => console.log(`Audit: login by ${user}`));
emitter.on("logout", (user) => console.log(`${user} logged out`));

emitter.emit("login", "Alice");
// Alice logged in
// Audit: login by Alice

emitter.off("login", secondHandler);   // removes one specific handler

emitter.once("payment", (amount) => console.log(`Payment: $${amount}`));
emitter.emit("payment", 99.99);   // fires
emitter.emit("payment", 50.00);   // does NOT fire — once = one time only

emitter.emit("logout", "Alice");  // Alice logged out
```

Requirements:
- `on(event, handler)` — register a handler; multiple handlers per event allowed
- `off(event, handler)` — remove one specific handler
- `emit(event, ...args)` — call all registered handlers for that event, passing args
- `once(event, handler)` — register a handler that auto-removes itself after firing once
- `listenerCount(event)` — returns how many handlers are registered for that event

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Syntax |
|---|---|
| Define class | `class Person { constructor(name) { this.name = name; } }` |
| Instantiate | `const p = new Person("Alice")` |
| Prototype method | Defined in class body (outside constructor): `greet() { ... }` |
| Getter | `get fullName() { return ... }` — accessed as `p.fullName` |
| Setter | `set fullName(v) { ... }` — triggered by `p.fullName = "..."` |
| Static method | `static create(...) { return new Person(...) }` — called as `Person.create()` |
| Class field | `score = 0;` in class body — own property on each instance |
| Private field | `#secret = "x";` — topic 51 |
| Method chaining | Return `this` from methods |
| Class expression | `const Foo = class { ... }` |
| Hoisting | Classes are NOT hoisted (TDZ applies) |
| Strict mode | Always active inside class body |
| Method enumerable? | `false` — won't appear in `for...in` |
| Call without `new` | Throws `TypeError` — safer than constructor functions |
| Under the hood | `typeof Person === "function"` — it's a constructor function |
| Inheritance | `class Dog extends Animal { ... }` — topic 48 |

---

## Connected topics

- **46 — Prototypes and prototype chain** — Everything in this doc runs on prototype lookup. Understanding the chain explains *why* class methods are efficient.
- **48 — Inheritance** — `extends` and `super` are the natural next step. That topic builds directly on this one.
- **50 — Getters and setters** — Covered briefly here; topic 50 goes into every edge case, including getters in object literals and computed getter names.
