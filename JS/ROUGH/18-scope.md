# 18 — Scope

## What is this?
Scope is the set of rules that determines **where a variable can be seen and used** in your code. Think of it like rooms in a house — a variable declared in the bedroom is only accessible inside that bedroom, not from the kitchen. JavaScript has three kinds of scope: global, function (local), and block.

## Why does it matter?
Scope controls which parts of your program can read or modify which variables. Without understanding scope, you'll run into "variable is not defined" errors or accidentally overwrite variables you didn't mean to touch — both are extremely common bugs.

## Syntax

```js
// --- GLOBAL scope ---
let appName = "ShopEasy";          // visible everywhere in the file

function showApp() {
  // --- FUNCTION (local) scope ---
  let discountCode = "SAVE10";     // only visible inside showApp()

  if (true) {
    // --- BLOCK scope ---
    let flashSale = true;          // only visible inside this if-block
    console.log(flashSale);        // ✅ works here
  }

  console.log(discountCode);       // ✅ works here
  // console.log(flashSale);       // ❌ ReferenceError — flashSale is block-scoped
}

console.log(appName);              // ✅ works here
// console.log(discountCode);      // ❌ ReferenceError — discountCode is function-scoped
```

## How it works — line by line

1. `let appName = "ShopEasy"` — declared at the top level (outside any function or block), so it lives in **global scope**. Every function and block in the file can read it.
2. `function showApp()` — creates a new **function scope** (also called local scope). Anything declared inside only lives here.
3. `let discountCode = "SAVE10"` — lives inside `showApp`, so it's only accessible within that function.
4. `if (true) { ... }` — curly braces create a **block**. `let` and `const` are block-scoped, meaning they live only inside the nearest `{ }` pair.
5. `let flashSale = true` — block-scoped to the `if` block. Gone the moment the `}` closes.
6. **The scope chain**: if JS can't find a variable in the current scope, it looks outward (to the parent scope), then outward again, until it reaches global scope. If it's still not found — `ReferenceError`.

## Example 1 — basic

```js
// Global variable — accessible everywhere
let siteName = "TechBlog";

function printHeader() {
  // Local variable — only accessible inside printHeader
  let headerText = "Welcome to " + siteName; // looks up to global scope to find siteName
  console.log(headerText); // "Welcome to TechBlog"
}

printHeader();

// console.log(headerText); // ❌ ReferenceError — headerText doesn't exist out here
```

## Example 2 — real world

```js
// Imagine a checkout system

let taxRate = 0.08; // global — applies to all calculations

function calculateTotal(productPrice, quantity) {
  let subtotal = productPrice * quantity;  // local to this function
  let taxAmount = subtotal * taxRate;      // taxRate found via scope chain (global)
  let total = subtotal + taxAmount;

  if (total > 100) {
    let discountMessage = "You qualify for free shipping!"; // block-scoped
    console.log(discountMessage); // ✅ works here
  }

  // console.log(discountMessage); // ❌ ReferenceError — block is already closed

  return total;
}

console.log(calculateTotal(45, 3)); // 146.34
// console.log(subtotal);           // ❌ ReferenceError — subtotal is local to the function
```

## Common mistakes

- **Using `var` and expecting block scope**

  `var` ignores block scope — it leaks out of `if` blocks and `for` loops into the surrounding function or global scope.

  ```js
  // ❌ Wrong — var leaks out of the block
  if (true) {
    var leakyMessage = "I escaped!";
  }
  console.log(leakyMessage); // "I escaped!" — var ignores block scope

  // ✅ Right — use let or const for block scope
  if (true) {
    let safeMessage = "I stay here.";
  }
  // console.log(safeMessage); // ❌ ReferenceError — as expected
  ```

- **Assuming inner scope can't access outer variables**

  Inner scopes CAN read variables from outer (parent) scopes — this is the scope chain working correctly.

  ```js
  // ❌ Wrong thinking — writing redundant code because you assume it's inaccessible
  let userRole = "admin";

  function checkAccess() {
    let userRole = "admin"; // ❌ unnecessary duplicate — just use the outer one
    if (userRole === "admin") console.log("Access granted");
  }

  // ✅ Right — inner function reads outer variable via scope chain
  function checkAccess() {
    if (userRole === "admin") console.log("Access granted"); // reads global userRole ✅
  }
  ```

- **Accidentally creating global variables by forgetting `let`/`const`/`var`**

  Assigning to a name with no declaration keyword creates an implicit global — a dangerous bug.

  ```js
  // ❌ Wrong — omitting let/const makes it a global variable
  function saveUser() {
    userName = "Alice"; // no let/const/var — becomes a global!
  }
  saveUser();
  console.log(userName); // "Alice" — pollutes global scope

  // ✅ Right — always declare with let or const
  function saveUser() {
    let userName = "Alice"; // scoped to the function
  }
  ```

## Practice exercises

### Exercise 1 — easy
Declare a variable `appVersion` in the global scope with the value `"2.1.0"`. Then write a function called `printVersion` that reads `appVersion` from the scope chain (do NOT pass it as a parameter or redeclare it inside the function) and logs a message like: `"Running version: 2.1.0"`.

```js
// Write your code here
```

### Exercise 2 — medium
Write a function called `processOrder` that takes `itemPrice` and `quantity` as parameters. Inside the function:
- Calculate `subtotal` (price × quantity) — keep it as a local variable
- Use an `if` block: if `subtotal` is greater than 200, declare a block-scoped variable `bonusPoints` set to 50, and log `"Bonus points earned: 50"`
- After the `if` block, try to log `bonusPoints` — observe the error, then comment that line out
- Return the `subtotal`

```js
// Write your code here
```

### Exercise 3 — hard
You're building a simple permission system. Do the following:
1. Declare a global variable `currentUserRole` set to `"editor"`
2. Write a function `canPublish` that:
   - Declares a local array `allowedRoles` containing `"admin"` and `"editor"`
   - Uses a loop with a block-scoped variable to check if `currentUserRole` exists in `allowedRoles`
   - Returns `true` if the role is found, `false` otherwise
3. Call `canPublish()` and log the result
4. Change `currentUserRole` to `"viewer"` and call it again — the result should now be `false`
5. After the function, try to access `allowedRoles` — it should throw a ReferenceError (comment it out after observing)

```js
// Write your code here
```

## Quick reference (cheat sheet)

| Scope type | Created by | Accessible from | Keyword |
|---|---|---|---|
| Global | Top-level code | Everywhere | `let`, `const`, `var` |
| Function (local) | `function` body | Only inside that function | `let`, `const`, `var` |
| Block | Any `{ }` (if, for, while, etc.) | Only inside that block | `let`, `const` only — NOT `var` |

**Scope chain rule:** JS always looks in the current scope first, then walks outward until it hits global. It never looks inward.

**Key rules:**
- `let` and `const` → block-scoped
- `var` → function-scoped (ignores blocks) — avoid it
- Inner scopes can read outer variables ✅
- Outer scopes cannot read inner variables ❌
- Forgetting `let`/`const` creates an accidental global — always declare variables

## Connected topics
- **19 — Closures**: closures are built directly on top of scope — a closure is a function that remembers the scope it was born in
- **14 — Function declarations vs expressions**: both create their own function scope
- **19 — Hoisting**: `var` hoisting is closely tied to how function scope works
