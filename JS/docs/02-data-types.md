# 02 — Data Types

## What is this?
A data type tells JavaScript what kind of value a variable holds — a piece of text, a number, a yes/no flag, or nothing at all. Think of it like the label on a storage box: "FRAGILE", "FROZEN", "DOCUMENTS". The label doesn't change what's inside, but it tells you — and JavaScript — how to handle it. JS has **8 data types** split into two categories: **primitives** (simple, single values) and **objects** (complex, structured values).

## Why does it matter?
JavaScript doesn't force you to say what type a variable holds — it figures it out automatically. That sounds convenient, but it causes silent bugs when types mix unexpectedly. Knowing your data types deeply means you understand why `"5" + 1` gives `"51"` instead of `6`, and you'll never be surprised again.

---

## The 7 primitive types

A **primitive** is a single, immutable value. When you copy it, you get a completely independent copy.

| Type | Example | Represents |
|------|---------|------------|
| `string` | `"hello"` | Text |
| `number` | `42`, `3.14`, `NaN`, `Infinity` | All numeric values |
| `boolean` | `true`, `false` | Yes/no, on/off |
| `null` | `null` | Intentional absence of a value |
| `undefined` | `undefined` | Value hasn't been assigned yet |
| `symbol` | `Symbol("id")` | Unique, hidden identifier |
| `bigint` | `9007199254740991n` | Integers too large for `number` |

The 8th type is **object** — arrays, functions, plain objects — everything else.

---

## Syntax

```js
// string — text, always wrapped in quotes (single, double, or backtick)
const firstName   = "Parsh";
const lastName    = 'Shah';
const greeting    = `Hello, ${firstName}`;     // template literal — can embed expressions

// number — integers and decimals, same type in JS
const age         = 25;
const temperature = -3.5;
const bigNumber   = Infinity;                  // valid number
const notANumber  = NaN;                       // also type "number" — watch out!

// boolean
const isLoggedIn  = true;
const hasDiscount = false;

// null — YOU explicitly set this to say "no value here on purpose"
const selectedItem = null;

// undefined — JS sets this automatically when no value has been assigned
let pendingOrder;                              // automatically undefined
console.log(pendingOrder);                     // undefined

// symbol — always unique, even if the description is the same
const id1 = Symbol("userId");
const id2 = Symbol("userId");
console.log(id1 === id2);                      // false — always unique

// bigint — append 'n' to the number
const maxSafeInt  = 9007199254740991n;
const hugeNumber  = 99999999999999999999n;
```

---

## How it works — line by line

- **string** — A sequence of characters. Backtick strings (template literals) let you embed variables and expressions with `${}`.
- **number** — JS uses one number type for everything: integers, floats, `NaN` (result of a failed math operation), and `Infinity`. There are no separate `int` or `float` types.
- **boolean** — Only two possible values: `true` or `false`. Used in every condition (`if`, `while`, etc).
- **null** — Means "this variable intentionally has no value." You write `null` yourself.
- **undefined** — Means "this variable exists but has never been given a value." JS writes this for you.
- **symbol** — A guaranteed-unique value. Mainly used as object property keys to avoid name collisions.
- **bigint** — Handles integers beyond `Number.MAX_SAFE_INTEGER` (2^53 - 1). Append `n` to create one.
- **object** — Anything that isn't a primitive. Arrays, functions, dates, and plain `{}` objects are all type `"object"` (with one exception: functions return `"function"` from `typeof`).

---

## Example 1 — basic

```js
// Declaring one of each type and checking it with typeof
const productName   = "Wireless Mouse";     // string
const productPrice  = 29.99;                // number
const inStock       = true;                 // boolean
const couponCode    = null;                 // null — no coupon applied yet
let   shippingDate;                         // undefined — not set yet

console.log(typeof productName);            // "string"
console.log(typeof productPrice);           // "number"
console.log(typeof inStock);               // "boolean"
console.log(typeof couponCode);            // "object"  ← famous bug, see Tricky Things below
console.log(typeof shippingDate);          // "undefined"
```

---

## Example 2 — real world

```js
// A user profile object — contains multiple types at once
const userProfile = {
  username:    "parsh_dev",          // string
  age:         22,                   // number
  isPremium:   true,                 // boolean
  referredBy:  null,                 // null — no referral
  lastLogin:   undefined,            // undefined — never logged in
};

// Checking a field's type before using it
if (typeof userProfile.age === "number") {
  console.log(`Age is valid: ${userProfile.age}`);   // "Age is valid: 22"
}

// Detecting null safely (can't use typeof — see Tricky Things)
if (userProfile.referredBy === null) {
  console.log("No referral code on this account.");
}
```

---

## Deep dive — concepts you MUST understand

### 1. What is `typeof`?

`typeof` is an operator that returns a **string** describing the type of a value.

