# 35 — Object Methods & `this`

---

## What is this?

An **object method** is simply a function stored as a property of an object. Think of an object as an employee profile — it has data (name, salary) but also *behaviours* (calculateBonus, promote). Those behaviours are methods.

The keyword **`this`** inside a method refers to the object the method was called on — it's how the function knows which object's data to use.

```js
const employee = {
  name:   'Alice',
  salary: 60000,
  greet() {                          // method
    return `Hi, I'm ${this.name}`; // 'this' = the employee object
  },
};

employee.greet(); // "Hi, I'm Alice"
```

---

## Why does it matter?

Methods let you attach behaviour directly to the data it belongs to — this is the foundation of Object-Oriented Programming. Understanding `this` is critical because it controls what data a method can access, and it behaves differently depending on *how* the function is called — one of the most common sources of bugs in JavaScript.

---

## Syntax

```js
const obj = {
  // Method shorthand (ES6 — preferred)
  methodName() {
    return this.property;
  },

  // Old style (function expression as value) — same behaviour
  methodName2: function() {
    return this.property;
  },

  // Arrow function — DO NOT use for methods (this won't work correctly)
  methodName3: () => {
    return this.property; // ❌ 'this' is NOT the object here
  },
};
```

---

## How it works — line by line

```js
const counter = {
  count: 0,              // data property

  increment() {          // method — shorthand syntax
    this.count += 1;     // 'this' refers to 'counter' (the object calling it)
    return this.count;   // returns the updated count
  },

  reset() {
    this.count = 0;      // 'this.count' → counter.count
  },
};

counter.increment(); // 1
counter.increment(); // 2
counter.increment(); // 3
counter.reset();
console.log(counter.count); // 0
```

When you write `counter.increment()`, JS sets `this` = `counter` automatically before running the function body.

---

## What `this` refers to — the core rules

`this` is determined by **how a function is called**, not where it is defined (except for arrow functions).

### Rule 1 — Method call: `this` = the object before the dot

```js
const user = {
  name: 'Alice',
  getName() { return this.name; },
};

user.getName(); // 'Alice' — this = user
```

### Rule 2 — Regular function call: `this` = `undefined` (strict mode) or global object

```js
function showThis() {
  console.log(this);
}

showThis();
// In browser (non-strict): Window object
// In Node.js (non-strict): global object
// In strict mode ('use strict'): undefined
```

### Rule 3 — Arrow function: `this` is inherited from the surrounding scope (lexical `this`)

```js
const timer = {
  seconds: 0,
  start() {
    // Regular function as callback — 'this' is lost
    setInterval(function() {
      this.seconds += 1; // ❌ 'this' is undefined/global here, not 'timer'
    }, 1000);

    // Arrow function as callback — inherits 'this' from start()
    setInterval(() => {
      this.seconds += 1; // ✅ 'this' = timer — correct!
    }, 1000);
  },
};
```

### Rule 4 — Explicit binding: `call`, `apply`, `bind`

You can manually set what `this` is (covered below).

---

## Example 1 — Object with multiple methods sharing data

```js
const bankAccount = {
  owner:   'Parsh',
  balance: 10000,

  deposit(amount) {
    this.balance += amount;
    console.log(`Deposited ₹${amount}. New balance: ₹${this.balance}`);
  },

  withdraw(amount) {
    if (amount > this.balance) {
      console.log('Insufficient funds.');
      return;
    }
    this.balance -= amount;
    console.log(`Withdrew ₹${amount}. New balance: ₹${this.balance}`);
  },

  getStatement() {
    return `Account owner: ${this.owner} | Balance: ₹${this.balance}`;
  },
};

bankAccount.deposit(5000);    // Deposited ₹5000. New balance: ₹15000
bankAccount.withdraw(3000);   // Withdrew ₹3000. New balance: ₹12000
bankAccount.withdraw(20000);  // Insufficient funds.
console.log(bankAccount.getStatement());
// Account owner: Parsh | Balance: ₹12000
```

---

## Example 2 — Method chaining (returning `this`)

Returning `this` from a method lets you chain multiple calls:

```js
const queryBuilder = {
  table:      '',
  conditions: [],
  limitVal:   null,

  from(tableName) {
    this.table = tableName;
    return this;                     // return this to allow chaining
  },

  where(condition) {
    this.conditions.push(condition);
    return this;
  },

  limit(n) {
    this.limitVal = n;
    return this;
  },

  build() {
    let query = `SELECT * FROM ${this.table}`;
    if (this.conditions.length > 0) {
      query += ` WHERE ${this.conditions.join(' AND ')}`;
    }
    if (this.limitVal) {
      query += ` LIMIT ${this.limitVal}`;
    }
    return query;
  },
};

