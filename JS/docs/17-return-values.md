# 17 — Return Values

## What is this?

A **return value** is the data a function sends back to the code that called it. The `return` keyword does two things simultaneously:

1. **Stops** the function — no code after `return` runs
2. **Hands back** a value to the caller

```js
function calculateTax(price, rate) {
  return price * rate;   // stops here, sends the result back
  console.log("done");   // ← this line NEVER runs
}

const tax = calculateTax(100, 0.08);  // tax = 8
```

Without `return`, a function performs an action but produces nothing — it returns `undefined` silently.

---

## Why does it matter?

The return value is how functions communicate results. Every function you write falls into one of two categories:

- **Command** — does something, no return value needed (e.g., log to console, save to database)
- **Query** — computes something and returns the result (e.g., calculate total, validate email)

Mixing these up — writing a query that forgets `return`, or expecting a return value from a command — is one of the most frequent sources of `undefined` and `NaN` bugs in real code.

Understanding return values deeply also unlocks:
- **Method chaining** — each method returns something the next can work on
- **Functional pipelines** — passing return values through a chain of functions
- **Returning functions** — the foundation of closures, currying, and factory functions

---

## Syntax

```js
function name() {
  return value;   // return with a value
}

function name() {
  return;         // return with no value — sends undefined
}

function name() {
  // no return    — implicitly returns undefined
}
```

---

## How it works — line by line

```js
function getOrderStatus(isPaid, isShipped) {
  if (!isPaid) {
    return "awaiting payment";   // exits here if not paid
  }
  if (!isShipped) {
    return "paid — processing";  // exits here if not shipped
  }
  return "shipped";              // only reached if both true
}

const status = getOrderStatus(true, false);
console.log(status);  // "paid — processing"
```

**Execution trace:**
```
isPaid = true, isShipped = false
→ !isPaid → false → skip first return
→ !isShipped → true → return "paid — processing"
→ function exits. "shipped" is never reached.
→ status = "paid — processing"
```

Each `return` is a possible exit point. The first one that executes ends the function entirely.

---

## Example 1 — basic

### Returning primitives

```js
function isValidAge(age) {
  return age >= 18 && age <= 120;
}

function getAgeGroup(age) {
  if (age < 13) return "child";
  if (age < 18) return "teen";
  if (age < 65) return "adult";
  return "senior";
}

console.log(isValidAge(25));   // true
console.log(isValidAge(200));  // false
console.log(getAgeGroup(10));  // "child"
console.log(getAgeGroup(17));  // "teen"
console.log(getAgeGroup(72));  // "senior"
```

### Returning computed values

```js
function calculateCompoundInterest(principal, annualRate, years) {
  return principal * Math.pow(1 + annualRate, years);
}

const futureValue = calculateCompoundInterest(1000, 0.07, 10);
console.log(futureValue.toFixed(2));  // "1967.15"
```

The caller gets the number back and can do further work with it (`.toFixed(2)` here). The function is pure — it just computes and returns.

### Returning early to prevent bad computations

```js
function safeDivide(numerator, denominator) {
  if (denominator === 0) {
    return null;  // early return for invalid input
  }
  return numerator / denominator;
}

console.log(safeDivide(10, 2));   // 5
console.log(safeDivide(10, 0));   // null — not Infinity
```

---

## Example 2 — real world

### Returning objects

```js
function parseUserAgent(uaString) {
  const isMobile = /Mobi|Android/i.test(uaString);
  const isTablet = /Tablet|iPad/i.test(uaString);
  const browser = uaString.includes("Chrome") ? "Chrome"
    : uaString.includes("Firefox") ? "Firefox"
    : uaString.includes("Safari") ? "Safari"
    : "Unknown";

  return {
    isMobile,
    isTablet,
    isDesktop: !isMobile && !isTablet,
    browser,
  };
}

const deviceInfo = parseUserAgent(navigator?.userAgent ?? "Chrome/Desktop");
console.log(deviceInfo);
// { isMobile: false, isTablet: false, isDesktop: true, browser: "Chrome" }
```

Returning an object lets a function communicate multiple pieces of related information in one call — without needing separate functions for each piece.

### Returning arrays

```js
function splitFullName(fullName) {
  const parts = fullName.trim().split(" ");
  const firstName = parts[0];
  const lastName = parts.slice(1).join(" ");  // handles "Mary Jane Watson"
  return [firstName, lastName];
}

const [first, last] = splitFullName("Mary Jane Watson");
console.log(first);  // "Mary"
console.log(last);   // "Jane Watson"
```

### Returning a function (factory pattern)

```js
function createMultiplier(factor) {
  return function(number) {
    return number * factor;
  };
}

const double = createMultiplier(2);
const triple = createMultiplier(3);
const tenX   = createMultiplier(10);

console.log(double(5));   // 10
console.log(triple(5));   // 15
console.log(tenX(5));     // 50
```

`createMultiplier` returns a *function* — not a number. That returned function "remembers" `factor` via closure (covered in topic 18). This is a **factory function** pattern.

