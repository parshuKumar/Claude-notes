# 19 — Hoisting

## What is this?

**Hoisting** is JavaScript's behaviour of processing certain declarations before any code runs. Before the first line of your code executes, the JS engine scans the entire scope and "hoists" (moves to the top) specific things.

What gets hoisted and *how* it gets hoisted depends on whether it's a `var`, `let`, `const`, function declaration, or function expression — and the differences are critical.

```js
console.log(greeting);  // undefined — not an error!
var greeting = "Hello";

sayHi();                // "Hi!" — works before the definition
function sayHi() {
  console.log("Hi!");
}
```

Both of those lines run before their declarations appear in the code. That is hoisting in action.

---

## Why does it matter?

Hoisting is one of the most common sources of bugs and confusion in JavaScript. It explains:

- Why `var` variables are accessible before their assignment (but are `undefined`)
- Why function declarations can be called before they're written
- Why `let` and `const` throw `ReferenceError` when accessed before their line
- What the **Temporal Dead Zone (TDZ)** is and why it exists
- Why function expressions and arrow functions behave differently from declarations

Getting this wrong leads to bugs that are extremely hard to track down — you access a variable and it's `undefined` or throws, and you don't know why. Understanding hoisting gives you the mental model to predict exactly what JavaScript will do.

---

## How JavaScript processes a scope — two phases

Before any code in a scope runs, the JS engine does a **creation phase**:

1. Scans for all declarations (`var`, `let`, `const`, `function`)
2. Sets them up in memory according to rules specific to each declaration type
3. **Then** starts executing code top to bottom

This is why access *before* declaration is even possible for some types — they've already been registered in memory.

---

## `var` hoisting

`var` declarations are hoisted to the top of their **function scope** (or global scope). The variable is created and initialized to `undefined` during the creation phase. The assignment stays in place.

```js
console.log(productName);  // undefined — NOT ReferenceError
var productName = "Laptop";
console.log(productName);  // "Laptop"
```

**What JavaScript actually sees (conceptually):**

```js
var productName;           // hoisted: declared and set to undefined
console.log(productName);  // undefined
productName = "Laptop";    // assignment stays here
console.log(productName);  // "Laptop"
```

The declaration moves up. The assignment does not.

### `var` is hoisted to function scope, not block scope

```js
function processCart() {
  console.log(discount);  // undefined — hoisted within the function

  if (true) {
    var discount = 0.1;   // var escapes the if-block, hoisted to function top
  }

  console.log(discount);  // 0.1
}

processCart();
```

`var` inside an `if` block is hoisted to the top of `processCart` — not to the top of the `if` block. This is the scope leak behaviour from topic 01/18.

---

## `let` and `const` hoisting — the Temporal Dead Zone

`let` and `const` ARE hoisted — but they are **not initialized**. They exist in memory from the start of the scope, but cannot be accessed until the line where they're declared is reached during execution. The period between the start of the scope and the declaration line is called the **Temporal Dead Zone (TDZ)**.

```js
console.log(sessionToken);  // ❌ ReferenceError: Cannot access 'sessionToken' before initialization
let sessionToken = "tok_abc123";
console.log(sessionToken);  // "tok_abc123"
```

Same with `const`:

```js
console.log(MAX_RETRIES);  // ❌ ReferenceError: Cannot access 'MAX_RETRIES' before initialization
const MAX_RETRIES = 3;
```

**The TDZ — visualized:**

```
Start of scope
│
│  ← TDZ begins for sessionToken (it exists but is uninitialized)
│
│  console.log(sessionToken)  ← ❌ still in TDZ — ReferenceError
│
│  let sessionToken = "tok_abc123"  ← TDZ ends here, variable initialized
│
│  console.log(sessionToken)  ← ✅ "tok_abc123"
│
End of scope
```

The TDZ exists to catch bugs. `var`'s silent `undefined` makes it easy to use a variable before you've set it up — you get no error, just a confusing `undefined`. `let`/`const` fail loudly so you know immediately that the access is before the assignment.

### TDZ in block scopes

```js
let outerValue = "outer";

{
  // TDZ for outerValue starts here in this inner block scope?
  // No — outerValue is from the outer scope, already initialized
  console.log(outerValue);  // ✅ "outer"

  // But a NEW let in this block:
  console.log(innerValue);  // ❌ ReferenceError — TDZ
  let innerValue = "inner";
}
```

### TDZ in function parameters — a subtle case