const sql = queryBuilder
  .from('users')
  .where('age > 18')
  .where('active = true')
  .limit(10)
  .build();

console.log(sql);
// SELECT * FROM users WHERE age > 18 AND active = true LIMIT 10
```

---

## Example 3 — `call`, `apply`, `bind` — explicit `this` control

Sometimes you want to borrow a method or set `this` manually.

### `call(thisArg, arg1, arg2, ...)`

Calls the function immediately with a specific `this` and individual arguments:

```js
const user1 = { name: 'Alice', score: 95 };
const user2 = { name: 'Bob',   score: 82 };

function describeUser(prefix, suffix) {
  return `${prefix} ${this.name} scored ${this.score}${suffix}`;
}

describeUser.call(user1, 'Player', '!'); // "Player Alice scored 95!"
describeUser.call(user2, 'Player', '!'); // "Player Bob scored 82!"
```

### `apply(thisArg, [argsArray])`

Same as `call` but arguments passed as an array:

```js
describeUser.apply(user1, ['Player', '!']); // "Player Alice scored 95!"

// Classic use: spreading an array into Math.max before spread operator existed
const scores = [88, 95, 72, 100, 61];
Math.max.apply(null, scores); // 100 (now use Math.max(...scores) instead)
```

### `bind(thisArg, arg1, ...)`

Returns a **new function** with `this` permanently bound — does NOT call immediately:

```js
const user = { name: 'Alice', score: 95 };

function describe() {
  return `${this.name}: ${this.score}`;
}

const describeAlice = describe.bind(user); // new function, this = user
describeAlice(); // "Alice: 95"
describeAlice(); // "Alice: 95" — always uses user as this

// Useful for event handlers and callbacks:
const timer = {
  name:  'Countdown',
  start() {
    // Without bind, 'this' inside the callback would be lost
    setTimeout(function() {
      console.log(`${this.name} finished`); // ❌ 'this' wrong without bind
    }.bind(this), 1000); // ✅ bind fixes it
  },
};
```

---

## Example 4 — The `this` problem: losing context

The most common `this` bug — pulling a method out of its object:

```js
const user = {
  name: 'Alice',
  greet() {
    return `Hello, ${this.name}`;
  },
};

user.greet(); // "Hello, Alice" ✅

// Extract method into a variable — LOSES context
const greetFn = user.greet;
greetFn(); // "Hello, undefined" ❌ — 'this' is no longer 'user'

// Fix 1: bind
const greetBound = user.greet.bind(user);
greetBound(); // "Hello, Alice" ✅

// Fix 2: arrow wrapper
const greetArrow = () => user.greet();
greetArrow(); // "Hello, Alice" ✅
```

This same problem happens with event listeners and callbacks:

```js
const button = {
  label: 'Submit',
  handleClick() {
    console.log(`Clicked: ${this.label}`);
  },
};

// ❌ Passes the function — 'this' inside will be the button element, not our object
document.querySelector('button').addEventListener('click', button.handleClick);

// ✅ Use bind to lock in 'this'
document.querySelector('button').addEventListener('click', button.handleClick.bind(button));

// ✅ Or use an arrow wrapper
document.querySelector('button').addEventListener('click', () => button.handleClick());
```

---

## Example 5 — Arrow functions inside methods (when to use them)

Arrow functions don't have their own `this` — they inherit it from the enclosing scope. This is a **feature**, not a bug, when used in the right place:

```js
const taskManager = {
  tasks: ['design', 'code', 'test'],

  // ✅ Arrow function in a method's inner callback — inherits 'this' from printAll
  printAll() {
    this.tasks.forEach(task => {             // arrow function here
      console.log(`${this.tasks.length} tasks | Current: ${task}`);
      // 'this' correctly refers to taskManager
    });
  },

  // ❌ Arrow function AS the method — 'this' doesn't refer to the object
  badMethod: () => {
    console.log(this.tasks); // undefined — 'this' is the outer scope (module/global)
  },
};

