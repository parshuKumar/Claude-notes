# 20 — Higher-Order Functions

## What is this?
A higher-order function (HOF) is a function that either **accepts another function as an argument**, **returns a function**, or both. Think of it like a machine that doesn't just do one thing — it takes instructions (in the form of another function) and runs them. A coffee machine is always a coffee machine, but you swap out the coffee pod to change what it brews. The machine is the HOF; the pod is the function you pass in.

## Why does it matter?
Higher-order functions are everywhere in JavaScript — `map`, `filter`, `forEach`, `reduce`, `setTimeout`, event listeners — all of these are HOFs. Understanding them unlocks the entire functional programming side of JS and makes your code dramatically more flexible and reusable.

## Syntax

```js
// ---- Pattern 1: HOF that ACCEPTS a function (callback) ----

function applyToNumber(number, transformFn) { // transformFn is a function passed in
  return transformFn(number);                 // HOF calls the function it received
}

function double(n) { return n * 2; }          // a regular function — will be passed in

applyToNumber(5, double);   // returns 10 — passes 5 into double()
applyToNumber(5, n => n * n); // returns 25 — passes an arrow fn directly


// ---- Pattern 2: HOF that RETURNS a function ----

function makeAdder(addAmount) {               // outer function — the HOF
  return function (number) {                  // inner function — returned to the caller
    return number + addAmount;               // closes over addAmount (closure!)
  };
}

const addTen = makeAdder(10);  // returns a new function with addAmount locked to 10
addTen(5);                     // 15
addTen(22);                    // 32
```

## How it works — line by line

**Pattern 1:**
1. `applyToNumber(number, transformFn)` — declares a function that takes TWO parameters: a regular value AND a function.
2. `return transformFn(number)` — inside the HOF, the received function is called with `number` as the argument. The HOF doesn't care what `transformFn` does — that's the caller's decision.
3. `applyToNumber(5, double)` — passes the `double` function by reference (no `()` — we're not calling it yet, we're handing it over).
4. `applyToNumber(5, n => n * n)` — passes an inline arrow function instead. Identical pattern.

**Pattern 2:**
1. `makeAdder(addAmount)` — outer function receives a value (`10`).
2. `return function(number) { ... }` — instead of returning a value, it returns a brand-new function. This is where closures and HOFs overlap.
3. `const addTen = makeAdder(10)` — `makeAdder` runs, `addAmount` is set to `10`, and the inner function is stored in `addTen`.
4. `addTen(5)` — calls the inner function with `number = 5`. It adds `5 + 10` using the closed-over `addAmount`.

## Example 1 — basic

```js
// A HOF that runs a function on every item in a list
// (simplified version of what .forEach does internally)

function runOnEach(itemList, actionFn) { // accepts an array AND a function
  for (let i = 0; i < itemList.length; i++) {
    actionFn(itemList[i]);               // calls the function for every item
  }
}

const productNames = ["Keyboard", "Mouse", "Monitor"];

// Pass a function that defines WHAT to do with each item
runOnEach(productNames, function (product) {
  console.log("In stock: " + product);
});
// "In stock: Keyboard"
// "In stock: Mouse"
// "In stock: Monitor"

// Swap the function out — the HOF stays the same, the behaviour changes
runOnEach(productNames, function (product) {
  console.log(product.toUpperCase()); // different action, same HOF
});
// "KEYBOARD"
// "MOUSE"
// "MONITOR"
```

## Example 2 — real world

```js
// --- HOF used to build a flexible discount system ---

function applyDiscount(calculateDiscountFn) { // accepts a function that knows how to discount
  return function (originalPrice) {           // returns a function that takes a price
    const discountedPrice = calculateDiscountFn(originalPrice);
    console.log(`Original: $${originalPrice} → After discount: $${discountedPrice.toFixed(2)}`);
    return discountedPrice;
  };
}

// Different discount strategies — each is just a function
const tenPercentOff  = price => price * 0.90;
const halfPrice      = price => price * 0.50;
const fixedFiveDollars = price => price - 5;

// Build specific discounters by passing a strategy into the HOF
const applyStudentDiscount = applyDiscount(tenPercentOff);
const applyFlashSale       = applyDiscount(halfPrice);
const applyLoyaltyDiscount = applyDiscount(fixedFiveDollars);

applyStudentDiscount(80);  // "Original: $80 → After discount: $72.00"
applyFlashSale(80);        // "Original: $80 → After discount: $40.00"
applyLoyaltyDiscount(80);  // "Original: $80 → After discount: $75.00"

// The HOF (applyDiscount) never needs to change — swap the strategy function to change behaviour
```

## Common mistakes