### Returning for method chaining

```js
class QueryBuilder {
  constructor() {
    this.conditions = [];
    this.limitValue = null;
  }

  where(condition) {
    this.conditions.push(condition);
    return this;  // ← returning 'this' enables chaining
  }

  limit(n) {
    this.limitValue = n;
    return this;
  }

  build() {
    const where = this.conditions.length
      ? "WHERE " + this.conditions.join(" AND ")
      : "";
    const limit = this.limitValue ? ` LIMIT ${this.limitValue}` : "";
    return `SELECT * FROM orders ${where}${limit}`.trim();
  }
}

const query = new QueryBuilder()
  .where("status = 'shipped'")
  .where("total > 100")
  .limit(10)
  .build();

console.log(query);
// "SELECT * FROM orders WHERE status = 'shipped' AND total > 100 LIMIT 10"
```

Returning `this` from each method lets you chain calls. This pattern is used by jQuery, Mongoose, and many query builder libraries.

---

## Tricky things you'll encounter in the real world

### 1. Forgetting `return` — the silent `undefined` bug

```js
function getFullName(firstName, lastName) {
  const full = firstName + " " + lastName;
  // ⚠️ forgot return
}

const name = getFullName("Sam", "Rhodes");
console.log(name);         // undefined
console.log(name.length);  // TypeError: Cannot read properties of undefined
```

The function ran correctly, built the string, then silently discarded it. This is the number one function bug for beginners — and it still trips up experienced developers.

**How to catch it:** If a function is supposed to produce a value and you're getting `undefined` back, check that every code path has a `return`.

### 2. `return` in a callback doesn't return from the outer function

```js
function findFirstExpired(products) {
  products.forEach(product => {
    if (product.isExpired) {
      return product;  // ⚠️ returns from the ARROW FUNCTION, not findFirstExpired
    }
  });
  // findFirstExpired always returns undefined!
}

// ✅ Fix: use for...of + return
function findFirstExpired(products) {
  for (const product of products) {
    if (product.isExpired) {
      return product;  // returns from findFirstExpired ✅
    }
  }
  return null;
}
```

`return` only exits the function it's directly inside. A `return` inside `.forEach()` callback exits that callback, not the enclosing function. This is one of the most confusing early bugs.

### 3. Automatic semicolon insertion breaks multi-line returns

```js
function getConfig() {
  return        // ⚠️ JS inserts semicolon here — returns undefined!
  {
    theme: "dark",
    lang: "en",
  };
}

console.log(getConfig());  // undefined — not the object!
```

JavaScript's Automatic Semicolon Insertion (ASI) treats a newline after `return` (with nothing else on that line) as `return;`. The object on the next line becomes unreachable code.

**Fix — opening brace on same line as `return`:**

```js
function getConfig() {
  return {       // ✅ brace on same line
    theme: "dark",
    lang: "en",
  };
}
```

This is why the convention of "opening brace on same line" isn't just style — it affects correctness with `return`.

### 4. Returning `undefined` vs returning nothing — they're the same

```js
function doTask() { }               // returns undefined
function doTask() { return; }       // returns undefined
function doTask() { return undefined; }  // returns undefined

// All three are identical in behaviour
```

Explicitly returning `undefined` is valid but unusual. If you see `return;` it's usually an early exit from a command function (stop here, don't do anything else), not an attempt to return a value.

### 5. Chained returns — each function must return for the chain to work

```js
// ❌ Broken pipeline — formatName doesn't return
function formatName(name) {
  name.trim().toLowerCase();  // computed but discarded
}
function capitalize(str) {
  return str.charAt(0).toUpperCase() + str.slice(1);
}

const result = capitalize(formatName("  SARAH  "));
// formatName returns undefined
// capitalize(undefined) → TypeError: Cannot read properties of undefined
```

In a pipeline where function A feeds into function B, every function in the chain must return its output.

### 6. Multiple return types — be consistent

```js
// ❌ Inconsistent — sometimes returns a number, sometimes a string
function getScore(passed) {
  if (passed) return 100;
  return "failed";  // different type!
}

const score = getScore(false);
const doubled = score * 2;  // NaN — "failed" * 2 is NaN
```

Functions should return the same type on all paths (or a clearly documented union type). Mixing numbers and strings silently causes `NaN` bugs downstream.

---

## Common mistakes

### Mistake 1: Returning without capturing the result

```js
function addTax(price) {
  return price * 1.08;
}

addTax(50);                 // ❌ result discarded — went nowhere
const final = addTax(50);   // ✅ captured
```

### Mistake 2: `return` inside `.forEach()` exits only the callback

```js
// ❌ This doesn't work as an early exit from processAll
function processAll(items) {
  items.forEach(item => {
    if (!item.isValid) return;  // exits forEach callback only
    process(item);
  });
}

// ✅
function processAll(items) {
  for (const item of items) {
    if (!item.isValid) continue;
    process(item);
  }
}
```

### Mistake 3: Return on its own line (ASI trap)