taskManager.printAll();
// 3 tasks | Current: design
// 3 tasks | Current: code
// 3 tasks | Current: test

taskManager.badMethod(); // undefined
```

**Rule**: Arrow functions are great *inside* methods (as callbacks). Never use them *as* methods.

---

## Tricky things you'll encounter in the real world

### 1. `this` in nested regular functions

```js
const obj = {
  value: 42,

  outer() {
    console.log(this.value); // 42 ✅ — this = obj

    function inner() {
      console.log(this.value); // undefined ❌ — regular function, this = global/undefined
    }
    inner();

    const innerArrow = () => {
      console.log(this.value); // 42 ✅ — arrow inherits this from outer()
    };
    innerArrow();
  },
};

obj.outer();
```

---

### 2. `this` in class vs object literal

```js
// In object literals, methods share 'this' pointing to the object
const obj = {
  x: 10,
  getX() { return this.x; },
};

// In classes (topic 47), 'this' works the same way for methods
// but the binding rules are identical
```

---

### 3. Chained method calls on different objects

```js
const a = {
  value: 1,
  getVal() { return this.value; },
};

const b = {
  value: 2,
  getVal: a.getVal,  // borrowing a's method
};

a.getVal(); // 1 — this = a
b.getVal(); // 2 — this = b (the object before the dot at call time)
```

---

### 4. `Object.assign()` — another way to copy/add properties

```js
const target = { a: 1 };
const source = { b: 2, c: 3 };

Object.assign(target, source); // mutates target
console.log(target); // { a: 1, b: 2, c: 3 }

// Copy without mutating (spread is cleaner for this):
const copy = Object.assign({}, source);
```

---

### 5. Getter and setter methods (preview of topic 50)

```js
const circle = {
  radius: 5,

  // 'get' makes this behave like a property, not a function call
  get area() {
    return Math.PI * this.radius ** 2;
  },

  get circumference() {
    return 2 * Math.PI * this.radius;
  },
};

circle.area;          // 78.53... — no () needed
circle.circumference; // 31.41...
circle.radius = 10;
circle.area;          // 314.15... — recomputed from new radius
```

---

## Common mistakes

### Mistake 1 — Arrow function as a method

```js
const user = { name: 'Alice' };

// ❌ Arrow function — 'this' is NOT the object
user.greet = () => {
  return `Hi, ${this.name}`; // 'this' is global/undefined
};
user.greet(); // "Hi, undefined"

// ✅ Regular function / shorthand
user.greet = function() {
  return `Hi, ${this.name}`;
};
user.greet(); // "Hi, Alice"
```

---

### Mistake 2 — Losing `this` when passing a method as a callback

```js
const validator = {
  minLength: 8,
  check(str) {
    return str.length >= this.minLength;
  },
};

const passwords = ['abc', 'securepass', 'hi'];

// ❌ 'this' inside check will be undefined
const valid = passwords.filter(validator.check);

// ✅ Bind it
const valid2 = passwords.filter(validator.check.bind(validator));
// or use arrow wrapper:
const valid3 = passwords.filter(str => validator.check(str));

console.log(valid2); // ['securepass']
```

---

### Mistake 3 — Calling a method without `this` to access own data

```js
const cart = {
  items:     ['laptop', 'mouse'],
  discount:  0.1,

  getTotal() {
    // ❌ Using bare variable name — ReferenceError
    return items.length * 1000;
  },

  // ✅ Use this to access the object's own property
  getTotalFixed() {
    return this.items.length * 1000 * (1 - this.discount);
  },
};

