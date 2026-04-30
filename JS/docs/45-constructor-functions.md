# 45 — Constructor Functions

## What is this?

A constructor function is a regular JavaScript function used as a **blueprint for creating multiple objects of the same shape**. Think of it like a cookie cutter — you define the shape once, and every time you call it with `new`, you stamp out a fresh object with the same structure but its own individual data. Before ES6 classes existed, constructor functions were *the* standard way to create reusable object templates in JavaScript.

## Why does it matter?

When you need to create 10, 100, or 1000 objects that all share the same structure (e.g., users, products, orders), repeating object literals is impractical. Constructor functions let you define the structure once and stamp out instances on demand — and understanding them is *required* to understand how ES6 classes actually work under the hood, because classes are just syntactic sugar over constructor functions.

---

## Syntax

```js
// Define the constructor (PascalCase by convention)
function Person(name, age, role) {   // parameters become the unique data per instance
  this.name = name;                  // 'this' refers to the new object being created
  this.age = age;
  this.role = role;
  this.greet = function () {         // a method attached to each instance
    return `Hi, I'm ${this.name}`;
  };
}

// Create instances using the 'new' keyword
const alice = new Person("Alice", 28, "Engineer");
const bob   = new Person("Bob",   34, "Designer");
```

---

## How it works — line by line

```js
function Person(name, age, role) {
```
A plain function. Nothing special about its definition — the magic is entirely in how it's *called*.

```js
  this.name = name;
```
Inside a constructor called with `new`, `this` refers to the **brand-new empty object** that JS creates automatically. You're attaching properties to it.

```js
}
```
The constructor has **no `return` statement** — JS automatically returns `this` (the new object) for you.

```js
const alice = new Person("Alice", 28, "Engineer");
```
The `new` keyword triggers a 4-step process (see below). `alice` now holds the freshly built object.

---

## The 4 things `new` does behind the scenes

When you write `new Person("Alice", 28, "Engineer")`, JavaScript silently does **four things**:

```
Step 1 — Creates a new empty object:      {}
Step 2 — Sets its prototype:              {}.__proto__ = Person.prototype
Step 3 — Runs the function body:          'this' = that new empty object
Step 4 — Returns the object:              return this  (implicit)
```

You never write any of this — `new` handles it all.

---

## Example 1 — basic

```js
function Car(make, model, year) {
  this.make  = make;   // manufacturer name
  this.model = model;  // car model
  this.year  = year;   // production year
  this.isRunning = false; // default state for every new car

  this.start = function () {
    this.isRunning = true;
    return `${this.make} ${this.model} started.`;
  };

  this.stop = function () {
    this.isRunning = false;
    return `${this.make} ${this.model} stopped.`;
  };
}

const myCar     = new Car("Toyota", "Camry", 2022);
const friendCar = new Car("Honda", "Civic", 2020);

console.log(myCar.start());          // "Toyota Camry started."
console.log(friendCar.isRunning);    // false — completely separate object
console.log(myCar.isRunning);        // true  — myCar's state unaffected by friendCar

// Each instance is independent:
console.log(myCar === friendCar);    // false
```

---

## Example 2 — real world

A shopping cart system where each cart belongs to a different user:

```js
function ShoppingCart(owner) {
  this.owner = owner;  // who owns this cart
  this.items = [];     // starts empty for every new cart

  // Add an item to this cart
  this.addItem = function (productName, price, quantity) {
    this.items.push({ productName, price, quantity });
  };

  // Remove an item by name
  this.removeItem = function (productName) {
    this.items = this.items.filter(item => item.productName !== productName);
  };

  // Calculate total price
  this.getTotal = function () {
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  };

  // Get a summary string
  this.getSummary = function () {
    if (this.items.length === 0) return `${this.owner}'s cart is empty.`;
    const list = this.items.map(i => `${i.productName} x${i.quantity}`).join(", ");
    return `${this.owner}'s cart: ${list} | Total: $${this.getTotal().toFixed(2)}`;
  };
}

const aliceCart = new ShoppingCart("Alice");
const bobCart   = new ShoppingCart("Bob");

aliceCart.addItem("Laptop",  999.99, 1);
aliceCart.addItem("Mouse",    29.99, 2);