```js
typeof "hello"        // "string"
typeof 42             // "number"
typeof true           // "boolean"
typeof undefined      // "undefined"
typeof null           // "object"   ← BUG IN JS (see below)
typeof {}             // "object"
typeof []             // "object"   ← arrays are objects too
typeof function(){}   // "function" ← special case
typeof Symbol()       // "symbol"
typeof 10n            // "bigint"
```

`typeof` always returns a lowercase string. You compare it like this:
```js
if (typeof userName === "string") { ... }   // correct
if (typeof userName === String)   { ... }   // WRONG — String (capital S) is a constructor, not a string
```

---

### 2. `null` vs `undefined` — the difference that trips everyone up

These both mean "no value" but they come from different sources:

| | `null` | `undefined` |
|-|--------|-------------|
| Who sets it? | **You** — intentionally | **JavaScript** — automatically |
| When? | "I want this to be empty right now" | "This was never given a value" |
| `typeof` result | `"object"` (bug) | `"undefined"` |
| Loose equality `==` | `null == undefined` is `true` | same |
| Strict equality `===` | `null === undefined` is `false` | same |

```js
let cartItem = null;        // you deliberately cleared the cart slot
let deliveryDate;           // JS set this to undefined — date not known yet

console.log(cartItem == deliveryDate);    // true  — both are "empty"
console.log(cartItem === deliveryDate);   // false — different types
```

**Rule of thumb:** Use `null` when YOU want to express "nothing here intentionally." Never manually assign `undefined` — let JS do that.

---

### 3. Primitives are copied by value — objects are copied by reference

This is one of the most important things in JavaScript.

**Primitives — copy by value:**
```js
let originalScore = 100;
let backupScore   = originalScore;   // a COPY is made — completely independent

backupScore = 999;

console.log(originalScore);   // 100 — unchanged. The copy didn't affect the original.
console.log(backupScore);     // 999
```

**Objects — copy by reference:**
```js
const order1 = { item: "Book", qty: 2 };
const order2 = order1;   // NOT a copy — both variables point to the SAME object in memory

order2.qty = 10;

console.log(order1.qty);   // 10 — order1 also changed! They share the same object.
console.log(order2.qty);   // 10
```

This is why `const` doesn't protect object contents — `const` only prevents rebinding the variable, not mutating what it points to.

---

### 4. `NaN` — the most confusing "number"

`NaN` stands for "Not a Number" but its type is... `"number"`. It's the result JS gives when a math operation fails.

```js
console.log(typeof NaN);          // "number" — yes, really
console.log(0 / 0);               // NaN
console.log("hello" * 5);         // NaN — can't multiply a string by a number
console.log(parseInt("hello"));   // NaN — couldn't parse

// THE TRAP: NaN is the only value in JS that is not equal to itself
console.log(NaN === NaN);         // false — always false, by spec

// RIGHT way to check for NaN
console.log(Number.isNaN(NaN));   // true
console.log(Number.isNaN(42));    // false
```

---

### 5. The `typeof null === "object"` bug

This is a **known bug in JavaScript** that has existed since 1995 and cannot be fixed because it would break existing code on the internet.

```js
console.log(typeof null);   // "object" — WRONG, but it's the spec
```

This means you can NEVER use `typeof` to check for `null`. Always use strict equality:
```js
// WRONG
if (typeof selectedItem === "object") { ... }   // this would also catch real objects and null

// RIGHT
if (selectedItem === null) { ... }              // the only reliable null check
```

---

### 6. Dynamic typing — JS changes types silently

JavaScript is **dynamically typed** — a variable can hold any type at any time. This is different from languages like Java or C# where you declare the type upfront.

```js
let sessionData = "active";    // string
console.log(typeof sessionData);   // "string"

sessionData = 42;              // now it's a number — JS allows this
console.log(typeof sessionData);   // "number"

sessionData = true;            // now it's a boolean
console.log(typeof sessionData);   // "boolean"
```

This flexibility is powerful but dangerous. It's why TypeScript (a superset of JS) was invented — to add type safety. For now, just know: **always be deliberate about what type a variable holds.**

---

## Common mistakes

### 1. Using `typeof` to check for `null`
```js
// WRONG
const selectedCategory = null;
if (typeof selectedCategory === "object") {
  console.log("Category is an object");   // this runs — but selectedCategory is null!
}

// RIGHT
if (selectedCategory === null) {
  console.log("No category selected");
}
```

### 2. Comparing `NaN` with `===`
```js
// WRONG
const userInput = parseInt("abc");   // NaN
if (userInput === NaN) {
  console.log("Invalid input");      // NEVER runs — NaN !== NaN
}

// RIGHT
if (Number.isNaN(userInput)) {
  console.log("Invalid input");      // works correctly
}
```

