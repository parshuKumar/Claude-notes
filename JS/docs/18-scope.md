# 18 — Scope

## What is this?
Scope is the set of rules that determines **where a variable can be seen and used** in your code. Think of it like rooms in a house — a variable declared in the bedroom is only accessible inside that bedroom (and any rooms nested inside it), not from the kitchen. JavaScript has three kinds of scope: **global**, **function (local)**, and **block**.

## Why does it matter?
Scope controls which parts of your program can read or modify which variables. Without understanding scope you will run into "variable is not defined" errors, accidentally overwrite variables you didn't mean to touch, and have a very hard time debugging anything — these are among the most common bugs beginners face.

---

## Syntax

```js
// --- Global scope ---
let appName = "MyApp";          // visible everywhere in this file

// --- Function scope ---
function greetUser(userName) {
  let message = "Hello, " + userName; // only visible inside greetUser
  console.log(message);
}
// console.log(message); // ❌ ReferenceError — message doesn't exist out here

// --- Block scope ---
if (true) {
  let blockOnly = "I live in this block"; // only visible inside the { }
  const alsoBlock = 42;
}
// console.log(blockOnly); // ❌ ReferenceError
```

---

## How it works — line by line

```js
let appName = "MyApp";
```
Declared at the top level (outside any function or block) → lives in the **global scope**. Every function and block inside this file can read `appName`.

```js
function greetUser(userName) {
  let message = "Hello, " + userName;
```
`message` is declared with `let` inside the function body. Its scope is the **function body** — it is created when `greetUser` is called and destroyed when the function returns.

```js
if (true) {
  let blockOnly = "I live in this block";
```
`let` and `const` are **block-scoped**: a block is any pair of `{ }`. `blockOnly` only exists inside that `if` block. `var` would *not* be block-scoped — it would leak out to the nearest function or global scope, which is why `var` causes so many bugs.

---

## Scope chain — how JS looks up a variable

When you use a variable, JS searches outward through scopes until it finds it or runs out of scopes (ReferenceError):

```
inner block → enclosing function → outer function (if any) → global
```

```js
let siteName = "ShopEasy";         // global

function showHeader() {
  let section = "header";          // function scope

  function renderTitle() {
    // JS looks here first → not found
    // JS looks in showHeader → not found
    // JS looks in global → found!
    console.log(siteName);         // "ShopEasy"  ✅

    // JS looks here first → found in showHeader
    console.log(section);          // "header"    ✅
  }

  renderTitle();
}
```

---

## Example 1 — basic

```js
// Global variable — accessible everywhere
let totalUsers = 0;

function registerUser(userName) {
  // Function-scoped variable — only lives inside registerUser
  let welcomeMessage = "Welcome, " + userName + "!";

  totalUsers = totalUsers + 1;     // can READ and WRITE global from inside a function
  console.log(welcomeMessage);     // "Welcome, Alice!"
  console.log(totalUsers);         // 1
}

registerUser("Alice");

// console.log(welcomeMessage);    // ❌ ReferenceError — not visible here
console.log(totalUsers);           // 1 ✅ — global was modified inside the function
```

---

## Example 2 — real world

A simple cart system showing how scope keeps internal details private while allowing controlled access.

```js
let cartItemCount = 0;          // global — tracks across the whole page

function addToCart(productName, quantity) {
  // These variables are private to addToCart — nothing outside can touch them directly
  let lineMessage = productName + " x" + quantity + " added to cart";

  if (quantity <= 0) {
    let errorMessage = "Quantity must be at least 1"; // block-scoped to this if
    console.error(errorMessage);
    return; // exit early — cartItemCount is NOT updated
  }

  cartItemCount += quantity;
  console.log(lineMessage);
  console.log("Cart total items:", cartItemCount);
}

addToCart("Wireless Mouse", 2);
// Wireless Mouse x2 added to cart
// Cart total items: 2

addToCart("Keyboard", 0);
// Quantity must be at least 1

addToCart("Monitor", 1);
// Monitor x1 added to cart
// Cart total items: 3

// console.log(lineMessage);     // ❌ ReferenceError
// console.log(errorMessage);    // ❌ ReferenceError
console.log(cartItemCount);      // 3 ✅
```