bobCart.addItem("Keyboard", 79.99, 1);

console.log(aliceCart.getSummary());
// Alice's cart: Laptop x1, Mouse x2 | Total: $1059.97

console.log(bobCart.getSummary());
// Bob's cart: Keyboard x1 | Total: $79.99

aliceCart.removeItem("Mouse");
console.log(aliceCart.getSummary());
// Alice's cart: Laptop x1 | Total: $999.99

// Crucially — Bob's cart is completely unaffected by Alice's changes
console.log(bobCart.items.length);  // 1
```

---

## The prototype problem — and why it matters

There is a **performance issue** with the method-in-constructor pattern above. Every time you call `new Car(...)`, JavaScript creates **brand new copies** of every method for each instance:

```js
const car1 = new Car("Toyota", "Camry", 2022);
const car2 = new Car("Honda",  "Civic", 2020);

// Each instance has its OWN copy of 'start'
console.log(car1.start === car2.start);  // false — different function objects in memory!
```

With 1000 cars, you have 1000 separate `start` functions in memory doing the exact same thing. The fix is to put shared methods on the **prototype** instead:

```js
function Car(make, model, year) {
  this.make  = make;   // unique data stays in the constructor
  this.model = model;
  this.year  = year;
  this.isRunning = false;
  // NO methods here
}

// Shared methods go on the prototype — created ONCE, shared by ALL instances
Car.prototype.start = function () {
  this.isRunning = true;
  return `${this.make} ${this.model} started.`;
};

Car.prototype.stop = function () {
  this.isRunning = false;
  return `${this.make} ${this.model} stopped.`;
};

const car1 = new Car("Toyota", "Camry", 2022);
const car2 = new Car("Honda",  "Civic", 2020);

console.log(car1.start === car2.start);  // true — same function, shared via prototype
```

This is exactly the pattern ES6 classes use internally — class methods are automatically placed on the prototype.

---

## Tricky things

### 1. Forgetting `new` — silent disaster

```js
function Person(name) {
  this.name = name;
}

const user = Person("Alice");  // forgot 'new'!

console.log(user);        // undefined  — constructor returns nothing without 'new'
console.log(window.name); // "Alice"    — 'this' was the global object! Property leaked.
```

Without `new`, `this` is `window` (in browsers) or `global` (in Node.js), not a new object. This silently pollutes the global scope — one of the most dangerous bugs in older JS code.

**Safety guard** — detect accidental non-`new` calls:

```js
function Person(name) {
  if (!(this instanceof Person)) {
    throw new Error("Person must be called with 'new'");
  }
  this.name = name;
}
```

### 2. `return` with a primitive vs an object

```js
function WeirdA(name) {
  this.name = name;
  return 42;  // returning a primitive — ignored by 'new'
}
const a = new WeirdA("Alice");
console.log(a.name);  // "Alice" — the primitive return is ignored

function WeirdB(name) {
  this.name = name;
  return { overridden: true };  // returning an OBJECT — replaces 'this'!
}
const b = new WeirdB("Alice");
console.log(b.name);       // undefined
console.log(b.overridden); // true — the returned object won the battle
```

Rule: returning a **primitive** from a constructor → ignored. Returning an **object** → that object is used instead of `this`.

### 3. Arrow functions cannot be constructors

```js
const Person = (name) => {
  this.name = name;
};

const p = new Person("Alice");  // TypeError: Person is not a constructor
```

Arrow functions have no `this` binding of their own and no `prototype` property, so `new` simply doesn't work on them.

### 4. Checking if something is an instance

```js
function Dog(name) { this.name = name; }

const myDog = new Dog("Rex");

console.log(myDog instanceof Dog);    // true
console.log(myDog instanceof Object); // true — everything is an Object
console.log(myDog.constructor === Dog); // true — also works
```

### 5. PascalCase naming convention

Constructor functions should always be written in PascalCase (`Person`, `ShoppingCart`, `DatabaseConnection`). This is not enforced by JS — it's a social contract that tells other developers "call this with `new`". If you see a PascalCase function, use `new`. If you see camelCase, don't.

---

## Common mistakes

### Mistake 1 — Putting methods directly inside (memory waste)

```js
// WRONG — creates a new copy of greet for every single instance
function User(name) {
  this.name = name;
  this.greet = function () {
    return `Hello, ${this.name}`;  // new function object every time new User() is called
  };
}

