# 15 — Arrow Functions

## What is this?

An **arrow function** is a shorter syntax for writing function expressions, introduced in ES6.

```js
// Regular function expression
const double = function(number) {
  return number * 2;
};

// Arrow function — same thing
const double = (number) => {
  return number * 2;
};

// Arrow function — even shorter (implicit return)
const double = number => number * 2;
```

All three do exactly the same thing. Arrow functions are now the dominant style in modern JavaScript — you will see them everywhere.

---

## Why does it matter?

Arrow functions aren't just shorter syntax. They have a fundamentally different behaviour with `this` compared to regular functions. That difference matters enormously once you work with objects, classes, and callbacks.

The two reasons arrow functions exist:
1. **Conciseness** — less ceremony for small utility functions and callbacks
2. **Lexical `this`** — they don't have their own `this`; they inherit it from the surrounding code (explained deeply below)

If you only learn the shorthand and ignore `this`, you will hit subtle bugs. This doc covers both.

---

## Syntax

### Full arrow function (with body)

```js
const functionName = (param1, param2) => {
  // body
  return value;
};
```

### Implicit return (single expression — no braces, no `return`)

```js
const functionName = (param1, param2) => expression;
```

The expression's value is automatically returned. No `return` keyword needed.

### Single parameter — parentheses are optional

```js
const double = number => number * 2;
const double = (number) => number * 2;  // same thing
```

### No parameters — parentheses required

```js
const getRandom = () => Math.random();
```

### Returning an object literal — must wrap in parentheses

```js
// ❌ JS thinks the {} is a function body, not an object
const makeUser = name => { name: name, active: true };  // SyntaxError

// ✅ Wrap the object in ()
const makeUser = name => ({ name: name, active: true });
```

---

## How it works — line by line

```js
const applyDiscount = (price, discountPercent) => {
  const discountAmount = price * (discountPercent / 100);
  return price - discountAmount;
};

const finalPrice = applyDiscount(80, 25);
console.log(finalPrice);  // 60
```

**Execution trace:**
```
1. Arrow function stored in const applyDiscount
2. applyDiscount(80, 25) called
3. price = 80, discountPercent = 25
4. discountAmount = 80 * (25 / 100) = 20
5. return 80 - 20 = 60
6. finalPrice = 60
```

Exactly the same as a regular function. The difference only appears when `this` is involved.

---

## Example 1 — basic

### Transforming data with implicit return

```js
const productNames = ["wireless mouse", "usb hub", "mechanical keyboard"];

// Old way
const capitalized = productNames.map(function(name) {
  return name.toUpperCase();
});

// Arrow function — cleaner
const capitalized = productNames.map(name => name.toUpperCase());

console.log(capitalized);
// ["WIRELESS MOUSE", "USB HUB", "MECHANICAL KEYBOARD"]
```

Arrow functions shine inside array methods like `.map()`, `.filter()`, `.reduce()` — the callback becomes a single readable line.

### Filtering and sorting

```js
const products = [
  { name: "Laptop",    price: 999, inStock: true  },
  { name: "Monitor",   price: 349, inStock: false },
  { name: "Keyboard",  price: 89,  inStock: true  },
  { name: "Webcam",    price: 129, inStock: true  },
  { name: "Headset",   price: 199, inStock: false },
];

const availableProducts = products
  .filter(product => product.inStock)
  .sort((a, b) => a.price - b.price);

console.log(availableProducts);
// [
//   { name: "Keyboard",  price: 89,  inStock: true },
//   { name: "Webcam",    price: 129, inStock: true },
//   { name: "Laptop",    price: 999, inStock: true },
// ]
```

### Utility arrow functions

```js
const clamp = (value, min, max) => Math.min(Math.max(value, min), max);
const isEven = number => number % 2 === 0;
const toPercent = (value, total) => ((value / total) * 100).toFixed(1) + "%";
const capitalize = str => str.charAt(0).toUpperCase() + str.slice(1);

console.log(clamp(150, 0, 100));         // 100
console.log(isEven(7));                  // false
console.log(toPercent(27, 60));          // "45.0%"
console.log(capitalize("javascript"));  // "Javascript"
```