cart.getTotalFixed(); // 1800
```

---

## Frequently asked questions

**Q: What's the difference between a method and a regular function?**
A method is just a function stored as a property of an object. The only difference is that when called as `obj.method()`, `this` is automatically set to `obj`.

**Q: Why does `this` work differently in arrow functions?**
Arrow functions don't have their own `this` binding. When you use `this` inside an arrow function, JS looks outward to the nearest enclosing regular function or the global scope. This is called "lexical `this`".

**Q: When should I use `bind` vs an arrow wrapper?**
- `bind` when you need to pass the bound function around and reuse it many times
- Arrow wrapper (`() => obj.method()`) for quick one-off usage like event listeners
- Both are correct — prefer arrow wrapper for readability in most cases

**Q: What is `this` in the global scope (outside any function)?**
- In browsers: `this === window`
- In Node.js modules: `this === module.exports` (an empty object `{}`)
- In Node.js scripts: `this === global`

**Q: Does `this` work differently in strict mode?**
Yes — in strict mode (`'use strict'`), `this` in a regular function call is `undefined` instead of the global object. ES6 modules are always in strict mode.

**Q: Can I use `call`/`apply`/`bind` on arrow functions?**
No — arrow functions ignore `call`, `apply`, and `bind`. Their `this` is fixed at creation time and cannot be changed.

---

## Practice exercises

### Exercise 1 — Easy: Product object with methods

Create an object `product` with:
- `name` — a product name string
- `price` — a number
- `quantity` — a number

Add three methods:
- `getTotalValue()` — returns `price × quantity`
- `applyDiscount(percent)` — reduces `price` by that percent (e.g. 10 → 10% off), updates `price` in place
- `describe()` — returns a string like: `"Laptop — ₹45000 (qty: 3)"`

Test all three methods:
```js
// Write your code here
```

---

### Exercise 2 — Medium: Shopping cart with method chaining

Build a `cart` object that starts with an empty `items` array and a `discountPercent` of 0. Add these methods — **each must return `this`** for chaining:

- `addItem(name, price, qty)` — pushes `{ name, price, qty }` into `items`
- `removeItem(name)` — removes item with matching name from `items`
- `setDiscount(percent)` — sets `discountPercent`
- `checkout()` — calculates total (sum of `price × qty` for all items), applies discount, logs a receipt summary, and returns `this`

It should work like this:
```js
cart
  .addItem('Keyboard', 1200, 1)
  .addItem('Mouse', 800, 2)
  .addItem('Monitor', 12000, 1)
  .removeItem('Mouse')
  .setDiscount(10)
  .checkout();
// Receipt should show items, subtotal, discount, and final total
```

```js
// Write your code here
```

---

### Exercise 3 — Hard: `this` binding in a timer object

Build a `stopwatch` object with:
- `name` — a string label (e.g. `'Workout Timer'`)
- `elapsed` — starts at `0`
- `intervalId` — starts as `null`

Methods:
- `start()` — uses `setInterval` to increment `elapsed` every 1 second. Must use an arrow function or `bind` so `this.elapsed` updates correctly. Stores the interval id in `this.intervalId`.
- `stop()` — clears the interval using `this.intervalId`
- `reset()` — stops and sets `elapsed` back to `0`
- `status()` — returns `"[name] — Xsec elapsed"` using `this.name` and `this.elapsed`

Then write a `borrowedStatus` — extract the `status` method into a standalone function and show that calling it plain loses `this`. Fix it with `bind`.

```js
// Write your code here
```

*(You can test start/stop with `setTimeout` to auto-stop after a few seconds.)*

---

## Quick reference — cheat sheet

```js
// Define a method (shorthand — preferred)
const obj = {
  method() { return this.prop; }
};

// Call
obj.method(); // this = obj

// Arrow function — lexical this (use INSIDE methods, not AS methods)
const obj2 = {
  items: [1, 2, 3],
  printAll() {
    this.items.forEach(item => console.log(this, item)); // this = obj2 ✅
  },
  bad: () => { console.log(this); }, // this = outer scope ❌
};

// Explicit this
fn.call(thisArg, arg1, arg2);    // call immediately, args separate
fn.apply(thisArg, [arg1, arg2]); // call immediately, args as array
fn.bind(thisArg);                // return new function, this locked in

// Method chaining — return this
const chain = {
  value: 0,
  add(n) { this.value += n; return this; },
  log()  { console.log(this.value); return this; },
};
chain.add(5).add(3).log(); // 8
```

| Rule | `this` value |
|---|---|
| `obj.method()` | `obj` |
| `fn()` (strict mode) | `undefined` |
| `fn()` (non-strict) | global object |
| Arrow function | inherited from outer scope |
| `fn.call(x, ...)` | `x` |
| `fn.apply(x, [...])` | `x` |
| `fn.bind(x)()` | `x` |

---

## Connected topics

- **[34] Object Basics** — properties and data that methods operate on
- **[36] Object Destructuring** — extracting properties cleanly (be careful with `this`)
- **[47] ES6 Classes** — `this` works the same way inside class methods
- **[19] Closures** — how arrow functions capture `this` from their outer scope
- **[50] Getters & Setters** — special methods using `get` and `set` keywords