```js
// ❌ b's default references a — but a is in TDZ when b is evaluated
function fn(a = b, b = 1) {
  return [a, b];
}
fn();  // ReferenceError — b is in TDZ when a's default runs

// ✅ Left-to-right: a is initialized before b's default can reference it
function fn(a = 1, b = a * 2) {
  return [a, b];
}
fn();  // [1, 2]
```

---

## Function declaration hoisting

Function declarations are **fully hoisted** — the entire function body is available from the very start of the scope. This is why you can call them before they appear:

```js
const result = calculateArea(5, 3);  // ✅ works — fully hoisted
console.log(result);  // 15

function calculateArea(width, height) {
  return width * height;
}
```

**What JS sees:**

```js
function calculateArea(width, height) {  // entire function hoisted to top
  return width * height;
}

const result = calculateArea(5, 3);
console.log(result);
```

This is why many developers put utility functions at the bottom of a file — they're callable from anywhere in that scope regardless of position.

---

## Function expression and arrow function hoisting

Function expressions and arrow functions are stored in variables. The variable is hoisted (as `undefined` for `var`, or TDZ for `let`/`const`), but the function itself is **not**.

```js
// With var
console.log(formatDate);  // undefined — var hoisted, function not yet assigned
formatDate("2026-01-01"); // ❌ TypeError: formatDate is not a function

var formatDate = function(date) {
  return new Date(date).toLocaleDateString();
};

// With const
console.log(formatPrice);  // ❌ ReferenceError — TDZ
const formatPrice = (amount) => `$${amount.toFixed(2)}`;
```

**What JS sees for the `var` case:**

```js
var formatDate;              // hoisted: undefined
console.log(formatDate);     // undefined
formatDate("2026-01-01");    // TypeError: undefined is not a function
formatDate = function(date) { ... };  // assignment stays here
```