Short, pure utility functions are where arrow functions are at their absolute best.

---

## Example 2 — real world

### Processing API response data

```js
const apiResponse = {
  status: "success",
  users: [
    { id: 1, fullName: "Elena Vasquez", email: "elena@co.com", role: "admin",  isActive: true  },
    { id: 2, fullName: "James Park",    email: "james@co.com", role: "viewer", isActive: false },
    { id: 3, fullName: "Nina Osei",     email: "nina@co.com",  role: "editor", isActive: true  },
    { id: 4, fullName: "Ravi Mehta",    email: "ravi@co.com",  role: "viewer", isActive: true  },
  ],
};

// Get only active users
const activeUsers = apiResponse.users.filter(user => user.isActive);

// Extract just the names
const activeUserNames = activeUsers.map(user => user.fullName);

// One-liner version using chaining
const adminEmails = apiResponse.users
  .filter(user => user.role === "admin" && user.isActive)
  .map(user => user.email);

console.log(activeUserNames);  // ["Elena Vasquez", "Nina Osei", "Ravi Mehta"]
console.log(adminEmails);      // ["elena@co.com"]
```

### Event handlers (DOM preview)

```js
const submitButton = document.querySelector("#submitBtn");

// Regular function — 'this' refers to the button element
submitButton.addEventListener("click", function() {
  console.log(this);  // <button id="submitBtn">
});

// Arrow function — 'this' is NOT the button; it's the surrounding scope
submitButton.addEventListener("click", () => {
  console.log(this);  // window (or undefined in strict mode)
});
```

This is why arrow functions are NOT always the right choice for event handlers when you need `this` to be the element. (Deep dive below.)

### Returning objects from arrow functions

```js
const createProductListing = (name, price, category) => ({
  id: Date.now(),
  name,
  price,
  category,
  createdAt: new Date().toISOString(),
  isAvailable: true,
});

const newProduct = createProductListing("Ergonomic Chair", 299, "Furniture");
console.log(newProduct);
// {
//   id: 1714305600000,
//   name: "Ergonomic Chair",
//   price: 299,
//   category: "Furniture",
//   createdAt: "2026-04-28T...",
//   isAvailable: true
// }
```

---

## Tricky things you'll encounter in the real world

### 1. Arrow functions have no `this` of their own — they inherit it

This is the most important thing to understand about arrow functions.

A regular function creates its own `this` binding depending on how it's called. An arrow function has **no `this`** — it uses the `this` from the scope where it was *defined*.

```js
const timer = {
  seconds: 0,

  startRegular: function() {
    setInterval(function() {
      this.seconds++;             // ❌ 'this' is NOT timer — it's window/undefined
      console.log(this.seconds);  // NaN or error
    }, 1000);
  },

  startArrow: function() {
    setInterval(() => {
      this.seconds++;             // ✅ 'this' IS timer — arrow inherited it
      console.log(this.seconds);  // 1, 2, 3...
    }, 1000);
  },
};
```

`setInterval` calls its callback as a plain function — so a regular function's `this` becomes `window`. The arrow function doesn't have its own `this`, so it looks outward and finds `timer`'s `this`. This is **lexical `this`** — it comes from the lexical (written) location of the function.

Before arrow functions, developers used `const self = this` or `.bind(this)` to work around this:

```js
startRegular: function() {
  const self = this;  // old workaround
  setInterval(function() {
    self.seconds++;  // works, but ugly
  }, 1000);
}
```

Arrow functions made this unnecessary.

### 2. Arrow functions cannot be used as constructors

```js
const Product = (name, price) => {
  this.name = name;
  this.price = price;
};

const item = new Product("Widget", 9.99);  // ❌ TypeError: Product is not a constructor
```

Arrow functions don't have a `prototype` property and can't be called with `new`. Use a regular function declaration or a `class` for constructors.

### 3. Arrow functions have no `arguments` object

```js
function regularSum() {
  console.log(arguments);  // Arguments [1, 2, 3]
  return Array.from(arguments).reduce((a, b) => a + b, 0);
}

const arrowSum = () => {
  console.log(arguments);  // ❌ ReferenceError: arguments is not defined
};
```