### 3. Confusing `null` and `undefined` when checking empty values
```js
// WRONG — loose equality catches both, which hides whether it's null or undefined
let orderStatus;
if (orderStatus == null) {    // true for both null AND undefined
  console.log("Not set");
}

// RIGHT — be explicit about which empty state you expect
if (orderStatus === undefined) {
  console.log("Order status was never set");
}
if (orderStatus === null) {
  console.log("Order status was intentionally cleared");
}
```

---

## Tricky things you'll encounter in the real world

### Trick 1 — Arrays and `typeof`
```js
const cartItems = ["Apple", "Banana"];
console.log(typeof cartItems);           // "object" — NOT "array"

// RIGHT way to check if something is an array
console.log(Array.isArray(cartItems));   // true
```

### Trick 2 — Empty string, 0, and false all look like "nothing" but they're real values
```js
const score   = 0;
const message = "";
const flag    = false;

// If you test these carelessly with falsy checks (covered in topic 08), you'll treat
// a legitimate score of 0 as "no score" — a very common bug.
// typeof is your friend here:
console.log(typeof score === "number");    // true — it's a valid number
```

### Trick 3 — `typeof` on an undeclared variable doesn't throw
```js
// Accessing an undeclared variable normally throws ReferenceError
console.log(unknownVar);             // ReferenceError

// BUT typeof on an undeclared variable returns "undefined" instead of throwing
console.log(typeof unknownVar);      // "undefined" — no error
// This is sometimes used to safely check if a global variable exists
```

### Trick 4 — BigInt and Number can't mix
```js
const regularNum = 10;
const bigNum     = 10n;

console.log(bigNum + regularNum);    // TypeError: Cannot mix BigInt and other types
console.log(bigNum + BigInt(regularNum));   // 20n — convert first, then add
```

### Trick 5 — `typeof` a function
```js
function sendEmail() {}
console.log(typeof sendEmail);   // "function" — not "object", even though functions are objects
// Functions are a special subtype of object that get their own typeof result
```

---

## Practice exercises

### Exercise 1 — easy
Without running any code first, predict the `typeof` result for each of these, then write the code to verify:
```
"42"
42
true
null
undefined
[1, 2, 3]
function() {}
```

Write a `console.log(typeof ...)` for each one. Then note which results surprised you.
```js
// Write your code here
```

---

### Exercise 2 — medium
You're building a user registration validator. Given this object:
```js
const newUser = {
  username:  "parsh_dev",
  email:     "parsh@example.com",
  age:       22,
  referral:  null,
  promoCode: undefined
};
```

Write code that checks and logs:
1. Whether `username` is a string
2. Whether `age` is a number AND is not `NaN`
3. Whether `referral` is explicitly `null` (use `=== null`, not `typeof`)
4. Whether `promoCode` is `undefined`
```js
// Write your code here
```

---

### Exercise 3 — hard
You're debugging a shopping cart. The following code has **three silent type-related bugs**. Find them, explain what each bug is, and rewrite the code correctly:
```js
const cart = {
  items:    ["Keyboard", "Mouse"],
  total:    0,
  voucher:  null,
  quantity: "3"
};

// Bug check 1 — is items an array?
if (typeof cart.items === "array") {
  console.log("Cart has items");
}

// Bug check 2 — is voucher empty?
if (typeof cart.voucher === "object") {
  console.log("No voucher applied");
}

// Bug check 3 — is total a valid number?
const adjustedTotal = cart.total + cart.quantity;
console.log(`Total: ${adjustedTotal}`);   // expects: "Total: 3"
```

Rewrite all three checks correctly and explain what each bug was.
```js
// Write your code here
```

---

## Quick reference (cheat sheet)

**The 8 types:**
```
string    → "text"
number    → 42, 3.14, NaN, Infinity
boolean   → true, false
null      → null   (intentional empty)
undefined → undefined   (unassigned)
symbol    → Symbol("key")
bigint    → 100n
object    → {}, [], function — everything else
```

**`typeof` quick results:**
```
typeof "hello"     → "string"
typeof 42          → "number"
typeof true        → "boolean"
typeof undefined   → "undefined"
typeof null        → "object"    ← BUG — always use === null instead
typeof {}          → "object"
typeof []          → "object"    ← use Array.isArray() instead
typeof function(){}→ "function"
typeof Symbol()    → "symbol"
typeof 10n         → "bigint"
```

**Key rules:**
- `null` → you set it. `undefined` → JS set it.
- `NaN !== NaN` always. Use `Number.isNaN()` to check.
- Arrays are `"object"` in `typeof`. Use `Array.isArray()`.
- Primitives are **copied by value**. Objects are **copied by reference**.
- Never use `typeof` to check for `null`. Always use `=== null`.

---

## Connected topics
- **03 — Type Coercion** — what happens when JS automatically converts one type to another (directly builds on this)
- **08 — Truthy and Falsy values** — how JS evaluates each type as true or false in conditions
- **01 — Variables** — variables are just named containers for these types