- **Calling the function instead of passing it**

  When you pass a function to a HOF, you pass the reference — no `()`. Adding `()` calls it immediately and passes the *result* (probably `undefined`) instead.

  ```js
  function greetUser(name) {
    console.log("Hello, " + name);
  }

  // ❌ Wrong — greetUser() is CALLED immediately, its return value (undefined) is passed
  runOnEach(["Alice", "Bob"], greetUser()); // TypeError or silent bug

  // ✅ Right — greetUser is passed as a reference, the HOF calls it at the right time
  runOnEach(["Alice", "Bob"], greetUser);
  ```

- **Forgetting that the HOF decides when/how to call your function**

  You define what the function does, but the HOF controls when it's invoked and what arguments it passes.

  ```js
  // ❌ Wrong assumption — thinking you control the arguments
  runOnEach(["Alice", "Bob"], function (name, index) {
    console.log(index + ": " + name); // index will always be undefined
  });
  // Our custom runOnEach above only passes itemList[i] — no index

  // ✅ Right — check what the HOF passes to your callback
  // JS built-ins like .forEach pass (item, index, array) — know your HOF's contract
  ["Alice", "Bob"].forEach(function (name, index) {
    console.log(index + ": " + name); // 0: Alice  1: Bob — works because forEach passes index
  });
  ```

- **Confusing a function that returns a function with a regular function**

  ```js
  function makeLogger(prefix) {
    return function (message) {
      console.log(prefix + ": " + message);
    };
  }

  // ❌ Wrong — calling with both arguments at once doesn't work
  makeLogger("ERROR", "File not found"); // prefix = "ERROR", "File not found" is ignored

  // ✅ Right — call in two steps: first to configure, second to use
  const logError = makeLogger("ERROR");   // step 1: get back the configured function
  logError("File not found");             // step 2: use it — "ERROR: File not found"
  logError("Disk full");                  // "ERROR: Disk full"
  ```

## Practice exercises

### Exercise 1 — easy
Write a higher-order function called `transformValue` that takes two arguments: a `value` (number) and a `transformFn` (function). It should return the result of calling `transformFn` with `value`.

Test it with three different functions passed in:
1. A function that squares the number
2. A function that adds 100 to it
3. A function that converts it to a string like `"Value: 42"`

```js
// Write your code here
```

### Exercise 2 — medium
Write a HOF called `createValidator` that takes a `validationFn` as its argument and **returns a new function**. The returned function should take a `value`, run `validationFn` on it, and:
- If the validation passes, log `"✅ Valid: [value]"` and return `true`
- If it fails, log `"❌ Invalid: [value]"` and return `false`

Use `createValidator` to create two validators:
1. `validateAge` — passes if the number is 18 or above
2. `validateUsername` — passes if the string length is 5 or more characters

Test each validator with both a passing and a failing value.

```js
// Write your code here
```

### Exercise 3 — hard
You're building a **pipeline system** — a function called `pipeline` that takes a value and an array of functions, then passes the value through each function in order, where the output of one becomes the input of the next.

For example:
```js
pipeline(10, [double, addFive, square])
// step 1: double(10)  → 20
// step 2: addFive(20) → 25
// step 3: square(25)  → 625
// final result: 625
```

Requirements:
1. Write the `pipeline(startValue, fnArray)` function
2. Define at least 4 individual transformation functions (e.g. double, halve, addTen, square)
3. Run two different pipelines with different combinations of functions on the same starting value
4. Log each step as it happens: `"Step 1: 10 → 20"`

```js
// Write your code here
```

## Quick reference (cheat sheet)

| Pattern | What it looks like | Common examples |
|---|---|---|
| Accepts a function | `fn(value, callbackFn)` | `forEach`, `setTimeout`, `addEventListener` |
| Returns a function | `fn(config) { return fn2 }` | `makeAdder`, `createValidator`, closures |
| Both | `fn(config) { return fn2(val, fn3) }` | `applyDiscount` in Example 2 |

**Key rules:**
- Pass functions **without `()`** — `greetUser` not `greetUser()`
- The HOF controls **when** your function is called and **what arguments** it gets
- HOFs + closures = the foundation of `map`, `filter`, `reduce`, and most of JS's power
- The function you pass in is called a **callback** (covered in depth in topic 21)

**Built-in HOFs you already know or will learn:**
- `array.forEach(fn)` — runs fn for each item
- `array.map(fn)` — transforms each item using fn
- `array.filter(fn)` — keeps items where fn returns true
- `array.reduce(fn)` — collapses array using fn
- `setTimeout(fn, delay)` — runs fn after a delay

## Connected topics
- **19 — Closures**: HOFs that return functions are always closures — the two concepts are inseparable
- **21 — Callback functions**: a callback IS the function you pass into a HOF — this topic and the next are one continuous idea
- **26–31 — Array methods**: `map`, `filter`, `reduce`, `forEach`, `find`, `some` are all HOFs built into arrays