Arrow functions don't have the `arguments` object. Use **rest parameters** instead (covered in topic 16):

```js
const arrowSum = (...numbers) => numbers.reduce((a, b) => a + b, 0);
arrowSum(1, 2, 3, 4);  // 10
```

### 4. Arrow functions as object methods — `this` problem

```js
const shoppingCart = {
  items: ["laptop", "mouse"],
  discount: 10,

  // ❌ Arrow function as method — 'this' is NOT shoppingCart
  getDiscount: () => {
    return `Discount: ${this.discount}%`;  // this is window/undefined
  },

  // ✅ Regular function as method — 'this' IS shoppingCart
  getDiscount: function() {
    return `Discount: ${this.discount}%`;
  },
};

console.log(shoppingCart.getDiscount());
```

**Rule:** Use regular functions for object methods that need `this`. Use arrow functions for callbacks inside those methods.

```js
const shoppingCart = {
  items: ["laptop", "mouse", "keyboard"],

  listItems: function() {
    // 'this' is shoppingCart here ✅
    this.items.forEach(item => {
      // Arrow callback — 'this' is still shoppingCart ✅ (inherited from listItems)
      console.log(`${item} — Cart ID: ${this.items.length} items`);
    });
  },
};
```

### 5. Implicit return with multiline expressions — use a block

```js
// ✅ Simple — one expression, implicit return works
const double = n => n * 2;

// ❌ Multi-step logic — implicit return is not possible
const processOrder = order => 
  const tax = order.total * 0.08;  // SyntaxError
  order.total + tax;

// ✅ Use a block with explicit return
const processOrder = order => {
  const tax = order.total * 0.08;
  return order.total + tax;
};
```

Implicit return only works for a **single expression**. The moment you need a variable or multiple statements, you need `{}` and `return`.

### 6. Chaining arrow functions — returns another function

```js
const multiply = a => b => a * b;

const triple = multiply(3);   // returns b => 3 * b
console.log(triple(5));       // 15
console.log(multiply(4)(6));  // 24
```

This is **currying** — a function that returns another function. It's a real pattern in functional programming and libraries like Lodash and Ramda. Arrow syntax makes it very compact. Covered deeply in topic 42.

---

## Common mistakes

### Mistake 1: Forgetting `()` when returning an object literal

```js
// ❌ JS reads {} as the function body, not an object
const makeSession = userId => { id: userId, active: true };

// ✅ Wrap in parentheses
const makeSession = userId => ({ id: userId, active: true });
```

### Mistake 2: Using arrow function as an object method that needs `this`

```js
const counter = {
  count: 0,
  increment: () => { this.count++; },  // ❌ this is not counter
};

counter.increment();
console.log(counter.count);  // still 0 — silent bug
```

### Mistake 3: Expecting `arguments` to work in an arrow function

```js
const logAll = () => {
  console.log(arguments);  // ❌ ReferenceError
};

// ✅ Use rest params
const logAll = (...args) => {
  console.log(args);
};
```

### Mistake 4: Overusing arrow functions for everything

Arrow functions are not always better. Use regular functions when:
- The function is a method on an object (needs `this`)
- The function is a constructor (needs `new`)
- You want the function declaration to be hoisted
- Readability suffers — very long arrow functions are not clearer than `function`

---

## Practice exercises

### Exercise 1 — easy

Convert each of these regular functions to arrow functions. Some should use implicit return (no braces, no `return`), and some should keep a block body — you decide which makes the code cleaner.

```js
// 1.
function square(n) {
  return n * n;
}

// 2.
function isAdult(age) {
  return age >= 18;
}

// 3.
function formatFullName(firstName, lastName) {
  const fullName = firstName + " " + lastName;
  return fullName.trim();
}

// 4.
function getInitials(firstName, lastName) {
  return firstName[0].toUpperCase() + lastName[0].toUpperCase();
}

// 5.
function getRandom() {
  return Math.floor(Math.random() * 100);
}
```

After converting, call each one and log the result.

---

### Exercise 2 — medium

You have this array of customer orders:

```js
const orders = [
  { orderId: "O-001", customerName: "Fatima Al-Amin", total: 239.99, status: "delivered", itemCount: 3 },
  { orderId: "O-002", customerName: "Luke Bennett",   total: 89.50,  status: "pending",   itemCount: 1 },
  { orderId: "O-003", customerName: "Yuna Kim",       total: 412.00, status: "delivered", itemCount: 5 },
  { orderId: "O-004", customerName: "Marco Silva",    total: 55.00,  status: "cancelled", itemCount: 2 },
  { orderId: "O-005", customerName: "Amira Hassan",   total: 178.25, status: "delivered", itemCount: 4 },
];
```

Using **only arrow functions** for all callbacks:

1. Filter to only `"delivered"` orders
2. Map the delivered orders to a simplified format: `{ orderId, customerName, total }`
3. Sort the simplified list by `total` descending (highest first)
4. Calculate the total revenue from delivered orders using `.reduce()`
5. Log the sorted list and the total revenue

All of the above should be done as a **method chain** on `orders`.

---

### Exercise 3 — hard

Build a **pipeline of transformation functions** for a product catalogue. Each function must be an arrow function.

```js
const rawCatalogue = [
  { id: 1, title: "  wireless MOUSE  ",  basePrice: 29.99, stock: 0,  category: "peripherals" },
  { id: 2, title: "mechanical KEYBOARD", basePrice: 89.99, stock: 15, category: "peripherals" },
  { id: 3, title: "27-INCH MONITOR",     basePrice: 319.99, stock: 3, category: "displays"    },
  { id: 4, title: "usb-c HUB",           basePrice: 49.99, stock: 22, category: "accessories" },
  { id: 5, title: "WEBCAM hd",           basePrice: 79.99, stock: 0,  category: "peripherals" },
];
```

Write these arrow functions:

1. `normalizeTitle` — trims whitespace and converts to title case  
   (`"  wireless MOUSE  "` → `"Wireless Mouse"`)

2. `applyMarkup` — adds a 20% markup to basePrice and rounds to 2 decimal places  
   (`29.99` → `35.99`)

3. `addAvailability` — returns a new product object with an added `isAvailable` property (`stock > 0`)

4. `formatProduct` — takes a raw product, runs it through all three transforms, and returns the final object:
   ```js
   { id: 1, title: "Wireless Mouse", price: 35.99, stock: 0, category: "peripherals", isAvailable: false }
   ```

5. Use `.map()` with `formatProduct` to transform the entire `rawCatalogue`, then:
   - Filter to only available products
   - Log each available product in the format:
     ```
     [peripherals] Mechanical Keyboard — $107.99 (15 in stock)
     ```
   - Log the count of unavailable products at the end

---

## Quick reference

```
Arrow functions cheat sheet
─────────────────────────────────────────────────────
FULL BODY        const fn = (a, b) => { return a + b; };
IMPLICIT RETURN  const fn = (a, b) => a + b;
SINGLE PARAM     const fn = x => x * 2;
NO PARAMS        const fn = () => value;
RETURN OBJECT    const fn = () => ({ key: value });
─────────────────────────────────────────────────────
this             NO own 'this' — inherits from surrounding scope (lexical)
arguments        NOT available — use rest params (...args) instead
new              CANNOT be used as constructor
hoisting         NOT hoisted — same as regular function expressions
─────────────────────────────────────────────────────
USE arrow for    callbacks, array methods, utility functions, currying
USE regular for  object methods needing 'this', constructors, methods needing 'arguments'
─────────────────────────────────────────────────────
arrow vs regular  arrow = shorter, lexical this
                  regular = own this, arguments, can be constructor
```

---

## Connected topics

- **14 — Function Declarations and Expressions** — the foundations arrow functions build on
- **16 — Parameters in depth** — rest parameters `...args` replace `arguments` in arrow functions
- **18 — Scope and closures** — lexical scoping is why arrow functions inherit `this`
- **34+ — Objects** — why arrow functions as object methods break `this`
- **41 — Higher-order functions** — arrow functions are used heavily as callbacks
- **42 — Currying and partial application** — `a => b => a + b` pattern
- **52+ — Async/Await** — arrow functions used in promise chains and async callbacks