```js
// ❌
function getUser() {
  return
    { name: "Sam" };  // unreachable — returns undefined
}

// ✅
function getUser() {
  return {
    name: "Sam"
  };
}
```

### Mistake 4: Forgetting that a function used in an expression must return

```js
// ❌ calculateDiscount doesn't return — total becomes NaN
function calculateDiscount(price) {
  const discount = price * 0.1;
  // forgot return
}

const total = 100 - calculateDiscount(100);  // 100 - undefined = NaN
```

---

## Practice exercises

### Exercise 1 — easy

Write three functions:

1. `getLetterGrade(score)` — returns `"A"` (90+), `"B"` (80–89), `"C"` (70–79), `"D"` (60–69), `"F"` (below 60), and `"invalid"` if score is not between 0–100
2. `isPassing(score)` — returns `true` if score ≥ 60, `false` otherwise, and `null` if score is not a valid number
3. `formatScoreReport(studentName, score)` — calls both functions above and returns a string: `"Priya Patel: 87/100 — Grade B (PASSED)"`

Test with at least 5 different score values. Make sure all three return types are consistent on every code path.

---

### Exercise 2 — medium

Build a **shopping cart calculator** using a chain of functions where each one's return value feeds into the next:

```js
const cartItems = [
  { name: "Laptop Stand",  price: 45.00, quantity: 1 },
  { name: "USB-C Cable",   price: 12.99, quantity: 3 },
  { name: "Mousepad XL",   price: 29.99, quantity: 2 },
];
```

Write these functions (each must `return` for the chain to work):

1. `calculateSubtotals(items)` — returns a new array where each item has a `subtotal` property added (`price × quantity`)
2. `sumCart(itemsWithSubtotals)` — returns the total sum of all `subtotal` values
3. `applyPromoCode(total, code)` — returns the discounted total:
   - `"SAVE10"` → 10% off
   - `"HALF"` → 50% off
   - Any other code → no discount (return total unchanged)
4. `addTax(total, taxRate = 0.08)` — returns total with tax added
5. `formatReceipt(finalTotal, itemCount)` — returns a string: `"Total: $123.45 for 6 items"`

Chain them: `formatReceipt(addTax(applyPromoCode(sumCart(calculateSubtotals(cartItems)), "SAVE10")), 6)`

---

### Exercise 3 — hard

Build a **factory function** that creates individual bank account objects:

```js
function createBankAccount(ownerName, initialBalance = 0) {
  // your code here
  // return an object with methods
}
```

The returned object must have these methods — each must return something meaningful:

- `deposit(amount)` — adds to balance, returns the new balance; throws `Error` if amount ≤ 0
- `withdraw(amount)` — subtracts from balance, returns the new balance; throws `Error` if amount ≤ 0; throws `Error` if amount > current balance (`"Insufficient funds"`)
- `getBalance()` — returns current balance
- `getStatement()` — returns a summary object:
  ```js
  { owner: "Layla Moss", balance: 420.50, transactionCount: 3 }
  ```
- `transfer(amount, targetAccount)` — withdraws `amount` from this account and deposits into `targetAccount`; returns `true` if successful

Track all transactions (deposit/withdraw/transfer) internally and include the count in `getStatement()`.

Test:
```js
const accountA = createBankAccount("Layla Moss", 500);
const accountB = createBankAccount("Noel Park", 100);

accountA.deposit(200);
accountA.withdraw(80);
accountA.transfer(150, accountB);

console.log(accountA.getStatement());
// { owner: "Layla Moss", balance: 470, transactionCount: 3 }

console.log(accountB.getStatement());
// { owner: "Noel Park", balance: 250, transactionCount: 1 }
```

---

## Quick reference

```
Return values cheat sheet
─────────────────────────────────────────────────────
return value      exits function, sends value to caller
return            exits function, sends undefined
no return         function implicitly sends undefined

MULTIPLE RETURNS  valid — first one reached exits the function
EARLY RETURN      return inside if = guard / early exit pattern
─────────────────────────────────────────────────────
RETURN IN CALLBACK  only exits the callback, not outer function
ASI TRAP            return on its own line = return undefined
                    → always put { on same line as return
INCONSISTENT TYPES  returning string/number on different paths = bugs
─────────────────────────────────────────────────────
CAN RETURN          primitives, objects, arrays, functions, null, undefined
RETURNING 'this'    enables method chaining
RETURNING function  factory pattern / closure
─────────────────────────────────────────────────────
command function    does something, no return needed
query function      computes something, MUST return
```

---

## Connected topics

- **14 — Function declarations and expressions** — where `return` is first introduced
- **15 — Arrow functions** — implicit return (no `{}`, no `return` keyword) for single expressions
- **18 — Scope and closures** — returning a function that remembers variables (closure)
- **41 — Higher-order functions** — functions that return functions
- **42 — Currying** — chained returns: `a => b => a + b`
- **Array methods (23+)** — `.map()`, `.filter()`, `.reduce()` all rely on the return value of their callbacks