The error is `TypeError: formatDate is not a function` — not `ReferenceError`. Because `formatDate` exists (it's `undefined`), it just isn't a function yet. This distinction matters when debugging.

---

## Hoisting in different scopes

Hoisting happens at **every** scope boundary, not just global:

```js
function outer() {
  console.log(localVar);  // undefined — hoisted within outer()
  var localVar = "I'm local";

  function inner() {
    console.log(innerVar);  // undefined — hoisted within inner()
    var innerVar = "I'm inner";
  }

  inner();
}

outer();
```

Each function creates its own scope, and hoisting applies independently within each one.

---

## The interaction: `var` in a block inside a function

This specific combination trips people up constantly:

```js
function checkEligibility(age) {
  if (age >= 18) {
    var status = "eligible";   // var hoists to checkEligibility, not the if-block
    var reason = "adult";
  }

  console.log(status);  // "eligible" if age >= 18, undefined if age < 18
  console.log(reason);  // same
}

checkEligibility(20);  // "eligible", "adult"
checkEligibility(15);  // undefined, undefined — no error!
```

The variables exist in the function scope regardless of whether the `if` block ran. If the block didn't execute, the variables are `undefined` — silently. This is one of the reasons `var` is avoided in modern code.

**With `let`:**

```js
function checkEligibility(age) {
  if (age >= 18) {
    let status = "eligible";  // block-scoped to the if-block
  }

  console.log(status);  // ❌ ReferenceError — status doesn't exist here
}
```

`let` forces the variable to stay inside the block. Cleaner, and the error is explicit rather than silent.

---

## Tricky things you'll encounter in the real world

### 1. `typeof` on a TDZ variable throws — but `typeof` on undeclared is safe

```js
// ❌ let in TDZ — typeof throws ReferenceError (unusual behaviour!)
console.log(typeof myLetVar);  // ReferenceError
let myLetVar = 5;

// ✅ Undeclared variable — typeof is safe (doesn't throw)
console.log(typeof undeclaredVariable);  // "undefined" — no error
```

This is a surprising inconsistency. `typeof` is normally the safe way to check if something exists. But in the TDZ, `typeof` throws — breaking the "typeof is always safe" rule people learn. This is intentional — the TDZ is there to prevent access, and `typeof` is no exception.

### 2. `var` inside a `switch` — all cases share the same scope

```js
function getLabel(type) {
  switch (type) {
    case "admin":
      var label = "Administrator";  // hoisted to getLabel scope
      break;
    case "user":
      var label = "Regular User";   // same variable! no conflict warning
      break;
  }
  return label;
}
```

Both `case` blocks declare the same `var label` — but because `var` is function-scoped, they're the same variable. No error. This is why you use `let` inside `switch` cases (and wrap in `{}` if needed to create a block scope).

### 3. Function declarations inside blocks — implementation-defined chaos

```js
if (true) {
  function hoistedHowExactly() {
    return "I'm in a block";
  }
}

console.log(hoistedHowExactly());  // ✅ in non-strict mode (most browsers)
                                    // ❌ ReferenceError in strict mode
```

Function declarations inside `if`/`for`/`while` blocks have **inconsistent behaviour** across environments and strict/non-strict mode. Some engines hoist them to the function scope; others don't. 

**Rule: never put function declarations inside blocks.** Use a function expression assigned to `let`/`const` instead:

```js
if (true) {
  const doTask = () => "I'm in a block";
  doTask();
}
```

### 4. Class declarations — hoisted but in TDZ (like `let`)

```js
const obj = new MyClass();  // ❌ ReferenceError — TDZ
class MyClass {
  constructor() { this.name = "example"; }
}
```

`class` declarations behave like `let` — they are hoisted but not initialized. You cannot use a class before its declaration. This is intentional — the language designers didn't want the confusing "callable before defined" behaviour that `function` has.

### 5. Redeclaration behaviour

```js
// var — allows redeclaration (no error, silently overwrites)
var username = "Alice";
var username = "Bob";  // ✅ no error — second declaration ignored, assignment runs
console.log(username);  // "Bob"

// let — throws on redeclaration in same scope
let email = "a@a.com";
let email = "b@b.com";  // ❌ SyntaxError: Identifier 'email' has already been declared

// const — same as let
const TAX_RATE = 0.08;
const TAX_RATE = 0.09;  // ❌ SyntaxError
```

`var` silently allows redeclaration — a historical mistake. `let` and `const` throw a `SyntaxError` immediately (before the code even runs — it's caught during compilation).

### 6. Hoisting doesn't move across module boundaries

Each ES6 module has its own scope. Hoisting happens within a module, not between modules. `import` statements are hoisted to the top of the module file they appear in — but the bindings are live (not `undefined`), and the module code runs after all imports are resolved.

```js
// main.js
doSomething();  // ✅ works — import is hoisted to top of module
import { doSomething } from "./utils.js";
```

`import` is always processed first regardless of where it appears in the file. This is different from `require()` in CommonJS (Node.js), which is NOT hoisted.

---

## Common mistakes

### Mistake 1: Accessing `let`/`const` before declaration and expecting `undefined`

```js
// ❌ Expecting undefined like var would give
console.log(orderStatus);  // ReferenceError — not undefined!
let orderStatus = "pending";

// ✅ Always declare before use with let/const
let orderStatus = "pending";
console.log(orderStatus);
```

### Mistake 2: Calling a function expression before assignment

```js
// ❌
processPayment(cart);
const processPayment = (cart) => { /* ... */ };  // ReferenceError (TDZ)

// Also ❌ with var:
processPayment(cart);                             // TypeError: not a function
var processPayment = (cart) => { /* ... */ };

// ✅ Use a function declaration if you need to call before definition
processPayment(cart);
function processPayment(cart) { /* ... */ }
```

### Mistake 3: Relying on `var` hoisting as a feature

```js
// ❌ Using a var before it's set — "works" but gives wrong results
function calculateTotal() {
  total = 100;        // assigning before var declaration — works because of hoisting
  var discount = 10;
  var total;          // declaration hoisted; but this is confusing
  return total - discount;  // 90
}
```

Don't write code that depends on `var` hoisting. It's not a feature to use — it's a quirk to be aware of.

### Mistake 4: Expecting class to be callable before its line

```js
// ❌
const instance = new EventEmitter();  // ReferenceError
class EventEmitter {
  constructor() { this.listeners = {}; }
}

// ✅ Classes must be declared before use
class EventEmitter {
  constructor() { this.listeners = {}; }
}
const instance = new EventEmitter();
```

---

## Summary comparison table

```
Declaration  | Hoisted? | Initialized?      | Before declaration gives
─────────────┼──────────┼───────────────────┼──────────────────────────
var          | ✅ yes   | ✅ undefined       | undefined (no error)
let          | ✅ yes   | ❌ TDZ            | ReferenceError
const        | ✅ yes   | ❌ TDZ            | ReferenceError
function     | ✅ yes   | ✅ full body       | works perfectly
function exp | only var | ❌ (var = undef)  | undefined / TypeError
arrow fn     | only var | ❌ (var = undef)  | undefined / TypeError
class        | ✅ yes   | ❌ TDZ            | ReferenceError
```

---

## Practice exercises

### Exercise 1 — easy

**Predict the output** of each snippet before running it. Write your prediction as a comment, then verify:

```js
// Snippet A
console.log(cityName);
var cityName = "Berlin";
console.log(cityName);

// Snippet B
console.log(temperature);
let temperature = 22;

// Snippet C
greet("Taylor");
function greet(name) {
  console.log("Hello, " + name);
}

// Snippet D
sayBye("Jordan");
const sayBye = function(name) {
  console.log("Bye, " + name);
};
```

For each one, explain in a comment *why* you get that result — not just what it is.

---

### Exercise 2 — medium

This code has 4 hoisting-related bugs. Find each one, explain what's wrong, fix it:

```js
// Bug hunt — find all 4 problems

function initializeDashboard() {
  renderHeader(userName);

  var userName = "Admin";

  const welcomeMsg = formatWelcome(userName);
  console.log(welcomeMsg);

  const formatWelcome = function(name) {
    return `Welcome back, ${name}!`;
  };

  for (var i = 0; i < 3; i++) {
    setTimeout(() => console.log("Tab", i), 0);
  }

  if (true) {
    var sessionActive = true;
  }
  console.log(sessionActive);
}

initializeDashboard();
```

Expected after fixes: header renders with `"Admin"`, welcome message logs correctly, tabs log 0, 1, 2, and session status logs without any hoisting surprises.

---

### Exercise 3 — hard

You're refactoring a legacy codebase that uses `var` throughout. The original code works but produces unexpected behaviour in two places due to hoisting. Your job is to:

1. Identify exactly why each piece of code behaves unexpectedly
2. Rewrite each section using `let`/`const` and function declarations/expressions appropriately so the behaviour is explicit and correct

```js
// Legacy code block 1 — cart discount system
function applyCartDiscounts(cart) {
  var finalPrices = [];

  for (var i = 0; i < cart.items.length; i++) {
    var discount = getDiscount(cart.items[i].category);
    var handlers = [];

    for (var j = 0; j < 3; j++) {
      handlers.push(function() {
        return cart.items[i].price * (1 - discount);
      });
    }

    finalPrices.push(handlers[0]());
  }

  return finalPrices;
}

// Legacy code block 2 — permission checker
function checkPermissions(role) {
  console.log("Checking role:", roleName);

  if (role === "admin") {
    var roleName = "Administrator";
    var canEdit = true;
  } else if (role === "editor") {
    var roleName = "Content Editor";
    var canEdit = true;
  } else {
    var roleName = "Viewer";
    var canEdit = false;
  }

  return { roleName, canEdit };
}
```

For each block: explain the hoisting problem, then write the corrected version.

---

## Quick reference

```
Hoisting — cheat sheet
─────────────────────────────────────────────────────
var             hoisted + initialized to undefined
                accessible before declaration (gives undefined)
                function-scoped (ignores blocks)
                can be redeclared (no error)

let/const       hoisted + NOT initialized (TDZ)
                accessing before declaration = ReferenceError
                block-scoped
                cannot be redeclared

function decl   fully hoisted (entire body)
                callable before it appears in code

function expr   variable hoisted (var=undefined, let/const=TDZ)
arrow function  function itself is NOT hoisted
                calling before assignment = TypeError or ReferenceError

class           hoisted but in TDZ (like let)
                cannot be used before declaration
─────────────────────────────────────────────────────
TDZ             Temporal Dead Zone
                starts at scope entry
                ends at the declaration line
                typeof in TDZ throws ReferenceError (unlike undeclared)
─────────────────────────────────────────────────────
RULES TO FOLLOW
  Always declare before use
  Prefer let/const over var (explicit errors > silent undefined)
  Use function declarations for utilities you want callable anywhere
  Never declare functions inside if/for/while blocks
─────────────────────────────────────────────────────
```

---

## Connected topics

- **01 — Variables** — `var`/`let`/`const` differences first introduced; hoisting is the deep explanation of why they behave differently
- **14 — Function declarations vs expressions** — the hoisting difference explained behaviourally
- **18 — Scope and closures** — hoisting determines what's in a scope at creation time; closures capture that scope
- **20 — The `this` keyword** — how `this` is determined at call time (next topic)
- **Classes (45+)** — class declarations are in TDZ; `extends` and `super` interact with hoisting
