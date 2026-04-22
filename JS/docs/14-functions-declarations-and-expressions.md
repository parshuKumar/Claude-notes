# 14 — Functions — Declarations and Expressions

## What is this?

A **function** is a named, reusable block of code that performs a task. You define it once and call it as many times as you need.

JavaScript has multiple ways to define a function. The two foundational forms are:

**Function declaration:**
```js
function greetUser(name) {
  return "Hello, " + name;
}
```

**Function expression:**
```js
const greetUser = function(name) {
  return "Hello, " + name;
};
```

Both define a function that does the same thing. The differences are in hoisting, when the function is available, and how they behave in different contexts.

---

## Why does it matter?

Functions are the single most important building block in JavaScript. Every meaningful program is made of functions.

They let you:
- **Avoid repeating code** — write logic once, call it anywhere
- **Name your intent** — `calculateTax(price)` is clearer than the raw math inline
- **Test in isolation** — a function with defined inputs and outputs is easy to verify
- **Build larger systems** — every framework, library, and pattern is built from functions

If you can't write functions confidently, you can't write real JavaScript. This is the topic where everything starts to come together.

---

## Syntax

### Function declaration

```js
function functionName(parameter1, parameter2) {
  // body — code that runs when called
  return value; // optional — if omitted, the function returns undefined
}
```

### Function expression

```js
const functionName = function(parameter1, parameter2) {
  // body
  return value;
};  // ← semicolon here — this is a variable assignment statement
```

### Named function expression

```js
const calculateTotal = function calculateTotal(price, tax) {
  return price + tax;
};
```

The inner name (`calculateTotal`) is only accessible inside the function body — useful for recursion and debugging stack traces.

### Calling a function

```js
functionName(argument1, argument2);

const result = functionName(argument1, argument2); // capture the return value
```

**Parameters vs arguments:**
- **Parameters** — the variable names in the function definition (`price`, `tax`)
- **Arguments** — the actual values passed when calling (`99.99`, `0.08`)

---

## How it works — line by line

```js
function calculateOrderTotal(itemPrice, quantity, discountPercent) {
  const subtotal = itemPrice * quantity;
  const discountAmount = subtotal * (discountPercent / 100);
  const total = subtotal - discountAmount;
  return total;
}

const orderTotal = calculateOrderTotal(25.00, 4, 10);
console.log(orderTotal);
```

**Execution trace:**

```
1. JS sees the function declaration → stores it in memory (hoisted)
2. calculateOrderTotal(25.00, 4, 10) is called
3. Parameters are assigned: itemPrice = 25.00, quantity = 4, discountPercent = 10
4. subtotal = 25.00 * 4 = 100.00
5. discountAmount = 100.00 * (10 / 100) = 10.00
6. total = 100.00 - 10.00 = 90.00
7. return 90.00 — execution leaves the function, hands back 90.00
8. orderTotal = 90.00
9. console.log → "90"
```

**What `return` does:**
- Immediately stops the function
- Sends a value back to the caller
- If you don't write `return`, or write `return;` with no value, the function returns `undefined`

---

## Example 1 — basic

### Temperature converter

```js
function celsiusToFahrenheit(celsius) {
  return (celsius * 9 / 5) + 32;
}

function fahrenheitToCelsius(fahrenheit) {
  return (fahrenheit - 32) * 5 / 9;
}

console.log(celsiusToFahrenheit(0));    // 32
console.log(celsiusToFahrenheit(100));  // 212
console.log(fahrenheitToCelsius(98.6)); // 37
```

### Default parameters

If a caller doesn't pass an argument, you can specify a default:

```js
function createWelcomeMessage(userName, platform = "the app") {
  return `Welcome to ${platform}, ${userName}!`;
}

console.log(createWelcomeMessage("Priya", "Dashboard"));
// "Welcome to Dashboard, Priya!"

console.log(createWelcomeMessage("Priya"));
// "Welcome to the app, Priya!"  ← default used
```

Default parameters were introduced in ES6. Before ES6, developers used `platform = platform || "the app"` — you'll see this in older codebases.

### Multiple return points (early return)