---

## Tricky things you'll encounter in the real world

### 1 — `var` leaks out of blocks (but NOT out of functions)

```js
// var ignores block boundaries
if (true) {
  var leakyFlag = true;  // ⚠️ leaks into the surrounding function/global scope
  let safeFlag = true;   // ✅ stays inside this block
}

console.log(leakyFlag);  // true  — probably not what you wanted
console.log(safeFlag);   // ❌ ReferenceError
```

**Rule:** Always use `let` or `const`. Never `var` in modern code.

---

### 2 — Variable shadowing

A variable declared in an inner scope with the same name as an outer one **shadows** the outer one inside that scope.

```js
let status = "active";           // outer

function checkOrder(orderId) {
  let status = "pending";        // shadows outer `status` — completely separate variable
  console.log(status);           // "pending"
}

checkOrder(101);
console.log(status);             // "active" — outer untouched ✅
```

Shadowing is not an error, but it can be confusing. Give your variables distinct names when possible.

---

### 3 — Modifying a global from inside a function (side effect)

```js
let isLoggedIn = false;

function logIn(password) {
  if (password === "secret123") {
    isLoggedIn = true;   // modifying global — a "side effect"
  }
}

logIn("secret123");
console.log(isLoggedIn); // true
```

This works, but relying heavily on modifying globals makes code hard to test and debug. Prefer returning values and assigning them outside.

---

### 4 — `let` and `const` in loops create a new binding per iteration

```js
// Each iteration of this loop has its OWN `i` (block-scoped)
for (let i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 0, 1, 2  ✅

// With var, there's ONE shared i — by the time callbacks run, i is already 3
for (var i = 0; i < 3; i++) {
  setTimeout(() => console.log(i), 100);
}
// Prints: 3, 3, 3  ⚠️
```

---

### 5 — Functions declared inside other functions have access to the outer function's scope (lexical scope)

```js
function buildGreeting(language) {
  let prefix = language === "es" ? "Hola" : "Hello"; // lives in buildGreeting's scope

  function greet(name) {
    // greet can see `prefix` because of lexical (static) scope
    console.log(prefix + ", " + name + "!");
  }

  greet("Maria");   // "Hola, Maria!"
}

buildGreeting("es");
```

This is the foundation of **closures** (topic 19).

---

## Common mistakes

### ❌ Mistake 1 — Using a variable before it's declared with `let`/`const`

```js
// WRONG
console.log(userName); // ❌ ReferenceError (Temporal Dead Zone)
let userName = "Parsh";

// RIGHT
let userName = "Parsh";
console.log(userName); // ✅ "Parsh"
```

---

### ❌ Mistake 2 — Expecting `var` to be block-scoped

```js
// WRONG — thinking var stays inside the if block
function processOrder() {
  if (true) {
    var discount = 0.1;  // ⚠️ leaks to function scope
  }
  console.log(discount); // 0.1 — probably unexpected
}

// RIGHT — use let so it stays in the block
function processOrder() {
  if (true) {
    let discount = 0.1;  // ✅ block-scoped
  }
  // console.log(discount); // ReferenceError — exactly what you want
}
```

---

### ❌ Mistake 3 — Accidentally creating a global by forgetting `let`/`const`/`var`

```js
// WRONG
function saveUser() {
  userName = "Parsh"; // ⚠️ no declaration keyword → creates an implicit global!
}
saveUser();
console.log(userName); // "Parsh" — leaked to global scope

// RIGHT
function saveUser() {
  let userName = "Parsh"; // ✅ scoped to the function
}
```

Use `"use strict"` mode to make JS throw an error on undeclared assignments.

---

## Frequently asked questions

**Q: What's the difference between scope and context (`this`)?**  
Scope determines which variables a function can access. Context (`this`) determines what object a function "belongs to" at call time. They are completely separate concepts.