// RIGHT — one shared greet function for all User instances
function User(name) {
  this.name = name;
}
User.prototype.greet = function () {
  return `Hello, ${this.name}`;
};
```

### Mistake 2 — Forgetting `new`

```js
// WRONG
const user = User("Alice");  // this = global; user = undefined

// RIGHT
const user = new User("Alice");  // this = fresh object; user = { name: "Alice" }
```

### Mistake 3 — Mutating shared arrays/objects on `this`

```js
// WRONG — all instances share the SAME items array!
function Cart() {}
Cart.prototype.items = [];  // ← Don't put mutable state on the prototype

const c1 = new Cart();
const c2 = new Cart();
c1.items.push("apple");
console.log(c2.items);  // ["apple"]  OOPS — c2 was contaminated!

// RIGHT — mutable state must be initialised in the constructor
function Cart() {
  this.items = [];  // each instance gets its own fresh array
}
```

---

## Practice exercises

### Exercise 1 — easy

Create a constructor function called `Book` that takes `title`, `author`, and `pages` as parameters. Each instance should also have a `isRead` property that defaults to `false`. Add a method `markAsRead()` that sets `isRead` to `true` and returns the string `"[title] marked as read."`. Create two different Book instances and call `markAsRead()` on one — verify the other is unaffected.

```js
// Write your code here
```

### Exercise 2 — medium

Create a constructor function called `BankAccount` that takes `ownerName` and `initialBalance`. Add the following methods **on the prototype** (not inside the constructor):
- `deposit(amount)` — adds to the balance, returns new balance
- `withdraw(amount)` — subtracts from balance only if sufficient funds exist; if not, return `"Insufficient funds"`, otherwise return the new balance
- `getStatement()` — returns a string like `"Alice | Balance: $250.00"`

Create two separate accounts and run a sequence of deposits and withdrawals to prove they're fully independent.

```js
// Write your code here
```

### Exercise 3 — hard

Build a `TaskManager` constructor. Each instance manages its own list of tasks. Each task is an object with `{ id, title, priority, done }`. Add prototype methods:
- `addTask(title, priority)` — auto-generates a numeric `id` (1, 2, 3...), sets `done: false`
- `completeTask(id)` — marks the matching task `done: true`; throws an error if id doesn't exist
- `getPending()` — returns array of tasks where `done === false`, sorted by priority (`"high"` first, then `"medium"`, then `"low"`)
- `getSummary()` — returns `"X of Y tasks complete"`

Test it with at least 5 tasks of mixed priorities, complete some, then call `getPending()` and `getSummary()`.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Concept | Detail |
|---|---|
| Define | `function Person(name) { this.name = name; }` — plain function, PascalCase |
| Instantiate | `const p = new Person("Alice")` — always use `new` |
| What `new` does | 1) Create `{}` 2) Set prototype 3) Run body with `this = {}` 4) Return `this` |
| Instance data | Goes inside constructor: `this.property = value` |
| Shared methods | Go on prototype: `Person.prototype.greet = function() {}` |
| Check type | `p instanceof Person` → `true` |
| Forgot `new`? | `this` = global object, function returns `undefined`, data leaks |
| Arrow function? | Cannot be used as constructor — no `this`, no `prototype` |
| Return primitive | Ignored — `new` still returns `this` |
| Return object | Replaces `this` — that object is returned instead |
| Naming convention | PascalCase signals "use `new` with me" |
| Modern alternative | ES6 class syntax (topic 47) — same thing, cleaner syntax |

---

## Connected topics

- **46 — Prototypes and the prototype chain** — The `Person.prototype` property mentioned here is the heart of how all JavaScript inheritance works. You need this next.
- **47 — ES6 Classes** — Classes are literally constructor functions + prototype methods written with cleaner syntax. Once you understand this topic, classes will feel obvious.
- **18 — Scope** — Understanding why `this` inside a constructor refers to the new object (vs why it doesn't in arrow functions) requires knowing how `this` binds in different call contexts.