```js
function getShippingCost(orderTotal) {
  if (orderTotal >= 100) {
    return 0;  // free shipping
  }
  if (orderTotal >= 50) {
    return 4.99;
  }
  return 9.99;  // standard shipping
}

console.log(getShippingCost(120));  // 0
console.log(getShippingCost(75));   // 4.99
console.log(getShippingCost(30));   // 9.99
```

A function can have multiple `return` statements. The first one that executes ends the function.

---

## Example 2 — real world

### Input validation function

```js
function validateEmail(email) {
  if (typeof email !== "string") {
    return { valid: false, reason: "Email must be a string" };
  }
  if (email.trim() === "") {
    return { valid: false, reason: "Email cannot be empty" };
  }
  if (!email.includes("@")) {
    return { valid: false, reason: "Email must contain @" };
  }
  if (!email.includes(".")) {
    return { valid: false, reason: "Email must contain a domain" };
  }
  return { valid: true, reason: null };
}

const result = validateEmail("sarah@example.com");
console.log(result);  // { valid: true, reason: null }

const badResult = validateEmail("notanemail");
console.log(badResult);  // { valid: false, reason: "Email must contain @" }
```

Returning an object with `{ valid, reason }` is a real pattern — gives the caller both the decision and the explanation.

### Reusable formatting functions

```js
function formatCurrency(amount, currencyCode = "USD") {
  return new Intl.NumberFormat("en-US", {
    style: "currency",
    currency: currencyCode,
  }).format(amount);
}

function formatDate(dateString) {
  const date = new Date(dateString);
  return new Intl.DateTimeFormat("en-US", {
    year: "numeric",
    month: "long",
    day: "numeric",
  }).format(date);
}

console.log(formatCurrency(1299.9));        // "$1,299.90"
console.log(formatCurrency(89.5, "EUR"));   // "€89.50"
console.log(formatDate("2026-04-21"));      // "April 21, 2026"
```

Utility functions like these get written once and used across an entire application.

### Function expression assigned to an object property

```js
const taxCalculator = {
  rate: 0.08,

  calculate: function(price) {
    return price * this.rate;
  },

  withTax: function(price) {
    return price + this.calculate(price);
  },
};

console.log(taxCalculator.calculate(100));  // 8
console.log(taxCalculator.withTax(100));    // 108
```

Function expressions stored in object properties are called **methods**. This is a preview of object-oriented patterns covered deeply in the Objects and Classes sections.

---

## Tricky things you'll encounter in the real world

### 1. Hoisting — declarations are available before they're written

```js
// ✅ This works — function declarations are hoisted
const result = double(5);
console.log(result);  // 10

function double(number) {
  return number * 2;
}
```

JavaScript hoists function declarations to the top of their scope before any code runs. The entire function body is hoisted — not just the name.

```js
// ❌ This fails — function expressions are NOT hoisted
const result = double(5);  // TypeError: double is not a function

const double = function(number) {
  return number * 2;
};
```

With a function expression, only the variable declaration is hoisted (as `undefined`), not the value. Calling `double` before assignment is like calling `undefined(5)`.

**Practical rule:** Declare helper functions at the top of the file, or use declarations (which hoist) if you need to call them before they appear.

### 2. Missing `return` — function silently returns `undefined`

```js
function getDiscountedPrice(price, discountPercent) {
  const discounted = price * (1 - discountPercent / 100);
  // ⚠️ forgot return!
}

const finalPrice = getDiscountedPrice(100, 20);
console.log(finalPrice);  // undefined — not 80!
```

This is one of the most frequent bugs for beginners. The function runs, does the math correctly, and then silently discards the result. Always check you're returning the value, not just computing it.

### 3. Parameters are copies — reassigning them doesn't affect the caller (for primitives)

```js
function applyDiscount(price, discountPercent) {
  price = price * (1 - discountPercent / 100);  // modifies the local copy
  return price;
}

let productPrice = 100;
applyDiscount(productPrice, 20);
console.log(productPrice);  // 100 — unchanged!

// You have to use the return value:
productPrice = applyDiscount(productPrice, 20);
console.log(productPrice);  // 80
```

For **primitives** (numbers, strings, booleans), JS passes a copy. Reassigning the parameter inside the function does nothing to the original. For **objects and arrays**, it's different — covered next point.