**Q: Is `let` in a `for` loop re-declared on every iteration?**  
Yes — each loop iteration gets a fresh `let` binding. This is why closures inside `for...let` loops work correctly (each callback captures its own `i`).

**Q: Does a function have access to variables declared after it in the same scope?**  
If both are in the same scope and the function is *called* after the variable is declared, yes. Function declarations are hoisted (topic coming up), so they can be called before they appear in the file, but the variable they reference must still be initialised before the call runs.

**Q: Can inner functions modify outer variables?**  
Yes. Inner functions can both read and write variables from their enclosing scopes. Writing to an outer variable is called a **side effect** — it works but should be done intentionally.

---

## Practice exercises

### Exercise 1 — easy

Write a function called `createProfile` that takes `firstName` and `age` as parameters.  
Inside it, declare a `let` variable called `profileSummary` that combines them into a readable string (e.g. `"Alice, age 30"`).  
Return `profileSummary`.  
After the function, try to `console.log(profileSummary)` and observe the error. (Comment it out once you've seen the error so the rest of your code can run.)

```js
// Write your code here
```

---

### Exercise 2 — medium

A global `let` variable called `loginAttempts` starts at `0`.  
Write a function `attemptLogin(password)` that:
- increments `loginAttempts` by 1 each time it is called
- if `password === "js-is-great"` logs `"Login successful after X attempt(s)"` (replace X with the actual count)
- otherwise logs `"Wrong password. Attempt X"` (same count)

Inside the function, declare a `let` variable `isCorrect` that holds the result of the password check.  
Call the function 3 times: twice with wrong passwords, once with the right one.

```js
// Write your code here
```

---

### Exercise 3 — hard

Demonstrate all three scope types and the scope chain in ONE block of code.

Requirements:
1. A global `const` called `appVersion` set to `"2.0"`.
2. A function `renderDashboard(userName)` that has:
   - A function-scoped `let` variable `dashboardTitle` set to `userName + "'s Dashboard — v" + appVersion`.
   - An inner function `renderWidget(widgetName)` that:
     - Has a block-scoped `const` called `widgetLabel` (inside an `if (widgetName)` block) set to `"[" + widgetName + "]"`.
     - Logs `dashboardTitle + " | Widget: " + widgetLabel` — it must reach up the scope chain to find `dashboardTitle`.
3. Call `renderDashboard("Parsh")` → inside, call `renderWidget("Analytics")`.
4. After `renderDashboard` returns, try to log `dashboardTitle` — observe the error.

```js
// Write your code here
```

---

## Quick reference (cheat sheet)

| Scope type | Created by | Keywords | Accessible from |
|---|---|---|---|
| Global | Top-level code | `let`, `const`, `var` | Everywhere in the file |
| Function (local) | Function body `{ }` | `let`, `const`, `var` | Only inside that function |
| Block | Any `{ }` (if, for, etc.) | `let`, `const` only | Only inside that block |

**Scope chain rule:** JS looks for a variable starting in the current scope, then moves outward one level at a time until it finds it or throws a `ReferenceError`.

| Keyword | Block-scoped? | Function-scoped? | Hoisted? |
|---|---|---|---|
| `var` | ❌ No | ✅ Yes | ✅ Yes (as `undefined`) |
| `let` | ✅ Yes | ✅ Yes | ✅ (TDZ — can't use before declaration) |
| `const` | ✅ Yes | ✅ Yes | ✅ (TDZ — can't use before declaration) |

**Key rules to remember:**
- Always use `let`/`const`, never `var`
- Inner scopes can see outer variables; outer scopes cannot see inner variables
- Same-name variables in inner scopes *shadow* outer ones — they don't modify them
- Missing a declaration keyword creates an implicit global (very bad — use `"use strict"`)

---

## Connected topics

- **19 — Closures** — closures are built directly on top of scope; a closure is a function that *remembers* the scope it was created in even after that scope closes
- **14 — Function declarations vs expressions** — understanding function scope requires knowing how functions create scope
- **Hoisting** — how `var`, `let`/`const`, and function declarations are processed at scope entry before any code runs (covered alongside closures)