### 4. Objects passed to functions ARE mutable — this surprises people

```js
function applyFreeShipping(order) {
  order.shippingCost = 0;  // ⚠️ modifies the ORIGINAL object
}

const myOrder = { total: 150, shippingCost: 9.99 };
applyFreeShipping(myOrder);
console.log(myOrder.shippingCost);  // 0 — the original was changed!
```

Objects are passed by reference. The function receives the same object in memory. Modifying properties inside the function changes the original. This can be intentional (mutation functions) or a nasty bug (unexpected side effects).

To avoid mutating, return a new object:

```js
function applyFreeShipping(order) {
  return { ...order, shippingCost: 0 };  // spread creates a new object
}

const myOrder = { total: 150, shippingCost: 9.99 };
const updatedOrder = applyFreeShipping(myOrder);
console.log(myOrder.shippingCost);    // 9.99 — unchanged
console.log(updatedOrder.shippingCost); // 0
```

### 5. Functions without `return` used in expressions give `undefined`

```js
function logTotal(total) {
  console.log("Total: $" + total);
  // no return statement
}

const displayValue = logTotal(99.99);
// prints: "Total: $99.99"
console.log(displayValue);  // undefined

// This causes a bug if you do:
const tax = logTotal(99.99) * 0.08;  // undefined * 0.08 = NaN
```

Functions that **do** something (side effects like logging) are different from functions that **produce** something (return a value). Mixing these up causes subtle `undefined`/`NaN` bugs.

### 6. Extra arguments are silently ignored; missing arguments are `undefined`

```js
function add(a, b) {
  return a + b;
}

console.log(add(2, 3, 100, 200));  // 5 — extra args ignored
console.log(add(2));               // NaN — b is undefined, 2 + undefined = NaN
```

JavaScript never throws an error for the wrong number of arguments. This means bugs from missing arguments are silent. Always consider what your function should do when called with missing data — use default parameters or validate at the top.

---

## Common mistakes

### Mistake 1: Forgetting to return

```js
// ❌ Computes but discards result
function formatUsername(rawName) {
  const formatted = rawName.trim().toLowerCase();
}

const name = formatUsername("  SARAH  ");
console.log(name);  // undefined

// ✅
function formatUsername(rawName) {
  return rawName.trim().toLowerCase();
}
```

### Mistake 2: Calling a function expression before it's defined

```js
// ❌ TypeError — processPayment is undefined at this point
processPayment(cart);

const processPayment = function(cart) {
  // ...
};

// ✅ Either move the call after the definition, or use a function declaration
function processPayment(cart) {
  // ...
}
processPayment(cart);  // now works — declarations are hoisted
```

### Mistake 3: Expecting to mutate a primitive via parameter

```js
// ❌ Does nothing to the original
function addTax(price) {
  price = price * 1.08;
}

let ticketPrice = 50;
addTax(ticketPrice);
console.log(ticketPrice);  // 50 — unchanged

// ✅ Return and reassign
function addTax(price) {
  return price * 1.08;
}
ticketPrice = addTax(ticketPrice);
```

### Mistake 4: Accidentally mutating an object parameter

```js
// ❌ Mutates the original — caller doesn't expect this
function resetScore(player) {
  player.score = 0;
}

// ✅ Return a new object if you want immutability
function resetScore(player) {
  return { ...player, score: 0 };
}
```

---

## Practice exercises

### Exercise 1 — easy

Write **three separate functions**:

1. `celsiusToFahrenheit(celsius)` — converts Celsius to Fahrenheit using `(c * 9/5) + 32`
2. `isHotWeather(celsius)` — returns `true` if celsius is above 30, otherwise `false`
3. `getWeatherLabel(celsius)` — returns:
   - `"freezing"` if below 0
   - `"cold"` if 0–14
   - `"mild"` if 15–29
   - `"hot"` if 30 and above

Test all three functions with at least 3 different temperatures each and log the results.

---

### Exercise 2 — medium

You work at an e-commerce company. Write these functions:

1. `calculateSubtotal(pricePerItem, quantity)` — returns price × quantity
2. `applyDiscount(subtotal, discountPercent)` — returns subtotal after discount; if `discountPercent` is not provided, default it to `0`
3. `calculateTax(amount, taxRate)` — returns the tax amount (amount × taxRate); default `taxRate` to `0.08`
4. `buildOrderSummary(pricePerItem, quantity, discountPercent)` — calls the above three functions and returns an object:
   ```js
   {
     subtotal: 80.00,
     discountAmount: 8.00,
     afterDiscount: 72.00,
     taxAmount: 5.76,
     orderTotal: 77.76,
   }
   ```

Test:
```js
console.log(buildOrderSummary(20, 4, 10));
// subtotal: 80, discount: 8, afterDiscount: 72, tax: 5.76, total: 77.76

console.log(buildOrderSummary(50, 2));
// no discount — discountPercent defaults to 0
```

---

### Exercise 3 — hard

Build a **student grade processing system** using only function declarations and expressions:

```js
const classResults = [
  { studentName: "Marcus Green",  scores: [72, 85, 91, 68, 79] },
  { studentName: "Layla Hassan",  scores: [95, 98, 92, 97, 100] },
  { studentName: "Tom Fischer",   scores: [40, 55, 48, 62, 51] },
  { studentName: "Anika Sharma",  scores: [83, 77, 88, 90, 85] },
  { studentName: "Ben Okafor",    scores: [60, 65, 58, 70, 63] },
];
```

Write these **five functions** — each must call only the previous ones (build up in layers):

1. `calculateAverage(scores)` — returns average rounded to 1 decimal place
2. `getLetterGrade(average)` — returns `"A"` (90+), `"B"` (80+), `"C"` (70+), `"D"` (60+), `"F"` (below 60)
3. `hasPassed(average)` — returns `true` if average ≥ 60
4. `buildStudentReport(student)` — takes one student object, calls the above functions, returns:
   ```js
   { studentName: "Marcus Green", average: 79.0, grade: "C", passed: true }
   ```
5. `generateClassReport(classResults)` — calls `buildStudentReport` for each student, returns array of report objects AND logs a class summary:
   ```
   Marcus Green   — Avg: 79.0 | Grade: C | PASSED
   Layla Hassan   — Avg: 96.4 | Grade: A | PASSED
   Tom Fischer    — Avg: 51.2 | Grade: F | FAILED
   Anika Sharma   — Avg: 84.6 | Grade: B | PASSED
   Ben Okafor     — Avg: 63.2 | Grade: D | PASSED

   Class summary: 4 passed, 1 failed | Class average: 74.9
   ```

Use `for...of` for all loops. Functions must be declared (not expressed) so the order of definition doesn't matter.

---

## Quick reference

```
Functions cheat sheet
─────────────────────────────────────────────────────
DECLARATION      function name(params) { return value; }
EXPRESSION       const name = function(params) { return value; };

HOISTING         declarations → fully hoisted (callable anywhere)
                 expressions  → variable hoisted as undefined (not callable before assignment)

DEFAULT PARAMS   function fn(x, y = 10) { }  — y defaults to 10 if not passed
RETURN           return exits immediately and sends a value back
NO RETURN        function implicitly returns undefined

PARAMETERS       local copies for primitives (reassigning doesn't affect caller)
OBJECTS          passed by reference — mutations affect the original
─────────────────────────────────────────────────────
CALL A FUNCTION  name(arg1, arg2)
STORE RESULT     const result = name(arg1, arg2)
EXTRA ARGS       silently ignored
MISSING ARGS     become undefined (use defaults or validate)
─────────────────────────────────────────────────────
declaration vs expression  declaration = hoisted; expression = not hoisted
```

---

## Connected topics

- **15 — Arrow Functions** — shorthand syntax for function expressions; different `this` behaviour
- **16 — Parameters in depth** — rest parameters `...args`, destructured parameters, argument validation
- **17 — Return values** — returning multiple values, early returns, returning functions
- **18 — Scope and closures** — how variables inside functions are isolated from the outside world
- **34+ — Objects** — function expressions as methods; `this` inside object methods
- **41 — Higher-order functions** — passing functions as arguments, returning functions from functions
